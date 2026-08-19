# 05. 쓰기 파이프라인

GitHub 쪽 파이프라인은 `F1 → F2 → F3 → (승인) → F4` 다.
Notion은 그 **오른쪽 바깥**에 붙는다.

```
FastAPI JobResult
    │
    ▼
Spring: proposals INSERT          ← 원본 커밋
    │
    ├─ 알림 / 미리보기 UI
    └─ 조건 맞으면 notion_sync_outbox INSERT
            │
            ▼
      worker CAS running
            │
            ▼
      F3 ArtifactMirror
            │
            ├─ duplicate → 기존 URL
            ├─ created   → 새 페이지
            ├─ skipped   → 이유 기록
            └─ failed    → backoff
```

## 1. 한 건의 수명

예: 매일 03:00 스케줄이 `progress_summary` 잡을 돌린다.

| t | 누가 | 무엇 |
| --- | --- | --- |
| 0 | Spring scheduler | FastAPI `POST /internal/jobs` + `X-GitHub-Pat` |
| 1 | FastAPI | GitHub 수집 + 진행 메모 `.md` |
| 2 | Spring | `proposals` 저장. `kind=progress`, `status=proposed` |
| 3 | Spring | 유저 Notion 연결 확인 후 outbox `pending` |
| 4 | Notion worker | F3 query `Proposal ID` |
| 5 | Notion worker | `POST /v1/pages` markdown |
| 6 | Spring | outbox `succeeded` + page_url |
| 7 | Frontend | 대시보드에 “Notion에 기록됨” 링크 |

t=5가 실패해도 t=2는 그대로다. 미리보기는 된다.

## 2. 미러할지 말지

워커 진입 전 필터 (코드로 그대로):

```
function shouldEnqueue(proposal, connection):
  if connection is null or connection.needs_reauth or connection.revoked_at: return false
  if proposal.kind not in {progress, portfolio, resume}: return false
  if proposal.status not in {proposed, partial}: return false
  if not proposal.body_markdown.strip(): return false
  return true
```

워커 진입 후 한 번 더:

```
function shouldMirror(proposal):
  if proposal.status == "no_change": return skip("no_change")
  if proposal.status in {"blocked","failed"}: return skip("not_mirrorable")
  if not proposal.body_markdown.strip(): return skip("empty_body")
  return go
```

enqueue 이후 proposal이 지워진 경우: FK 때문에 못 지우거나, 지웠으면 outbox도 cascade. 워커가 join 실패하면 `skipped`.

## 3. F3 내부 순서 (이 순서가 계약)

아래 1–2는 **워커(조립기)** 가 한다. F3는 토큰·맵을 인자로만 받는다 (`DESIGN.md` — 피처끼리 import 금지).

```
1. decrypt access via F1.accessFor(user)
     not_connected / needs_reauth → 즉시 종료
2. load WorkspaceMap
     없거나 root 404 → 조립기가 F2.provision 한 번
     또 실패 → workspace_missing retryable=false
3. pick data_source_id by kind
4. query Proposal ID equals
     hit → duplicate, page_url, stop
5. build properties (04)
6. prepend meta callout to markdown (07) — 원문 삭제 아님
7. if len(markdown) > 80000: allow_async=true
8. POST /v1/pages
9. 202 async: poll GET /v1/async_tasks/{id}
     timeout 60s → failed retryable=true
10. persist created
```

## 4. 속성 채우기 예 (progress)

```json
{
  "parent": {
    "type": "data_source_id",
    "data_source_id": "{progress_data_source_id}"
  },
  "icon": { "type": "emoji", "emoji": "📝" },
  "properties": {
    "Name": {
      "title": [{ "type": "text", "text": { "content": "2026-08-19 진행 메모" } }]
    },
    "Date": { "date": { "start": "2026-08-19" } },
    "Status": { "select": { "name": "proposed" } },
    "Proposal ID": {
      "rich_text": [{ "type": "text", "text": { "content": "2f1c0a6e-...." } }]
    },
    "Job ID": {
      "rich_text": [{ "type": "text", "text": { "content": "9aa1...." } }]
    },
    "Digest": {
      "rich_text": [{ "type": "text", "text": { "content": "a3f2..." } }]
    },
    "Complete": { "checkbox": true },
    "Repos": { "number": 3 }
  },
  "markdown": "<07에서 만든 문자열>"
}
```

title 2000자 제한. 우리는 짧은 제목만.

## 5. 동시성

같은 `proposal_id`가 두 번 enqueue되지 않게 UNIQUE.
그래도 워커 두 대가 같은 행을 집으면:

1. CAS `pending → running` 이 한 명만 성공
2. 진 쪽은 0행, POST 안 함
3. 이긴 쪽이 POST 직전에 죽으면 행이 `running`에 남음
4. 안전망: `running` 이 10분 이상이면 `pending`으로 되돌리는 스케줄

```sql
UPDATE notion_sync_outbox
   SET status = 'pending',
       next_attempt_at = now(),
       updated_at = now()
 WHERE status = 'running'
   AND updated_at < now() - interval '10 minutes';
```

Notion에 페이지가 이미 생긴 뒤 워커가 죽으면, 재시도 때 query가 `duplicate`를 준다. **이게 멱등의 본체다.** CAS만 믿으면 안 된다.

## 6. GitHub `no_change`와의 관계

FastAPI가 `status=no_change`, `body_markdown=""` 을 주면 Spring은 알림만 하고 outbox를 만들지 않는다.
어제와 같은 커밋이면 Notion에 빈 페이지가 매일 쌓이면 안 된다.

부분 수집 `partial`은 남긴다. 본문에 경고 callout이 있다 (`07`).

## 7. 수동 “지금 동기화”

`POST /api/integrations/notion/sync/{proposal_id}` (JWT)

용도: 데모 중 워커를 기다리기 싫을 때, 실패 건 재시도.

동작:

1. proposal이 본인 것인지 확인
2. outbox가 없으면 enqueue 조건으로 insert
3. 있으면 `next_attempt_at=now()`, `status=pending` (succeeded는 건드리지 않음. duplicate 경로)
4. 워커를 동기 호출해도 된다. HTTP 타임아웃 25초. 넘으면 202 `{queued:true}`

## 8. 관측

로그 (토큰 없이):

```
notion.mirror start user=… proposal=… kind=progress
notion.mirror duplicate page=…
notion.mirror created page=… ms=…
notion.mirror failed code=rate_limited retryable=true attempt=2
```

메트릭이 있으면:

- `notion_mirror_total{status}`
- `notion_mirror_latency_ms`
- `notion_outbox_backlog`

없어도 된다. 해커톤은 로그 + outbox 테이블 조회면 충분.

```sql
SELECT status, count(*) FROM notion_sync_outbox GROUP BY 1;
```

# 08. 구현 플레이북 — 오늘부터 체크리스트

이 순서대로 하면 설계를 다시 해석할 필요가 없다.
하루는 해커톤 기준 3–5시간이다. 앞에서 막히면 다음 날로 넘기지 말고 Day 0을 통과할 때까지 멈춘다.

코드 위치: `Blocki-Backend` `com.blocki.notion`.
`Blocki-AI`에는 파일을 만들지 않는다.

---

## Day 0 — Notion API가 내 손에서 살아 있는지

목표: 토큰 하나로 페이지 하나를 만든다. Spring 없음.

1. https://app.notion.com/developers/connections 에서 **Internal connection** 하나 (개발 전용)
2. Capabilities: Read, Insert, Update
3. 테스트 페이지에서 `···` → Add connections → 방금 만든 연결
4. 토큰을 환경변수로만

```bash
export NOTION_TOKEN="ntn_..."   # Configuration 탭
export NOTION_VERSION="2026-03-11"
export PARENT_PAGE_ID="xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
```

페이지 ID: 페이지 링크의 32 hex. 대시 넣기: `8-4-4-4-12`.

```bash
curl -sS https://api.notion.com/v1/users/me \
  -H "Authorization: Bearer $NOTION_TOKEN" \
  -H "Notion-Version: $NOTION_VERSION" | jq .
```

합격: `"object": "user"` 또는 bot.

```bash
curl -sS https://api.notion.com/v1/pages \
  -H "Authorization: Bearer $NOTION_TOKEN" \
  -H "Notion-Version: $NOTION_VERSION" \
  -H "Content-Type: application/json" \
  -d "{
    \"parent\": {\"type\":\"page_id\",\"page_id\":\"$PARENT_PAGE_ID\"},
    \"properties\": {
      \"title\": {\"title\":[{\"type\":\"text\",\"text\":{\"content\":\"Blocki smoke\"}}]}
    },
    \"markdown\": \"# smoke\\n\\n- ok\\n\"
  }" | jq '{id,url}'
```

합격: Notion 앱에서 `Blocki smoke` 페이지가 보이고 본문에 `smoke`가 있다.

실패 시:

| 증상 | 원인 |
| --- | --- |
| 401 | 토큰 오타, 헤더 이름 |
| 404 object_not_found | 페이지에 connection을 Add 하지 않음 |
| 400 validation | `Notion-Version` 빠짐, parent JSON 오타 |
| markdown 무시 | 구버전 헤더. `2026-03-11` 확인 |

이 토큰은 **프로덕션 금고에 넣지 않는다.** Day 1부터 Public OAuth.

---

## Day 1 — Public connection + F1 뼈대

1. `02-oauth.md` §0 대로 Public connection 생성. redirect `http://localhost:8080/api/integrations/notion/callback`
2. Flyway `Vxx__create_notion_tables.sql` — `03-data-model.md` SQL 그대로
3. `application.yml` notion 블록
4. 클래스:
   - `NotionConfig`
   - `NotionTokenVault` (기존 encrypt 호출)
   - `NotionOAuthService.start / handleCallback / disconnect / status`
   - `NotionConnectController`
5. 로그 필터: `ntn_`, `nrt_`, `Authorization`

수동 확인:

1. 로그인
2. `/api/integrations/notion/connect` → Notion 동의 → 페이지 한 개 선택
3. callback 후 `/api/integrations/notion/status` 가 `connected:true`
4. DB `notion_connections` 1행. 컬럼을 SELECT * 로 터미널에 붙여 넣지 말 것
5. disconnect → `connected:false`
6. 다시 connect → UPSERT

테스트:

- 잘못된 state → 400
- deny → 기존 행 유지 (연결 후 두 번째 창에서 deny)
- status JSON에 `access` / `token` 키가 없다

아직 F2/F3 없음. `handleCallback` 끝에서 provision은 no-op 또는 TODO 로그.

---

## Day 2 — F2 워크스페이스

1. `NotionRestClient`: `get`, `post`, 429/529 retry (`10`의 자바 스케치)
2. `NotionWorkspaceProvisioner.provision`
3. callback 성공 직후 호출
4. `notion_workspaces` 저장

확인:

- 유저 Notion에 `Blocki` / `Progress Logs` / `Portfolio Versions` / `Resume Versions`
- 같은 유저로 disconnect → connect 해도 루트가 **하나**
- `GET /v1/databases/{id}` 로 `data_sources[0].id` 가 비어 있지 않음
- 페이지를 하나도 안 고르고 동의하면 status가 `pick_page` 또는 `workspace_missing` 메시지

search fallback (`02` §7)을 이 날 넣는다.

---

## Day 3 — outbox + progress 미러

1. proposal 저장 코드 위치를 찾는다. Spring 팀에게 파일 이름을 묻는다. 없으면 임시로

```
POST /internal/debug/fake-proposal
```

바디:

```json
{
  "user_id": "...",
  "proposal_id": "11111111-1111-1111-1111-111111111111",
  "job_id": "22222222-2222-2222-2222-222222222222",
  "kind": "progress",
  "status": "proposed",
  "body_markdown": "## 오늘\n- repo X 커밋 3\n",
  "proposal_digest": "abc123",
  "snapshot_digest": "def456",
  "collected_at": "2026-08-19T00:00:00Z"
}
```

이 디버그 엔드포인트는 `X-Internal-Key` 뒤에 두고, 데모 전에 막는다.

2. enqueue 훅
3. `@Scheduled` 워커 5초
4. `NotionArtifactMirror` — progress만
5. 멱등 query

확인:

- fake-proposal 1회 → Progress Logs 1행, 본문에 커밋 목록
- 같은 proposal_id 재전송 → 페이지가 늘지 않음. outbox `duplicate` 또는 `succeeded` 유지
- `status=no_change`, 빈 본문 → Notion 호출 0 (WireMock/로그로 확인)
- 연결 해제 후 fake → outbox 없음

---

## Day 4 — portfolio / resume + 실패 경로

1. `MIRROR_TARGETS`에 두 kind
2. 제목 규칙 `04`
3. `allow_async` 분기 (80_000자 fixture)
4. refresh 단일 비행 테스트 (두 스레드 `accessFor`)
5. 401 스텁 → refresh → 성공
6. `invalid_grant` → `needs_reauth`, 프론트 배지

프론트:

- settings에 연결 버튼
- `?notion=connected|denied|error|pick_page` 토스트
- 마지막 동기화 링크

---

## Day 5 — FastAPI 실데이터 연결 +  Harden

1. debug 엔드포인트 끄기
2. 진짜 `POST /internal/jobs` 성공 커밋에 enqueue
3. 진행 메모 스케줄 1회를 로컬에서 수동 트리거
4. GitHub 잡이 실패한 채 Notion만 성공하는 테스트는 **만들지 말 것**. 반대만 만든다: GitHub 성공, Notion 5xx → proposal 유지
5. 로그에 토큰이 없는지 `rg "ntn_|nrt_|Bearer "` 로 테스트 로그를 훑는다
6. EC2 redirect URI를 portal에 추가
7. Docker 환경변수 `NOTION_CLIENT_*`

데모 스크립트:

1. 로그인
2. GitHub 연결 (다른 팀)
3. Notion 연결, 페이지 선택
4. “지금 수집” 또는 스케줄
5. 서비스 미리보기에 진행 메모
6. Notion Progress Logs에 같은 글
7. 포폴 생성 → Portfolio Versions에 버전 1

---

## 클래스 책임 (복붙용 시그니처)

```java
public interface NotionOAuthService {
  URI start(UUID userId, String returnTo);
  NotionStatus handleCallback(String code, String state, String error);
  NotionStatus disconnect(UUID userId);
  NotionStatus status(UUID userId);
  String accessFor(UUID userId); // 평문. 호출자 스택에서 저장 금지
}

public interface NotionWorkspaceProvisioner {
  WorkspaceMap provision(UUID userId, ConnectionIdentity id, String accessToken);
  WorkspaceMap ensure(UUID userId, String accessToken);
}

public interface NotionArtifactMirror {
  SyncResult mirror(MirrorRequest req);
}
```

조립 (`NotionConfig` 또는 애플리케이션 서비스):

```java
@Transactional
public void onProposalSaved(Proposal p) {
  if (!shouldEnqueue(p, connections.find(p.userId()))) return;
  outbox.insertIgnore(p.proposalId(), p.userId(), p.kind());
}

@Scheduled(fixedDelay = 5000)
public void drain() {
  for (var row : outbox.due(10)) {
    if (!outbox.casRunning(row.id())) continue;
    SyncResult r = mirror.mirror(toRequest(row));
    outbox.complete(row.id(), r);
  }
}
```

피처 클래스가 서로를 `new` 하지 않는다. 위 조립기만 둘을 안다.

---

## 하지 말 것 (구현 중 유혹)

- FastAPI에 notion 키 추가
- LLM에게 `notion-create-pages` 를 맡김
- 페이지를 매일 같은 제목으로 overwrite
- refresh를 `@Scheduled` 로 전 유저 일괄 (만료 직전 on-demand만)
- Notion Java 비공식 SDK를 급하게 추가 (RestClient로 끝낸 뒤, 여유 있으면)
- 속성 키를 한글로 (`진행일` 등)
- `Blocki-AI/DESIGN.md`의 Job 타입을 Notion 때문에 바꿈

---

## 완료 정의 (1차)

- [ ] 유저가 OAuth로 자기 워크스페이스를 연결/해제할 수 있다
- [ ] 연결 시 루트+DB 3개가 생기고, 두 번째 연결에서 복제되지 않는다
- [ ] `progress` / `portfolio` / `resume` 의 `proposed|partial` 이 각 DB에 한 페이지로 남는다
- [ ] 같은 `proposal_id`는 페이지 하나
- [ ] `no_change`와 미연결은 Notion 호출 0
- [ ] Notion 장애가 GitHub 미리보기를 깨지 않는다
- [ ] 응답·로그에 토큰이 없다
- [ ] `needs_reauth` 시 다시 연결 CTA

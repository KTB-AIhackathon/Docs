# 04. Notion 워크스페이스 스키마 v1

유저가 OAuth로 연결하면 F2가 **한 번** 만든다.
속성 **이름(API key)** 은 영어 고정이다. 구현체가 한글 키를 쓰면 쿼리가 깨진다.

## 1. 트리

```text
Blocki                            ← page (root)
├─ Progress Logs                  ← database
├─ Portfolio Versions             ← database
└─ Resume Versions                ← database
```

아이콘 권장:

| 객체 | emoji |
| --- | --- |
| 루트 `Blocki` | 🧱 |
| Progress Logs | 📝 |
| Portfolio Versions | 📁 |
| Resume Versions | 📄 |

루트 페이지 본문 (처음 한 번만, 이미 본문이 있으면 덮어쓰지 않음):

```markdown
이 페이지는 Blocki가 만든 로그 폴더입니다.

- **Progress Logs**: GitHub 활동을 요약한 진행 메모
- **Portfolio Versions**: 포트폴리오 초안 버전
- **Resume Versions**: 이력서 초안 버전

원본은 Blocki 서비스에 있습니다. 여기 페이지를 고쳐도 서비스 미리보기에는 반영되지 않습니다.
```

## 2. 공통 규칙

- DB 생성: `POST /v1/databases`
- 페이지 생성 parent: `{ "type": "data_source_id", "data_source_id": "<id>" }`
- 생성 직후 `GET /v1/databases/{database_id}` 응답의 `data_sources[0].id` 를 Spring에 저장
- `data_sources`가 없고 `id`만 있으면, 그 `id`를 data_source_id로도 저장한다 (구응답 호환)
- 속성 값은 모두 요청 한도에 맞게 자른다 (`07-markdown-and-pages.md`)

## 3. Progress Logs

### 3.1 속성

| API key | Notion type | 필수 | 출처 |
| --- | --- | --- | --- |
| `Name` | title | 예 | `{yyyy-MM-dd} 진행 메모` UTC 날짜. 같은 날 여러 건이면 뒤에 ` {HH:mm}` |
| `Date` | date | 예 | `collected_at` 또는 proposal `created_at`. UTC date |
| `Status` | select | 예 | `proposed` / `partial` |
| `Proposal ID` | rich_text | 예 | `proposal_id` 전체 UUID. **멱등 키** |
| `Job ID` | rich_text | 예 | `job_id` |
| `Digest` | rich_text | 예 | `proposal_digest` hex |
| `Complete` | checkbox | 예 | snapshot.complete. 없으면 true |
| `Repos` | number | 아니오 | snapshot_summary.repo_count |

select 옵션을 미리 만든다.

```
proposed  — green
partial   — yellow
```

`no_change` 행은 만들지 않으므로 옵션에 넣지 않는다.

### 3.2 생성 페이로드

```json
{
  "parent": { "type": "page_id", "page_id": "{ROOT_PAGE_ID}" },
  "icon": { "type": "emoji", "emoji": "📝" },
  "title": [{ "type": "text", "text": { "content": "Progress Logs" } }],
  "properties": {
    "Name": { "title": {} },
    "Date": { "date": {} },
    "Status": {
      "select": {
        "options": [
          { "name": "proposed", "color": "green" },
          { "name": "partial", "color": "yellow" }
        ]
      }
    },
    "Proposal ID": { "rich_text": {} },
    "Job ID": { "rich_text": {} },
    "Digest": { "rich_text": {} },
    "Complete": { "checkbox": {} },
    "Repos": { "number": { "format": "number" } }
  }
}
```

### 3.3 보드 뷰 (선택)

시간 있으면 생성 직후 `POST` view API로 Date 내림차순 table view 이름을 `All`로.
없어도 1차 합격. Notion 기본 뷰로 충분하다.

## 4. Portfolio Versions / Resume Versions

두 DB는 속성 키가 같다. title만 다르다.

| API key | type | 출처 |
| --- | --- | --- |
| `Name` | title | `{kind} {template_version} {yyyy-MM-dd HH:mm}` |
| `Kind` | select | `portfolio` 또는 `resume` |
| `Template` | rich_text | `v1` 등 `template_ref.version` |
| `Template SHA` | rich_text | `template_ref.sha256` |
| `Proposal ID` | rich_text | 멱등 키 |
| `Job ID` | rich_text | |
| `Digest` | rich_text | `proposal_digest` |
| `Snapshot` | rich_text | `snapshot_digest` |
| `Date` | date | created_at |

Kind 옵션: `portfolio` blue, `resume` purple.

Portfolio DB에는 Kind를 `portfolio`로만 넣고, Resume DB에는 `resume`만 넣는다.
속성을 공유하는 이유는 코드 한 함수로 쓰기 위해서다. 한 DB에 섞지 않는다. 유저가 찾을 때 헷갈린다.

생성 페이로드 (portfolio):

```json
{
  "parent": { "type": "page_id", "page_id": "{ROOT_PAGE_ID}" },
  "icon": { "type": "emoji", "emoji": "📁" },
  "title": [{ "type": "text", "text": { "content": "Portfolio Versions" } }],
  "properties": {
    "Name": { "title": {} },
    "Kind": {
      "select": {
        "options": [
          { "name": "portfolio", "color": "blue" },
          { "name": "resume", "color": "purple" }
        ]
      }
    },
    "Template": { "rich_text": {} },
    "Template SHA": { "rich_text": {} },
    "Proposal ID": { "rich_text": {} },
    "Job ID": { "rich_text": {} },
    "Digest": { "rich_text": {} },
    "Snapshot": { "rich_text": {} },
    "Date": { "date": {} }
  }
}
```

Resume은 title `Resume Versions`, icon `📄`. properties 동일.

## 5. 멱등 조회

Notion DB에 unique index가 없다. 쓰기 전에 query 한다.

```http
POST /v1/data_sources/{data_source_id}/query
Notion-Version: 2026-03-11
Content-Type: application/json

{
  "filter": {
    "property": "Proposal ID",
    "rich_text": { "equals": "{proposal_id}" }
  },
  "page_size": 1
}
```

엔드포인트가 404면 구버전 호환:

```http
POST /v1/databases/{database_id}/query
```

같은 필터.

`results.length >= 1` 이면 `SyncResult.status=duplicate`, `page_id=results[0].id`.
페이지를 또 만들지 않음.

query 자체도 3 rps에 들어간다. 한 미러 = query 1 + create 1.

## 6. 프로비저닝 멱등

이미 `notion_workspaces` 행이 있을 때:

1. `GET /v1/pages/{root_page_id}`
2. 200이고 `archived=false` 이면 DB 3개도 GET
3. 하나라도 404/archived 이면 **그 객체만** 다시 만들고 행을 UPDATE
4. 전부 정상이면 API 쓰기 없이 맵 반환

루트 제목으로 search 해서 중복 `Blocki` 페이지를 만들지 말 것.
search는 유저 워크스페이스에 동명 페이지가 있을 수 있다. **저장한 id만** 믿는다.

예외: 저장 id가 전부 404이고 workspace_id는 같음 → 유저가 루트를 지움. 새로 만들고 행을 교체. 옛 로그는 복구하지 않음.

## 7. curl로 스키마만 검증

PAT/Internal 토큰 + 공유된 페이지 ID가 있을 때.

```bash
ROOT=....   # dashes 있는 UUID

curl -sS https://api.notion.com/v1/pages \
  -H "Authorization: Bearer $NOTION_TOKEN" \
  -H "Notion-Version: 2026-03-11" \
  -H "Content-Type: application/json" \
  -d "{
    \"parent\": {\"type\":\"page_id\",\"page_id\":\"$ROOT\"},
    \"icon\": {\"type\":\"emoji\",\"emoji\":\"🧱\"},
    \"properties\": {
      \"title\": {\"title\":[{\"type\":\"text\",\"text\":{\"content\":\"Blocki\"}}]}
    },
    \"markdown\": \"이 페이지는 Blocki가 만든 로그 폴더입니다.\\n\"
  }"
```

응답 `id`를 `ROOT_PAGE_ID`로 두고 위 databases POST를 반복한다.

## 8. 스키마 변경 규칙

속성을 추가할 때는:

1. `schema_version`을 `v2`로
2. F2가 기존 DB에 `update data source properties` (있으면)
3. 없는 속성만 추가. 이름 변경/삭제는 하지 않음
4. 이 문서와 DESIGN.md를 같이 수정

1차에서 속성을 “나중에 쓰려고” 미리 넣지 않는다. 일정, 마일스톤, 팀원 relation은 `12-phase-2.md`.

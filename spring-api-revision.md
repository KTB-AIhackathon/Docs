# Spring ↔ FastAPI — 필수 계약

작성: Blocki-AI  
대상: `api-specification.md` §1.1  
공개 `/api/v1` 경로·DTO는 바꾸지 않는다.

---

## 책임 (이 선을 깨지 않는다)

```text
Front (웹)  ↔  Spring  ↔  DB
                 ↕ JSON REST
              FastAPI (AI)  ↔  Git
                     ↓ md
                  Notion
```

| 담당 | 한다 | 하지 않는다 |
| --- | --- | --- |
| **Front** | Spring `/api/v1` 만 호출 | FastAPI, Git, Notion, DB |
| **Spring** | 공개 API, JWT, 토큰 금고, **DB**, 202 Job | GitHub API, Notion API, LangGraph, MCP |
| **FastAPI** | Git 수집·산출, Notion에 md 쓰기 | DB, 로그인, 공개 API |

**markdown 원본은 Spring DB다.** FastAPI는 DB를 열지 않는다. 생성한 `.md` 는 JSON REST로 Spring에 돌려주고, **Spring이 INSERT/UPDATE 한다.**

```text
React ── generate ──► Spring                FastAPI
                        │                      │
                        │  POST /internal/jobs │
                        │  + X-Internal-Key    │
                        │  + X-GitHub-Pat      │
                        │ ───────────────────► │  Git 수집, md 생성
                        │                      │  (Notion md 쓰기도 FastAPI)
                        │  200 JobResult       │
                        │  artifact.body_markdown
                        │ ◄─────────────────── │
                        │                      │
                        ▼                      │
                       DB  ← Spring만 저장
```

콜백 URL을 새로 만들 필요 없다. Spring이 먼저 치고, 응답의 `artifact.body_markdown` 을 저장하면 된다.

---

## 공개 API에서 유지할 것

연결 대상은 생성 3종이다. URL·202 Job·FE DTO는 그대로다.

- `POST /api/v1/daily-scrums/{date}/generate`
- `POST /api/v1/reflections/{date}/generate`
- `POST /api/v1/documents/generations`

Spring이 비동기 Job을 소유하고, 워커가 아래 내부 호출을 한다.

---

## 반드시 고칠 것 (명세 §1.1 + 구현)

`api-specification.md` 25–30행이 틀렸다. 아래를 버린다.

- Spring이 GitHub·Notion API를 직접 호출한다
- Python은 토큰을 받지 않는다
- Python은 GitHub·Notion을 치지 않는다
- Python은 정제된 업무 데이터만 받는다

교체:

1. GitHub·Notion 외부 호출은 **FastAPI** 가 한다.
2. Spring은 유저 토큰을 암호화해 DB에 두고, 내부 호출 헤더로만 넘긴다. 응답에 토큰을 넣지 않는다.
3. FastAPI는 생성 `.md` 를 REST 응답으로 돌려준다. **저장은 Spring.**

---

## 내부 호출 (구현됨)

`POST {FASTAPI}/internal/jobs`  
헤더: `X-Internal-Key` (필수, 틀리면 401), `X-GitHub-Pat` (없으면 200 + `missing_pat`)

생성 3종 매핑:

| 공개 generate | `job_type` | Spring이 추가로 넣는 것 | DB에 넣을 필드 |
| --- | --- | --- | --- |
| 데일리 스크럼 | `progress_summary` | `job_id`, `user_id` | `artifact.body_markdown` |
| TIL·회고 Draft | `progress_summary` | 동일 | `artifact.body_markdown` |
| 이력서·포폴 | `profile_document` | `document.kind` = `resume`\|`portfolio`, `profile_fields` | `artifact.body_markdown` |

`ok=true` (`proposed` / `partial` / `no_change`) 이면 markdown을 저장한다. `no_change` 는 본문이 빈 문자열일 수 있다. `ok=false` 이면 저장하지 않는다.

이력서/포폴 body 예:

```json
{
  "job_id": "uuid",
  "user_id": "uuid",
  "job_type": "profile_document",
  "document": {
    "kind": "resume",
    "template_version": "v1",
    "profile_fields": {
      "name": "김블로",
      "contact_md": "",
      "experience_md": "",
      "education_md": ""
    }
  }
}
```

응답에서 저장할 것:

```json
{
  "ok": true,
  "artifact": {
    "kind": "resume",
    "title": "이력서",
    "body_markdown": "# ...",
    "proposal_id": "uuid"
  }
}
```

공개 스크럼의 `goals/priorities/warnings` 같은 FE 필드는 Spring이 이 markdown으로 채운다. FastAPI DTO로 공개 스키마를 바꾸지 않는다.

---

## 하지 않는 것

- 공개 API에 PAT 등록 엔드포인트를 추가하라고 하지 않는다. 기존 GitHub 연동 토큰을 `X-GitHub-Pat` 로 넘기면 된다.
- README PR 공개 API는 이번 연결 대상이 아니다.
- FastAPI에 DB URL을 주지 않는다.
- FastAPI Job 테이블을 만들지 않는다. 202/재시도는 Spring.

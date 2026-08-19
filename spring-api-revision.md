# Spring 팀 — `api-specification.md` 수정 요청

작성: Blocki-AI (GitHub 워커)  
대상 문서: `Docs/api-specification.md` (커밋 `7d6550c`)  
구현: `Blocki-AI` 레포 FastAPI (`POST /internal/jobs`, `POST /internal/executions`)

공개 `/api/v1` 경로·DTO는 프론트 계약이니 **그대로 둬도 된다.**  
고쳐야 하는 것은 Spring ↔ FastAPI **책임 경계**와 GitHub 토큰 전달이다.

---

## 1. 한 줄

GitHub 수집·README PR은 FastAPI가 한다. Spring은 토큰 금고와 공개 API만 맡는다. FastAPI에 정제된 GitHub JSON만 넘기는 구조는 쓰지 않는다.

---

## 2. 바꾸지 말 것

| 항목 | 이유 |
| --- | --- |
| 브라우저 → `GET/POST /api/v1/**` | FE 계약. FastAPI는 공개하지 않음 |
| `I01–I03` 연동 목록·OAuth 시작/콜백 URL | 공개 UX. 구현은 Spring |
| `D*` 스크럼, `R*` 회고, `M*` 문서의 **공개** 스키마 | FE가 그 모양으로 그린다 |
| 202 Job + 재시도 소유가 Spring | FastAPI는 동기 1회. Spring이 감싼다 |
| 응답에 토큰·내부 URL을 넣지 않는 규칙 | 그대로 |

---

## 3. 지금 틀린 문장 — 이렇게 고쳐라

### 3.1 §1.1 책임 경계 (핵심)

**현재** (`api-specification.md` 25–30행)

- Spring이 Notion·GitHub API를 직접 호출하고 수집 실패를 분류한다
- Python은 OAuth 토큰·API 키를 받거나 보관하지 않는다
- Python은 Notion·GitHub에 직접 호출하지 않는다
- Python은 사용자 식별자와 **정제된 업무 데이터만** 입력으로 받는다

**수정**

| 구간 | Spring | FastAPI (Blocki-AI) |
| --- | --- | --- |
| 공개 API·보안 | `/api/v1/**`, JWT, 소유권, 공통 오류 | 공개하지 않음 |
| GitHub | OAuth/PAT **저장·암호화**, 스케줄이 job을 침, proposal/버전 DB, 승인 UI | **유저 PAT로 GitHub MCP 읽기**, 진행 메모·이력서/포폴·README 초안 생성, 승인 후 README PR |
| Notion | OAuth, 토큰 금고, TIL 발행(`R05`), 산출 `.md` 미러 | **호출하지 않음. 토큰도 받지 않음** |
| Job | 공개 202 Job, 재시도, DB 상태 | 내부 동기 `200 JobResult`. Job 테이블을 만들지 않음 |

Python 입력 문장을 아래로 교체한다.

> Spring은 내부망에서만 FastAPI를 호출한다.  
> `POST /internal/jobs`, `POST /internal/executions`  
> 헤더: `X-Internal-Key`, `X-GitHub-Pat`  
> Body에 토큰을 넣지 않는다. FastAPI는 PAT를 저장하지 않고 요청이 끝나면 버린다.

**이유**

1. 기능명세 `Docs/README.md` 4.2: GitHub는 “PAT 또는 OAuth”. 4.3 PERS-001: **GitHub MCP로** 커밋·이슈·PR 조회. PERS-004: README 개선안 후 PR. 수집 주체는 MCP 워커다.
2. 팀이 잠근 흐름은 `Spring → FastAPI → GitHub MCP` 이다. `Blocki-AI/DESIGN.md` §3, §7.1.
3. GitHub remote MCP (`https://api.githubcopilot.com/mcp/`)는 요청마다 `Authorization: Bearer {user_token}` 이 필요하다. Spring이 먼저 GitHub를 긁어 FastAPI에 “정제 데이터만” 주면 MCP 워커가 할 일이 없어진다.
4. FastAPI가 PAT를 body/DB에 넣지 않고 헤더로만 받는 이유는 유저 교차 사용을 막기 위해서다. 클라이언트를 job마다 새로 연다.

다이어그램도 바꿔라.

```text
React ── /api/v1 ──> Spring ── /internal/jobs|executions + X-GitHub-Pat ──> FastAPI
                       │                                              │
                       ├── PostgreSQL (유저·토큰금고·proposal·Job)     └── GitHub MCP
                       └── Notion OAuth/API (발행·미러만)
```

### 3.2 GitHub 연결: OAuth만으로 고정하지 말 것

**현재** I02/I03 `provider=github` 는 OAuth만 있다. PAT 등록 API가 없다.

**수정 (택1, 해커톤은 A 권장)**

- **A.** 공개 API에 PAT 등록을 추가한다.  
  `PUT /api/v1/integrations/github/token`  
  body: `{ "token": "ghp_..." }`  
  저장은 Spring 암호화. 응답에 토큰 금지.  
  I01의 `GITHUB` 상태를 `CONNECTED`로 바꾼다.
- **B.** 공개 UX는 GitHub OAuth를 유지한다. 그 경우 Spring이 받은 access token을 FastAPI에 `X-GitHub-Pat` 로 그대로 넘긴다. 이름은 PAT지만 Bearer 토큰이면 된다. README PR을 쓰려면 그 토큰에 Contents + Pull requests 쓰기 권한이 있어야 한다.

**이유**

- 기능명세 4.2가 “PAT 또는 OAuth”다.
- Remote GitHub MCP는 브라우저 Copilot OAuth가 아니라 요청 헤더 토큰이다. 크론/배치에 브라우저 세션을 쓸 수 없다 (`DESIGN.md` §7.3).
- 쓰기 없는 사용자는 read-only 토큰으로 `/internal/jobs` 만 동작하면 된다. PR은 그때 401 → `github_auth`.

### 3.3 FastAPI 산출 범위를 §1.1 AI 기능 칸에 반영

**현재:** FastAPI는 데일리 스크럼·TIL Draft·이력서·포트폴리오만 생성한다.

**수정:** FastAPI `job_type` 은 아래 셋이다. 공개 리소스로의 **매핑은 Spring** 이 한다.

| 공개 API | Spring이 FastAPI에 보내는 `job_type` | FastAPI가 돌려주는 것 |
| --- | --- | --- |
| `POST /daily-scrums/{date}/generate` | `progress_summary` | `artifact.body_markdown` + `snapshot_summary` (커밋/이슈/PR 수) |
| `POST /reflections/{date}/generate` | `progress_summary` (같은 수집, 다른 프롬프트가 필요하면 이후 `job_type` 추가) | 동일. Spring이 TIL Draft markdown으로 저장 |
| `POST /documents/generations` `{type:RESUME\|PORTFOLIO}` | `profile_document` + `document.kind` + `profile_fields` | 템플릿이 채운 markdown + `template_ref` |
| (공개 API 없음, 데모/설정) | `readme_proposal` → 유저 승인 후 `/internal/executions` | README 초안 + `proposed_action` |

공개 스크럼 응답의 `goals/priorities/warnings` 는 Spring이 markdown에서 나누거나, 1차는 markdown을 그대로 넣고 배열은 요약 문장 1개로 채워도 된다. **공개 스키마를 FastAPI DTO로 바꾸지 마라.**

`missingSources: ["GITHUB"]` 매핑:

- FastAPI `error.code == missing_pat | github_auth` → GitHub 미연동/실패
- `snapshot_summary.complete == false` 또는 `proposal.status == partial` → `PARTIALLY_SUCCEEDED` + `missingSources` 필요 시 `GITHUB`
- `proposal.status == no_change` → 성공. 새 본문 없음. cursor 유지 가능

**이유:** 공개 제품은 스크럼/회고/문서가 맞고, GitHub 수집 단위는 레포 활동 스냅샷이다. 두 레이어를 한 DTO로 합치면 FastAPI가 Spring Job 테이블을 알게 된다.

### 3.4 README PR은 빠진 기능이다 — 지우라는 뜻이 아님

기능명세 PERS-004~005 와 `DESIGN.md` F3-readme / F4 가 있다. 공개 명세 6장 제외 목록에도 README가 없다. 그냥 엔드포인트가 안 적혀 있다.

**수정 (데모에 넣을 경우, 최소)**

- 문서 생성과 별도. 예: `POST /api/v1/readme-proposals` → Spring이 `readme_proposal` job
- 승인: `POST /api/v1/readme-proposals/{id}/approve` → Spring이 저장해 둔 `action` + `action_digest` 를 `/internal/executions` 로 전달
- FastAPI는 `approved: true` 불리언을 보지 않는다. digest와 expected SHA만 본다

데모에 안 넣으면 공개 API는 비워 두되, §1.1에서 “FastAPI는 GitHub에 쓰지 않는다”고 쓰지 마라. 쓰면 F4와 모순이다.

### 3.5 Notion 발행 주체

**현재 §1.1:** FastAPI는 Notion을 직접 쓰지 않는다. `R05` 는 Spring.

**유지.** Blocki-AI 구현에서 Notion 헤더·클라이언트를 제거했다.

Spring은 FastAPI 응답의 `artifact.body_markdown` 을 DB에 저장한 뒤, Notion 미러/TIL 발행을 **Spring Notion 모듈**에서 한다. FastAPI에 `X-Notion-Token` 을 보내지 마라.

---

## 4. 내부 HTTP 계약 (구현됨, 이 레포가 소스)

Base: FastAPI. 내부망만. OpenAPI `/docs`.

공통 헤더

| Header | 필수 | 규칙 |
| --- | --- | --- |
| `X-Internal-Key` | 예 | env `INTERNAL_API_KEY` 와 불일치면 **HTTP 401**. 그래프 진입 전 |
| `X-GitHub-Pat` | job/execution | 공백이면 HTTP 200 + `error.code=missing_pat`, `ok=false` |
| `X-Notion-Token` | **금지** | 있어도 무시한다 |

### `POST /internal/jobs` → `JobResult`

```json
{
  "job_id": "uuid",
  "user_id": "uuid",
  "job_type": "progress_summary",
  "repos": [{"owner": "acme", "name": "demo"}],
  "since": null,
  "cursor": [{"owner": "acme", "name": "demo", "head_sha": "abc", "last_success_at": "2026-08-19T09:00:00Z"}],
  "document": null,
  "readme": null
}
```

`profile_document` 이면 `document` 필수:

```json
{
  "kind": "resume",
  "template_version": "v1",
  "profile_fields": {
    "name": "김블로",
    "contact_md": "me@example.com",
    "experience_md": "...",
    "education_md": "..."
  }
}
```

`readme_proposal` 이면 `readme` 필수: `{ "owner", "repo", "path": "README.md" }`  
`path` 는 `README` / `docs/README` 계열만. 그 외 422.

성공 시 Spring이 저장할 필드

- `artifact.body_markdown` — 원본. FastAPI는 DB를 열지 않는다
- `proposal.proposal_id`, `proposal_digest`, `action_digest`, `proposed_action`
- `snapshot_summary` (`complete`, `repo_count`, `commit_count`, `issue_count`, `pr_count`)
- `next_cursor` — **`snapshot_summary.complete == true` 일 때만** 덮어쓴다. false면 이전 cursor 유지

`ok=true`: `proposed` / `no_change` / `partial`  
`ok=false`: `failed` / `blocked` 또는 수집 실패

에러 코드: `missing_pat`, `github_auth`, `github_rate_limit`, `mcp_unavailable`, `llm_failed`, `blocked`, `validation`, `internal`

### `POST /internal/executions` → `ExecuteResult`

```json
{
  "execution_id": "uuid",
  "proposal_id": "<저장한 proposal_id>",
  "action_digest": "<저장한 action_digest>",
  "idempotency_key": "<proposal_id와 동일>",
  "action": { "type": "create_readme_pr", "owner": "", "repo": "", "path": "README.md", "base_branch": "main", "expected_base_sha": "", "expected_blob_sha": "", "replacement_markdown": "", "pr_title": "", "pr_body": "" }
}
```

`status`: `created` | `duplicate` | `rejected`  
클라이언트가 `action` 을 고치면 digest 불일치로 `rejected`.

---

## 5. Spring 체크리스트

1. §1.1 표·다이어그램·“정제 데이터만” 문장을 위 3.1으로 교체
2. GitHub 토큰을 유저별로 암호화 저장. FastAPI URL·키는 서버 env
3. 공개 generate API → 내부 `/internal/jobs` 동기 호출 → 결과를 Job/문서/스크럼 테이블에 저장 → 공개 202/폴링은 기존대로
4. FastAPI에 Notion 토큰을 넘기지 않음. `R05` 는 Spring
5. README를 데모할지 공개 API에 최소 승인 엔드포인트를 넣을지 결정. 내부 executions 계약은 이미 있다
6. `complete=false` 이면 cursor 미갱신

질문 있으면 Blocki-AI `DESIGN.md` §7이 우선이다.

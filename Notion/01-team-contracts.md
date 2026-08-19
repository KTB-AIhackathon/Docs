# 01. 팀 계약 — Notion이 지키고 깨면 안 되는 선

이 문서는 `Docs/README.md`와 `Blocki-AI/DESIGN.md` v0.2 §7을 Notion 관점에서 다시 쓴 것이다.
구현 중에 “편하니까 FastAPI에서 한 줄 더”가 생각나면 이 파일을 다시 연다.

## 1. 제품에서 Notion의 자리

기능명세 한 줄: GitHub + Notion 데이터로 진행·포폴을 자동 관리한다.

이미 팀이 나눈 실제 흐름은 이것이다.

```
유저
  ├─ Frontend (로그인, 미리보기, 승인 버튼, Notion 연결 버튼)
  └─ Spring
        ├─ 유저/JWT/토큰 금고/스케줄/proposal 원본 DB
        ├─ POST /internal/jobs  ──────────────► Blocki-AI (GitHub 수집·산출)
        ├─ POST /internal/executions ─────────► Blocki-AI (README PR)
        └─ (연결돼 있으면) Notion 모듈 ────────► api.notion.com (로그 미러)
```

Notion은 **세 번째 화살표**만 담당한다.

## 2. 담당 표

| 담당 | 한다 | 하지 않는다 |
| --- | --- | --- |
| **Notion 팀원** | Public OAuth, 토큰 암호화 저장(Spring 금고 재사용), 루트/DB 생성, proposal `.md`를 Notion 페이지로 남김, 동기화 상태, 재연결 안내 | GitHub PAT, GitHub MCP, LangGraph, 템플릿 문구 생성, 승인 게이트, FastAPI 라우트 추가 |
| **AI / GitHub** | `JobRequest`/`JobResult`/`ExecuteRequest`, GitHub MCP 읽기·쓰기, 템플릿 치환, digest | Notion 페이지, Notion 토큰, `JobRequest.notion` |
| **Spring 공통** | JWT, 스케줄이 FastAPI를 때리는 시계, proposal 테이블, 알림 | Notion API 세부 — `com.blocki.notion`에 위임 |
| **Frontend** | `/api/integrations/notion/*` 호출, 연결 배지 | 토큰을 localStorage에 넣기 |

「깃허브 파트 전부」는 GitHub 읽기·산출·승인 후 PR이다.
「Notion 파트」는 OAuth와 과거 산출 `.md` 로그다.
토큰 금고는 둘 다 Spring이다. Notion 팀원이 금고를 새로 만들지 말고, GitHub PAT와 **같은 암호화기**에 키만 다른 row로 넣는다.

## 3. 산출물 소유권 (잠김)

`Blocki-AI/DESIGN.md` §7.4 그대로다.

1. FastAPI는 완성 `body_markdown` + `template_ref` + `snapshot_digest` + `proposal_digest`를 반환한다.
2. **원본 버전은 Spring DB.** 미리보기·승인·이력의 기준이다.
3. Notion은 같은 `.md`를 로그로 붙인다.
4. FastAPI는 Notion을 호출하지 않는다.

따라서:

- Notion 페이지를 유저가 지워도 Spring 미리보기는 살아 있어야 한다.
- Spring 행을 지워야 이력이 사라진다. Notion만 지웠다고 원본이 사라지면 안 된다.
- 포폴을 “Notion에서 고치면 서비스에 반영”하는 양방향 동기화는 **1차 금지**. 하면 원본이 두 개가 된다.

## 4. 승인과의 관계

승인은 GitHub README PR 전용이다.

| 산출 | 승인 필요 | Notion |
| --- | --- | --- |
| `progress_summary` | 없음. 미리보기 + 알림 | 연결돼 있으면 바로 미러 |
| `profile_document` (portfolio/resume) | 서비스 미리보기. GitHub 쓰기는 없음 | 연결돼 있으면 바로 미러 |
| `readme_proposal` | 있음. 승인 후에만 F4 PR | 1차는 Notion에 안 남김 |

Notion 쓰기에 `approved: true` 같은 필드를 만들지 않는다.
유저가 Notion 연결을 했다는 것 자체가 “내 워크스페이스에 로그를 남겨도 된다”는 동의다.

## 5. FastAPI 계약을 건드리지 않는 법

아래를 **추가하거나 되살리지 않는다.**

- `JobRequest.notion`
- `ArtifactProposal`의 Notion URL 필드 (필요하면 Spring이 자기 테이블에 둔다)
- FastAPI 헤더 `X-Notion-Token`
- FastAPI에서 `https://mcp.notion.com/mcp` 클라이언트

Spring이 FastAPI 응답을 받은 뒤 하는 일:

```
JobResult.ok && proposal.status in {proposed, partial}
  && proposal.body_markdown not empty
  && proposal.kind in {progress, portfolio, resume}
  && user has Notion connection
→ insert notion_sync_outbox
```

이 분기문은 Spring(조립기)에만 있다.

## 6. 토큰 규칙

GitHub PAT 규칙과 같은 정신이다.

| 항목 | GitHub | Notion |
| --- | --- | --- |
| 저장 | Spring 암호화 | Spring 암호화 |
| 전달 | FastAPI에 `X-GitHub-Pat` | Notion 모듈 메모리에만. HTTP 응답 금지 |
| 수명 | 유저가 만든 PAT | OAuth access + refresh |
| 크론 | Spring이 헤더로 주입 | Spring이 refresh 후 REST 호출 |
| 로그 | PAT·Authorization 금지 | access_token·refresh_token·Authorization 금지 |

Notion 토큰을 FastAPI에 넘길 이유가 없다.

## 7. 실패 격리

| 실패 | GitHub 잡 | Spring proposal | Notion outbox | 유저 |
| --- | --- | --- | --- | --- |
| FastAPI 실패 | 실패 | 안 생김 | 안 생김 | 잡 실패 알림 |
| Spring 저장 실패 | 이미 200이어도 재호출 가능 | 없음 | 없음 | 재시도 |
| Notion 429/5xx | 영향 없음 | 유지 | `failed` retryable | 대시보드에 “Notion 동기화 실패” |
| Notion 재인증 필요 | 영향 없음 | 유지 | `failed` + `needs_reauth` | “Notion 다시 연결” |
| Notion 미연결 | 영향 없음 | 유지 | 없음 또는 `skipped` | 연결 유도 |

데모에서 GitHub 진행 메모가 보이면 성공이다. Notion이 죽어 있어도 데모는 살아야 한다.

## 8. 기능명세 ID 매핑

명세를 모두 1차에 넣지 않는다. Notion 파트가 지금 닫는 ID만 적는다.

| 명세 ID | 1차 Notion | 처리 |
| --- | --- | --- |
| AUTH-001~004 | 간접 | 프로필에 연결 상태만 표시. 로그인 자체는 Spring |
| INTEG-001 | 안 함 | GitHub 파트 |
| INTEG-002 | 함 | Notion OAuth (`02-oauth.md`) |
| INTEG-003~004 | 함 | 상태 조회·해제 |
| INTEG-005~006 | 함 | 유저별 암호화 저장 |
| PERS-001 | 안 함 | GitHub 수집 |
| PERS-002 | 함 | 진행 메모 미러 |
| PERS-003 | 안 함 | 일정 대비 진행률. 캘린더 제외와 충돌. 2차 |
| PERS-004~005 | 안 함 | README는 GitHub |
| TEAM-001~007 | 안 함 | 2차 (`12-phase-2.md`) |
| PROJ-001~006 | 안 함 | 2차 |
| PORT-001~005 | 일부 | 카드 JSON이 아니라 `.md` 버전 로그. 공개 페이지는 Frontend/Spring |
| AGENT-001~005 | 간접 | 스케줄은 Spring. Notion은 outbox worker |
| NOTI / DASH | 일부 | 연결 상태·마지막 동기화만 |

## 9. 프론트와 맞출 문장

설정 화면에 이렇게 쓴다. 구현 문구를 바꾸지 말 것.

- 연결 전: “Notion을 연결하면 진행 메모와 포트폴리오 초안이 내 워크스페이스에 로그로 쌓입니다. 원본은 이 서비스에 있습니다.”
- 연결 후: “연결됨 · {workspace_name}” + 루트 페이지 링크
- 재인증: “Notion 권한이 만료되었습니다. 다시 연결해 주세요.”
- 해제: “연결을 끊어도 Notion에 이미 만들어진 페이지는 삭제되지 않습니다.”

## 10. 충돌이 생기면

우선순위:

1. `Blocki-AI/DESIGN.md`의 HTTP/산출/승인 계약
2. 이 폴더의 `DESIGN.md`
3. 기능명세의 “2차 이후” 항목
4. 편의를 위한 새 필드

기능명세가 “Notion MCP로 진행 메모를 자동 추가”라고 해도, 호출 주체는 FastAPI가 아니라 **Spring Notion 모듈**이다.
호스트 MCP 대신 REST를 쓰는 이유는 `11-mcp-vs-rest.md`.

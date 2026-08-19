# Blocki Notion 파트 — 구현 핸드북

이 폴더는 **Notion 담당 팀원**이 바로 구현할 수 있도록 쓴 설계다.
기능명세(`Docs/README.md`)와 GitHub/AI 설계(`Blocki-AI/DESIGN.md` v0.2)를 다시 읽고, 이미 잠긴 경계를 깨지 않는 범위에서 Notion만 설계했다.

코드는 아직 없다. 이 문서가 구현의 원본이다.

## 한 줄

Spring이 저장한 산출물 `.md`를, 유저가 연결한 Notion 워크스페이스에 **로그로 남긴다.**
원본은 Spring DB다. Notion은 미러다. FastAPI는 Notion을 호출하지 않는다.

## 누구를 위한 문서인가

| 사람 | 이 폴더에서 할 일 |
| --- | --- |
| Notion 담당 | 위에서 아래 순서로 읽고, `08-implementation-playbook.md` 체크리스트대로 구현한다 |
| Spring 담당 | `01-team-contracts.md`, `03-data-model.md`, `06-http-api.md`만 먼저 맞춘다 |
| AI / GitHub 담당 | Notion을 호출하지 않는다. 계약은 `01-team-contracts.md`만 확인한다 |
| Frontend 담당 | `06-http-api.md`의 공개 API만 본다 |

## 읽는 순서

구현 전 **반드시 1 → 2 → 3**을 읽는다. 나머지는 구현할 때 해당 파일만 연다.

| 순서 | 파일 | 내용 |
| --- | --- | --- |
| 1 | [DESIGN.md](./DESIGN.md) | 아키텍처, 모듈, 파이프라인, 결정 로그 |
| 2 | [01-team-contracts.md](./01-team-contracts.md) | 팀 경계, 하지 말 것, 산출물 소유권 |
| 3 | [11-mcp-vs-rest.md](./11-mcp-vs-rest.md) | 왜 호스트 MCP가 크론 쓰기가 아닌지 |
| 4 | [02-oauth.md](./02-oauth.md) | Public connection OAuth, 토큰 금고, refresh |
| 5 | [03-data-model.md](./03-data-model.md) | Spring 테이블, 엔티티, 상태 기계 |
| 6 | [04-workspace-schema.md](./04-workspace-schema.md) | Notion 루트 페이지·DB 스키마 |
| 7 | [05-write-pipeline.md](./05-write-pipeline.md) | outbox, 멱등, 어떤 산출을 쓰는지 |
| 8 | [07-markdown-and-pages.md](./07-markdown-and-pages.md) | `.md` → Notion 페이지, 크기 한도 |
| 9 | [06-http-api.md](./06-http-api.md) | Spring 공개 API 요청/응답 전문 |
| 10 | [08-implementation-playbook.md](./08-implementation-playbook.md) | 0일차부터 복붙 가능한 구현 순서 |
| 11 | [09-testing.md](./09-testing.md) | 계약 테스트, E2E, 합격 기준 |
| 12 | [10-errors-and-runbook.md](./10-errors-and-runbook.md) | 에러 코드, 운영, 재연결 |
| 13 | [12-phase-2.md](./12-phase-2.md) | 팀 보드·일정 비교 (1차에 하지 않음) |

## 잠긴 전제 (여기서 뒤집지 않는다)

이미 `Blocki-AI/DESIGN.md`와 기능명세에서 확정된 값이다.

1. FastAPI는 Notion을 호출하지 않는다. `JobRequest`에 `notion` 필드가 없다.
2. 포폴/이력서/진행 메모의 **원본 버전은 Spring DB**다.
3. Notion은 같은 `.md`를 나중에 로그로 붙인다.
4. GitHub 수집·README PR·승인 게이트는 Notion 파트가 아니다.
5. 캘린더는 제외다.
6. 유저별 토큰은 완전히 격리하고, Spring이 암호화해서 보관한다.
7. 배포는 EC2 + Docker다.

이 전제를 깨면 GitHub 파트와 계약이 깨진다. 바꾸고 싶으면 문서부터 고치고 팀에 말한다.

## 1차(MVP)에서 Notion이 하는 일

기능명세 4.2 / 4.3 / 4.6 / 5절의 Notion 조각을 **개인 사용자** 범위로만 구현한다.

| 한다 | 하지 않는다 (1차) |
| --- | --- |
| Notion Public OAuth 연결·해제·상태 | 호스트 MCP로 크론 쓰기 |
| 유저별 Blocki 루트 + 3개 DB 자동 생성 | 팀 공유 보드 |
| 진행 메모 `.md`를 Progress Logs에 한 행으로 남김 | GitHub MCP 호출 |
| 포폴/이력서 `.md` 버전을 로그 페이지로 남김 | 승인 UI, PAT 금고 |
| 같은 `proposal_id`는 한 페이지만 | 일정·마일스톤 계산 |
| Notion 쓰기 실패해도 GitHub 잡은 성공 유지 | 캘린더 |

## 코드가 들어갈 위치

| 위치 | 이유 |
| --- | --- |
| `Blocki-Backend` 패키지 `com.blocki.notion` | OAuth·토큰·스케줄·버전 DB가 이미 Spring 소유 |
| `Blocki-Frontend` 설정 화면의 Notion 연결 버튼 | 공개 API만 호출 |
| `Blocki-AI` | **파일 추가 금지** |

별도 `Blocki-Notion` 워커는 1차에 만들지 않는다. 배포 단위를 하나 더 늘리지 않기 위해서다. 나중에 분리할 계약은 `DESIGN.md` §7.2에 적어 두었다.

## 구현을 시작하는 최소 명령

로컬에서 OAuth 없이 Notion API가 살아 있는지 먼저 확인한다. 토큰은 본인 개발용 Internal connection 또는 PAT다. **이 토큰을 커밋하지 않는다.**

```bash
export NOTION_TOKEN="ntn_..."   # 또는 secret_...
export NOTION_VERSION="2026-03-11"

curl -sS https://api.notion.com/v1/users/me \
  -H "Authorization: Bearer $NOTION_TOKEN" \
  -H "Notion-Version: $NOTION_VERSION"
```

200과 bot/user JSON이 나오면 `08-implementation-playbook.md` Day 0으로 간다.

## 문서 버전

- 작성일: 2026-08-19
- 기준 문서: `Docs/README.md`, `Blocki-AI/DESIGN.md` v0.2
- Notion API 버전: `2026-03-11`
- 이 폴더 버전: 1.0

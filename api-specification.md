---
description: Blocki의 인증, Notion·GitHub 연동, 데일리 스크럼, TIL·회고, AI 문서 생성·버전 조회를 프론트엔드와 백엔드가 구현·검증할 수 있는 REST API 계약으로 정의합니다.
---

# Blocki REST API 명세

## 1. 문서 범위와 요구사항 점검

이 문서는 `docs/project-plan.md`와 `docs/page-specification.md`의 MVP 사용자 흐름을 공개 REST API로 구체화한다. 브라우저는 Spring Boot의 `/api/v1`만 호출하며, Spring Boot만 Notion·GitHub 및 내부 FastAPI를 호출한다. OAuth 토큰·API 키·내부 FastAPI URL은 어떤 응답에도 포함하지 않는다.

이 문서가 Blocki 공개 API의 유일한 계약 문서다.

### 1.1 Spring Boot와 Python(FastAPI) 책임 경계

```text
React Browser ── 공개 REST API ──> Spring Boot ── 내부 요청 ──> Python FastAPI / LangGraph
                                      │                                  │
                                      ├── PostgreSQL·Redis                └── AI 분석·구조화된 생성 결과
                                      └── Notion·GitHub OAuth/API
```

| 구간 | Spring Boot 책임 | Python(FastAPI) 책임 |
| --- | --- | --- |
| 공개 API·보안 | `/api/v1/**` 전체 라우팅, 회원가입·로그인, Bearer 인증, 사용자별 소유권 검사, 공통 응답·오류 변환 | 공개하지 않음. 브라우저 인증·세션·OAuth state를 처리하지 않음 |
| 외부 연동 | Notion·GitHub OAuth 시작·콜백, 토큰 암호화 보관, 외부 API 호출, 수집 실패 분류 | OAuth 토큰·API 키를 받거나 보관하지 않음. Notion·GitHub에 직접 호출하지 않음 |
| 데이터·상태 | 사용자·연동·Todo·Draft·문서·버전·Job을 PostgreSQL에 저장하고 Redis로 중복 작업을 제어, 작업 상태·재시도·발행 결과를 관리 | Spring Boot가 전달한 정제 데이터의 유효성을 검사하고 AI 파이프라인 실행 결과(성공·부분 성공·실패, 누락 근거)를 반환 |
| AI 기능 | 요청 대상·사용자 소유권을 검증하고 수집 데이터를 정제해 내부 요청을 만들며, 검증된 Markdown만 저장 | 데일리 스크럼·TIL Draft·이력서·포트폴리오의 근거 기반 생성, Pydantic 검증, 사실 외 내용 미생성 |
| Notion 발행 | 확정된 TIL Draft를 Notion에 저장하고 URL·식별자·발행 상태를 반환 | Notion에 직접 쓰지 않음 |

- Python은 Spring Boot가 전달한 사용자 식별자와 **정제된 업무 데이터**만 입력으로 받는다. 비밀번호, access token, OAuth token, API key는 전달하지 않는다.
- Spring Boot는 Python 결과를 신뢰하지 않고 구조·길이·허용 상태를 검증한 뒤 공개 응답과 저장 모델로 변환한다.
- Spring Boot가 비동기 Job·자동 재시도·결과 저장을 소유한다. FastAPI는 Job ID·DB 상태를 만들지 않고, Spring Boot 작업 워커의 동기 내부 요청에 생성 결과 또는 구조화된 실패만 반환한다.

### 1.2 사용자 기능·리소스 추출

| 사용자 기능 | 핵심 리소스 | 근거 |
| --- | --- | --- |
| 계정 생성·로그인·내 정보 확인 | User, AuthToken | 로그인·회원가입·사이드바 |
| Notion·GitHub 연결과 상태 확인 | Integration | 워크스페이스·설정 |
| 오늘의 요약 확인 | Workspace | 워크스페이스 |
| 날짜별 할 일·AI 스크럼 조회, 생성, 완료 처리 | DailyScrum, Todo, DailyScrumJob | 데일리 스크럼 |
| 날짜별 근거·TIL Draft 조회, 생성, 편집, Notion 발행 | Reflection, ReflectionJob | TIL·회고 |
| 이력서·포트폴리오 생성, 작업 상태·버전 조회 | Document, DocumentVersion, DocumentGenerationJob | 내 문서 |

### 1.3 기능-API 매핑표

| 기능 | API | 비고 |
| --- | --- | --- |
| 회원가입 | `POST /auth/sign-up` | 비인증 |
| 로그인 | `POST /auth/login` | 비인증, Bearer 토큰 발급 |
| 내 정보 조회 | `GET /users/me` | 인증 사용자 자신만 |
| 연동 상태 조회·OAuth 시작·콜백 | `GET /integrations`, `GET /integrations/{provider}/authorize`, `GET /integrations/{provider}/callback` | 콜백은 제공자가 호출 |
| 워크스페이스 요약 | `GET /workspace` | 오늘 상태·연동 요약 |
| 스크럼 조회·생성·작업 확인·할 일 완료 | `GET /daily-scrums/{date}`, `POST /daily-scrums/{date}/generate`, `GET /daily-scrum-jobs/{jobId}`, `PATCH /daily-scrums/{date}/todos/{todoId}` | 생성은 비동기 |
| 회고 조회·생성·작업 확인·Draft 저장·Notion 발행 | `GET /reflections/{date}`, `POST /reflections/{date}/generate`, `GET /reflection-jobs/{jobId}`, `PUT /reflections/{date}/draft`, `POST /reflections/{date}/publish` | 생성은 비동기 |
| 문서 생성·작업 확인·목록·본문·버전 조회·PDF 다운로드 | `POST /documents/generations`, `GET /document-generation-jobs/{jobId}`, `GET /documents`, `GET /documents/{documentId}`, `GET /documents/{documentId}/versions`, `GET /documents/{documentId}/versions/{versionId}`, `GET /documents/{documentId}/versions/{versionId}/pdf` | 본문은 읽기 전용, PDF는 요청 시 생성 |

## 2. 공통 규칙·스키마

### 2.1 HTTP와 데이터 규칙

| 항목 | 계약 |
| --- | --- |
| Base URL | `/api/v1` |
| 콘텐츠 형식 | 요청·JSON 응답은 `application/json; charset=utf-8` |
| 인증 | 인증 API는 `Authorization: Bearer {accessToken}` 필수다. access token은 발급 후 1시간에 만료되며 Refresh token·cookie는 사용하지 않는다. 만료 시 프론트는 토큰을 삭제하고 로그인으로 이동한다. |
| 역할 | MVP에는 `USER` 단일 역할만 존재한다. 모든 소유 리소스는 인증된 `userId`와 일치해야 하며 관리자 역할은 제공하지 않는다. |
| ID | 모든 식별자는 UUID v4 문자열. 예: `018f8f5a-7e55-7b7b-9c7d-1a2b3c4d5e6f` |
| 시간·날짜 | 시간은 UTC ISO 8601 RFC 3339 (`2026-08-19T09:00:00Z`), 대상 날짜는 KST 기준 `YYYY-MM-DD` (`2026-08-19`). 서버가 KST 날짜로 해석한다. |
| 열거값 | 대문자 영문 스네이크 케이스. 정의되지 않은 값은 허용하지 않는다. |
| 미제공 값 | 선택 필드는 생략하거나 `null`일 수 있으며, 표의 null 허용 여부를 따른다. 배열은 `null` 대신 빈 배열을 사용한다. |
| 페이지네이션 | 목록은 `page` 0부터, `size` 1~100(기본 20), `sort`는 `field,DESC|ASC`. `PageMeta`를 반환한다. |
| 검색·필터 | 명세에 정의된 Query만 허용한다. 지원하지 않는 필드·값은 `400 INVALID_PARAMETER`이다. |
| 멱등성 | 생성·발행 API는 `Idempotency-Key` 헤더를 지원한다. 같은 사용자·같은 키·같은 요청 본문은 최초 성공/수락 응답을 재반환하며, 키는 24시간 보관한다. 같은 키에 다른 본문은 `409 IDEMPOTENCY_KEY_REUSED`다. |

### 2.2 공통 응답 형식

성공은 항상 아래 형식이다. `data`는 필수이며 `null`이 될 수 없다.

```json
{ "data": {} }
```

실패는 항상 아래 형식이고 HTTP 상태가 4xx/5xx다. `fieldErrors`와 `missingSources`는 해당하지 않으면 생략한다.

```json
{
  "error": {
    "code": "INVALID_PARAMETER",
    "message": "요청 값이 올바르지 않습니다.",
    "traceId": "018f8f5a-7e55-7b7b-9c7d-1a2b3c4d5e6f",
    "fieldErrors": [{ "field": "email", "reason": "유효한 이메일 형식이 아닙니다." }],
    "missingSources": ["GITHUB"]
  }
}
```

### 2.3 공통 객체 필드

| 객체 | 필드(타입 / 필수 / null 허용 / 제약) |
| --- | --- |
| `PageMeta` | `page`(integer/필수/불가/0 이상), `size`(integer/필수/불가/1~100), `totalElements`(integer/필수/불가/0 이상), `totalPages`(integer/필수/불가/0 이상), `sort`(string/필수/불가/요청 정렬값) |
| `FieldError` | `field`(string/필수/불가/JSON 필드 경로), `reason`(string/필수/불가/1~200자) |
| `Integration` | `provider`(enum/필수/불가/`NOTION`,`GITHUB`), `status`(enum/필수/불가/`NOT_CONNECTED`,`CONNECTING`,`CONNECTED`,`ERROR`), `accountLabel`(string/필수/허용/1~100자), `connectedAt`(date-time/필수/허용), `errorCode`(string/필수/허용/공개 오류 코드) |
| `SourceStatus` | `source`(enum/필수/불가/`NOTION`,`GITHUB`,`TODO`), `status`(enum/필수/불가/`AVAILABLE`,`MISSING`,`FAILED`), `count`(integer/필수/불가/0 이상), `reason`(string/필수/허용/1~200자) |
| `Job` | `id`(uuid/필수/불가), `type`(enum/필수/불가/아래 API별 값), `status`(enum/필수/불가/`QUEUED`,`RUNNING`,`SUCCEEDED`,`PARTIALLY_SUCCEEDED`,`FAILED`), `progress`(integer/필수/불가/0~100), `attempt`(integer/필수/불가/1 이상), `maxAttempts`(integer/필수/불가/1 이상), `createdAt`,`startedAt`,`completedAt`(date-time/필수/`startedAt`,`completedAt`만 허용), `retryable`(boolean/필수/불가), `nextRetryAt`(date-time/필수/허용), `errorCode`(string/필수/허용), `missingSources`(Source enum 배열/필수/불가) |
| `Todo` | `id`(uuid/필수/불가), `title`(string/필수/불가/1~500자), `priority`(enum/필수/불가/`HIGH`,`MEDIUM`,`LOW`,`UNSPECIFIED`), `dueDate`(date/필수/허용/원본 마감 날짜), `dueAt`(date-time/필수/허용/원본에 시각이 있을 때만 UTC 반환), `source`(enum/필수/불가/`NOTION`), `completed`(boolean/필수/불가), `updatedAt`(date-time/필수/불가) |
| `EvidenceSummary` | `notionNotes`(integer/필수/불가/0 이상), `githubCommits`(integer/필수/불가/0 이상), `githubPullRequests`(integer/필수/불가/0 이상), `todosCompleted`(integer/필수/불가/0 이상), `todosIncomplete`(integer/필수/불가/0 이상) |
| `DocumentVersion` | `id`(uuid/필수/불가), `version`(integer/필수/불가/1 이상), `createdAt`(date-time/필수/불가), `source`(literal/필수/불가/`AI_GENERATED`) |

### 2.4 비동기 작업·재시도 규칙

- 생성 요청은 `202 Accepted`와 `Job`을 반환한다. `Location: /api/v1/{job-path}/{jobId}` 헤더를 반환한다.
- 같은 사용자·같은 `type`·같은 대상 날짜(스크럼/회고) 또는 문서 유형에 `QUEUED`·`RUNNING` 작업이 하나 있으면 새 작업을 만들지 않고 `409 JOB_ALREADY_RUNNING`과 기존 `jobId`를 반환한다.
- `SUCCEEDED`는 모든 필수 처리가 성공한 상태, `PARTIALLY_SUCCEEDED`는 결과는 생성했지만 `missingSources`가 있는 상태, `FAILED`는 결과 리소스가 생성되지 않은 상태다. `PARTIALLY_SUCCEEDED`는 성공 응답으로 취급한다.
- 일시적 외부·AI 오류만 서버가 자동 재시도한다. 최초 시도 후 30초, 2분 간격으로 재시도해 최대 3회 시도하며, 모두 실패하면 `FAILED`다. 클라이언트는 `retryable=true`인 `FAILED` 작업에 대해 같은 생성 API를 새 `Idempotency-Key`로 호출해 재시도한다. `retryable=false`이면 입력·연동을 보완해야 한다.

## 3. API 요약표

| 이름 | Method / URL | 인증 | 성공 |
| --- | --- | --- | --- |
| 회원가입 | `POST /auth/sign-up` | 없음 | 201 |
| 로그인 | `POST /auth/login` | 없음 | 200 |
| 내 정보 | `GET /users/me` | USER | 200 |
| 연동 목록 | `GET /integrations` | USER | 200 |
| OAuth 시작 | `GET /integrations/{provider}/authorize` | USER | 302 |
| OAuth 콜백 | `GET /integrations/{provider}/callback` | 제공자 state | 302 |
| 워크스페이스 | `GET /workspace` | USER | 200 |
| 스크럼 조회 | `GET /daily-scrums/{date}` | USER | 200 |
| 스크럼 생성 | `POST /daily-scrums/{date}/generate` | USER | 202 |
| 스크럼 작업 | `GET /daily-scrum-jobs/{jobId}` | USER | 200 |
| 할 일 완료 변경 | `PATCH /daily-scrums/{date}/todos/{todoId}` | USER | 200 |
| 회고 조회 | `GET /reflections/{date}` | USER | 200 |
| 회고 생성 | `POST /reflections/{date}/generate` | USER | 202 |
| 회고 작업 | `GET /reflection-jobs/{jobId}` | USER | 200 |
| Draft 저장 | `PUT /reflections/{date}/draft` | USER | 200 |
| Notion 발행 | `POST /reflections/{date}/publish` | USER | 200 |
| 문서 생성 | `POST /documents/generations` | USER | 202 |
| 문서 생성 작업 | `GET /document-generation-jobs/{jobId}` | USER | 200 |
| 문서 목록 | `GET /documents` | USER | 200 |
| 최신 문서 | `GET /documents/{documentId}` | USER | 200 |
| 버전 목록 | `GET /documents/{documentId}/versions` | USER | 200 |
| 특정 버전 | `GET /documents/{documentId}/versions/{versionId}` | USER | 200 |
| 버전 PDF 다운로드 | `GET /documents/{documentId}/versions/{versionId}/pdf` | USER | 200 |

## 4. 상세 명세

아래 표에서 `—`는 해당 입력이 없음을 뜻한다. 모든 인증 API의 미인증 실패 예시는 공통 `401 UNAUTHENTICATED` 형식을 사용한다.

### 4.1 인증·사용자

#### A01. 회원가입 — 계정 생성

- **Method / URL:** `POST /api/v1/auth/sign-up`
- **인증·권한:** 불필요. 동일 `loginId` 또는 `email`은 생성할 수 없다.
- **Path·Query·Header:** 없음. `Content-Type` 필수.
- **Request Body:** `loginId`(string/필수/불가/`^[A-Za-z0-9_-]{4,30}$`), `password`(string/필수/불가/8~72자), `name`(string/필수/불가/trim 후 1~50자), `email`(string/필수/불가/유효 이메일, 최대 254자).

```json
{ "loginId": "blocki_user", "password": "example-password", "name": "김블로", "email": "blocki@example.com" }
```

- **성공:** `201 Created`; `data.id`(uuid), `data.loginId`(string), `data.name`(string), `data.email`(string), `data.createdAt`(date-time)는 모두 필수·null 불가. `Location: /api/v1/users/{id}`.

```json
{ "data": { "id": "018f8f5a-7e55-7b7b-9c7d-1a2b3c4d5e6f", "loginId": "blocki_user", "name": "김블로", "email": "blocki@example.com", "createdAt": "2026-08-19T09:00:00Z" } }
```

- **실패:** 중복 ID `409 LOGIN_ID_ALREADY_EXISTS`, 중복 이메일 `409 EMAIL_ALREADY_EXISTS`, 형식 오류 `400 INVALID_PARAMETER`.

```json
{ "error": { "code": "LOGIN_ID_ALREADY_EXISTS", "message": "이미 사용 중인 로그인 아이디입니다.", "traceId": "018f8f5a-7e55-7b7b-9c7d-1a2b3c4d5e6f" } }
```

#### A02. 로그인 — 액세스 토큰 발급

- **Method / URL:** `POST /api/v1/auth/login`
- **인증·권한:** 불필요.
- **Path·Query·Header:** 없음. `Content-Type` 필수.
- **Request Body:** `loginId`(string/필수/불가/4~30자), `password`(string/필수/불가/1~72자).

```json
{ "loginId": "blocki_user", "password": "example-password" }
```

- **성공:** `200 OK`; `accessToken`(string/필수/불가/비어 있지 않음), `tokenType`(literal/필수/불가/`Bearer`), `expiresAt`(date-time/필수/불가/발급 시각 + 1시간), `user`(A01 성공의 사용자 필드/필수/불가). Refresh token·cookie·서버 로그아웃 API는 제공하지 않는다.

```json
{ "data": { "accessToken": "<redacted>", "tokenType": "Bearer", "expiresAt": "2026-08-19T10:00:00Z", "user": { "id": "018f8f5a-7e55-7b7b-9c7d-1a2b3c4d5e6f", "loginId": "blocki_user", "name": "김블로", "email": "blocki@example.com" } } }
```

- **실패:** 잘못된 자격 증명·비활성 계정은 구분 없이 `401 INVALID_CREDENTIALS`.

```json
{ "error": { "code": "INVALID_CREDENTIALS", "message": "아이디 또는 비밀번호가 올바르지 않습니다.", "traceId": "018f8f5a-7e55-7b7b-9c7d-1a2b3c4d5e6f" } }
```

#### A03. 내 정보 — 현재 사용자 조회

- **Method / URL:** `GET /api/v1/users/me`
- **인증·권한:** Bearer USER. 본인만 가능.
- **Path·Query·Body:** 없음. **Header:** `Authorization` 필수.
- **성공:** `200 OK`; `id`(uuid), `loginId`(string), `name`(string), `email`(string), `createdAt`(date-time)는 필수·null 불가.

```json
{ "data": { "id": "018f8f5a-7e55-7b7b-9c7d-1a2b3c4d5e6f", "loginId": "blocki_user", "name": "김블로", "email": "blocki@example.com", "createdAt": "2026-08-19T09:00:00Z" } }
```

- **실패:** 토큰 누락·만료·위조 `401 UNAUTHENTICATED`.

```json
{ "error": { "code": "UNAUTHENTICATED", "message": "인증이 필요하거나 만료되었습니다.", "traceId": "018f8f5a-7e55-7b7b-9c7d-1a2b3c4d5e6f" } }
```

### 4.2 외부 연동·워크스페이스

#### I01. 연동 목록 — 제공자별 연결 상태 조회

- **Method / URL:** `GET /api/v1/integrations`
- **인증·권한:** Bearer USER, 본인의 두 제공자만 조회.
- **Path·Query·Body:** 없음. **Header:** `Authorization` 필수.
- **성공:** `200 OK`; `items`(Integration 배열/필수/불가/항상 `NOTION`,`GITHUB` 각각 하나), 필드는 [2.3](#23-공통-객체-필드)의 `Integration`을 따른다.

```json
{ "data": { "items": [{ "provider": "NOTION", "status": "CONNECTED", "accountLabel": "Blocki Workspace", "connectedAt": "2026-08-19T09:00:00Z", "errorCode": null }, { "provider": "GITHUB", "status": "NOT_CONNECTED", "accountLabel": null, "connectedAt": null, "errorCode": null }] } }
```

- **실패:** 인증 실패 `401 UNAUTHENTICATED`.

```json
{ "error": { "code": "UNAUTHENTICATED", "message": "인증이 필요하거나 만료되었습니다.", "traceId": "018f8f5a-7e55-7b7b-9c7d-1a2b3c4d5e6f" } }
```

#### I02. OAuth 인가 시작 — 제공자 인가 화면으로 이동

- **Method / URL:** `GET /api/v1/integrations/{provider}/authorize`
- **인증·권한:** Bearer USER, 본인 연결만 시작.
- **Path:** `provider`(string/필수/불가/소문자 `notion` 또는 `github`). **Query·Body:** 없음. **Header:** `Authorization` 필수.
- **성공:** `302 Found`; `Location`(string/필수/불가/제공자 인가 URL). JSON body 없음.
- **실패:** 지원하지 않는 제공자 `400 UNSUPPORTED_PROVIDER`, 이미 연결됨 `409 INTEGRATION_ALREADY_CONNECTED`, 인증 실패 `401 UNAUTHENTICATED`.

```json
{ "error": { "code": "UNSUPPORTED_PROVIDER", "message": "지원하지 않는 연동 제공자입니다.", "traceId": "018f8f5a-7e55-7b7b-9c7d-1a2b3c4d5e6f" } }
```

#### I03. OAuth 콜백 — 제공자 인가 결과 처리

- **Method / URL:** `GET /api/v1/integrations/{provider}/callback`
- **인증·권한:** 브라우저 Bearer 인증 없음. 제공자가 검증 가능한 `state`를 전달해야 하며, state의 사용자 소유권으로 인가한다.
- **Path:** `provider`(string/필수/불가/`notion`,`github`). **Query:** `code`(string/성공 시 필수/불가/1~2,000자), `state`(string/필수/불가/1~2,000자), `error`(string/실패 시 선택/불가/제공자 값), `error_description`(string/선택/허용/최대 1,000자). **Body·Header:** 없음.
- **성공:** `302 Found`; `/workspace?integration={provider}&result=success`로 리다이렉트. JSON body 없음.
- **실패:** state 누락·위조·만료 `400 OAUTH_STATE_INVALID`, 토큰 교환 실패 `502 EXTERNAL_SOURCE_FAILED`. 브라우저 콜백 실패는 `/workspace?integration={provider}&result=failed&error={publicErrorCode}`로 리다이렉트하며, 제공자 원문 오류는 노출하지 않는다. 제공자 서버 직접 오류에는 공통 JSON을 보장하지 않는다.

```json
{ "error": { "code": "OAUTH_STATE_INVALID", "message": "연동 요청을 확인할 수 없습니다. 다시 시도해주세요.", "traceId": "018f8f5a-7e55-7b7b-9c7d-1a2b3c4d5e6f" } }
```

#### W01. 워크스페이스 — 연동·오늘 상태 요약

- **Method / URL:** `GET /api/v1/workspace`
- **인증·권한:** Bearer USER, 본인 데이터만.
- **Path·Query·Body:** 없음. **Header:** `Authorization` 필수.
- **성공:** `200 OK`; `sourceSummary.connectedCount`(integer/필수/불가/0~2), `sourceSummary.totalCount`(literal/필수/불가/`2`), `today.date`(date/필수/불가), `today.dailyScrumStatus`(enum/필수/불가/`NOT_STARTED`,`QUEUED`,`RUNNING`,`SUCCEEDED`,`PARTIALLY_SUCCEEDED`,`FAILED`), `today.reflectionStatus`(동일 enum/필수/불가), `today.documentJobs`(Job 요약 배열/필수/불가).

```json
{ "data": { "sourceSummary": { "connectedCount": 1, "totalCount": 2 }, "today": { "date": "2026-08-19", "dailyScrumStatus": "SUCCEEDED", "reflectionStatus": "NOT_STARTED", "documentJobs": [] } } }
```

- **실패:** 인증 실패 `401 UNAUTHENTICATED`.

```json
{ "error": { "code": "UNAUTHENTICATED", "message": "인증이 필요하거나 만료되었습니다.", "traceId": "018f8f5a-7e55-7b7b-9c7d-1a2b3c4d5e6f" } }
```

### 4.3 데일리 스크럼

#### D01. 스크럼 조회 — 날짜별 근거·할 일·최신 스크럼 조회

- **Method / URL:** `GET /api/v1/daily-scrums/{date}`
- **인증·권한:** Bearer USER, 본인 날짜 데이터만.
- **Path:** `date`(date/필수/불가/`YYYY-MM-DD`). **Query:** 없음. **Header:** `Authorization` 필수. **Body:** 없음.
- **성공:** `200 OK`; `date`(date), `sourceStatuses`(SourceStatus 배열), `todos`(Todo 배열), `dailyScrum`(object/필수/허용: `id` uuid, `goals` string 배열 0~10, `priorities` string 배열 0~10, `warnings` string 배열 0~10, `generatedAt` date-time), `latestJob`(Job/필수/허용). 배열은 필수·null 불가이며 객체 내부 필드는 모두 필수·null 불가다.

```json
{ "data": { "date": "2026-08-19", "sourceStatuses": [{ "source": "NOTION", "status": "AVAILABLE", "count": 3, "reason": null }], "todos": [{ "id": "018f8f5a-7e55-7b7b-9c7d-1a2b3c4d5e6f", "title": "API 명세 작성", "priority": "HIGH", "dueDate": "2026-08-19", "dueAt": "2026-08-19T10:00:00Z", "source": "NOTION", "completed": false, "updatedAt": "2026-08-19T09:00:00Z" }], "dailyScrum": null, "latestJob": null } }
```

- **실패:** 잘못된 날짜 `400 INVALID_PARAMETER`, 인증 실패 `401 UNAUTHENTICATED`.

```json
{ "error": { "code": "INVALID_PARAMETER", "message": "요청 값이 올바르지 않습니다.", "traceId": "018f8f5a-7e55-7b7b-9c7d-1a2b3c4d5e6f", "fieldErrors": [{ "field": "date", "reason": "YYYY-MM-DD 형식이어야 합니다." }] } }
```

#### D02. 스크럼 생성 — AI 생성 작업 수락

- **Method / URL:** `POST /api/v1/daily-scrums/{date}/generate`
- **인증·권한:** Bearer USER, 본인 날짜만.
- **Path:** `date`(date/필수/불가). **Query·Body:** 없음. **Header:** `Authorization`, `Idempotency-Key`(string/필수/불가/UUID 권장, 1~255자).
- **성공:** `202 Accepted`; `Job`의 `type`은 `DAILY_SCRUM_GENERATION`, 결과 식별자는 완료 전 `null`이다.

```json
{ "data": { "id": "018f8f5a-7e55-7b7b-9c7d-1a2b3c4d5e6f", "type": "DAILY_SCRUM_GENERATION", "status": "QUEUED", "progress": 0, "attempt": 1, "maxAttempts": 3, "createdAt": "2026-08-19T09:00:00Z", "startedAt": null, "completedAt": null, "retryable": true, "nextRetryAt": null, "errorCode": null, "missingSources": [] } }
```

- **실패:** 진행 중 작업 `409 JOB_ALREADY_RUNNING`, Notion 미연동·근거 부족 `422 DATA_INSUFFICIENT`, 날짜 오류 `400 INVALID_PARAMETER`, 멱등 키 충돌 `409 IDEMPOTENCY_KEY_REUSED`.

```json
{ "error": { "code": "DATA_INSUFFICIENT", "message": "데일리 스크럼을 생성할 기록이 부족합니다.", "traceId": "018f8f5a-7e55-7b7b-9c7d-1a2b3c4d5e6f", "missingSources": ["NOTION"] } }
```

#### D03. 스크럼 작업 조회 — 생성 진행 상태 확인

- **Method / URL:** `GET /api/v1/daily-scrum-jobs/{jobId}`
- **인증·권한:** Bearer USER, 생성자만.
- **Path:** `jobId`(uuid/필수/불가). **Query·Body:** 없음. **Header:** `Authorization` 필수.
- **성공:** `200 OK`; [2.3](#23-공통-객체-필드)의 `Job` 필수 필드에 `dailyScrumId`(uuid/필수/허용/`SUCCEEDED` 또는 `PARTIALLY_SUCCEEDED`일 때 필수)를 더한다.

```json
{ "data": { "id": "018f8f5a-7e55-7b7b-9c7d-1a2b3c4d5e6f", "type": "DAILY_SCRUM_GENERATION", "status": "PARTIALLY_SUCCEEDED", "progress": 100, "attempt": 1, "maxAttempts": 3, "createdAt": "2026-08-19T09:00:00Z", "startedAt": "2026-08-19T09:00:01Z", "completedAt": "2026-08-19T09:00:05Z", "retryable": false, "nextRetryAt": null, "errorCode": null, "missingSources": ["GITHUB"], "dailyScrumId": "018f8f5a-7e55-7b7b-9c7d-1a2b3c4d5e6f" } }
```

- **실패:** 존재하지 않음 `404 JOB_NOT_FOUND`, 타 사용자 `403 FORBIDDEN`, 인증 실패 `401 UNAUTHENTICATED`.

```json
{ "error": { "code": "JOB_NOT_FOUND", "message": "생성 작업을 찾을 수 없습니다.", "traceId": "018f8f5a-7e55-7b7b-9c7d-1a2b3c4d5e6f" } }
```

#### D04. 할 일 완료 변경 — Blocki 추적 상태만 수정

- **Method / URL:** `PATCH /api/v1/daily-scrums/{date}/todos/{todoId}`
- **인증·권한:** Bearer USER, 본인의 해당 날짜 To-do만.
- **Path:** `date`(date/필수/불가), `todoId`(uuid/필수/불가). **Query:** 없음. **Header:** `Authorization` 필수. **Request Body:** `completed`(boolean/필수/불가).

```json
{ "completed": true }
```

- **성공:** `200 OK`; `Todo` 전체 필드(2.3)를 반환. 같은 값 반복 PATCH는 상태를 유지하고 200을 반환한다. Notion 쓰기는 부수 효과가 없다.

```json
{ "data": { "id": "018f8f5a-7e55-7b7b-9c7d-1a2b3c4d5e6f", "title": "API 명세 작성", "priority": "HIGH", "dueDate": "2026-08-19", "dueAt": "2026-08-19T10:00:00Z", "source": "NOTION", "completed": true, "updatedAt": "2026-08-19T09:10:00Z" } }
```

- **실패:** 날짜/ID 형식 `400 INVALID_PARAMETER`, 대상 없음 `404 TODO_NOT_FOUND`, 타 사용자 `403 FORBIDDEN`.

```json
{ "error": { "code": "TODO_NOT_FOUND", "message": "할 일을 찾을 수 없습니다.", "traceId": "018f8f5a-7e55-7b7b-9c7d-1a2b3c4d5e6f" } }
```

### 4.4 TIL·회고

#### R01. 회고 조회 — 근거와 현재 Draft·발행 결과 조회

- **Method / URL:** `GET /api/v1/reflections/{date}`
- **인증·권한:** Bearer USER, 본인 날짜만.
- **Path:** `date`(date/필수/불가). **Query·Body:** 없음. **Header:** `Authorization` 필수.
- **성공:** `200 OK`; `date`(date/필수/불가), `sourceStatuses`(SourceStatus 배열/필수/불가), `evidenceSummary`(EvidenceSummary/필수/불가), `reflection`(object/필수/허용: `id` uuid, `status` enum `DRAFT|PUBLISHING|PUBLISHED`, `markdown` string 1~100,000자, `evidenceSummary` EvidenceSummary, `missingSources` Source 배열, `notionPageId` string 1~255 nullable, `notionPageUrl` uri nullable, `publishedAt` date-time nullable, `updatedAt` date-time; 모두 필수), `latestJob`(Job/필수/허용).

```json
{ "data": { "date": "2026-08-19", "sourceStatuses": [{ "source": "GITHUB", "status": "AVAILABLE", "count": 4, "reason": null }], "evidenceSummary": { "notionNotes": 2, "githubCommits": 3, "githubPullRequests": 1, "todosCompleted": 2, "todosIncomplete": 1 }, "reflection": null, "latestJob": null } }
```

- **실패:** 날짜 오류 `400 INVALID_PARAMETER`, 인증 실패 `401 UNAUTHENTICATED`.

```json
{ "error": { "code": "INVALID_PARAMETER", "message": "요청 값이 올바르지 않습니다.", "traceId": "018f8f5a-7e55-7b7b-9c7d-1a2b3c4d5e6f", "fieldErrors": [{ "field": "date", "reason": "YYYY-MM-DD 형식이어야 합니다." }] } }
```

#### R02. 회고 생성 — TIL Draft 생성 작업 수락

- **Method / URL:** `POST /api/v1/reflections/{date}/generate`
- **인증·권한:** Bearer USER, 본인 날짜만.
- **Path:** `date`(date/필수/불가). **Query·Body:** 없음. **Header:** `Authorization`, `Idempotency-Key`(string/필수/불가/1~255자).
- **성공:** `202 Accepted`; `Job`이며 `type`은 `REFLECTION_GENERATION`.

```json
{ "data": { "id": "018f8f5a-7e55-7b7b-9c7d-1a2b3c4d5e6f", "type": "REFLECTION_GENERATION", "status": "QUEUED", "progress": 0, "attempt": 1, "maxAttempts": 3, "createdAt": "2026-08-19T09:00:00Z", "startedAt": null, "completedAt": null, "retryable": true, "nextRetryAt": null, "errorCode": null, "missingSources": [] } }
```

- **실패:** 진행 중 `409 JOB_ALREADY_RUNNING`, 근거 부족 `422 DATA_INSUFFICIENT`, 이미 PUBLISHED `409 REFLECTION_ALREADY_PUBLISHED`, 멱등 키 충돌 `409 IDEMPOTENCY_KEY_REUSED`.

```json
{ "error": { "code": "REFLECTION_ALREADY_PUBLISHED", "message": "발행된 회고는 다시 생성할 수 없습니다.", "traceId": "018f8f5a-7e55-7b7b-9c7d-1a2b3c4d5e6f" } }
```

#### R03. 회고 작업 조회 — 생성 진행 상태 확인

- **Method / URL:** `GET /api/v1/reflection-jobs/{jobId}`
- **인증·권한:** Bearer USER, 생성자만.
- **Path:** `jobId`(uuid/필수/불가). **Query·Body:** 없음. **Header:** `Authorization` 필수.
- **성공:** `200 OK`; `Job`에 `reflectionId`(uuid/필수/허용/성공 또는 부분 성공 시 필수)를 더한다.

```json
{ "data": { "id": "018f8f5a-7e55-7b7b-9c7d-1a2b3c4d5e6f", "type": "REFLECTION_GENERATION", "status": "SUCCEEDED", "progress": 100, "attempt": 1, "maxAttempts": 3, "createdAt": "2026-08-19T09:00:00Z", "startedAt": "2026-08-19T09:00:01Z", "completedAt": "2026-08-19T09:00:05Z", "retryable": false, "nextRetryAt": null, "errorCode": null, "missingSources": [], "reflectionId": "018f8f5a-7e55-7b7b-9c7d-1a2b3c4d5e6f" } }
```

- **실패:** 없음 `404 JOB_NOT_FOUND`, 타 사용자 `403 FORBIDDEN`.

```json
{ "error": { "code": "FORBIDDEN", "message": "이 리소스에 접근할 권한이 없습니다.", "traceId": "018f8f5a-7e55-7b7b-9c7d-1a2b3c4d5e6f" } }
```

#### R04. Draft 저장 — 사용자 편집 Markdown 전체 저장

- **Method / URL:** `PUT /api/v1/reflections/{date}/draft`
- **인증·권한:** Bearer USER, 본인 DRAFT 상태만.
- **Path:** `date`(date/필수/불가). **Query:** 없음. **Header:** `Authorization` 필수. **Request Body:** `markdown`(string/필수/불가/trim 후 1~100,000자; HTML 태그를 포함할 수 없는 Markdown 원문).

```json
{ "markdown": "# 오늘의 TIL\n\nAPI 명세를 작성했습니다." }
```

- **성공:** `200 OK`; R01의 `reflection` 객체와 같은 필드이며 `status`는 `DRAFT`.

```json
{ "data": { "id": "018f8f5a-7e55-7b7b-9c7d-1a2b3c4d5e6f", "status": "DRAFT", "markdown": "# 오늘의 TIL\n\nAPI 명세를 작성했습니다.", "evidenceSummary": { "notionNotes": 2, "githubCommits": 3, "githubPullRequests": 1, "todosCompleted": 2, "todosIncomplete": 1 }, "missingSources": [], "notionPageId": null, "notionPageUrl": null, "publishedAt": null, "updatedAt": "2026-08-19T09:10:00Z" } }
```

- **실패:** Draft 없음 `404 REFLECTION_NOT_FOUND`, 발행됨 `409 REFLECTION_ALREADY_PUBLISHED`, 본문 오류 `400 INVALID_PARAMETER`.

```json
{ "error": { "code": "REFLECTION_ALREADY_PUBLISHED", "message": "발행된 회고는 수정할 수 없습니다.", "traceId": "018f8f5a-7e55-7b7b-9c7d-1a2b3c4d5e6f" } }
```

#### R05. Notion 발행 — 확정 Draft를 Notion에 저장

- **Method / URL:** `POST /api/v1/reflections/{date}/publish`
- **인증·권한:** Bearer USER, 본인 DRAFT 상태만.
- **Path:** `date`(date/필수/불가). **Query·Body:** 없음. **Header:** `Authorization`, `Idempotency-Key`(string/필수/불가/1~255자).
- **성공:** `200 OK`; `id`(uuid), `status`(literal `PUBLISHED`), `notionPageId`(string), `notionPageUrl`(uri), `publishedAt`(date-time), `updatedAt`(date-time)는 필수·null 불가. 같은 키·요청의 반복은 같은 결과를 반환한다.

```json
{ "data": { "id": "018f8f5a-7e55-7b7b-9c7d-1a2b3c4d5e6f", "status": "PUBLISHED", "notionPageId": "notion-page-id", "notionPageUrl": "https://www.notion.so/notion-page-id", "publishedAt": "2026-08-19T09:20:00Z", "updatedAt": "2026-08-19T09:20:00Z" } }
```

- **실패:** Draft 없음 `404 REFLECTION_NOT_FOUND`, 이미 발행 `409 REFLECTION_ALREADY_PUBLISHED`, Notion 미연동 `422 DATA_INSUFFICIENT`, 외부 저장 실패 `502 EXTERNAL_SOURCE_FAILED`. 실패해도 Draft는 보존된다.

```json
{ "error": { "code": "EXTERNAL_SOURCE_FAILED", "message": "Notion 저장에 실패했습니다. Draft는 보존되었습니다.", "traceId": "018f8f5a-7e55-7b7b-9c7d-1a2b3c4d5e6f" } }
```

### 4.5 AI 문서·버전

#### M01. 문서 생성 — 새 버전 생성 작업 수락

- **Method / URL:** `POST /api/v1/documents/generations`
- **인증·권한:** Bearer USER, 본인 문서만.
- **Path·Query:** 없음. **Header:** `Authorization`, `Idempotency-Key`(string/필수/불가/1~255자). **Request Body:** `type`(enum/필수/불가/`RESUME`,`PORTFOLIO`).

```json
{ "type": "RESUME" }
```

- **성공:** `202 Accepted`; `Job`이며 `type`은 `DOCUMENT_GENERATION`. 같은 문서 유형의 진행 중 작업은 중복 생성하지 않는다.

```json
{ "data": { "id": "018f8f5a-7e55-7b7b-9c7d-1a2b3c4d5e6f", "type": "DOCUMENT_GENERATION", "status": "QUEUED", "progress": 0, "attempt": 1, "maxAttempts": 3, "createdAt": "2026-08-19T09:00:00Z", "startedAt": null, "completedAt": null, "retryable": true, "nextRetryAt": null, "errorCode": null, "missingSources": [] } }
```

- **실패:** 유형 오류 `400 UNSUPPORTED_DOCUMENT_TYPE`, 진행 중 `409 JOB_ALREADY_RUNNING`, 근거 부족 `422 DATA_INSUFFICIENT`, 키 충돌 `409 IDEMPOTENCY_KEY_REUSED`.

```json
{ "error": { "code": "UNSUPPORTED_DOCUMENT_TYPE", "message": "지원하지 않는 문서 유형입니다.", "traceId": "018f8f5a-7e55-7b7b-9c7d-1a2b3c4d5e6f" } }
```

#### M02. 문서 생성 작업 조회 — 생성 결과 식별자 확인

- **Method / URL:** `GET /api/v1/document-generation-jobs/{jobId}`
- **인증·권한:** Bearer USER, 생성자만.
- **Path:** `jobId`(uuid/필수/불가). **Query·Body:** 없음. **Header:** `Authorization` 필수.
- **성공:** `200 OK`; `Job`에 `documentId`(uuid/필수/허용), `versionId`(uuid/필수/허용)를 더하며, 두 값은 성공/부분 성공 때 모두 필수다.

```json
{ "data": { "id": "018f8f5a-7e55-7b7b-9c7d-1a2b3c4d5e6f", "type": "DOCUMENT_GENERATION", "status": "SUCCEEDED", "progress": 100, "attempt": 1, "maxAttempts": 3, "createdAt": "2026-08-19T09:00:00Z", "startedAt": "2026-08-19T09:00:01Z", "completedAt": "2026-08-19T09:00:05Z", "retryable": false, "nextRetryAt": null, "errorCode": null, "missingSources": [], "documentId": "018f8f5a-7e55-7b7b-9c7d-1a2b3c4d5e6f", "versionId": "018f8f5a-7e55-7b7b-9c7d-1a2b3c4d5e6f" } }
```

- **실패:** 없음 `404 JOB_NOT_FOUND`, 타 사용자 `403 FORBIDDEN`.

```json
{ "error": { "code": "JOB_NOT_FOUND", "message": "생성 작업을 찾을 수 없습니다.", "traceId": "018f8f5a-7e55-7b7b-9c7d-1a2b3c4d5e6f" } }
```

#### M03. 문서 목록 — 유형별 문서·버전 메타데이터 조회

- **Method / URL:** `GET /api/v1/documents`
- **인증·권한:** Bearer USER, 본인 문서만.
- **Path·Body:** 없음. **Query:** `type`(enum/선택/불가/`RESUME`,`PORTFOLIO`), `page`(integer/선택/불가/기본 0, 0 이상), `size`(integer/선택/불가/기본 20, 1~100), `sort`(string/선택/불가/기본 `latestVersionCreatedAt,DESC`; `latestVersionCreatedAt,ASC|DESC`만). 검색어는 문서 제목 정책이 없어 **미지원**이며, `q`는 `400 INVALID_PARAMETER`. **Header:** `Authorization` 필수.
- **성공:** `200 OK`; `items`(DocumentSummary 배열/필수/불가), `page`(PageMeta/필수/불가). `DocumentSummary`: `id`(uuid), `type`(`RESUME|PORTFOLIO`), `title`(string 1~200), `latestVersion`(DocumentVersion/허용), `versionCount`(integer 0 이상), `createdAt`(date-time), `updatedAt`(date-time)는 모두 필수이다.

```json
{ "data": { "items": [{ "id": "018f8f5a-7e55-7b7b-9c7d-1a2b3c4d5e6f", "type": "RESUME", "title": "김블로 이력서", "latestVersion": { "id": "018f8f5a-7e55-7b7b-9c7d-1a2b3c4d5e6f", "version": 2, "createdAt": "2026-08-19T09:00:00Z", "source": "AI_GENERATED" }, "versionCount": 2, "createdAt": "2026-08-18T09:00:00Z", "updatedAt": "2026-08-19T09:00:00Z" }], "page": { "page": 0, "size": 20, "totalElements": 1, "totalPages": 1, "sort": "latestVersionCreatedAt,DESC" } } }
```

- **실패:** 잘못된 필터·정렬 `400 INVALID_PARAMETER`, 인증 실패 `401 UNAUTHENTICATED`.

```json
{ "error": { "code": "INVALID_PARAMETER", "message": "지원하지 않는 정렬 조건입니다.", "traceId": "018f8f5a-7e55-7b7b-9c7d-1a2b3c4d5e6f", "fieldErrors": [{ "field": "sort", "reason": "latestVersionCreatedAt,ASC 또는 DESC만 사용할 수 있습니다." }] } }
```

#### M04. 최신 문서 조회 — 최신 Markdown 본문 조회

- **Method / URL:** `GET /api/v1/documents/{documentId}`
- **인증·권한:** Bearer USER, 소유자만.
- **Path:** `documentId`(uuid/필수/불가). **Query·Body:** 없음. **Header:** `Authorization` 필수.
- **성공:** `200 OK`; `id`(uuid), `type`(enum), `title`(string 1~200), `version`(integer 1 이상), `markdown`(string 1~100,000), `createdAt`(date-time), `source`(literal `AI_GENERATED`)는 필수·null 불가.

```json
{ "data": { "id": "018f8f5a-7e55-7b7b-9c7d-1a2b3c4d5e6f", "type": "RESUME", "title": "김블로 이력서", "version": 2, "markdown": "# 김블로\n\n## 프로젝트 경험", "createdAt": "2026-08-19T09:00:00Z", "source": "AI_GENERATED" } }
```

- **실패:** 문서 없음 `404 DOCUMENT_NOT_FOUND`, 타 사용자 `403 FORBIDDEN`, 인증 실패 `401 UNAUTHENTICATED`.

```json
{ "error": { "code": "DOCUMENT_NOT_FOUND", "message": "문서를 찾을 수 없습니다.", "traceId": "018f8f5a-7e55-7b7b-9c7d-1a2b3c4d5e6f" } }
```

#### M05. 버전 목록 — 특정 문서의 버전 메타데이터 목록

- **Method / URL:** `GET /api/v1/documents/{documentId}/versions`
- **인증·권한:** Bearer USER, 소유자만.
- **Path:** `documentId`(uuid/필수/불가). **Query:** `page`(integer/선택/불가/기본 0, 0 이상), `size`(integer/선택/불가/기본 20, 1~100), `sort`(string/선택/불가/기본 `version,DESC`; `version,ASC|DESC`만). 검색·필터는 없음. **Header:** `Authorization` 필수. **Body:** 없음.
- **성공:** `200 OK`; `items`(DocumentVersion 배열/필수/불가), `page`(PageMeta/필수/불가).

```json
{ "data": { "items": [{ "id": "018f8f5a-7e55-7b7b-9c7d-1a2b3c4d5e6f", "version": 2, "createdAt": "2026-08-19T09:00:00Z", "source": "AI_GENERATED" }], "page": { "page": 0, "size": 20, "totalElements": 2, "totalPages": 1, "sort": "version,DESC" } } }
```

- **실패:** 문서 없음 `404 DOCUMENT_NOT_FOUND`, 정렬 오류 `400 INVALID_PARAMETER`, 타 사용자 `403 FORBIDDEN`.

```json
{ "error": { "code": "DOCUMENT_NOT_FOUND", "message": "문서를 찾을 수 없습니다.", "traceId": "018f8f5a-7e55-7b7b-9c7d-1a2b3c4d5e6f" } }
```

#### M06. 특정 버전 조회 — 선택 버전의 Markdown 본문 조회

- **Method / URL:** `GET /api/v1/documents/{documentId}/versions/{versionId}`
- **인증·권한:** Bearer USER, 소유자만.
- **Path:** `documentId`(uuid/필수/불가), `versionId`(uuid/필수/불가). **Query·Body:** 없음. **Header:** `Authorization` 필수.
- **성공:** `200 OK`; M04 성공 응답과 동일한 필드. `version`은 요청 `versionId`의 번호다.

```json
{ "data": { "id": "018f8f5a-7e55-7b7b-9c7d-1a2b3c4d5e6f", "type": "RESUME", "title": "김블로 이력서", "version": 1, "markdown": "# 김블로\n\n## 프로젝트 경험", "createdAt": "2026-08-18T09:00:00Z", "source": "AI_GENERATED" } }
```

- **실패:** 문서 없음 `404 DOCUMENT_NOT_FOUND`, 버전 없음 `404 VERSION_NOT_FOUND`, 타 사용자 `403 FORBIDDEN`.

```json
{ "error": { "code": "VERSION_NOT_FOUND", "message": "문서 버전을 찾을 수 없습니다.", "traceId": "018f8f5a-7e55-7b7b-9c7d-1a2b3c4d5e6f" } }
```

#### M07. 버전 PDF 다운로드 — 선택한 읽기 전용 문서를 즉시 PDF로 반환

- **Method / URL:** `GET /api/v1/documents/{documentId}/versions/{versionId}/pdf`
- **인증·권한:** Bearer USER, 소유자만.
- **Path:** `documentId`(uuid/필수/불가), `versionId`(uuid/필수/불가). **Query·Body:** 없음. **Header:** `Authorization` 필수.
- **성공:** `200 OK`; JSON이 아닌 `application/pdf` binary body를 반환한다. `Content-Disposition`(string/필수/불가/`attachment; filename="{type}-v{version}.pdf"`), `Content-Type`(literal/필수/불가/`application/pdf`) 헤더를 포함한다. PDF는 요청 시 선택 버전 Markdown으로 생성하며 파일·생성 Job을 별도로 저장하지 않는다.
- **성공 응답 예시:** HTTP body는 PDF binary이므로 JSON 예시가 없다.
- **실패:** 문서 없음 `404 DOCUMENT_NOT_FOUND`, 버전 없음 `404 VERSION_NOT_FOUND`, 타 사용자 `403 FORBIDDEN`, PDF 변환 실패 `500 INTERNAL_ERROR`.

```json
{ "error": { "code": "VERSION_NOT_FOUND", "message": "문서 버전을 찾을 수 없습니다.", "traceId": "018f8f5a-7e55-7b7b-9c7d-1a2b3c4d5e6f" } }
```

## 5. 에러 코드표

| HTTP | 코드 | 발생 상황 |
| --- | --- | --- |
| 400 | `INVALID_PARAMETER` | Path·Query·Header·Body 형식, 허용값, 범위 오류 |
| 400 | `UNSUPPORTED_PROVIDER` | Notion·GitHub 외 provider |
| 400 | `UNSUPPORTED_DOCUMENT_TYPE` | RESUME·PORTFOLIO 외 type |
| 400 | `OAUTH_STATE_INVALID` | OAuth state 누락·만료·위조 |
| 401 | `UNAUTHENTICATED` | Bearer 토큰 누락·만료·위조 |
| 401 | `INVALID_CREDENTIALS` | 로그인 실패 |
| 403 | `FORBIDDEN` | 타 사용자 소유 리소스 접근 |
| 404 | `TODO_NOT_FOUND` | 날짜 내 Todo 없음 |
| 404 | `REFLECTION_NOT_FOUND` | Draft 없음 |
| 404 | `DOCUMENT_NOT_FOUND` | 문서 없음 |
| 404 | `VERSION_NOT_FOUND` | 문서 버전 없음 |
| 404 | `JOB_NOT_FOUND` | 작업 없음 또는 다른 작업 유형 조회 |
| 409 | `LOGIN_ID_ALREADY_EXISTS` | 가입 loginId 중복 |
| 409 | `EMAIL_ALREADY_EXISTS` | 가입 email 중복 |
| 409 | `INTEGRATION_ALREADY_CONNECTED` | 이미 연결된 제공자 인가 시작 |
| 409 | `JOB_ALREADY_RUNNING` | 같은 대상의 QUEUED/RUNNING 작업 존재 |
| 409 | `IDEMPOTENCY_KEY_REUSED` | 같은 키에 다른 요청 본문 |
| 409 | `REFLECTION_ALREADY_PUBLISHED` | PUBLISHED 회고 생성·수정·발행 |
| 422 | `DATA_INSUFFICIENT` | 생성·발행에 필요한 연동/근거 부족 |
| 502 | `EXTERNAL_SOURCE_FAILED` | Notion·GitHub·OAuth 외부 제공자 호출 실패 |
| 502 | `AI_PIPELINE_FAILED` | 내부 AI 파이프라인 최종 실패 |
| 500 | `INTERNAL_ERROR` | 처리하지 못한 서버 오류 |

## 6. 확정된 MVP 제외 범위

| 항목 | 정책 |
| --- | --- |
| Discord·공지 연동 | Notion·GitHub·Blocki Todo 외 데이터 소스는 MVP에서 제공하지 않는다. |
| 포트폴리오 카드 반영·문서 본문 편집 | 이력서·포트폴리오는 Markdown 읽기 전용이며, 새 생성만 새 버전을 추가한다. |
| 연동 해제·계정 정보 수정·비밀번호 변경 | MVP API와 설정 화면에서 제공하지 않는다. |
| 서버 로그아웃·Refresh token | 프론트가 access token을 삭제해 로그아웃하며, 서버 차단 목록과 Refresh token은 제공하지 않는다. |

## 7. 자체 검수

| 완료 기준 | 결과 | 근거 |
| --- | --- | --- |
| 모든 사용자 기능이 API와 매핑됨 | 통과 | MVP 핵심 기능과 선택 버전 PDF 다운로드를 1.3에 매핑했고, 제외 기능은 6장에 명시했다. |
| 모든 API에 요청·응답·에러 예시 존재 | 통과 | 4장의 23개 API에 요청(없는 경우 없음 명시), 성공·실패 JSON 또는 binary body 규칙을 포함했다. |
| 모든 필드의 타입·필수 여부·제약조건 정의 | 통과 | 2.3 공통 객체와 각 API의 입력·응답 필드에 정의했다. |
| 인증·권한·페이지네이션·비동기 상태 정의 | 통과 | 2.1, 2.4 및 각 목록·작업 API에 정의했다. |
| 프론트엔드와 백엔드가 추가 질문 없이 DTO와 테스트 작성 가능 | 통과 | 인증·OAuth·재시도·멱등성·날짜·Markdown·PDF의 구현 정책을 확정했다. |
| API 간 필드명·상태 코드·에러 형식 일관 | 통과 | 공통 응답·오류, UUID, UTC timestamp, Job·PageMeta를 단일 정의로 사용한다. |

모든 완료 기준을 통과했다.

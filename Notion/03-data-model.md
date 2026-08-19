# 03. Spring 데이터 모델

원본 proposal 테이블은 Spring 공통(GitHub 파트와 공유)이다.
이 문서는 **Notion 모듈이 추가하는 테이블만** 정의한다.
proposal 원본 스키마는 `Blocki-AI/DESIGN.md` §7.5를 따른다.

## 0. 기존 테이블과의 관계

Spring이 이미 갖고 있어야 하는 것 (Notion이 만들지 않음):

```
users
proposals
  proposal_id PK
  user_id
  job_id
  kind                  -- progress | portfolio | resume | readme
  status
  artifact_markdown
  template_ref jsonb
  snapshot_digest
  proposal_digest
  action_json
  action_digest
  created_at
  expires_at
```

Notion은 `proposals.proposal_id`를 FK로만 본다. markdown을 복사해 두지 않는다. outbox 워커가 읽을 때 join 한다.

## 1. ERD

```text
users 1──1 notion_connections
users 1──1 notion_workspaces
users 1──* notion_sync_outbox
proposals 1──0..1 notion_sync_outbox     (proposal_id UNIQUE)
notion_sync_outbox 1──* notion_sync_attempts
```

유저당 연결 1개, 워크스페이스 맵 1개.
한 proposal은 Notion에 최대 한 번 성공한다.

## 2. `notion_connections`

```sql
CREATE TABLE notion_connections (
  user_id              UUID PRIMARY KEY REFERENCES users(id),
  access_token_enc     BYTEA NOT NULL,
  refresh_token_enc    BYTEA NOT NULL,
  token_type           TEXT NOT NULL DEFAULT 'bearer',
  access_expires_at    TIMESTAMPTZ,
  bot_id               TEXT NOT NULL,
  workspace_id         TEXT NOT NULL,
  workspace_name       TEXT,
  workspace_icon       TEXT,
  notion_user_id       TEXT,
  duplicated_template_id TEXT,
  needs_reauth         BOOLEAN NOT NULL DEFAULT FALSE,
  revoked_at           TIMESTAMPTZ,
  created_at           TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at           TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

| 컬럼 | 설명 |
| --- | --- |
| `access_token_enc` | GitHub PAT와 같은 AES/KMS. 평문 컬럼 금지 |
| `needs_reauth` | refresh `invalid_grant` 후 true. 프론트 CTA |
| `revoked_at` | 우리 쪽 disconnect. 행을 지워도 되지만 감사 로그를 남기려면 이 컬럼 |

재연결: 같은 `user_id`에 UPSERT. workspace_id가 바뀌면 F2를 처음부터.

## 3. `notion_workspaces`

```sql
CREATE TABLE notion_workspaces (
  user_id                    UUID PRIMARY KEY REFERENCES users(id),
  workspace_id               TEXT NOT NULL,
  root_page_id               TEXT NOT NULL,
  progress_db_id             TEXT NOT NULL,
  progress_data_source_id    TEXT NOT NULL,
  portfolio_db_id            TEXT NOT NULL,
  portfolio_data_source_id   TEXT NOT NULL,
  resume_db_id               TEXT NOT NULL,
  resume_data_source_id      TEXT NOT NULL,
  schema_version             TEXT NOT NULL DEFAULT 'v1',
  created_at                 TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at                 TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

API 2026-03-11에서 페이지 parent는 `data_source_id`다.
`*_db_id`는 사람이 링크 만들 때, `*_data_source_id`는 POST /v1/pages 때 쓴다.

루트 URL:

```
https://www.notion.so/{root_page_id without dashes}
```

status API는 이 URL을 내려준다.

## 4. `notion_sync_outbox`

```sql
CREATE TABLE notion_sync_outbox (
  id              BIGSERIAL PRIMARY KEY,
  proposal_id     UUID NOT NULL UNIQUE REFERENCES proposals(proposal_id),
  user_id         UUID NOT NULL REFERENCES users(id),
  kind            TEXT NOT NULL CHECK (kind IN ('progress','portfolio','resume')),
  status          TEXT NOT NULL CHECK (status IN (
                    'pending','running','succeeded','failed','skipped'
                  )),
  skip_reason     TEXT,
  page_id         TEXT,
  page_url        TEXT,
  error_code      TEXT,
  error_message   TEXT,
  retryable       BOOLEAN NOT NULL DEFAULT FALSE,
  attempt_count   INT NOT NULL DEFAULT 0,
  next_attempt_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX notion_outbox_due_idx
  ON notion_sync_outbox (status, next_attempt_at)
  WHERE status IN ('pending','failed');
```

상태 기계:

```
          insert
            │
            ▼
        pending ──────────────────────────────► skipped
            │                                   (not_connected, empty, disconnected)
            │ CAS status=running
            ▼
         running
            │
     ┌──────┼──────────────┐
     ▼      ▼              ▼
 succeeded  failed      skipped
            │
            │ retryable && attempt_count < 8
            ▼
         pending  (next_attempt_at = now + backoff)
```

CAS (이중 미러 방지):

```sql
UPDATE notion_sync_outbox
   SET status = 'running',
       updated_at = now(),
       attempt_count = attempt_count + 1
 WHERE proposal_id = :id
   AND status IN ('pending','failed')
   AND next_attempt_at <= now()
 RETURNING *;
```

0행이면 다른 워커가 가져갔거나 이미 성공. 그때 Notion POST 하지 않음.

백오프 (attempt_count는 CAS 이후 값):

| attempt | next_attempt_at |
| --- | --- |
| 1 | +30s |
| 2 | +2m |
| 3 | +10m |
| 4 | +30m |
| 5–8 | +2h |
| >8 | 멈춤. retryable은 그대로 true여도 워커가 안 집음. 수동 `sync-now`만 |

`failed && retryable=false` (`needs_reauth`, `validation`, `not_mirrorable`): 재시도 없음.

## 5. `notion_sync_attempts` (선택, 권장)

해커톤에서 디버깅이 급하면 생략 가능. 있으면 데모 장애 때 산다.

```sql
CREATE TABLE notion_sync_attempts (
  id           BIGSERIAL PRIMARY KEY,
  outbox_id    BIGINT NOT NULL REFERENCES notion_sync_outbox(id),
  attempted_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  http_status  INT,
  notion_code  TEXT,
  result       TEXT NOT NULL,  -- created|duplicate|skipped|failed
  detail       TEXT            -- 토큰 없이 메시지. Authorization 헤더 저장 금지
);
```

## 6. Enqueue 조건 (proposal 저장 트랜잭션 안)

```
if proposal.status in ('proposed','partial')
   and proposal.kind in ('progress','portfolio','resume')
   and proposal.body_markdown is not null
   and length(trim(proposal.body_markdown)) > 0
   and exists (notion_connections where user_id=... and revoked_at is null and needs_reauth=false)
then
   insert into notion_sync_outbox (proposal_id, user_id, kind, status)
   values (..., 'pending')
   on conflict (proposal_id) do nothing
```

`no_change`, `blocked`, `failed` 는 insert 자체를 하지 않는다.
연결이 없으면 insert하지 않는다. 나중에 연결해도 **과거 proposal을 자동 소급하지 않는다.**
소급이 필요하면 `POST /api/integrations/notion/backfill?limit=20` 을 2차에 둔다.

트랜잭션: proposal INSERT와 outbox INSERT는 **같은 커밋**. 커밋 후 워커가 본다.

## 7. 워커

Spring `@Scheduled` 5초 또는 proposal 커밋 후 `ApplicationEvent`.

둘 다 해도 된다. 스케줄은 안전망, 이벤트는 지연 축소.

```
every 5s:
  rows = SELECT ... WHERE status IN ('pending','failed')
                     AND retryable IS NOT FALSE
                     AND next_attempt_at <= now()
                     AND attempt_count < 8
          ORDER BY created_at
          LIMIT 10
  for row in rows:
    CAS running
    if 0: continue
    result = F3.mirror(...)
    persist result
```

한 틱에 10개. Notion 3 rps를 한 유저가 혼자 다 쓰지 않게, **유저당 동시 1개**.

```
in-flight set per user_id. 이미 running 이면 그 유저 다음 행은 이번 틱 skip
```

## 8. JPA 스케치

이름은 팀 컨벤션에 맞춘다. 필드만 지키면 된다.

```java
@Entity
@Table(name = "notion_connections")
class NotionConnection {
  @Id UUID userId;
  byte[] accessTokenEnc;
  byte[] refreshTokenEnc;
  Instant accessExpiresAt;
  String botId;
  String workspaceId;
  String workspaceName;
  boolean needsReauth;
  Instant revokedAt;
}

@Entity
@Table(name = "notion_workspaces")
class NotionWorkspace {
  @Id UUID userId;
  String workspaceId;
  String rootPageId;
  String progressDataSourceId;
  String portfolioDataSourceId;
  String resumeDataSourceId;
  String schemaVersion;
}

@Entity
@Table(name = "notion_sync_outbox")
class NotionSyncOutbox {
  @Id Long id;
  UUID proposalId;
  UUID userId;
  String kind;
  String status;
  String skipReason;
  String pageId;
  String pageUrl;
  String errorCode;
  boolean retryable;
  int attemptCount;
  Instant nextAttemptAt;
}
```

토큰 getter가 `toString()`/JSON 직렬화에 노출되지 않게 `@JsonIgnore` + 로그 라이브러리 마스킹.

## 9. 마이그레이션 파일 이름

```
V{next}__create_notion_tables.sql
```

Flyway/Liquibase 중 Spring이 쓰는 것을 따른다. 새 도구를 들이지 않는다.

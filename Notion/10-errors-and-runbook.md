# 10. 에러 코드와 운영

## 1. `NotionError.code`

| code | HTTP (공개 API) | retryable | 유저 메시지 | 서버 동작 |
| --- | --- | --- | --- | --- |
| `oauth_denied` | 302 `denied` | false | 연결이 취소되었습니다 | 기존 토큰 유지 |
| `oauth_state` | 400 | false | 연결을 다시 시작해 주세요 | 교환 안 함 |
| `missing_code` | 400 | false | 연결을 다시 시작해 주세요 | |
| `token_exchange` | 502 | true | 잠시 후 다시 연결 | |
| `not_connected` | 409 | false | Notion을 먼저 연결해 주세요 | |
| `needs_reauth` | 409 | false | Notion 권한이 만료되었습니다 | 토큰 삭제 |
| `notion_auth` | 502 | false | 다시 연결해 주세요 | 401이 refresh 실패로 끝난 경우 |
| `notion_forbidden` | 403 | false | Blocki가 쓸 페이지를 선택해 다시 연결해 주세요 | |
| `workspace_missing` | 409 | false | 위와 같음 | 맵 무효 |
| `rate_limited` | 429 | true | (UI에 잘 안 띄움) | Retry-After |
| `payload_too_large` | 400 | false | 문서가 너무 큽니다 | 자르지 않음 |
| `validation` | 400 | false | 로그 확인 | 속성 키 오타 등 |
| `internal` | 500 | true | 잠시 후 | |

공개 API는 토큰·Notion 원문 에러를 `message`에 넣지 않는다. 서버 로그만.

```
log.warn("notion error code={} notion_code={} status={} proposal={}", ...);
```

## 2. Notion HTTP → 우리 코드

| Notion | 우리 |
| --- | --- |
| 400 `validation_error` | `validation` 또는 `payload_too_large` (message에 size/limit) |
| 401 `unauthorized` | refresh 시도. 실패 시 `needs_reauth` |
| 403 `restricted_resource` | `notion_forbidden` |
| 404 `object_not_found` | provision 재시도 1회 후 `workspace_missing` |
| 409 | 드묾. `internal` retryable |
| 429 `rate_limited` | `rate_limited` |
| 500/502/503/504 | `internal` retryable. GET만 자동 재시도, POST는 outbox 멱등으로 재시도 |
| 529 `service_overload` | 429와 동일. `Retry-After` |

`Retry-After` 가 초 정수. 없으 면 `min(2^attempt, 30)` + jitter 0–250ms. 최대 5회(클라이언트) + outbox 8회.

## 3. RestClient 재시도 (복붙)

`NotionRestClient` 한곳에만 둔다. F1/F2/F3가 각자 재시도하면 폭풍이 난다.

```java
HttpResponse<String> exchange(HttpRequest req) throws Exception {
  boolean idempotent = req.method().equals("GET") || req.method().equals("DELETE");
  for (int attempt = 0; attempt < 6; attempt++) {
    HttpResponse<String> res = http.send(req, HttpResponse.BodyHandlers.ofString());
    int s = res.statusCode();
    boolean retry = s == 429 || s == 529
        || (idempotent && Set.of(500, 502, 503, 504).contains(s));
    if (!retry || attempt == 5) return res;
    long retryAfter = res.headers().firstValue("Retry-After")
        .map(Long::parseLong)
        .orElse(Math.min(1L << attempt, 30L));
    Thread.sleep(retryAfter * 1000 + ThreadLocalRandom.current().nextLong(250));
  }
  throw new IllegalStateException("unreachable");
}
```

POST /pages 는 idempotent가 아니다. 429만 여기서 재시도하고, 500은 outbox가 다시 집는다. query(duplicate)가 두 번째 POST를 막는다.

## 4. 운영 증상표

| 보이는 것 | 볼 곳 | 조치 |
| --- | --- | --- |
| 연결 직후 pick_page | 유저가 페이지 0개 | 다시 연결, 페이지 선택 |
| 미리보기는 있는데 Notion 빈 DB | outbox `pending` 적체 | 워커 로그, `next_attempt_at` |
| outbox `failed` rate_limited | 3 rps | 유저당 동시 1 확인 |
| 전원 needs_reauth | client_secret 교체, 재등록 | 유저에게 재연결 공지. DCR이 아님(REST) |
| 페이지가 매일 복제 | no_change 필터 실패 또는 Proposal ID query 실패 | query 엔드포인트/속성 키 |
| 루트 페이지 여러 개 | F2 멱등 실패 | id 저장 여부 |
| 데모 중 401 행렬 | refresh 레이스 | 락 유무 |
| GitHub 잡 실패로 보임 | 사실 Notion 예외가 트랜잭션을 롤백 | enqueue를 커밋 **후** 이벤트로. proposal INSERT와 Notion HTTP를 한 트랜잭션에 넣지 말 것 |

마지막 줄이 가장 중요하다.

```
@Transactional
save proposal
insert outbox
commit
then event → worker   // HTTP는 트랜잭션 밖
```

워커가 트랜잭션 안에서 Notion을 치면 3 rps + DB 락이 묶인다.

## 5. 재연결 플레이북

1. 유저: 설정 → 다시 연결 (기존 disconnect를 강제하지 않아도 됨. UPSERT)
2. 운영: `needs_reauth=true` 인 유저 수

```sql
SELECT count(*) FROM notion_connections
 WHERE needs_reauth AND revoked_at IS NULL;
```

3. 대량 만료(180일 MCP가 아니라 REST refresh 정책)면 공지 배너
4. 토큰 유출 의심: portal에서 connection secret rotate + 전원 재연결. 로그에 토큰이 찍혔으면 그 커밋을 되돌리고 시크릿 교체

## 6. 데이터 삭제 요청

유저가 계정 삭제:

1. `notion_connections` 삭제
2. `notion_workspaces` 삭제
3. outbox 삭제
4. **Notion 페이지는 지우지 않음** (다른 제품도 보통 이렇게). 정책 문구:

> 서비스에서 연결을 끊거나 계정을 삭제해도, 이미 Notion에 만들어진 페이지는 남아 있습니다. Notion에서 직접 삭제해 주세요.

## 7. 로컬에서 토큰 빼기

실수로 로그에 남겼을 때:

```bash
# 히스토리에서 찾기
rg -n "ntn_|nrt_" --hidden -g '!.git' || true
```

git에 들어갔으면 그 커밋을 공개 레포에 푸시하지 말 것. secret rotate.

## 8. 대시보드 SQL

```sql
-- 오늘 미러
SELECT kind, status, count(*)
  FROM notion_sync_outbox
 WHERE created_at >= date_trunc('day', now())
 GROUP BY 1, 2
 ORDER BY 1, 2;

-- 죽은 running
SELECT * FROM notion_sync_outbox
 WHERE status = 'running'
   AND updated_at < now() - interval '10 minutes';
```

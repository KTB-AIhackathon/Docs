# 02. Notion Public OAuth — 따라 하기

멀티유저 서비스이므로 **Public connection**만 쓴다.
Internal connection 토큰 하나로는 유저 워크스페이스에 들어가지 못한다.
개발 중 본인 워크스페이스 실험만 Internal/PAT를 쓴다 (`08-implementation-playbook.md` Day 0).

공식 문서: [Authorization](https://developers.notion.com/guides/get-started/authorization), [Public connections](https://developers.notion.com/guides/get-started/public-connections).

## 0. Developer portal에서 한 번 만들기

1. https://app.notion.com/developers/connections 접속 (워크스페이스 소유자)
2. **Public connections** → **Create new connection**
3. 채울 값:

| 필드 | 값 |
| --- | --- |
| 이름 | `Blocki` (또는 해커톤 팀 이름) |
| Development workspace | 팀 개발용 워크스페이스 |
| Redirect URI (로컬) | `http://localhost:8080/api/integrations/notion/callback` |
| Redirect URI (EC2) | `https://{도메인}/api/integrations/notion/callback` |
| Installation scope | **Any workspace** (해커톤 심사위원 워크스페이스가 우리 목록에 없다) |
| Capabilities | Read content, Update content, Insert content |
| User capabilities | 없어도 된다. 1차는 유저 이메일이 필요 없다 |

4. **Configuration** 탭에서 복사:
   - OAuth client ID
   - OAuth client secret
5. 둘 다 Spring 시크릿. 레포에 커밋 금지.

선택: **Notion URL for optional template**
- 공개 페이지 하나를 “Blocki 시작 페이지”로 만들어 두면, 유저가 OAuth 때 복제할 수 있다.
- 복제되면 token 응답의 `duplicated_template_id`가 루트가 된다.
- 해커톤 시간 없으면 비워 둔다. F2가 빈 페이지를 만든다.

## 1. 환경 변수

```yaml
# application.yml
notion:
  api-base: https://api.notion.com
  api-version: "2026-03-11"
  oauth:
    client-id: ${NOTION_CLIENT_ID}
    client-secret: ${NOTION_CLIENT_SECRET}
    redirect-uri: ${NOTION_REDIRECT_URI}
    authorize-url: https://api.notion.com/v1/oauth/authorize
    token-url: https://api.notion.com/v1/oauth/token
  state-ttl: 10m
```

로컬 `.env` 예:

```
NOTION_CLIENT_ID=463558a3-...
NOTION_CLIENT_SECRET=secret_...
NOTION_REDIRECT_URI=http://localhost:8080/api/integrations/notion/callback
```

`redirect_uri`는 portal에 등록한 문자열과 **한 글자도 같아야** 한다. trailing slash 포함.

## 2. 연결 시작

프론트: 로그인된 유저가 “Notion 연결” 클릭
→ `GET /api/integrations/notion/connect` (JWT 쿠키/헤더)

서버:

1. `user_id` = SecurityContext
2. `nonce` = 32 bytes hex
3. `state` = HMAC 또는 signed JWT `{user_id, nonce, exp, return_to}`
4. `return_to`는 allowlist (`/settings`, `/dashboard` 만)
5. Redis 또는 DB `notion_oauth_state`에 `nonce` 저장. TTL 10분
6. 302:

```
https://api.notion.com/v1/oauth/authorize
  ?owner=user
  &client_id={CLIENT_ID}
  &redirect_uri={URL_ENCODED_REDIRECT}
  &response_type=code
  &state={state}
```

필수 쿼리:

| 파라미터 | 값 |
| --- | --- |
| `client_id` | portal |
| `redirect_uri` | portal과 동일 |
| `response_type` | `code` |
| `owner` | `user` |
| `state` | 위에서 만든 값 |

유저 화면:

1. capability 설명
2. **Select pages** → 페이지 피커
3. **Allow access** 또는 **Cancel**

안내 문구를 프론트에 미리 띄운다.

> Blocki가 로그를 남길 페이지를 골라 주세요. 새로 만들 “Blocki” 페이지를 고르거나, 빈 페이지 하나를 고르면 됩니다. 고르지 않은 페이지에는 접근하지 않습니다.

## 3. 콜백

Notion이 보낸다.

성공:

```
GET {redirect_uri}?code={임시코드}&state={우리가보낸state}
```

취소:

```
GET {redirect_uri}?error=access_denied&state=
```

서버 처리 순서. **이 순서를 바꾸지 말 것.**

```
1. error 쿼리가 있으면
     access_denied → 프론트 /settings?notion=denied
     그 외 → /settings?notion=error
     기존 토큰을 지우지 않음
2. state 서명 검증. 실패 → 400 oauth_state
3. state.user_id 가 현재 세션 user_id 와 같아야 함
     콜백을 세션 없이 열면 state 안의 user_id 만 믿지 말 것.
     해커톤 현실: 콜백도 JWT 쿠키가 따라오게 같은 사이트 리다이렉트로 둔다.
4. nonce 가 Redis에 있으면 consume (한 번만). 없으면 400
5. POST token exchange (아래)
6. vault에 access/refresh 저장. 예전 행이 있으면 덮어씀 (재연결)
7. F2 provision
8. 302 {return_to}?notion=connected
```

### 3.1 token exchange

```http
POST /v1/oauth/token HTTP/1.1
Host: api.notion.com
Authorization: Basic {base64(CLIENT_ID:CLIENT_SECRET)}
Content-Type: application/json
Accept: application/json

{
  "grant_type": "authorization_code",
  "code": "{code}",
  "redirect_uri": "{NOTION_REDIRECT_URI}"
}
```

curl:

```bash
ENCODED=$(printf '%s:%s' "$NOTION_CLIENT_ID" "$NOTION_CLIENT_SECRET" | base64)

curl -sS https://api.notion.com/v1/oauth/token \
  -H "Authorization: Basic $ENCODED" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -d "$(jq -n --arg c "$CODE" --arg r "$NOTION_REDIRECT_URI" \
      '{grant_type:"authorization_code", code:$c, redirect_uri:$r}')"
```

성공 응답에서 **전부 저장**한다. 나중에 UI가 필요해져도 유저에게 OAuth를 다시 시킬 수 없다.

```json
{
  "access_token": "ntn_...",
  "refresh_token": "nrt_...",
  "token_type": "bearer",
  "bot_id": "...",
  "duplicated_template_id": null,
  "owner": { "type": "user", "user": { "id": "...", "name": "..." } },
  "workspace_icon": "https://...",
  "workspace_id": "...",
  "workspace_name": "Alice's Workspace"
}
```

`expires_in`이 있으면:

```
access_expires_at = now_utc + expires_in - 60s
```

없으면 `access_expires_at = null`. 호출 401일 때만 refresh.

### 3.2 저장 컬럼

`03-data-model.md`의 `notion_connections`에 대응.

암호화 대상:

- `access_token`
- `refresh_token`

암호화하지 않아도 되는 것 (비밀이 아님):

- `bot_id`, `workspace_id`, `workspace_name`, `workspace_icon`, `notion_user_id`, `duplicated_template_id`

## 4. Refresh

```http
POST /v1/oauth/token HTTP/1.1
Authorization: Basic {base64(CLIENT_ID:CLIENT_SECRET)}
Content-Type: application/json

{
  "grant_type": "refresh_token",
  "refresh_token": "{현재 refresh}"
}
```

규칙:

1. 유저당 락. 키 `notion:refresh:{user_id}`. 30초 TTL.
2. 락 안에서 읽기 → POST → `(access, refresh, expires_at)`를 **한 UPDATE**로 저장 → 락 해제.
3. 응답에 새 `refresh_token`이 오면 반드시 그걸 저장. 예전 것으로 다시 refresh하면 `invalid_grant`가 나고, 어떤 제공자는 그랜트 전체를 죽인다.
4. `invalid_grant` → 토큰 삭제, `needs_reauth=true`. **재시도 금지.**
5. 네트워크/`temporarily_unavailable` → 백오프 후 재시도. 최대 3회.
6. 두 워커가 동시에 미러하면 둘 다 이 함수를 거친다. 두 번째 워커는 락을 기다렸다가 **DB에서 새 access를 다시 읽는다.** 자기 메모리의 헌 토큰으로 호출하지 않는다.

의사코드:

```
function accessFor(userId):
  row = vault.read(userId)
  if row is null: raise not_connected
  if row.access_expires_at != null and row.access_expires_at > now+30s:
    return decrypt(row.access_token)
  lock(userId):
    row = vault.read(userId)          # 다시
    if still fresh: return decrypt(access)
    resp = POST refresh
    if invalid_grant:
      vault.delete(userId)
      raise needs_reauth
    vault.update(access, refresh, expires)  # 원자
    return new access
```

## 5. 연결 해제

`POST /api/integrations/notion/disconnect` (JWT)

1. `notion_connections` 삭제 (또는 `revoked_at` 세팅)
2. `notion_workspaces` 는 남긴다. 같은 workspace_id로 재연결하면 F2가 기존 루트를 재사용
3. 진행 중인 outbox `pending`/`running` 은 `skipped/disconnected`
4. Notion 쪽 페이지·DB는 API로 삭제하지 않음
5. 응답 `connected: false`

Notion 앱에서 유저가 연결을 직접 끊으면 다음 API 호출이 401/`unauthorized`가 된다. 그때 refresh가 `invalid_grant` → 우리 쪽도 `needs_reauth`.

## 6. 상태 조회

`GET /api/integrations/notion/status` (JWT)

토큰을 절대 넣지 않는다. 예시는 `06-http-api.md`.

`connected` 판정:

```
row exists
AND revoked_at is null
AND needs_reauth is false
AND access 또는 refresh 가 있다
```

워크스페이스 이름은 저장값을 보여 준다. 매번 Notion을 치지 않는다.
“지금 살아 있나”가 필요하면 `GET /v1/users/me` 를 status에 `?ping=true` 일 때만.

## 7. 페이지 피커가 비면

Public connection은 **유저가 고른 페이지 밖을 못 만든다.**
Internal처럼 “Add connections”로 나중에 공유할 수도 있지만, 유저 경험상 OAuth 때 고르는 게 맞다.

F2 전략:

1. `duplicated_template_id`가 있으면 그걸 루트로 쓴다 (이미 공유됨)
2. 없으면 `parent.workspace = true` 로 private 페이지 생성 시도
3. 403이면 `POST /v1/search` `{filter:{property:"object", value:"page"}, page_size:1}`
   - 결과의 첫 페이지를 parent로 `Blocki` 자식 페이지 생성
4. search도 비면 `workspace_missing`. 프론트 메시지:

> Notion에서 Blocki가 사용할 페이지를 하나 이상 선택해 다시 연결해 주세요.

## 8. 보안 체크리스트

- [ ] client_secret, access_token, refresh_token 이 git / 로그 / 프론트 응답에 없다
- [ ] 로그 필터가 `Authorization`, `ntn_`, `nrt_`, `secret_` 를 마스킹한다
- [ ] state TTL 10분, 1회용
- [ ] redirect_uri 고정
- [ ] token exchange는 서버만
- [ ] refresh 단일 비행
- [ ] `invalid_grant` 재시도 루프 없음
- [ ] disconnect 후에도 Notion 페이지 삭제 API 없음
- [ ] 유저 A의 토큰으로 유저 B provision/mirror 호출이 테스트에서 실패한다

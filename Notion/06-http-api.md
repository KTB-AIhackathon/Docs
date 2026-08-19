# 06. HTTP API (Spring 공개)

모두 JWT 필요. FastAPI `/internal/*` 가 아니다.
Gateway가 `/api/integrations/notion` 을 Spring에 붙인다.

에러 바디 공통:

```json
{
  "ok": false,
  "error": {
    "code": "oauth_state",
    "message": "invalid state",
    "retryable": false
  }
}
```

`code` 목록은 `10-errors-and-runbook.md`. 토큰 필드 없음.

---

## GET `/api/integrations/notion/status`

연결 상태. Notion을 기본으로 안 친다.

**200**

```json
{
  "ok": true,
  "connected": true,
  "needs_reauth": false,
  "workspace_name": "Alice's Workspace",
  "workspace_icon": "https://...",
  "root_page_url": "https://www.notion.so/…",
  "schema_version": "v1",
  "last_sync_at": "2026-08-19T12:03:11Z",
  "last_sync_status": "succeeded",
  "last_sync_page_url": "https://www.notion.so/…",
  "outbox_backlog": 0
}
```

미연결:

```json
{
  "ok": true,
  "connected": false,
  "needs_reauth": false,
  "workspace_name": null,
  "workspace_icon": null,
  "root_page_url": null,
  "schema_version": null,
  "last_sync_at": null,
  "last_sync_status": null,
  "last_sync_page_url": null,
  "outbox_backlog": 0
}
```

쿼리 `?ping=true` 이면 `GET /v1/users/me` 한 번. 실패 시 `connected`는 true로 두되 `needs_reauth`를 갱신할 수 있다.

프론트 배지:

| 조건 | UI |
| --- | --- |
| `connected=false` | 연결 버튼 |
| `connected=true && needs_reauth=false` | 초록 · workspace_name · 루트 링크 |
| `needs_reauth=true` | 다시 연결 |

---

## GET `/api/integrations/notion/connect`

302 to Notion authorize URL.

쿼리:

| 이름 | 필수 | 설명 |
| --- | --- | --- |
| `return_to` | 아니오 | `/settings` 또는 `/dashboard` 만 |

401: JWT 없음. 302 하지 않음.

프론트는 `window.location = "/api/integrations/notion/connect?return_to=/settings"` 로 충분하다. CORS preflight를 피하려면 같은 오리진 링크.

---

## GET `/api/integrations/notion/callback`

Notion이 브라우저를 돌려보내는 곳. JWT 쿠키가 따라오는 전제.

처리 후 302:

| 결과 | Location |
| --- | --- |
| 성공 | `{return_to}?notion=connected` |
| 유저 취소 | `{return_to}?notion=denied` |
| state/code 실패 | `{return_to}?notion=error` |
| provision 403 | `{return_to}?notion=pick_page` |

바디 없음. JSON을 여기서 주지 않는다.

프론트 settings는 `notion` 쿼리를 토스트로 바꾼다.

---

## POST `/api/integrations/notion/disconnect`

바디 없음.

**200**

```json
{
  "ok": true,
  "connected": false,
  "needs_reauth": false
}
```

이미 미연결이어도 200. 멱등.

---

## POST `/api/integrations/notion/sync/{proposalId}`

그 proposal을 지금 미러. 본인 것만.

**200** 동기 성공

```json
{
  "ok": true,
  "result": {
    "proposal_id": "2f1c0a6e-1111-2222-3333-444444444444",
    "status": "created",
    "page_id": "b55c9c91-384d-452b-81db-d1ef79372b75",
    "page_url": "https://www.notion.so/…",
    "skip_reason": null
  }
}
```

duplicate:

```json
{
  "ok": true,
  "result": {
    "proposal_id": "2f1c0a6e-1111-2222-3333-444444444444",
    "status": "duplicate",
    "page_id": "…",
    "page_url": "https://www.notion.so/…",
    "skip_reason": null
  }
}
```

**202** 시간 초과로 큐만

```json
{
  "ok": true,
  "queued": true,
  "proposal_id": "2f1c0a6e-1111-2222-3333-444444444444"
}
```

**404** proposal 없음 또는 남의 것 (존재 여부 숨김, 둘 다 404)

**409** `not_connected` / `needs_reauth`

```json
{
  "ok": false,
  "error": { "code": "needs_reauth", "message": "reconnect Notion", "retryable": false }
}
```

---

## GET `/api/integrations/notion/syncs`

최근 미러 20개. 대시보드.

```json
{
  "ok": true,
  "items": [
    {
      "proposal_id": "…",
      "kind": "progress",
      "status": "succeeded",
      "page_url": "https://…",
      "updated_at": "2026-08-19T12:03:11Z"
    }
  ]
}
```

마크다운 원문은 여기 안 준다. 원문은 기존 proposal 미리보기 API.

---

## Frontend 호출 스케치

```ts
type NotionStatus = {
  ok: true
  connected: boolean
  needs_reauth: boolean
  workspace_name: string | null
  root_page_url: string | null
  last_sync_status: "succeeded" | "failed" | "skipped" | "running" | null
}

export function startNotionConnect() {
  window.location.assign("/api/integrations/notion/connect?return_to=/settings")
}

export async function fetchNotionStatus(): Promise<NotionStatus> {
  const r = await fetch("/api/integrations/notion/status", { credentials: "include" })
  if (r.status === 401) throw new Error("login")
  return r.json()
}

export async function disconnectNotion() {
  await fetch("/api/integrations/notion/disconnect", {
    method: "POST",
    credentials: "include",
  })
}
```

토큰, bot_id, workspace_id를 화면에 뿌리지 않아도 된다. 이름은 필요.

## CORS / 쿠키

callback은 브라우저 탑 레벨 네비게이션이다. CORS 해당 없음.
status/disconnect는 프론트 오리진에서 호출. Spring이 이미 JWT 쿠키 SameSite를 정했을 것이다. 새 예외를 만들지 말 것.

로컬: 프론트 `localhost:5173`, API `localhost:8080` 이면 기존 프록시(`vite proxy /api`)를 탄다. connect 링크도 프록시를 타야 쿠키가 붙는다.

```
/api → http://localhost:8080
```

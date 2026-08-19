# 11. 호스트 Notion MCP를 크론 쓰기에 쓰지 않는 이유

`Blocki-AI/DESIGN.md` §7.6에는 이런 예시가 있다.

```yaml
mcp_servers:
  notion:
    url: "https://mcp.notion.com/mcp"
```

그리고 “호스트 Notion MCP는 OAuth만. 크론은 Spring이 access token을 갱신해야 한다.”

이 문장은 **의도는 맞다** (브라우저 OAuth + 서버가 토큰을 들고 크론).
다만 그 YAML의 호스트 MCP를 프로덕션 쓰기 클라이언트로 쓰면 **구현이 불가능에 가깝다.**
Notion 파트는 REST를 쓰고, 저 YAML은 로컬 디버그용으로만 남긴다.

## 1. 토큰이 두 종류다

| | Notion REST Public connection | 호스트 MCP (`mcp.notion.com`) |
| --- | --- | --- |
| 인가 화면 | `https://api.notion.com/v1/oauth/authorize` | MCP OAuth discovery + PKCE + DCR |
| 토큰 발급 | `POST /v1/oauth/token` (Basic client_id:secret) | MCP token_endpoint (RFC 8414) |
| audience | `api.notion.com` | MCP 리소스 |
| 사용처 | `Authorization: Bearer` + `Notion-Version` | MCP `initialize` / `tools/call` |
| 교차 사용 | MCP 토큰으로 `GET /v1/users/me` **거절** | REST access_token을 MCP에 넣으면 **보장 없음** (다른 audience) |
| access 수명 | 응답 `expires_in` (문서화, refresh 지원) | 현재 약 8시간. `expires_in`을 따라야 함 |
| refresh 수명 | refresh_token 회전 | 최대 180일(슬라이딩 아님) 또는 30일 유휴. 회전 + 재사용 시 그랜트 전체 폐기 |

공식 문서(`Build an MCP client for Notion`)가 분명히 말한다.

> public REST API의 `GET /v1/users/me` 는 MCP-audienced 토큰을 받지 않는다.

즉 **Spring이 들고 갱신하는 REST 토큰**과 **호스트 MCP 세션 토큰**은 같은 물건이 아니다.
“OAuth 한 번 하고 그 토큰으로 MCP를 돌린다”는 그림이 성립하지 않는다.

## 2. 호스트 MCP가 잘 하는 일 / 못 하는 일

잘 하는 일:

- Cursor, Claude Code, Codex처럼 **사람 앞에 브라우저가 있는** 클라이언트
- 한 개발자가 자기 워크스페이스를 탐색
- `notion-search`, `notion-fetch` 같은 대화형 도구

못 하는 일 (우리 제품):

- 멀티유저. 유저 A 크론이 유저 B 세션을 쓰면 안 된다
- 헤드리스 크론. DCR로 받은 `client_id`를 재등록하면 이전 그랜트가 고아가 된다
- FastAPI 무상태 워커. MCP 세션을 job마다 새로 열려면 유저별 refresh를 우리가 들고 있어야 하고, 그건 결국 REST OAuth를 한 번 더 만드는 일이다
- 결정적 쓰기. GitHub F2와 같이 **코드가 tool을 고정 호출**해야 한다. LLM에게 Notion tool을 열면 페이지를 엉뚱한 곳에 만든다

## 3. 오픈소스 `notion-mcp-server`도 1차 거절

- Notion이 “더 이상 적극 유지하지 않는다. 호스트 MCP를 써라”고 안내한다.
- Bearer(Internal/PAT)를 받으므로 크론에는 기술적으로 맞다.
- 그래도 우리가 서버를 띄우고 버전 `2025-09-03` data source 깨짐을 떠안는다.
- 해커톤에서 프로세스 + 이미지 + 장애 포인트가 하나 는다.
- REST가 같은 Bearer로 같은 일을 한다.

2차에 “에이전트가 Notion을 읽어서 답한다”가 진짜 생기기 전에는 도입하지 않는다.

## 4. 우리가 쓰는 경로

```
유저 브라우저
    │  Public connection OAuth
    ▼
Spring F1  (access_token + refresh_token 암호화 저장)
    │  크론 / 잡 이후
    ▼
Spring F3  RestClient
    │  Authorization: Bearer {access}
    │  Notion-Version: 2026-03-11
    ▼
https://api.notion.com/v1/pages   (markdown 필드)
```

이건 DESIGN.md가 말한 “크론은 Spring이 access token을 갱신한다”를 **그대로** 구현한다.
도구만 MCP가 아니라 REST다.

## 5. 호스트 MCP를 남기는 곳

로컬에서 스키마를 눈으로 보고 싶을 때:

```bash
# Cursor / Claude Code
# MCP 설정에만 추가. 프로덕션 코드 경로에 넣지 말 것
{
  "mcpServers": {
    "notion": {
      "url": "https://mcp.notion.com/mcp"
    }
  }
}
```

용도:

- Progress Logs DB가 잘 생겼는지 검색
- 페이지 마크다운이 어떻게 보이는지 확인
- 도구 목록 학습

금지:

- Spring이 이 URL에 유저 토큰을 넣기
- FastAPI `MultiServerMCPClient`에 notion 키를 추가
- 데모 서버 환경변수에 MCP OAuth 세션을 넣기

## 6. 나중에 MCP를 쓰고 싶어지면

조건이 모두 참일 때만 재검토한다.

1. “유저가 채팅으로 Notion 문서를 묻는다”가 제품에 있다
2. 그 읽기는 결정적 REST search로 부족하다
3. 유저별 MCP 토큰 금고와 refresh 단일 비행을 따로 만들 시간이 있다

그때도 **쓰기는 REST를 유지**한다. GitHub가 수집은 고정 tool, 쓰기는 F4인 것과 같다.
검색 에이전트와 미러는 같은 클라이언트를 공유하지 않는다.

# 09. 테스트와 합격 기준

모듈 경계마다 **계약 테스트 하나**가 있어야 한다. DESIGN.md §9와 같다.
외부 Notion을 모든 테스트가 치면 안 된다. 가짜 HTTP와 실연동을 가른다.

## 1. 레이어

| 레이어 | 도구 | 네트워크 |
| --- | --- | --- |
| 단위 | JUnit 5 | 없음. vault/client는 테스트 더블 |
| 계약 | WireMock 또는 `MockRestServiceServer` | localhost |
| E2E | 수동 + 선택 `@EnabledIfEnvironmentVariable` | 실 Notion (개발 워크스페이스) |

CI에는 단위+계약을 넣는다. 실토큰 E2E는 로컬/스테이징만.

## 2. F1 계약

파일: `NotionOAuthServiceTest`

| 케이스 | 기대 |
| --- | --- |
| `start` | authorize URL에 `owner=user`, `client_id`, `redirect_uri`, `state` |
| callback 정상 | vault에 암호문 저장. status.connected=true |
| callback `error=access_denied` | vault 변경 없음 |
| callback state 불일치 | 예외 `oauth_state`, token URL 호출 0 |
| callback nonce 재사용 | 두 번째 400 |
| disconnect | connected=false, 암호문 삭제 |
| `accessFor` 만료 + refresh 200 | 새 토큰 저장, 평문 반환 |
| `accessFor` 동시 2회 | token URL refresh는 1회 |
| refresh `invalid_grant` | needs_reauth=true, 저장된 토큰 없음 |
| status JSON | `access_token` 키 없음 (직렬화 스냅샷) |

refresh 동시성:

```
thread A, B = accessFor(sameUser)
join
verify(tokenApi, times(1)).refresh(...)
```

## 3. F2 계약

파일: `NotionWorkspaceProvisionerTest`

| 케이스 | 기대 |
| --- | --- |
| 맵 없음 | pages POST 1, databases POST 3, GET database 3 |
| 맵 있고 root GET 200 | POST 0 |
| root GET 404 | 다시 만들고 행 UPDATE |
| workspace parent 403 후 search 1페이지 | 그 page_id를 parent로 재시도 |
| search 0 | `workspace_missing` |

픽스처: `04`의 JSON을 `src/test/resources/notion/create-database-progress.json` 로 두고 요청 바디를 비교한다. 속성 키가 한글이면 실패.

## 4. F3 계약

파일: `NotionArtifactMirrorTest`

| 케이스 | 기대 |
| --- | --- |
| query 0건 | pages POST 1, status=created |
| query 1건 | pages POST 0, status=duplicate, url=기존 |
| body 빈 문자열 | Notion 호출 0, skipped/empty_body |
| kind=readme | 호출 0, skipped/not_mirrorable (1차) |
| 429 후 200 | retry ≥1, created |
| 401 → refresh → 200 | created |
| 401 → invalid_grant | failed needs_reauth, pages POST 0 (재시도 루프 없음) |
| 같은 proposal CAS 실패 | POST 0 |
| markdown에 proposal_id 머리글 포함 | 요청 바디 검증 |

`no_change`는 F3 바깥(enqueue) 테스트:

```
onProposalSaved(no_change) → outbox count 0
```

## 5. Controller

| 케이스 | 기대 |
| --- | --- |
| status 401 무토큰 | 401 |
| disconnect 멱등 | 두 번 200 |
| sync 남의 proposal | 404 |
| sync 미연결 | 409 not_connected |
| callback 302 Location 쿼리 | `notion=connected` |

## 6. 로그 회귀

테스트 출력과 감사:

```
rg -n "ntn_|nrt_|secret_|Bearer [A-Za-z0-9]" build/reports test-logs || true
```

매칭이 있으면 실패로 만든다. CI 한 줄.

## 7. 실연동 (로컬만)

```
NOTION_E2E=1
NOTION_TOKEN=...
NOTION_PARENT=...
```

시나리오 하나면 충분하다.

1. provision
2. progress mirror
3. 같은 proposal 다시 → 페이지 수 +0
4. 사람 눈으로 본문 확인

자동화 주장: 페이지 수 query count. 시각 품질은 사람이 본다.

## 8. 데모 합격

심사 자리에서 3분.

1. 새 유저(또는 시연 계정) 로그인
2. Notion 연결, 빈 페이지 선택
3. GitHub 잡으로 진행 메모 생성 (다른 팀)
4. 서비스 미리보기 = Notion 페이지 본문 (머리글 제외)
5. 같은 잡 재실행이 `no_change`면 Notion에 새 행이 안 생긴다
6. Notion 탭을 꺼도 미리보기는 남는다

하나라도 깨지면 Notion 파트가 원본을 침범한 것이다. 5번이 새 행을 만들면 멱등/필터 버그.

## 9. 테스트가 없는 채로 merge 하지 말 것

최소 세트 (이것만 있어도 Day 5 합격):

1. state 위조 callback
2. duplicate proposal
3. no_change enqueue 0
4. status 직렬화에 토큰 없음
5. refresh 단일 비행

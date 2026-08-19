# 12. 2차 — 지금 구현하지 말 것

기능명세 4.4 / 4.5 / PERS-003 / PORT 공개 페이지는 제품에 적혀 있다.
1차 완료 정의(`08`)를 넘기 전에 이 파일의 항목을 코드에 넣지 않는다.
넣기 시작하면 GitHub 계약과 원본 이중화가 다시 열린다.

## 1. 후보와 왜 미뤘는지

| 항목 | 명세 | 미룬 이유 | 착수 조건 |
| --- | --- | --- | --- |
| README 미러 | PERS-004 | 승인과 PR이 본체. 로그는 부가 | F4 데모가 안정 + MIRROR_TARGETS에 `readme` 한 줄 |
| 일정 vs 진행률 | PERS-003 | 캘린더 CUT. 날짜 원본이 없음 | 팀이 마일스톤을 GitHub milestone만으로 쓰기로 합의 |
| 친구/그룹 | TEAM-* | 공유 페이지 ACL, 토큰 주체가 유저 단위 | 그룹 테이블이 Spring에 생긴 뒤 |
| 팀 보드 | PROJ-* | GitHub 이벤트 → DB 행 업데이트는 쓰기 폭주 | 개인 미러 3 rps 운영이 안정 |
| 양방향 동기화 | 없음 | 원본이 두 개 | 하지 않는 쪽으로 유지하는 것을 권장 |
| 호스트 MCP 읽기 에이전트 | 채팅 | audience가 REST와 다름 | 채팅이 제품이 되고 MCP 금고를 따로 만들 때 |
| 포폴 공개 URL | PORT-003 | Frontend/Spring 정적 렌더 | Notion 공개 링크에 의존하지 말 것 (ACL이 유저 선택 페이지) |
| 파일 업로드 | 템플릿 이미지 | 20MiB 플로우, 복잡도 | 텍스트 포폴이 먼저 |
| backfill | 운영 | 과거 소급은 폭주 | 연결 직후 최근 20개만, 명시 버튼 |

## 2. README 미러 (가장 싼 2차)

`NotionConfig.MIRROR_TARGETS`에:

```
readme → 새 DB "README Proposals" 또는 Progress에 Kind select 추가
```

속성: repo (`owner/name`), path, proposal_id, status(`proposed`만, PR URL은 실행 후).
실행 후 PR URL을 쓰려면 Spring이 F4 `ExecuteResult.pr_url`을 받을 때 outbox가 아니라 **별 UPDATE**. 1차 스키마에 PR URL을 미리 넣지 말 것.

승인 전 초안만 남긴다면 GitHub 승인 게이트와 혼동된다. 남긴다면 페이지 머리에:

```
이 문서는 제안입니다. 레포에 반영된 README가 아닐 수 있습니다.
```

## 3. 팀 공간

그룹당 Notion 연결을 **리더 OAuth 하나**로 두면 멤버 퇴장 때 그랜트가 죽는다.
멤버마다 OAuth 하면 같은 보드를 다섯 번 만들게 된다.

합의안 (착수 시):

- 그룹 보드의 토큰 주체는 리더 connection
- 멤버는 Spring 권한만 본다 (미리보기)
- Notion 공유는 리더가 팀 페이지를 수동 공유
- 자동 “팀원 invite into Notion” 은 API 한계 + 권한 사고로 하지 않음

## 4. 일정

GitHub milestone due date만 쓰고, Notion Date 속성에 복사한다.
Google Calendar는 계속 제외.

경고 알림은 Spring이 계산하고, Notion에는 `Status=at_risk` select 정도만.

## 5. 확장 시 DESIGN 수정

맵에 kind를 추가할 때:

1. 이 파일에서 줄을 지우고 `DESIGN.md` §5에 variant를 적는다
2. `04` 스키마 `v2`
3. enqueue 필터 kind 목록
4. 계약 테스트 한 줄

그 전에 코드를 넣지 않는다.

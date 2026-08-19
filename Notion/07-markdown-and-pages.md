# 07. 마크다운 → Notion 페이지

FastAPI는 이미 완성된 `.md`를 준다. Notion 파트는 그 문자열을 **거의 그대로** 페이지 본문으로 넣는다.
블록 객체 트리를 우리 손으로 조립하지 않는다.

API: `POST /v1/pages` body 필드 `markdown` (Notion-flavored Markdown).
공식 한도: 요청당 블록 1000개, 페이로드 500KB, rich text 2000자.

## 1. 본문 조립

```
final_markdown = meta_callout + "\n\n" + body_markdown
```

`body_markdown`은 Spring `proposals.artifact_markdown` 그대로. 정규화·윤문·번역 금지.
증거 링크를 우리가 지어내지 않는다. FastAPI `evidence_refs`는 1차 본문에 안 넣어도 된다.

### 1.1 메타 callout

Notion markdown에서 callout 문법이 지원되면 그걸 쓰고, 애매하면 인용 + 수평선.

안전하게 호환되는 머리글 (모든 kind 공통):

```markdown
> Blocki 로그입니다. 원본은 서비스 미리보기에 있습니다. 이 페이지를 고쳐도 원본은 바뀌지 않습니다.
>
> proposal `{proposal_id}` · job `{job_id}` · digest `{proposal_digest 앞 12자}`

---

```

`partial`이면 한 줄 더:

```markdown
> 수집이 부분 성공했습니다 (`complete=false`). 내용이 비어 있을 수 있습니다.
```

이 머리글을 붙인 뒤에도 FastAPI가 준 제목(`# ...`)은 유지한다.

## 2. POST

동기 (기본):

```bash
curl -sS https://api.notion.com/v1/pages \
  -H "Authorization: Bearer $ACCESS" \
  -H "Notion-Version: 2026-03-11" \
  -H "Content-Type: application/json" \
  --data "$(jq -n \
    --arg ds "$DATA_SOURCE_ID" \
    --arg md "$FINAL_MARKDOWN" \
    '{
      parent: {type:"data_source_id", data_source_id:$ds},
      properties: { /* 04 */ },
      markdown: $md
    }')"
```

JSON 문자열 안의 개행은 `\n`이다. 파일에서 읽을 때:

```java
String body = objectMapper.createObjectNode()
    .putObject("parent")
        .put("type", "data_source_id")
        .put("data_source_id", ds)
        .end() // 실제로는 Map/DTO
    ...
    .put("markdown", finalMarkdown) // ObjectMapper가 이스케이프
    .toString();
```

프론트나 로그에 `finalMarkdown` 전체를 찍지 말 것. 길이만.

```
notion.mirror markdown_chars=1820
```

## 3. 큰 문서

포폴 템플릿(`Docs/developer_portfolio_template.md`)이 채워지면 수천 자가 된다. 보통 80KB 아래다.

| 조건 | 동작 |
| --- | --- |
| `len(utf8) <= 80_000` 이고 예상 500KB 미만 | 동기 POST |
| `len > 80_000` 또는 직전 요청이 timeout | `"allow_async": true` |
| 202 `async_task` | `poll_after_seconds` 만큼 쉬고 `GET {status_url}` 또는 `GET /v1/async_tasks/{id}` |
| 60초 안에 succeeded 아님 | `failed` retryable=true |
| 400 `validation_error` payload 한도 | 아래 분할 |

### 3.1 분할 (한도에 걸릴 때만)

1. `final_markdown`을 `\n## ` 기준으로 자른다
2. 첫 청크(머리글 + 첫 섹션들)로 부모 페이지를 만든다
3. 나머지 청크는 같은 페이지에 `PATCH /v1/blocks/{page_id}/children` 로 이어 붙이지 말고, **자식 페이지**로 만든다
   - 부모 markdown 끝에 `## 이어지는 섹션` 링크를 못 넣으면 제목만 `… (2)`, `… (3)`
   - 자식의 `Proposal ID` 는 `{uuid}#2` 처럼 쓰지 말고, **부모와 같은 Proposal ID를 넣지 않는다** (멱등 query가 여러 히트). 자식은 일반 페이지, DB 행 아님
4. 1차에서 분할이 필요해지면 구현. 그 전에는 `payload_too_large` 로 실패하고 본문을 자른 채 쓰지 **말 것**. 원본이 잘리면 안 된다

해커톤 템플릿 분량에서는 분할이 안 나올 가능성이 크다. 분기는 코드에 두되 테스트는 80_001자 fixture 한 개만.

## 4. 지원되는 마크다운 / 깨지는 것

FastAPI 템플릿이 쓰는 것:

- `#` ~ `####` 제목
- 목록 `-`
- 표 `| ... |` — Notion markdown이 표를 받으면 테이블 블록이 된다. 실패하면 코드블록으로 감싸지 말고, 그냥 원문을 넣는다. 깨지면 그 섹션만 코드펜스
- 코드펜스
- 링크
- 인라인 `` `code` ``

넣지 말 것:

- HTML `<div>`
- Jinja
- 로컬 이미지 `./images/architecture.png` — 템플릿에 있으면 링크로 남고, 업로드하지 않음 (1차 파일 업로드 생략)
- 2000자를 넘는 한 단락 — Notion이 거절할 수 있다. 거절되면 단락을 2000자에서 줄바꿈. **문장을 요약하지 않고** 자른 뒤 다음 단락으로

```
function splitRichText(s):
  if len(s) <= 2000: return [s]
  chunks = []
  while s:
    chunks.append(s[:2000])
    s = s[2000:]
  return chunks
```

`markdown` 필드를 쓰면 서버가 이 분할을 할 수도 있다. 400이 나면 그때 우리가 나눈다. 미리 모든 단락을 나누지 말 것.

## 5. 페이지 URL

응답:

```json
{
  "id": "b55c9c91-384d-452b-81db-d1ef79372b75",
  "url": "https://www.notion.so/b55c9c91..."
}
```

`url`이 있으면 그대로 outbox에 저장.
없으면:

```
https://www.notion.so/{id without dashes}
```

프론트 “Notion에서 보기”는 이 URL.

## 6. 아카이브 / 삭제

유저가 Notion에서 행을 지우거나 아카이브하면:

- 다음 같은 `proposal_id` 재미러는 query miss → 새 행이 생긴다
- 원본 Spring은 그대로
- 일부러 다시 만들고 싶지 않으면 outbox `succeeded`를 유지하고 재미러하지 않음. **succeeded를 재실행하지 않는 것이 기본**

수동 sync는 succeeded를 건너뛴다 (`05` §7).

## 7. 로컬 스모크 (OAuth 없이)

공유된 빈 데이터베이스(또는 data source) ID가 있을 때:

```bash
python3 - <<'PY'
import json, os, urllib.request
token = os.environ["NOTION_TOKEN"]
ds = os.environ["DS"]
md = "# Hello from Blocki\n\n- item 1\n- item 2\n"
body = {
  "parent": {"type": "data_source_id", "data_source_id": ds},
  "properties": {
    "Name": {"title": [{"type":"text","text":{"content":"smoke"}}]}
  },
  "markdown": md,
}
req = urllib.request.Request(
  "https://api.notion.com/v1/pages",
  data=json.dumps(body).encode(),
  headers={
    "Authorization": f"Bearer {token}",
    "Notion-Version": "2026-03-11",
    "Content-Type": "application/json",
  },
  method="POST",
)
print(urllib.request.urlopen(req).read().decode()[:500])
PY
```

`Name` 속성이 없는 테스트 DB면 속성 키를 그 DB의 title 속성 이름에 맞춘다.
프로덕션 코드는 우리 스키마의 `Name`만 쓴다.

---
name: spec-from-thread
description: Take a Slack thread URL, classify each message into [decision / requirement / open question / chatter], and create a Notion spec page from the non-chatter messages with the original thread permalink attached.
---

# spec-from-thread

PM이 신규 기능 논의를 Slack thread에서 진행하면 결정·요구사항·오픈 질문이 50-100개 메시지에 흩어진다. 이를 Notion 스펙 페이지로 옮기는 데 1-2시간 → 결국 안 함. URL 1개 입력으로 fetch + 분류 + Notion 작성이 한 명령으로 끝남.

## Input

- `--thread-url <slack-permalink>` (필수) — Slack thread 영구 링크 (`https://<workspace>.slack.com/archives/<channel>/p<ts>` 형식)
- `--notion-parent-id <id>` (선택, 환경변수 `CLAUDE_CODE_TOOLKIT_SPEC_PARENT_ID`로 fallback) — Notion 부모 페이지 ID
- `--title <text>` (선택) — 생성할 스펙 페이지 제목. 누락 시 thread 첫 메시지에서 LLM 추출
- 입력 누락 → AskUserQuestion (추측 금지)

## Output

콘솔 1줄 요약 + Notion 페이지 URL:

```
Thread <permalink> → 메시지 N건 분류 (결정 N / 요구 N / 질문 N / 잡담 N) → Notion 스펙 페이지 생성
- Notion: <page URL>
```

생성되는 Notion 페이지 본문 형식:

```
# <제목>

## 출처
- Slack thread: <permalink>
- 추출 일시: YYYY-MM-DD HH:MM KST
- 메시지 수: N건 (결정 N / 요구 N / 질문 N / 잡담 제외)

## 목적
<thread 첫 메시지 또는 LLM 추정 1단락>

## 결정사항
- <결정 1> (작성자: @<display_name>, <시각>)
- ...

## 요구사항
- <요구 1> (작성자: ...)
- ...

## 오픈 이슈
- <질문 1> (작성자: ...)
- ...

## 다음 액션 (제안)
- [ ] <액션 1>
- [ ] <액션 2>

## 검토 한계
- 잡담 분류된 메시지 N건은 본문에서 제외됨 (원문은 Slack thread에서 확인)
- 결정/요구 구분이 모호한 경우 결정 우선
```

## 진행 순서

### 1. 입력 파싱

- `--thread-url` 파싱: workspace·channel-id·ts 추출. 형식 아니면 거부
- `--notion-parent-id` 누락 + env 누락 → AskUserQuestion
- `--title` 누락 → step 3 후 LLM 추정

### 2. Thread 메시지 수집

`mcp__slack__get_channel_history` 호출:
- `channel`: 추출한 channel-id
- `oldest`: thread_ts (parent 메시지 시각)
- `latest`: thread_ts + 30일 (thread 응답 모두 커버)
- `limit`: 200

또는 thread 전용 도구가 있으면 우선 사용 (Slack MCP 도구명은 `docs/mcp-setup.md` dry-run으로 사전 확정).

각 메시지에 `mcp__slack__get_user_info`로 user_id → display_name 변환 (캐싱).

### 3. LLM 분류

각 메시지를 [결정 / 요구 / 질문 / 잡담] 1개로 분류:

| 카테고리 | 신호 |
|---|---|
| 결정 | "X로 정함", "Y하기로", "최종적으로 Z", 결재 이모지 |
| 요구 | "Y가 필요함", "X 해야 함", "반드시", spec 형태 |
| 질문 | `?`로 끝남, "어떻게", "가능한가", "예외는?" |
| 잡담 | 인사, 동의("ㅇㅋ"), 감사, 무관 정보 |

신뢰도 모호 시 결정 우선 (요구·질문에 결정성 발화가 섞이면 결정).

### 4. 제목 추정 (필요 시)

`--title` 누락이면 thread 첫 메시지 + 결정 사항 1-2건을 LLM 입력으로 50자 이내 제목 생성.

### 5. Notion 페이지 생성

`mcp__notion__create_page` 호출:
- `parent`: `page_id` = 입력된 부모 페이지 ID
- `properties`: `title` = 제목
- `children`: 위 본문 형식대로 paragraph/heading_2/bulleted_list_item/to_do block 조합

**hook 동작:** plugin의 `PreToolUse(mcp__notion__create_page)` hook이 발화 — 사용자에게 "Notion에 신규 페이지 생성합니다. 진행?" 1회 확인. **본 릴리스에서는 인터랙티브 confirm 전용 — bypass 사용 금지** (현 hook은 audit log를 `decision-log-YYYY-MM-DD.jsonl`로만 기록하므로 spec-from-thread bypass 시 다른 skill의 audit trail에 섞여버린다). cron 자동화는 caller-aware audit log 분리 후 활성화 검토.

### 6. 출력

콘솔 1줄 요약 + Notion 페이지 URL. 끝.

## 함정

- **Thread parent 메시지가 #channel root이고 thread 응답이 거의 없음**: thread 자체가 thread가 아닌 단발 메시지일 수 있음. parent 1건만 있으면 "스펙 추출 의미 없음" 안내 후 종료
- **분류 prompt 너무 느슨**: 결정과 요구가 같은 문장에 섞이면 둘 다로 잘못 분류됨. system prompt에 "한 메시지 당 1개 카테고리만, 결정 우선" 강제
- **LLM이 잡담 메시지에서 잡담 외 정보 누락**: 잡담 분류된 메시지에 사실 정보가 섞여 있을 수 있음 — "검토 한계" 섹션에 명시
- **동일 thread 재실행 시 중복 페이지**: 매 호출마다 신규 Notion 페이지 생성. 중복 방지 로직 미포함 — 사용자가 알아서 기존 페이지 삭제 또는 `--update` 모드 (TBD)
- **Slack MCP의 thread 전용 도구 부재**: `docs/mcp-setup.md` dry-run에서 도구명 확정 (예: `get_thread_replies`가 별도 노출되는지). 부재 시 `get_channel_history` + thread_ts로 우회

## 톤

- 결정/요구 인용 시 원문 유지, 의역 금지
- 작성자 익명 처리 X — display_name 그대로
- "추측" 표현 환영 (잡담 분류 신뢰도 낮은 경우)
- 가짜 자신감 ("이게 모든 결정입니다" 등) 금지

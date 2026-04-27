---
name: ticket-triage-router
description: Classify a Slack #support message and produce an answer draft. Spawns ticket-classifier (Slack read-only) and ticket-draft-writer (GitHub + Notion search read-only) subagents in series. No auto-post — draft only, for human review.
---

# ticket-triage-router

**워크숍 픽 #2 (V+X 와우).** CS 팀이 Slack #support 채널에 들어오는 외부 사용자 메시지를 [버그/기능 요청/사용법 문의] 분류하고 답변 초안을 작성하는 작업을 자동화. 도구 권한별 격리(Pattern C)로 분류 단계와 답변 작성 단계를 audit trace에서 분리 가능. 시연 시퀀스는 `docs/workshop-flow.md` 참조.

## Input

- `--message-url <slack-permalink>` (필수) — Slack 메시지 영구 링크 (`https://<workspace>.slack.com/archives/<channel>/p<ts>` 형식)
- 입력 없으면 AskUserQuestion으로 받기 — 추측 금지

## Output

```
## Ticket #<auto-id>

### 분류 결과 (ticket-classifier)
- 카테고리: bug | feature_request | usage_question
- 신뢰도: 높음 / 중간 / 낮음
- 근거: <1-2문장>

### 답변 초안 (ticket-draft-writer)
<카테고리별 검색 결과 기반 초안 (200-400자)>

### 관련 자료
- GitHub issue: #N — <제목> (중복 가능성 / 해결됨 / 미해결)
- Notion 페이지: <제목> — <URL>

### 수동 검토 필요 항목
- <항목 1>
- <항목 2>
```

## 진행 순서

### 1. 입력 검증

- `--message-url` 누락 → AskUserQuestion
- URL이 Slack permalink 형식이 아니면 거부

### 2. ticket-classifier subagent 실행

Task 툴로 ticket-classifier 호출:

```
description: "Classify support ticket"
subagent_type: "ticket-classifier"
prompt: |
  Slack message URL: <message-url>

  메시지 본문 fetch 후 [bug / feature_request / usage_question] 중 1개로 분류.
  신뢰도와 근거 1-2문장 포함.
```

ticket-classifier는 `mcp__slack__get_message` (또는 동등 도구) read-only만 가짐. 분류 결과 1개 + 신뢰도 + 근거 반환.

### 3. ticket-draft-writer subagent 실행

ticket-classifier 결과를 받아 ticket-draft-writer 호출:

```
description: "Draft support answer"
subagent_type: "ticket-draft-writer"
prompt: |
  메시지 본문: <classifier가 fetch한 내용>
  카테고리: <classifier 결과>

  카테고리별로:
  - bug: GitHub issue 중복 검색 (mcp__github__search_issues), 같은 증상 issue 있으면 링크 + 상태(open/closed) 인용
  - feature_request: Notion 로드맵 페이지 검색 (mcp__notion__search), 이미 계획에 있으면 일정 인용, 없으면 "검토 후 답변" 답변
  - usage_question: Notion FAQ/docs 검색, 가장 가까운 페이지 1-2개 인용

  답변 초안 200-400자 + 관련 자료 링크 출력.
  자동 게시 X — 초안만.
```

ticket-draft-writer는 `mcp__github__search_issues` + `mcp__notion__search` read-only만 가짐. **write 도구 0개**.

### 4. 결과 합성

ticket-classifier + ticket-draft-writer 결과를 SKILL output 형식으로 합쳐 출력. 자동 Slack 답변 게시 **금지** — CS 담당자가 검토 후 직접 게시.

## 자동화 (선택)

Slack Events API receiver 별도 인프라에서 `#support` 채널 신규 메시지 수신 시 webhook이 `claude --print "/claude-code-toolkit:ticket-triage-router --message-url <url>"` 외부 호출 → 결과를 thread에 봇이 미리보기로 첨부 (게시는 사람이).

Claude Code 내부 hook은 외부 Slack 이벤트를 받지 못하므로 별도 receiver 필요.

## 워크숍 시연 (T1 명시 호출)

```
> /claude-code-toolkit:ticket-triage-router --message-url https://<workspace>.slack.com/archives/C123/p1234567890
```

사전 셋업:
- 더미 `#support` 채널 + 메시지 1건 시드 (예: "기능 X에서 Y 동작 후 에러가 납니다. 어떻게 해결하나요?")
- sandbox GitHub에 더미 issue 1건 (같은 증상)
- Notion FAQ 페이지 1건

시연 1분 흐름:
1. 명령 입력 (5초)
2. ticket-classifier subagent 실행 — 분류: bug, 신뢰도 높음 (15초)
3. ticket-draft-writer subagent 실행 — GitHub issue #N 인용 + 답변 초안 (30초)
4. 결과 출력 + 진행자 설명: "subagent 2개가 도구 권한 분리 — Slack read만 / GitHub+Notion search만. 자동 게시 X" (10초)

핵심 메시지: "subagent가 각자 다른 MCP read 권한만 갖고 격리 실행. 답변은 사람이 검토 후 게시 — write 도구 0개"

## 함정

- **classifier가 모든 메시지를 usage_question으로 분류**: LLM 기본값 편향. system prompt에 "감정/의견 표현은 분류 신호 아님 — 키워드(crash, error, doesn't work = bug; "제안", "있으면 좋겠다" = feature)에 우선" 강제
- **draft-writer가 GitHub/Notion을 둘 다 검색**: 카테고리에 따라 적절한 source만 검색해야 함. classifier 결과를 prompt에 명시적으로 분기 요구
- **답변 자동 게시 유혹**: write 도구 부여하면 봇 답변이 잘못된 정보 게시 위험. 절대 write 부여 X
- **메시지 URL이 thread 안 메시지**: thread parent 메시지가 더 중요한 컨텍스트일 수 있음. classifier가 thread_ts 감지 시 parent도 함께 fetch

## 톤

- 답변 초안은 정중·친절. 사용자 메시지의 감정 톤 무시
- 미해결 case는 "현재 검토 중입니다. <issue link>에서 진행 상황 확인 가능합니다" 식으로 honest
- "확실치 않음" 표현 환영 — 가짜 자신감 금지

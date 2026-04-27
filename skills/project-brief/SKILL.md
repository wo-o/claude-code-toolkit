---
name: project-brief
description: Generate a one-page project brief by gathering recent activity from Slack and project status from Notion. Spawns slack-scout and notion-scout subagents in parallel, then synthesizer to merge. Input - project name (string).
---

# project-brief

특정 프로젝트의 Slack 최근 활동 + Notion 진행 상황을 수집해 한 페이지 브리프로 정리한다.

## Input

- 프로젝트명 (필수, 자연어 문자열). 예: `mobile-relaunch`, `payments-v2`, `Q2 OKR`
- 입력 없으면 AskUserQuestion으로 받을 것 — 추측 금지

## Output

다음 형식의 한 페이지 브리프:

```
# <프로젝트명> 브리프 (YYYY-MM-DD)

## 한 줄 요약
<현재 상태 1-2문장>

## Slack 최근 활동
- <YYYY-MM-DD HH:MM> #channel — 발화자: 핵심 메시지
- (최대 5개, 최신순)

## Notion 진행 상황
- 페이지/DB: <제목>
- 마일스톤: <달성 / 진행중 / 지연>
- 핵심 결정: <목록>

## 발견된 이슈
- <이슈 1>
- <이슈 2>

## 다음 액션 (제안)
- [ ] <액션 1>
- [ ] <액션 2>
```

## 진행 순서

### 1. 입력 검증
- 프로젝트명 비어있으면 AskUserQuestion으로 받기
- 너무 추상적이면 (예: "회사") 구체화 요청

### 2. 병렬 정찰 (Subagent 2개 동시 spawn)

Task 툴로 두 서브에이전트를 한 메시지에서 동시 호출.

**slack-scout 호출:**
```
description: "Scout Slack for <프로젝트명>"
subagent_type: "slack-scout"
prompt: "프로젝트명: <프로젝트명>. 관련 채널/메시지 검색해서 최근 7일 핵심 활동 정리. 응답은 SKILL.md output 섹션의 'Slack 최근 활동' 형식으로."
```

**notion-scout 호출:**
```
description: "Scout Notion for <프로젝트명>"
subagent_type: "notion-scout"
prompt: "프로젝트명: <프로젝트명>. 관련 페이지/DB 검색해서 진행 상황 정리. 응답은 SKILL.md output 섹션의 'Notion 진행 상황' 형식으로."
```

병렬로 한 메시지에서 두 Task tool call. 절대 순차 실행 금지 — Pattern B(fan-out scout)의 핵심.

### 3. 통합 (Subagent synthesizer)

slack-scout / notion-scout 결과를 받아 synthesizer subagent 호출:

```
description: "Synthesize project brief"
subagent_type: "synthesizer"
prompt: |
  프로젝트명: <프로젝트명>

  [Slack 정찰 결과]
  <slack-scout 응답>

  [Notion 정찰 결과]
  <notion-scout 응답>

  위 두 입력을 SKILL.md output 섹션 전체 형식으로 합쳐 한 페이지 브리프 생성.
  특히 다음을 강조:
  - Slack 발언과 Notion 문서가 어긋나는 지점 (이슈 후보)
  - Slack에서 결정됐지만 Notion에 반영 안 된 것 (다음 액션 후보)
```

### 4. 출력

synthesizer 결과를 그대로 사용자에게 출력. 추가 가공 금지 — 형식은 synthesizer가 강제한다.

(선택) 사용자가 "Notion에 저장" 요청하면 `mcp__notion__create_page`로 페이지 생성. 부모 페이지는 사용자에게 묻기. 이 경로는 plugin hook (`PreToolUse(mcp__notion__create_page)`)에서 1회 사용자 확인 후 진행.

## 함정

- **두 정찰 subagent를 순차 호출**: Pattern B fan-out 의의 사라짐. 반드시 한 메시지에 두 Task call 동시
- **synthesizer를 거치지 않고 메인이 합침**: 메인 컨텍스트에 두 결과가 그대로 들어와 컨텍스트 격리 의미가 없다. 통합도 분리된 컨텍스트에서
- **MCP를 메인이 직접 호출**: 메인이 Slack/Notion에 직접 접근하면 subagent 의의가 사라진다. MCP 호출은 정찰 subagent 안에서만
- **결과 형식 깨짐**: synthesizer 프롬프트에서 형식 강제하지 않으면 매번 다르게 나옴. 출력 섹션 형식을 synthesizer 입력 프롬프트에 명시

## 톤

- 사실 위주, 감상 금지
- 마일스톤 진행 상태는 "달성/진행중/지연" 셋 중 하나로만 표기
- Slack 메시지 인용 시 발화자 + 날짜 필수

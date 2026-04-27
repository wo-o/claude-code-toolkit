---
name: decision-log-keeper
description: Extract decision messages from a Slack channel for a given date and append them as rows to a Notion "Decision Log" database. Designed to run daily via launchd cron, but can be invoked manually with --date for backfill or workshop demo.
---

# decision-log-keeper

**워크숍 픽 #1 (H+X 워밍업).** PM 회의에서 Slack #decisions 채널에 던져진 결정사항을 매일 자동으로 Notion DB로 옮긴다. 분기 회고 시점에 "그때 왜 이렇게 결정했지?"를 추적할 수 있도록 영구 보존. 시연 시퀀스는 `docs/workshop-flow.md` 참조.

## Input

- `--date YYYY-MM-DD` (선택, 기본 오늘) — 추출 대상 날짜
- `--channel #channel-name` (선택, 기본 `#decisions`) — Slack 채널
- `--notion-db-id <id>` (선택, 환경변수 `CLAUDE_CODE_TOOLKIT_DECISION_DB_ID`로 fallback) — Notion DB ID

## Output

추가된 row 개수 + 각 row의 Notion URL 출력. 콘솔 1줄 요약:

```
2026-04-26: #decisions 채널 결정 N건 추출 → Notion DB row N건 추가
- <Notion row URL 1>
- <Notion row URL 2>
- ...
```

## 진행 순서

### 1. 입력 파싱

- `--date` 누락 → 오늘 날짜 (timezone Asia/Seoul 기준 KST)
- `--channel` 누락 → `#decisions`
- `--notion-db-id` 누락 + `CLAUDE_CODE_TOOLKIT_DECISION_DB_ID` env 누락 → AskUserQuestion으로 받기

### 2. Slack 메시지 수집

`mcp__slack__get_channel_history` 호출:
- `channel`: 입력된 채널의 ID (`mcp__slack__list_channels`로 이름→ID 변환)
- `oldest`: 입력 날짜 00:00 KST의 Unix timestamp
- `latest`: 입력 날짜 23:59 KST의 Unix timestamp
- `limit`: 200

응답 메시지 N건. 0건이면 "그날 메시지 없음" 출력하고 종료.

### 3. 결정 메시지 필터링 (LLM)

수집된 메시지 N건을 LLM으로 분류:
- **결정**: "X로 정함", "Y하기로 결정", "최종적으로 Z" 등 결정성 발화. user reaction `:white_check_mark:` `:approved:` 같은 결재 이모지가 붙은 메시지 우대
- **잡담**: 인사·날씨·점심 등
- **질문**: `?`로 끝나거나 답을 구하는 발화

결정만 추출 — 잡담/질문 제외.

각 결정에 다음 메타 부착:
- `작성자`: `mcp__slack__get_user_info`로 user_id → display_name 조회
- `시각`: 메시지 timestamp ISO 8601
- `맥락 1줄`: 같은 thread의 이전 메시지 1-2개 또는 채널 직전 메시지 1개에서 결정 배경 1줄 요약
- `Slack 영구 링크`: `https://<workspace>.slack.com/archives/<channel-id>/p<ts>` 형식

### 4. Notion DB row 추가

각 결정마다 `mcp__notion__create_page` 호출:
- `parent`: `database_id` = 입력된 DB ID
- `properties`:
  - `Title` (title): 결정 본문 1줄 요약 (50자 이내)
  - `Date` (date): 결정 시각
  - `Author` (rich_text): Slack display_name
  - `Context` (rich_text): 맥락 1줄
  - `Slack Link` (url): Slack 영구 링크
- `children`: 결정 본문 전문 (paragraph block)

**hook 동작:** 첫 row 생성 시 plugin의 `PreToolUse(mcp__notion__create_page)` hook이 발화 — 사용자에게 "Notion DB에 N건 row 추가합니다. 진행?" 1회 확인. cron 모드에서는 `CLAUDE_CODE_TOOLKIT_CRON_MODE=1` 환경변수로 hook bypass + audit log만 기록 (audit log 경로 = `~/.claude-code-toolkit/audit/decision-log-YYYY-MM-DD.jsonl`).

### 5. 출력

추가된 row 개수와 각 row Notion URL을 콘솔 1줄 요약 형식으로 출력. 끝.

## 자동 운영 (cron)

매일 18:00 KST launchd cron 권장:

```
# ~/Library/LaunchAgents/<reverse-domain>.claude-code-toolkit.decision-log-keeper.plist
# Program: /usr/local/bin/claude
# ProgramArguments: -p "/claude-code-toolkit:decision-log-keeper"
# StartCalendarInterval: { Hour=18, Minute=0 }
# EnvironmentVariables:
#   CLAUDE_CODE_TOOLKIT_CRON_MODE=1
#   CLAUDE_CODE_TOOLKIT_DECISION_DB_ID=<id>
```

상세 plist 예시는 `docs/cron-setup.md`.

## 워크숍 시연 (T1 명시 호출)

```
> /claude-code-toolkit:decision-log-keeper --date 2026-04-26
```

사전 셋업:
- Notion "Decision Log" DB 1회 생성 (Title / Date / Author / Context / Slack Link 컬럼)
- `#decisions` 채널에 더미 결정 메시지 1-2건 시드 (예: "토큰 vesting 6개월 → 12개월로 변경하기로 결정")
- 진행자 PC에 `CLAUDE_CODE_TOOLKIT_DECISION_DB_ID` 환경변수 + Slack/Notion MCP 인증 완료

시연 30초 흐름:
1. 명령 입력 (5초)
2. Slack 메시지 fetch + LLM 결정 추출 (10초)
3. Notion DB row 생성 + 사용자 확인 prompt → y (5초)
4. 진행자 Notion UI 새로고침해서 row 노출 확인 (10초)

핵심 메시지: "원래는 매일 18:00 cron이 자동. 워크숍에선 `--date`로 그 명령 직접 한 번 돌렸음"

## 함정

- **Slack timezone 혼동**: `oldest`/`latest`는 Unix timestamp(UTC). `--date 2026-04-26` 입력 시 KST 00:00-23:59을 UTC로 환산 후 전달
- **결정/잡담 LLM 분류 prompt 너무 느슨**: false positive (잡담을 결정으로 추출) 발생 시 Notion DB가 노이즈로 가득 참. system prompt에 "결정성 발화 = 명시적 의사결정 동사(`정함/결정/확정/택함`) 또는 결재 이모지(:white_check_mark: :approved:) 동반" 강제
- **Notion DB schema drift**: DB의 컬럼명·타입이 변경되면 `create_page` 실패. 첫 호출 시 `mcp__notion__fetch <db-id>`로 schema 검증 1회 후 진행
- **cron 모드에서 hook prompt가 떠 멈춤**: `CLAUDE_CODE_TOOLKIT_CRON_MODE=1` 환경변수로 hook bypass + audit log만 기록. 이 환경변수는 plugin hooks.json에서 검사

## 톤

- 결정 본문 인용 시 원문 유지, 의역 금지
- 작성자 익명 처리 X — display_name 그대로
- 맥락 1줄은 "왜 이 결정이 나왔는지" 한 문장, 추측 금지

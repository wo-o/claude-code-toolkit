# Workshop Flow

5분 워크숍 시연 가이드. 픽 3종(보수→V→X 클라이맥스) 순서 고정.

## 시연 시퀀스

| 시각 | 명령 | 의도 | 톤 |
|---|---|---|---|
| T+0:30 | `/claude-code-toolkit:decision-log-keeper --date 2026-04-26` | 결정사항 Slack→Notion 자동 정리 | H+X 워밍업 — "오늘 결정 흩어졌죠? 한 줄로 끝." |
| T+1:30 | `/claude-code-toolkit:ticket-triage-router --message-url <slack-permalink>` | CS 메시지 분류 + 답변 초안 (Pattern C 격리) | V+X 와우 — subagent 권한 격리 시각화 |
| T+3:30 | `/claude-code-toolkit:pr-pattern-auditor --pr 42 --repo <owner>/<repo>` | PR diff → 안티패턴 + cross-module 영향 → PR comment write-back | X 클라이맥스 — hook prompt 1회 발화, 사용자 확인 후 게시 |

총 5분. 각 데모 사이 30초 전환 멘트.

## 사전 준비

- demo Slack workspace 준비: `#decisions` 채널에 시연용 메시지 3-5건, `#support` 채널에 사용자 메시지 1건 (분류 대상)
- demo Notion workspace 준비: "Decision Log" DB 1개 (Title/Date/Channel/Owner/Decision 컬럼)
- demo PR 준비: sandbox repo에 의도적 안티패턴 1-2개 포함 (예: 권한 검사 누락된 mutation 엔드포인트, 에러 swallow, 시그니처 변경 1건)
- 시연자 환경: GitHub PAT, Slack/Notion OAuth 사전 완료. `CLAUDE_CODE_TOOLKIT_CI_MODE` / `CLAUDE_CODE_TOOLKIT_CRON_MODE`는 **시연 중 export 금지** — hook prompt를 보여주는 것이 시연 핵심.

## 발표자 노트

- decision-log-keeper: "매일 18:00 launchd cron으로 자동 실행 — 워크숍은 manual 호출 시연"
- ticket-triage-router: classifier(Slack read-only) → draft-writer(GitHub+Notion search read-only) 권한 격리 강조. "한 에이전트가 모든 권한 들고 있으면 audit이 깨진다"
- pr-pattern-auditor: hook prompt가 발화하는 순간을 일부러 보여주기. "이게 plugin이 하는 일 — write 직전 사용자 confirm을 강제."

## 부적합 픽 (시연 X)

| 스킬 | 부적합 사유 |
|---|---|
| project-brief | scout 2개 + synthesizer 합성으로 시연 시간 초과 (3분+) |
| pr-storybook | Slack `chat:write` sandbox 분배 부담 |
| spec-from-thread | thread URL 사전 준비 의존, 합성 결과가 길어서 무대 가독성 낮음 |
| battle-card-builder | HubSpot sandbox 부재 (Drift H, GA 신생) |
| deal-context-loader | 동일 (HubSpot Drift H) |

## 사후 운영

- 시연 후 워크숍 참석자에게 install 가이드 공유 (`README.md` §설치)
- Slack `#claude-toolkit` 채널 개설 → Q&A + bug 리포트
- 1주 후 사용 통계 수집 (Notion DB row 수, audit log line 수) → 운영 검토

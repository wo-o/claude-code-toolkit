# Workshop Flow

5분 워크숍 시연 가이드. 픽 3종(보수→V→X 클라이맥스) 순서 고정.

## 시연 시퀀스

| 시각 | 명령 | 의도 | 톤 |
|---|---|---|---|
| T+0:30 | `/claude-code-toolkit:decision-log-keeper --date 2026-04-26` | 결정사항 Slack→Notion 자동 정리 | H+X 워밍업 — "오늘 결정 흩어졌죠? 한 줄로 끝." |
| T+1:30 | `/claude-code-toolkit:ticket-triage-router --message-url <slack-permalink>` | CS 메시지 분류 + 답변 초안 (Pattern C 격리) | V+X 와우 — subagent 권한 격리 시각화 |
| T+3:30 | `/claude-code-toolkit:spec-from-thread --thread-url <slack-permalink>` | Slack 스레드 raw 대화 → Notion 스펙 페이지 자동 생성 | X 클라이맥스 — Notion write 직전 hook prompt 발화 |

총 5분. 각 데모 사이 30초 전환 멘트.

## 사전 준비

- demo Slack workspace 준비: `#decisions` 채널에 시연용 메시지 3-5건, `#support` 채널에 사용자 메시지 1건 (분류 대상), 별도 채널에 의사결정/요구사항이 오간 thread 1개 (10-20개 메시지, spec-from-thread용)
- demo Notion workspace 준비: "Decision Log" DB 1개 (Title/Date/Channel/Owner/Decision 컬럼) + "Specs" 페이지 부모 1개 (spec-from-thread가 child 페이지로 추가)
- 시연자 환경: claude.ai/customize/connectors에서 Slack/Notion OAuth 사전 완료. `CLAUDE_CODE_TOOLKIT_CRON_MODE`는 **시연 중 export 금지** — hook prompt를 보여주는 것이 시연 핵심.

## 발표자 노트

- decision-log-keeper: "매일 18:00 launchd cron으로 자동 실행 — 워크숍은 manual 호출 시연"
- ticket-triage-router: classifier(Slack read-only) → draft-writer(Notion search read-only) 권한 격리 강조. "한 에이전트가 모든 권한 들고 있으면 audit이 깨진다"
- spec-from-thread: hook prompt가 발화하는 순간을 일부러 보여주기. "이게 plugin이 하는 일 — Notion write 직전 사용자 confirm을 강제. raw 대화가 구조화된 스펙 문서로 변환되는 과정을 가장 직관적으로 보여줌."

## 부적합 픽 (시연 X)

| 스킬 | 부적합 사유 |
|---|---|
| project-brief | scout 2개 + synthesizer 합성으로 시연 시간 초과 (3분+) |
| battle-card-builder | HubSpot sandbox 부재 (Drift H, GA 신생) |
| deal-context-loader | 동일 (HubSpot Drift H) |

## 사후 운영

- 시연 후 워크숍 참석자에게 install 가이드 공유 (`README.md` §설치 + §Connectors 활성)
- Slack `#claude-toolkit` 채널 개설 → Q&A + bug 리포트
- 1주 후 사용 통계 수집 (Notion DB row 수, audit log line 수) → 운영 검토

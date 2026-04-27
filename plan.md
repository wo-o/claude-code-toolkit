---
created: 2026-04-27
updated: 2026-04-27
type: deep-plan-output
spec_hash: e7c57100
status: legacy
---

> **LEGACY NOTICE (2026-04-27):** 본 문서는 `story-toolkit` 초기 설계 단계의 산출물로, **Story Protocol 사내용** 으로 작성됨. 이후 plugin은 (a) 식별자 일체 제거, (b) Solidity/블록체인 도메인 제거 → 범용 팀 생산성 plugin으로 재정렬됨 (smart-contract-radar → pr-pattern-auditor 등). **현재 시점 정합성 X** — Section 1 원문, smart-contract-radar 1페이지 설계, 일부 워크숍 픽 묘사는 이전 식별자 시대 상태. 현행 정의는 `README.md` + `skills/*/SKILL.md` + `docs/workshop-flow.md` 참조. 본 문서는 설계 의도·9축 평가·Drift 분석 추적용으로만 보존.

---

# story-toolkit 플러그인 전체 설계도

**Status:** Legacy (superseded — see notice above)
**Date:** 2026-04-27
**Spec hash:** e7c57100 (frozen — 도메인 일반화 후 재산출 안 함)

---

## 1. 내 의도 (원문)

_사용자 원문 verbatim. Claude 재진술 없음._

**Goal:** "Story Protocol 사내 전사용 plugin `story-toolkit` 전체 설계도. plugin 컨테이너 안에 어떤 skills/subagents가 들어가야 하는지, 각 후보 1페이지 설계서, 워크숍 시연용 추천 2-3개 + 시연 흐름까지."

**Intent items:**
- I1: "사내에서 활용할 수 있는 skills, subagents를 플러그인화" — verification: `git clone story-toolkit` → `/plugin install` → 엔지/PM/마케/세일즈/CS 각 직군이 자기 직군 스킬을 한 번에 호출 가능
- I2: "여기서 만든 것들을 기반으로 실습에서 와우가 나올 수 있도록 세팅" — verification: 워크숍 비개발자 참가자가 시연 1건당 30초 내 결과물 보고 "와우" 반응 + 본인 PC에서 동일 명령 재현 가능
- I3: "사내에서 사용하기 유용한 전체 플러그인에 대한 설계도" — verification: 본 문서를 읽은 사람이 plugin 디렉토리 트리를 그대로 scaffold 할 수 있음 (manifest, 후보 목록, 의존 MCP, 워크숍 픽 포함)
- I4: "워크숍 시연용 추천 2-3개 + 시연 흐름" — verification: 추천된 후보를 순서대로 실행하면 5분 이내 와우 도달, 각 단계 1줄 설명 가능

**Non-goals:**
- project-brief 재설계 X — 그대로 포함만
- Story 사내 private, 외부 공개 X
- 사내 GitHub repo 생성·배포·MCP credential 발급 같은 사내 인프라 의존 task X (이번 단계는 동작 가능한 dev artifact 작성까지)

**Constraints:**
- story-toolkit 플러그인 컨테이너 (단일 plugin)
- 워크숍 시연 가능 (비개발자, macOS+Claude Pro, 3시간)
- MCP 신규 셋업 자유
- 시연 1건당 시간 제한 없음

**Success criteria:**
- 후보 skills/subagents 8-12개
- 각 후보 1페이지 설계서 (이름/한 줄 요약/입력/출력/MCP/난이도/와우임팩트/사내효용/직군)
- 워크숍 시연용 추천 2-3개 + 시연 흐름
- story-toolkit 디렉토리 레이아웃 + manifest 설계

**Context:**
- Story Protocol = IP L1 블록체인. 전사 — 엔지/PM/마케/세일즈/CS
- 사내 가용 스택: GitHub, Slack, Notion, Google Drive, HubSpot
- 첫 스킬 `project-brief` 완료 (parallel scout + synthesizer 패턴)
- 워크숍 = 문토 AI meetup 3시간 비개발자

**Anti-patterns:**
- 단순 ChatGPT 래퍼
- parallel scout + synthesizer 패턴 또 반복 (project-brief와 차별화)
- 5분+ 셋업 (시연용 후보 한정)
- 주관적 결과 (와우 검증 안 됨)

---

## 2. 제안 구조

```
story-toolkit/                              # 단일 plugin repo (Story Protocol private)
├── .claude-plugin/
│   ├── plugin.json                         # name, description, author (no version field — commit SHA)
│   └── marketplace.json                    # private marketplace manifest
├── skills/
│   ├── project-brief/                      # I1, I2 (already built — scout + synthesizer)
│   ├── pr-storybook/                       # I1 eng (hook cron daily)
│   ├── smart-contract-radar/               # I1 eng/security, I2 워크숍 클라이맥스 ★
│   ├── decision-log-keeper/                # I1 PM, I2 워크숍 보수 워밍업 픽 ★
│   ├── spec-from-thread/                   # I1 PM
│   ├── battle-card-builder/                # I1 sales
│   ├── deal-context-loader/                # I1 sales
│   └── ticket-triage-router/               # I1 CS, I2 워크숍 V+X 와우 픽 ★
├── agents/                                 # plugin-namespaced subagents
│   ├── synthesizer.md                      # project-brief가 사용 (재사용)
│   ├── slack-scout.md                      # project-brief가 사용
│   ├── notion-scout.md                     # project-brief가 사용
│   ├── static-pattern-auditor.md           # smart-contract-radar 도구 격리 #1
│   ├── cross-module-impact.md              # smart-contract-radar 도구 격리 #2
│   ├── ticket-classifier.md                # ticket-triage-router
│   └── ticket-draft-writer.md              # ticket-triage-router
├── hooks/
│   └── hooks.json                          # PostToolUse, cron-via-launchd 안내 등
├── commands/                               # 슬래시 단축 (선택)
│   └── (없음 — skill 자체가 /story-toolkit:<name>으로 호출됨)
├── .mcp.json                               # GitHub/Slack/Notion/GDrive/HubSpot 5종 정의
├── README.md                               # 사내 설치 가이드 (gh auth + /plugin install)
└── docs/
    ├── workshop-flow.md                    # 워크숍 시연 5분 시나리오
    └── mcp-setup.md                        # 사내 OAuth/PAT 분배 가이드 (관리자용)
```

**데이터 흐름 (시연):**
```
워크숍 참가자
  → /story-toolkit:decision-log-keeper --date 2026-04-26 (보수 워밍업 — H+X, 30초)
  → /story-toolkit:ticket-triage-router --message-url <slack-msg> (V+X 와우, 1분 subagent 격리)
  → /story-toolkit:smart-contract-radar --pr 42        (X 클라이맥스, PR write-back 클로즈드 루프)
```

**Skill→Subagent 토폴로지 패턴 (3종, 통일):**
- Pattern A — flat: skill 단독 (battle-card-builder, deal-context-loader, decision-log-keeper, spec-from-thread, pr-storybook)
- Pattern B — fan-out scout: skill → N개 정찰 subagent → synthesizer (project-brief 1건만)
- Pattern C — tool-isolated chain: skill → 도구 권한별 subagent 직렬 (smart-contract-radar 2개, ticket-triage-router 2개)

`release-radio` (4-subagent fan-out) 는 anti-pattern "scout 패턴 반복" 위반으로 제외. Pattern B는 project-brief 1건으로 한정 — anti-pattern 회피 + 첫 skill로 이미 검증된 패턴 재사용 최소화.

**호스트 환경 전제 (D5 — Section 6.5):**
- `~/.claude/settings.json` = [piplabs/claude-code-setup](https://github.com/piplabs/claude-code-setup) `personas/l1-auditor`(엔지/보안) 또는 `personas/developer`(비엔지) 템플릿 (사내 표준 baseline)
- `~/.claude/CLAUDE.md` = 같은 persona 글로벌 지침
- 위 둘이 plugin 외부에 선행 설치된 상태에서 `/plugin install story-toolkit`
- plugin은 baseline의 deny rule(60+) / sandbox / `rm -rf` · `git push main` PreToolUse hook을 재정의 X — plugin 고유 hook(예: PR comment write-back 사전 확인)만 `story-toolkit/hooks/hooks.json`에 추가

---

## 3. 의도→구조 매핑

| # | 내 원문 | 대응 구조 | 상태 |
|---|---|---|---|
| I1 | "사내에서 활용할 수 있는 skills, subagents를 플러그인화" | `story-toolkit/skills/*` 8종 + `agents/*` 7종 + `.mcp.json` | OK |
| I2 | "실습에서 와우가 나올 수 있도록 세팅" | `docs/workshop-flow.md` + 픽 3종 (decision-log-keeper / ticket-triage-router / smart-contract-radar) | OK |
| I3 | "전체 플러그인에 대한 설계도" | Section 2 디렉토리 트리 + Section 6 1-pagers + Section 7 구현 단계 | OK |
| I4 | "시연 추천 2-3개 + 시연 흐름" | Section 6.7 워크숍 픽 3종 + 5분 시나리오 (Section 7 Phase 3) | OK |

**합계:** 누락 0 / 왜곡 0 / 부분 0 / OK 4

---

## 4. 갈림길 (고려했으나 택하지 않음)

| # | 대안 | 기각 이유 |
|---|---|---|
| A1 | 직군별 plugin 분리 (eng-toolkit / pm-toolkit / sales-toolkit ...) | I3 "전체 플러그인" 단일 컨테이너 의도와 충돌. 9축 7(모듈화) 강화 + 신규 입사자 1직군 1 plugin 단순화 효과 있음. 다만 사내 13명 규모에서 cross-skill 변경(synthesizer agent 공유 등) 빈도가 더 큰 비용 — 5 repo PR 동기화 마찰 > 1 repo 디렉토리 관리 마찰 |
| A2 | release-radio (cross-부서 4-subagent fan-out) 워크숍 픽 채택 | anti-pattern "scout + synthesizer 반복" 정면 위반. 시연 4분+ 소요 (3시간 워크숍 페이스 깨짐). 9축 8(테스트 용이): subagent 4개 동시 실패 시 재현하려면 4 MCP trace 모두 보존 필요 — 워크숍에서 trace UI 보여줄 슬롯 없음 |
| A3 | Hook 없이 전부 명시 호출만 | 9축 4(확장성) 약화. PM의 decision-log-keeper, 엔지의 pr-storybook 같이 매일 자동 돌아야 가치 있는 후보가 죽음. I1 사내 효용 절반 손실 |
| A4 | 단일 거대 skill (router 패턴) — `/story` 하나로 전부 | I1 직군별 호출 명료성 파괴. 9축 7(모듈화) 0점. 9축 6(리팩토링) 위험 — 한 skill 수정 시 5개 직군 전부 영향 |

---

## 5. 보류 (내 결정 필요)

| # | 보류 항목 | 관련 의도 | 필요한 입력 |
|---|---|---|---|
| B1 | HubSpot/Slack/Notion MCP 도구 ID dry-run 결과 | I1, I2 | 사내 워크스페이스에서 `mcp__hubspot__*`, `mcp__slack__*`, `mcp__notion__*` 실제 노출 도구명 측정 후 1-pager의 "MCP tool" 필드 확정 |
| B2 | Plugin marketplace 호스팅 위치 | I1 | (a) Story 사내 GitHub Enterprise (b) GitHub.com private repo + PAT (c) GitLab self-hosted — 인프라 팀 결정 필요 |
| B3 | 워크숍 참가자 MCP 인증 분배 방식 | I2 | (a) 데모 sandbox 워크스페이스 미리 OAuth 발급 (b) 참가자 본인 워크스페이스 + PAT 사전 안내 (c) read-only 더미 데이터 + 모킹 — 셋 중 택일 |
| B4 | 첫 릴리스 후보 범위 (8개 전부 vs 1차 4개) | I1 | (a) 8개 한 번에 (b) 코어 4개 (project-brief, decision-log-keeper, ticket-triage-router, smart-contract-radar) 먼저 + 나머지 4개 분기별 추가 — 운영 팀 의사결정 |
| B5 | smart-contract-radar 워크숍 노출 범위 | I2 | Story 사내 컨트랙트 일부가 아직 비공개. 워크숍에선 (P1) 공개 컨트랙트 PR 사용 (P2) 더미 PR 미리 제작 — 둘 중 |
| B6 | piplabs/claude-code-setup persona 분배 정책 | I1 | (a) 엔지/보안 = `l1-auditor`, 그 외 = `developer` (b) 전원 `l1-auditor` (가장 엄격, geth/foundry deny 포함) (c) 전원 `developer` (l1 deny rule 누락) — 사내 보안팀 결정 필요 |

---

## 6. 기술 결정 및 9축 근거

### 6.1 결정 D1 — 단일 plugin 컨테이너 (vs 직군별 분리)

**선택:** 단일 `story-toolkit`
**대안:** 5개 plugin 분리 (Y), 모노레포 multi-plugin (Z)

| 축 | X (단일) | Y (5개 분리) | Z (모노레포 multi) |
|---|---|---|---|
| 1. 비용 효율 | 1 repo 호스팅, MCP credential 1세트 공유 | 5 repo, OAuth 5회 발급 | 1 repo, 5 manifest 동시 발행 |
| 2. 보안 | `.mcp.json` 1곳 — secret 누출면 전체 | 직군별 격리, 누출 범위 좁음 | 단일 secret 풀 + manifest 분리 |
| 3. 유지보수 | 1 PR로 cross-skill 변경 가능 | 5 repo PR — 변경 추적 분산 | 단일 history, manifest별 release |
| 4. 확장성 | 13→30 skill 시 디렉토리 비대화 | 신규 직군 = 신규 plugin 자연스러움 | 신규 plugin 추가 = 신규 manifest |
| 5. 수정 용이 | 1 clone, 1 install | clone 5회, install 5회 | 1 clone, install N회 |
| 6. 리팩토링 | shared agents (synthesizer) 자유 이동 | 직군 간 agent 공유 시 복제 필요 | shared agent 1곳, manifest 참조 |
| 7. 모듈화 | skill 폴더 단위 | plugin 단위 — 가장 강함 | manifest 단위 — Y와 X 중간 |
| 8. 테스트 용이 | E2E 1세트 | E2E 5세트 | E2E 1세트 + manifest별 |
| 9. 디버그 로깅 | 단일 plugin namespace `story-toolkit:*` | 5 namespace 추적 | manifest별 namespace |
| 추가축. MCP drift | 5 MCP 1개 도구명 변경 = plugin 1개 재배포로 13 skill 모두 영향 | drift 격리 — 1 plugin 영향 = 1 직군 한정 | 5 MCP 1개 변경 = plugin 1개 재배포, X와 동등 |

**근거:** I1 "전사용 단일 컨테이너" 명시 + Constraint "story-toolkit 플러그인 컨테이너". Y의 모듈화 이점은 사내 13명 규모에서 비용 대비 작음.
**연결 의도:** I1, I3
**참고:** Anthropic plugin docs (claude.ai/docs/plugins) — single plugin can hold N skills, accessed via `/<plugin>:<skill>`

### 6.2 결정 D2 — Subagent 토폴로지 3패턴 통일

**선택:** Pattern A (flat) / B (fan-out scout) / C (tool-isolated chain) — 3종으로 한정
**대안:** Pattern free-form 후보별 자유 (Y), 단일 패턴 강제 (Z)

| 축 | X (3패턴 통일) | Y (free-form) | Z (단일 강제) |
|---|---|---|---|
| 1. 비용 효율 | subagent context fork 횟수 예측 가능 | 후보별 비용 들쭉날쭉 | 단순 — 전체 subagent 0회 |
| 2. 보안 | tool 권한 매트릭스 3종만 검토 | N종 매트릭스 | 단일 — 권한 격리 불가 |
| 3. 유지보수 | 신규 후보 = 3패턴 중 매핑만 | 신규 후보 = 새 패턴 발명 가능 | 후보 다양성 막음 |
| 4. 확장성 | 30 skill까지 3패턴으로 흡수 | 패턴 폭주 | 단일 — 표현력 부족 |
| 5. 수정 용이 | 패턴 단위 일괄 변경 | 후보별 개별 변경 | 1곳 수정으로 끝 |
| 6. 리팩토링 | 3패턴 이전 가능성 명확 | 분류 체계 없음 | 리팩토링 필요 없음 |
| 7. 모듈화 | 패턴이 경계 명시 | 경계 모호 | 모듈화 약함 — 모든 skill 동일 |
| 8. 테스트 용이 | 패턴별 픽스처 3종 | 후보별 픽스처 N종 | 픽스처 1종 |
| 9. 디버그 로깅 | 패턴별 trace 표준화 가능 | 표준화 불가 | trace 표준화 자동 |
| 추가축. MCP drift | 패턴 A/B/C 표 1개로 MCP 결합 매트릭스 1눈에 확인 | 후보별 결합 — 13건 1:1 추적 필요 | drift 시 전체 N skill 영향 |

**근거:** Pattern A는 프롬프트 단순/와우 즉시, Pattern B는 데이터 합치기 가치 큼, Pattern C는 도구 권한 격리 가치(보안).

**Anti-pattern "scout 반복" 정량 정의 (사용자 원문 "차별화" qualifier 해석):**
- 위반 = subagent 4개 이상 fan-out scout (비대화) **AND/OR** 기존 Pattern B 후보(project-brief)와 차별점 0 (같은 MCP 조합 + 같은 산출물 + 같은 직군)
- Pattern B 1건 정당화:
  - **project-brief**: Slack + Notion 스카웃, 산출물 = 텍스트 브리프, 직군 = 전사 (이미 구현됨)
  → 사용자 실용성 검토(2026-04-27) 결과 추가 Pattern B 후보(repo-radar / blog-post-forge / social-recap / runbook-finder)는 토이 데모 비중 높음(GitHub Insights / Claude.ai 단독 대체 가능)으로 판단되어 8개 코어에서 제외. release-radio는 4-subagent 비대화로 위반.

**연결 의도:** I1, I2 (워크숍 시연 토폴로지 변화 보여주기)

### 6.3 결정 D3 — Hook + 명시 호출 hybrid

**선택:** Hook 자동 (cron + PostToolUse) + 명시 슬래시 호출 둘 다 지원
**대안:** 명시 호출만 (Y), Hook only (Z)

| 축 | X (hybrid) | Y (명시만) | Z (hook만) |
|---|---|---|---|
| 1. 비용 효율 | hook 호출 = 매일 N회 백그라운드 LLM | 호출 시점만 비용 | 매일 N회 (사용 안 해도) |
| 2. 보안 | hook PostToolUse가 자동 trigger — secret 노출 시점 trace 1차 + audit log 2차 grep 필요 | 호출 시점이 사용자 명령 1개 — secret 검토 grep 1회로 끝 | 자동 trigger = 사일런트 노출, trace 의존 |
| 3. 유지보수 | hook config 분리 (hooks.json) | 단순 | hook 깨지면 모든 skill 정지 |
| 4. 확장성 | 두 모드 동시 — 후보 다양성 흡수 | 자동화 후보 죽음 | on-demand 후보 죽음 |
| 5. 수정 용이 | hook 끄기 = hooks.json 한 줄 | hook 자체 없음 | hook config 의존 |
| 6. 리팩토링 | skill ↔ hook 결합 분리 | 결합 없음 | 결합 강함 |
| 7. 모듈화 | skill이 hook 무관하게 독립 호출 가능 | 명시 호출 단일 모드 | hook 없으면 호출 불가 |
| 8. 테스트 용이 | 명시 호출로 단위 테스트 가능 | 단위 테스트 직접 | hook 시뮬 필요 |
| 9. 디버그 로깅 | trigger source 로그 (manual/hook) | trigger 단일 | trigger 단일 (hook only) |
| 추가축. MCP drift | hook이 MCP 자동 호출 — drift 시 사일런트 실패 | 사용자가 즉시 실패 인지 | 사일런트 실패 위험 큼 |

**근거:** decision-log-keeper / pr-storybook 같은 매일 자동 후보는 hook 필수. 워크숍 시연은 명시 호출이 와우 명료. 둘 다 지원해야 I1+I2 동시 충족.

### 6.4 결정 D4 — 워크숍 픽 3종 + 보수 1건 원칙

**선택:** decision-log-keeper (보수 워밍업, H+X) → ticket-triage-router (V+X 와우, Pattern C) → smart-contract-radar (X 클라이맥스, Pattern C 클로즈드 루프)
**대안:** release-radio 포함 (Y), project-brief만 사용 (Z)
**선정 변경 이력:** 2026-04-27 사용자 실용성 검토 — 초안 픽(repo-radar/blog-post-forge/smart-contract-radar)에서 repo-radar(GitHub Insights 대체 가능, 토이 데모)·blog-post-forge(Claude.ai 단독 가능, MCP 합성 가치 입증 약함) 제외, 사내 ★★★ 효용 등급 후보(decision-log-keeper, ticket-triage-router) 승격. I1(사내 production skill) 우선, I2(워크숍 와우)는 선정된 ★★★ 픽이 H+X / V+X / X 와우 태그 그대로 충족.

| 축 | X (현재 픽) | Y (release-radio 포함) | Z (project-brief만) |
|---|---|---|---|
| 1. 비용 효율 | 시연 1건 30초~1분 | release-radio 1건 4분+ | 1 skill 반복 — 와우 단조 |
| 2. 보안 | sandbox MCP credential 1세트(GitHub PAT + Slack token + GDrive OAuth)로 시연 3건 모두 커버 | release-radio = 4 MCP 동시 인증 + write scope 추가 필요 | 단일 MCP credential |
| 3. 유지보수 | 패턴 A/B/C 1개씩 — 다양성 학습 | 패턴 B 비대화 학습 | 패턴 다양성 학습 X |
| 4. 확장성 | 워크숍 후 참가자가 패턴 A/B/C 중 자기 회사 케이스 1개 골라 1시간 내 적용 가능 | 4-subagent 자기 회사 적용 시 4 MCP credential + hook 4개 분배 = 비개발자 30분+ | 단일 패턴 — 적용 케이스 1종 한정 |
| 5. 수정 용이 | 시연 실패 시 1건씩 fallback | 4 subagent 중 1 실패 = 전체 망가짐 | fallback 없음 |
| 6. 리팩토링 | 픽 1건 교체 시 다른 2건 영향 X | release-radio 교체 시 cross-부서 narrative 깨짐 | 교체 불가 |
| 7. 모듈화 | 3 skill 독립 | release-radio 4 subagent 결합 | 단일 |
| 8. 테스트 용이 | 사전 리허설 3회 (각 5분, 시연 3종 1:1 매칭) | release-radio 리허설 = 4 MCP 모킹 + hook trigger 시뮬 = 회당 15분+ | 단일 리허설 5분 |
| 9. 디버그 로깅 | 각 시연 trace 명확 | trace 4갈래 — 워크숍에서 못 보여줌 | trace 단일 |
| 추가축. MCP drift | 1 시연당 1 MCP — drift 격리 | 4 MCP 중 1개만 깨져도 시연 실패 | drift 영향 단일 |
| 워크숍 와우 패턴 태그 | H(hook) + V(시각 산출) + X(cross-MCP) 균형 | X 4중 — 비대화 | H/V/X 결핍 |

본 9축 평가는 '픽 3종' 묶음 단위 평가다 — 후보별(decision-log-keeper / ticket-triage-router / smart-contract-radar)로 9축을 분리할 경우 7. 모듈화·8. 테스트 용이의 셀 값이 3건 모두 동일해지므로 묶음 표기로 정보 손실 없음, 개별 후보 9축은 6.6 1-pagers의 'MCP drift 위험'·'와우 패턴 태그' 필드가 대체한다.

**근거:** Bias 해소 — "전부 새것" 대신 보수 워밍업(decision-log-keeper, Pattern A flat + cron 자동 메타포) → V+X(ticket-triage-router, Pattern C subagent 2개 도구 격리 시각화) → X 클라이맥스(smart-contract-radar, Pattern C + GitHub PR write-back 클로즈드 루프) 점층 구조. 픽 3종이 사내 매일 실사용 가치(★★★ 등급)도 가져 I1+I2 동시 충족.
**연결 의도:** I2, I4

### 6.5 결정 D5 — 보안 베이스라인 = piplabs/claude-code-setup 상속 (자체 작성 X)

**선택:** 사내 `~/.claude/settings.json` + `~/.claude/CLAUDE.md` = [piplabs/claude-code-setup](https://github.com/piplabs/claude-code-setup) `personas/l1-auditor`(엔지·보안) 또는 `personas/developer`(비엔지) 템플릿을 그대로 사용. story-toolkit plugin의 `hooks/hooks.json`은 baseline을 재정의 X, plugin 고유 흐름(예: smart-contract-radar PR comment write-back 사전 확인)만 추가.
**대안:** 자체 작성 (Y), Anthropic 기본값 그대로 (Z)

**piplabs 베이스라인이 가져오는 것 (`l1-auditor` 기준):**
- `sandbox.enabled: true` — macOS Seatbelt / Linux bubblewrap OS-수준 격리
- `enableAllProjectMcpServers: false` — 프로젝트 `.mcp.json` 자동 활성화 차단 (compromised repo의 MCP 주입 방어)
- `permissions.deny` 60+ 룰 (요약):
  - 시스템 파괴: `Bash(rm -rf *)` `Bash(sudo *)` `Bash(mkfs *)` `Bash(dd *)` `Bash(curl *|bash*)` `Bash(git push --force*)` `Bash(git reset --hard*)`
  - L1 한정: `Bash(geth --unlock *)` `Bash(geth account import *)`
  - 자격증명: `Read(~/.ssh/**)` `Read(~/.gnupg/**)` `Read(~/.aws/**)` `Read(~/.azure/**)` `Read(~/.config/gh/**)` `Read(~/.git-credentials)` `Read(~/.docker/config.json)` `Read(~/.kube/**)` `Read(~/.npmrc)` `Read(~/.pypirc)` `Read(~/Library/Keychains/**)`
  - 지갑: `Read(~/Library/Application Support/**/metamask*/**)` + electrum / exodus / phantom / solflare
  - 프로젝트 secret: `Read(./.env*)` `Read(./.envrc)` `Read(./**/*.pem)` `Read(./**/*.key)` `Read(./**/*.p12)` `Read(./secrets/**)` `Read(./**/*credentials*.json)` `Read(./**/*apikey*)`
  - Web3: `Read(./**/keystore/**)` `Read(./.foundry/keystores/**)` `Read(./**/broadcast/**)` `Read(./**/*.mnemonic)` `Read(./**/wallet.json)` `Read(./**/private_key.txt)`
  - IaC: `Read(./**/*.tfvars)` `Read(./**/*.tfstate*)` `Read(./.terraform/**)`
  - K8s: `Read(./**/kubeconfig)` `Read(./**/*.kubeconfig)`
- `hooks.PreToolUse(Bash)` 2개: `rm -rf` 차단(trash 권유), `git push main/master` 차단(feature branch 강제)
- `hooks.PostToolUse(Edit)` 1개 (`l1-auditor` 한정): consensus path 편집 시 `warn-consensus-path.sh` 경고
- `env`: `DISABLE_TELEMETRY=1` `DISABLE_ERROR_REPORTING=1` `CLAUDE_CODE_DISABLE_FEEDBACK_SURVEY=1` `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`

**대안 평가 (9축 + drift):**

| 축 | X (piplabs 상속) | Y (자체 작성) | Z (기본값 그대로) |
|---|---|---|---|
| 1. 비용 효율 | 0줄 작성, fork 1개 | 60+ deny rule 검토·작성 비용 | 0줄, 사고 시 비용 무한 |
| 2. 보안 | 사내 보안팀 검증 deny set 60+ + sandbox + git push main 차단 | 첫 배포 누락 위험 (메타마스크/keystore/.env 중 1개 빠지면 노출) | 무방비 — `Read(./.env)` 자유, `Bash(rm -rf *)` 자유 |
| 3. 유지보수 | upstream 변경 사항 git pull로 흡수 | 위협 트렌드마다 수동 추가 | 보수 0 — 무방비 영구 |
| 4. 확장성 | 신규 직군 = 신규 persona 디렉터리 1개 | 직군별 deny set 5종 매번 분기 | 분기 없음 |
| 5. 수정 용이 | persona 템플릿 1줄 수정 후 사내 13명 재배포 | 60줄 중 1줄 수정 = 사내 PR 13건 분산 | 수정할 게 없음 |
| 6. 리팩토링 | persona 분리 = developer / l1-auditor 4종 이미 분리 | 단일 파일 비대화 | 리팩토링 대상 없음 |
| 7. 모듈화 | 사내 baseline ↔ plugin hooks 명확 분리 | 둘 섞임 — story-toolkit 변경이 baseline 변경 가능 | 단일 — 모듈화 0 |
| 8. 테스트 용이 | sandbox 켠 상태로 deny rule 즉시 검증 (`Read(./.env)` 시도 → 차단 확인) | 자체 검증 시나리오 직접 작성 | 검증 불가 |
| 9. 디버그 로깅 | hook 차단 시 명시 메시지 (`BLOCKED: Use trash instead of rm -rf`) | 자체 메시지 작성 | 로그 없음 (silent allow) |
| 추가축. MCP drift | `enableAllProjectMcpServers: false`로 .mcp.json 자동 노출 방어 — drift 시 plugin 1개만 재배포 | 같은 정책 자체 작성 시 누락 위험 | 모든 .mcp.json 자동 활성 — drift = 즉시 사고 |
| Story L1 적합성 | `l1-auditor` 페르소나가 geth keystore / foundry / consensus path 정확히 커버 | Story L1 컨텍스트 자체 학습 비용 | 컨텍스트 0 |

**근거 3가지:**
1. **PIP Labs = Story Protocol GitHub 조직.** 본 plugin이 Story 사내용이라는 점에서 사내 표준 baseline과 일치하지 않으면 plugin install 시점에 `enableAllProjectMcpServers: false`와 `.mcp.json` 정책이 충돌하거나 기존 deny rule과 plugin 동작이 어긋남. 같은 조직이 검증·운영 중인 설정을 재사용.
2. **`l1-auditor`의 geth/foundry/keystore deny rule은 smart-contract-radar 시연과 1:1.** 베이스라인이 `Read(./.foundry/keystores/**)` 차단을 보장하므로, smart-contract-radar의 `static-pattern-auditor` subagent가 read-only로 PR diff만 보고 keystore 절대 미접근 — D2 Pattern C(도구 격리)가 OS-수준에서도 강제됨.
3. **README "Adapted from Trail of Bits."** 외부 보안 컨설팅사가 검증한 설정의 fork → 자체 작성 대비 1차 근거(공개 git history) 보유. piplabs/claude-code-setup README는 sandboxing/hooks/MCP/skills 운영 가이드도 포함.

**plugin 레벨에서 추가만 하는 hooks (`story-toolkit/hooks/hooks.json`, baseline 미정의 영역만):**
- `PreToolUse(mcp__github__create_pull_request_comment)` — smart-contract-radar PR comment write-back 직전 사용자 확인 1회 (baseline엔 GitHub MCP write hook 없음)
- `PreToolUse(mcp__notion__create_page)` — decision-log-keeper의 Notion DB row 추가 직전 사용자 확인 1회 (cron 모드에서는 skip 옵션 + audit log) (baseline엔 Notion MCP write hook 없음)
- baseline의 `rm -rf` / `git push main` 차단 hook은 재정의 X — 이미 동작

**연결 의도:** I1 (사내 plugin이 사내 보안 표준에 정렬), I3 (설계도가 환경 전제 명시)
**참고:**
- piplabs/claude-code-setup — https://github.com/piplabs/claude-code-setup (`personas/l1-auditor/settings.json`, `personas/developer/settings.json`, README "Sandboxing" / "Hooks" 섹션)
- Trail of Bits 원본 — https://github.com/trailofbits/claude-code-config

### 6.6 8개 채택 후보 1-pagers + 1개 제외 분석

각 후보: 이름 / 한 줄 요약 / 직군 / MCP / Topology Pattern / Tool 권한 / 와우 패턴 태그(H=Hook, V=Visual, X=Cross-MCP) / MCP drift 위험(L/M/H) / 사내 효용 / 워크숍 픽 여부
Drift 위험 등급 기준: L = 단일 MCP 도구 1개 사용·write 없음 / M = 2개 MCP 또는 write 1건 / H = HubSpot처럼 서드파티 GA 6개월 내 + 다중 호출

#### 6.6.1 project-brief ★ (이미 구현)
- **시나리오:** 월요일 아침 PM이 "지난주 X 프로젝트 어디까지 갔지?" 묻는 상황. Slack 채널 4-5개와 Notion 스펙/회의록 페이지 5-10개를 직접 훑는 데 30-45분. 신규 입사자가 합류한 첫 주 컨텍스트 따라잡기에서도 동일한 페인.
- **동작 흐름:** 프로젝트명 입력 → (1) slack-scout subagent가 채널명·태그·키워드 매칭으로 관련 채널 N개 식별 + 최근 7일 메시지 검색 → (2) notion-scout subagent가 페이지·DB 검색 + 마일스톤·결정·미해결 체크리스트 추출 → (3) synthesizer가 두 정찰 결과를 병합해 한 페이지(요약·진척·결정·미해결·다음 액션) 마크다운 브리프 출력.
- **왜 이 스킬인가:** Slack 검색만 / Notion 검색만 단독으로는 "현황"이 안 보임 — 결정은 Slack에 있고 진척은 Notion에 있다. 두 출처를 LLM이 교차해 "Slack에서 결정됐는데 Notion에 반영 안 된 항목" 같은 갭을 자동 식별하는 게 단독 검색 대비 차별점. 사내 13명 전 직군이 매주 1회+ 사용하는 보편 케이스.
- **요약:** 프로젝트명 입력 → Slack 최근 활동 + Notion 진행 상황 한 페이지 브리프
- **직군:** 전사 공통
- **MCP:** Slack, Notion
- **Topology:** Pattern B (skill → slack-scout + notion-scout 병렬 → synthesizer)
- **Tool 권한:** mcp__slack__list_channels / search_messages / get_channel_history / get_user_info, mcp__notion__search / fetch
- **와우 태그:** X (Slack+Notion 교차)
- **Drift 위험:** L (Slack/Notion MCP 모두 GA, 도구명 안정)
- **사내 효용:** 매주, 모든 직군
- **워크숍 픽:** ○ 시연 가능하나 D4 픽에선 제외 (이미 알려진 패턴, 와우 약함)

#### 6.6.2 pr-storybook
- **시나리오:** 비-엔지(PM/마케/세일즈)가 "오늘 엔지가 뭘 만들었는지" 따라가려면 GitHub 수십 건 PR 제목/디프를 직접 읽어야 한다. PR 제목은 "fix: edge case in foo"처럼 영어 + 내부 코드 식별자로만 작성되어 비-엔지가 1건당 1-2분 해석 시간 필요. 그래서 매일 누군가가 수동으로 "오늘의 변경 요약" Slack 포스팅 → 작성 부담으로 결국 안 함.
- **동작 흐름:** launchd cron 매일 18:00 → (1) `mcp__github__list_pull_requests state=closed merged_at >= today` 호출 → (2) skill 본문이 PR 제목/본문/디프 통계를 받아 비-엔지 가독성 위주 한 단락 스토리(예: "오늘 토큰 라이센스 만료 처리 버그 1건 수정 + Vault 신규 endpoint 1개 추가")로 LLM 합성 → (3) `mcp__slack__post_message` write로 #eng-changelog 자동 포스팅.
- **왜 이 스킬인가:** GitHub Slack 공식 통합도 PR-merge 알림을 보내지만 제목·작성자만 노출되고 비-엔지 가독 요약이 없음. 사내 13명 중 비-엔지 9명이 "엔지가 오늘 뭐 했는지"를 알 수 있게 만드는 게 가치. 매일 자동 = hook 없으면 죽는 후보 (D3 hybrid의 H 모드 핵심 사례).
- **요약:** 매일 18:00 cron — 그날 머지된 PR을 비-엔지 가독 한 단락 스토리로 #eng-changelog Slack 자동 포스팅
- **직군:** 엔지니어 (작성 자동화) + 비-엔지 (소비)
- **MCP:** GitHub, Slack
- **Topology:** Pattern A (flat skill, hook trigger)
- **Tool 권한:** mcp__github__list_pull_requests + state=closed, mcp__slack__post_message (write 권한 필요)
- **와우 태그:** H (hook 자동)
- **Drift 위험:** M (Slack write scope `chat:write` 별도 분배 필요)
- **사내 효용:** 매일, 엔지+PM
- **워크숍 픽:** X (write 권한 워크숍 sandbox 부담)

#### 6.6.3 smart-contract-radar ★ (워크숍 픽 #3 — 클라이맥스)
- **시나리오:** Story Protocol L1 코어 컨트랙트 PR이 올라오면 시니어 audit 1명이 (a) 자주 보는 패턴(reentrancy, unchecked external call, storage collision) 스캔 + (b) 변경 모듈이 호출되는 다른 컨트랙트 영향 분석을 수동으로 회당 30-60분 한다. 신규 PR 빈도가 주 5-10건 → audit 1명이 병목, 1차 패턴 스캔 정도는 자동화 후 사람은 의미 있는 트레이드오프 검토에만 집중.
- **동작 흐름:** PR 번호 입력 → (1) static-pattern-auditor subagent 실행: 도구는 `mcp__github__get_pull_request_files / get_pull_request_diff` (read-only)만 — 변경된 .sol 파일에서 알려진 안티패턴 N종(reentrancy 가드 누락, 외부 호출 후 state mutation, low-level call 결과 무시 등) 패턴 매칭으로 의심 위치 보고 → (2) cross-module-impact subagent 실행: 도구는 `mcp__github__list_commits + Bash grep` (sandboxed) — 변경된 함수/이벤트 시그니처를 다른 컨트랙트 파일에서 grep으로 호출처 역추적, 영향 모듈 리스트 작성 → (3) skill 본문이 두 결과 합쳐 리스크 리포트 마크다운 작성 → (4) `mcp__github__create_pull_request_comment` (write)로 PR에 자동 게시. PreToolUse hook이 사용자 확인 1회.
- **왜 이 스킬인가:** GitHub Copilot/Claude.ai 단독은 "PR diff 보여주고 검토해 달라" 1턴 호출 — 패턴 스캐닝 단계와 cross-module 영향 단계가 한 컨텍스트에 섞여서 도구 권한 분리 불가. Pattern C로 두 단계를 도구 권한별로 격리하면: 스캐닝 단계는 PR 파일만 read, 영향 분석 단계는 list_commits만 read — 한쪽이 망가져도 다른 쪽 결과 살아 있음 + audit log에서 어느 단계가 어디까지 봤는지 추적 가능. write 권한은 마지막 합성 단계에만 부여. Story L1 음의 자산 가치(audit 1명 시간) × PR 빈도 = 명확한 ROI.
- **직군:** 엔지니어 / 보안
- **MCP:** GitHub
- **Topology:** Pattern C (skill → static-pattern-auditor [read-only file tools] → cross-module-impact [grep + git log tools] → 직렬 합성)
- **Tool 권한:** static-pattern-auditor = mcp__github__get_pull_request_files / get_pull_request_diff (read-only). cross-module-impact = mcp__github__list_commits + Bash grep (sandboxed). 최종 mcp__github__create_pull_request_comment (write).
- **Trigger 메커니즘 (3옵션):**
  - (T1) 워크숍 시연용: 명시 슬래시 호출 `/story-toolkit:smart-contract-radar --pr 42` — Claude Code 내부, hook 무관
  - (T2) 사내 자동화 옵션 1: GitHub Actions workflow `pull_request: opened` 이벤트 → runner에서 `claude --print "/story-toolkit:smart-contract-radar --pr <PR_NUMBER>"` 외부 호출 [`--skill` 플래그 불존재 — 슬래시 명령은 `--print` 인자로 전달, Section 8 Q2 참조]
  - (T3) 사내 자동화 옵션 2: 별도 webhook receiver 서버에서 Claude SDK 호출
  - **주의:** Claude Code 자체 hook 시스템(PostToolUse/UserPromptSubmit 등)은 외부 GitHub PR open 이벤트를 받을 수 없음. (T2)/(T3)는 plugin 외부 인프라.
- **와우 태그:** X (GitHub PR write-back 클로즈드 루프) + (사내 운영 시) H
- **Drift 위험:** M (GitHub MCP write scope `pull_requests:write` 필요, GitHub Actions runner는 plugin 외부)
- **사내 효용:** 매주 (PR 빈도), 엔지/보안
- **워크숍 픽:** ★ 클라이맥스 — 옵션 (T1) 사용. 진행자가 Claude 안에서 명시 호출 → Pattern C subagent 격리 + PR comment 자동 게시 시연. 사내 적용 시 옵션 (T2) 권장 안내.

#### 6.6.4 decision-log-keeper ★ (워크숍 픽 #1 — 보수 워밍업)
- **시나리오:** PM 회의에서 "토큰 vesting 6개월 → 12개월로 변경" 같은 결정이 Slack #decisions 채널에 던져진 뒤 누가 정리하지 않으면 일주일 안에 묻힌다. 분기 회고 시점에 "그때 왜 이렇게 결정했지?"를 추적할 출처가 없어 Notion에 다시 손으로 옮겨야 함 — 결국 안 함 → 결정 이력 단절.
- **동작 흐름:** launchd cron 매일 18:00 → (1) `mcp__slack__get_channel_history` 호출로 그날 #decisions 채널 메시지 N건 수집 → (2) skill 본문이 LLM으로 결정성 발화만 추출(질문/잡담 필터링) + 각 결정에 작성자·시각·맥락 1줄 메타 부착 → (3) `mcp__notion__create_page` write로 Notion "Decision Log" DB에 row N개 추가 (제목·날짜·작성자·맥락·Slack 영구 링크 컬럼).
- **왜 이 스킬인가:** Slack 검색만으로는 분기 후 90일 retention 정책 또는 메시지 깊이로 사실상 사라짐. 사내 위키/Confluence 결정 페이지 수동 작성은 작성 부담으로 결국 안 함이 default. cron + LLM 추출 + Notion DB write 자동화로 "작성 부담 0 + 영구 보존 + 검색 가능"의 3박자 — 단독 도구로는 안 됨. 워크숍 보수 워밍업으로 적합한 이유: 도구 단순(Slack read + Notion write 2개), Pattern A flat, 시연 30초로 짧고 효과 명확.
- **요약:** 매일 18:00 cron — 그날 Slack #decisions 채널 결정사항 추출 → Notion DB row 자동 추가
- **직군:** PM
- **MCP:** Slack, Notion
- **Topology:** Pattern A (flat, hook cron)
- **Tool 권한:** mcp__slack__get_channel_history, mcp__notion__create_page (write)
- **와우 태그:** H + X
- **Drift 위험:** M (Notion DB schema drift 시 create_page 실패)
- **사내 효용:** 매일, PM 전부
- **워크숍 픽:** ★ 보수 워밍업 — 사전 Notion DB 1회 셋업 + #decisions 더미 메시지 1-2건 시드 후, 워크숍에선 `--date` 명시 호출(시연용 T1 패턴, cron 미발화) 30초 시연. 핵심 메시지: "매일 18:00 cron이 그날 결정 자동 보존"

#### 6.6.5 spec-from-thread
- **시나리오:** PM이 신규 기능 논의를 Slack thread에서 진행하면 결정·요구사항·오픈 질문이 50-100개 메시지에 흩어진다. 이를 Notion 스펙 페이지로 옮기려면 thread 처음부터 끝까지 다시 읽고 카테고리별로 손으로 정리하는 데 1-2시간 — PM 1명당 주 2-3건 발생. 결국 스펙 페이지가 thread보다 늦거나 누락되어 엔지가 thread를 직접 읽어야 함.
- **동작 흐름:** Slack thread URL 입력 → (1) `mcp__slack__get_channel_history` + thread_ts 파라미터로 thread 전체 메시지 수집 → (2) skill 본문이 LLM으로 메시지를 [결정 / 요구사항 / 오픈 질문 / 잡담] 4분류 → (3) 결정/요구/질문만 골라 스펙 템플릿(목적·결정사항·요구사항·오픈 이슈·다음 액션) 마크다운 작성 → (4) `mcp__notion__create_page` write로 지정 부모 페이지 아래 신규 스펙 페이지 생성 + Slack thread 영구 링크 자동 첨부.
- **왜 이 스킬인가:** Slack 자체 "thread 요약" 기능은 일반 요약일 뿐 [결정/요구/질문] 분류 + Notion 페이지 작성까지 자동화 안 됨. ChatGPT 단독 호출은 thread 메시지를 손으로 복사해 넣어야 함 — 결국 작업량 동일. URL 1개 입력으로 fetch + 분류 + Notion 작성 3단계가 한 명령으로 끝나는 게 가치. PM이 thread 종료 직후 1초 호출 → 회의 끝과 동시에 스펙 초안 존재.
- **요약:** Slack thread URL 입력 → 결정/요구/오픈 질문 추출 → Notion 스펙 페이지 초안 생성
- **직군:** PM
- **MCP:** Slack, Notion
- **Topology:** Pattern A
- **Tool 권한:** mcp__slack__search_messages, mcp__slack__get_channel_history, mcp__notion__create_page
- **와우 태그:** X
- **Drift 위험:** M
- **사내 효용:** 매주, PM
- **워크숍 픽:** X

#### 6.6.6 battle-card-builder
- **시나리오:** 세일즈가 신규 lead와 첫 미팅 직전 "이 회사 우리 경쟁사 어디랑 비교 중인가?"를 따라잡아야 한다. HubSpot에 들어 있는 과거 동일 경쟁사 deal 메모(승/패 사유) + Notion에 PM이 정리해 둔 경쟁사 비교 페이지를 각각 검색해 미팅 노트로 합치는 데 30-60분. 미팅 빈도가 주 5-10건 → 매주 5시간 단순 검색·복붙으로 소진.
- **동작 흐름:** 경쟁사명 입력 → (1) `mcp__hubspot__search_companies / get_company`로 그 경쟁사가 등장한 과거 deal 메모·승패 결과·딜 액수 수집 → (2) `mcp__notion__search`로 Notion 내 그 경쟁사 비교 페이지·기능 차이 문서 fetch → (3) skill 본문이 두 출처 합쳐 1페이지 battle card(우리 강점 3 / 그 경쟁사 강점 3 / 과거 승패 사유 / 추천 화법) 마크다운 작성 → (4) 출력만, write 없음 (세일즈가 미팅 노트로 직접 복사).
- **왜 이 스킬인가:** HubSpot 검색 단독은 과거 deal 결과만 — "기능 비교" 정보는 Notion에. Notion 검색 단독은 PM이 작성한 정태적 비교 — "우리가 그 경쟁사한테 실제 어떻게 졌는지"는 HubSpot에. 두 출처를 LLM이 교차해 "이 경쟁사한테 가격 사유로 2회 졌음 → 가격 화법 준비" 같은 화법 추천까지 한 번에 합성하는 게 단독 검색 대비 차별점. Drift H 위험으로 워크숍은 제외하지만 사내 분기 운영 가치는 명확.
- **요약:** 경쟁사명 입력 → HubSpot CRM 기록 + Notion 비교 페이지 → 1페이지 battle card
- **직군:** 세일즈
- **MCP:** HubSpot, Notion
- **Topology:** Pattern A
- **Tool 권한:** mcp__hubspot__* (search_companies / get_company), mcp__notion__search (B1 dry-run으로 도구명 확정)
- **와우 태그:** V
- **Drift 위험:** H (HubSpot MCP GA 신생 — 2026-04-13, 도구명 변경 가능성)
- **사내 효용:** 분기, 세일즈
- **워크숍 픽:** X (HubSpot 사내 sandbox 부재)

#### 6.6.7 deal-context-loader
- **시나리오:** 세일즈가 진행 중 딜 30-50건을 동시에 들고 있을 때 "오후 3시 콜 5분 전, A사 딜 어디까지 갔지?"를 떠올려야 한다. HubSpot deal 페이지의 활동 로그 + 사내 Slack에서 그 회사명이 언급된 최근 메시지(엔지·PM이 fit 검토하며 남긴 의견)가 흩어져 있어 콜 직전 컨텍스트 따라잡기에 매번 5-10분 — 콜 자체보다 준비가 더 길어지는 비효율.
- **동작 흐름:** 딜 ID(또는 회사명) 입력 → (1) `mcp__hubspot__get_deal / get_contact`으로 deal 단계·소유자·최근 활동·연관 컨택트 수집 → (2) `mcp__slack__search_messages`로 그 회사명·컨택트 이름이 멘션된 최근 14일 사내 채널 메시지 검색 → (3) skill 본문이 HubSpot 정량(스테이지·액수·last activity 일자) + Slack 정성(엔지/PM 의견·우려·추가 정보) 합쳐 5분 브리프(딜 한줄 상태·고객 컨택트 N명·최근 활동·사내 의견·다음 액션 추천) 출력.
- **왜 이 스킬인가:** HubSpot 단독은 공식 활동 로그만 — 사내에서 누가 그 딜을 어떻게 보고 있는지 정성 정보는 Slack에. Slack 검색 단독은 정량 컨텍스트(스테이지·액수) 부재로 "그래서 어디까지 갔지?"가 안 보임. 두 출처를 5분 안에 합성하는 게 콜 직전 가치. HubSpot drift H 때문에 워크숍 픽 제외이나 사내 매주 운영 가치는 battle-card보다 더 빈번(주 1회+ vs 분기 1회).
- **요약:** 딜 ID 입력 → HubSpot 활동 + Slack 멘션 → 5분 상담 직전 컨텍스트 브리프
- **직군:** 세일즈
- **MCP:** HubSpot, Slack
- **Topology:** Pattern A
- **Tool 권한:** mcp__hubspot__* (deal/contact), mcp__slack__search_messages
- **와우 태그:** X
- **Drift 위험:** H (HubSpot drift)
- **사내 효용:** 매주, 세일즈
- **워크숍 픽:** X

#### 6.6.8 ticket-triage-router ★ (워크숍 픽 #2 — V+X 와우)
- **시나리오:** CS 팀이 Slack #support 채널에 들어오는 외부 사용자 메시지를 [버그/기능 요청/사용법 문의] 분류 → 각각 GitHub issue 검색 또는 Notion FAQ 검색 → 답변 초안 작성을 매일 20-50건 처리. 각 메시지당 5-10분, 전체 1.5-3시간/일. 분류·검색 단순 노동에 시간을 다 쓰고 정작 어려운 케이스에 집중을 못 함.
- **동작 흐름:** Slack 메시지 URL 입력 → (1) ticket-classifier subagent 실행 (도구 = `mcp__slack__get_message` read-only): 메시지 본문 fetch + LLM 분류 → [bug / feature_request / usage_question] 카테고리 출력 → (2) ticket-draft-writer subagent 실행 (도구 = `mcp__github__search_issues` + `mcp__notion__search` read-only, write 도구 부재): 카테고리에 따라 bug면 GitHub 기존 issue 중복 검색, usage면 Notion FAQ 검색, feature면 Notion 로드맵 검색 → 검색 결과 합성한 답변 초안 출력 → (3) skill 본문이 분류 + 초안을 사람 검토용으로 출력. 자동 게시 X.
- **왜 이 스킬인가:** Zendesk·Intercom 자동 분류 도구는 사내 GitHub/Notion 지식과 분리됨 — 답변에 "GitHub issue #123 같은 케이스" 인용을 자동 생성 못 함. 단일 LLM 1턴 호출은 도구 권한 분리 불가 → classifier 단계가 잘못 분류해도 검증 어렵고 audit log에서 어느 단계 실수인지 추적 불가. Pattern C로 분류 단계와 답변 작성 단계를 도구 권한별 격리 → 분류만 잘못됐는지 / 검색이 빈약했는지를 실행 trace에서 분리 가능. write 도구 0개 정책으로 워크숍 시연도 부담 없음. CS 매일 가장 빈번 + 분류 시간 절약 직접 ROI.
- **요약:** Slack #support 메시지 URL → ticket-classifier subagent 분류 → 카테고리별 GitHub issue 또는 Notion FAQ 검색 → ticket-draft-writer subagent 답변 초안
- **직군:** CS
- **MCP:** GitHub, Slack, Notion
- **Topology:** Pattern C (classifier → draft-writer 직렬, 도구 권한 격리)
- **Tool 권한:** classifier = mcp__slack__get_message + 분류 LLM. draft-writer = mcp__github__search_issues + mcp__notion__search (read-only). 답변 자동 게시 X — 초안만.
- **와우 태그:** V (subagent 격리 시각화 + 초안 비교) + X (3 MCP read-only)
- **Drift 위험:** M
- **사내 효용:** 매일, CS
- **워크숍 픽:** ★ V+X 와우 — 사전 더미 #support 채널 + 메시지 1건 시드 후 워크숍에선 message URL 명시 호출(T1) 1분 시연. 포인트: "subagent 2개가 도구 권한 분리해서 답변 초안"

#### 6.6.9 release-radio (제외 분석 — 채택 X)
- **시나리오:** 매주 금요일 전사 공지 작성을 누군가 1명이 수동으로 한다 — GitHub releases 4-5건 / Slack 결정 10-15건 / Notion 일정 변경 / HubSpot 신규 딜 합쳐 1페이지 공지로 합성. 작성자 1명에게 주 2-3시간 부담 + 정보 누락 빈번. "전사 한 화면" 자동화는 직관적으로 매력적 후보였음.
- **동작 흐름 (이론):** 매주 금 17:00 cron → 4 부서 scout subagent(github-scout / slack-scout / notion-scout / hubspot-scout) 병렬 실행 → 각자 그 주 데이터 수집 → synthesizer subagent가 4 결과를 합쳐 부서별 1단락씩 한 페이지 공지 → Slack #all-hands 채널에 자동 포스팅.
- **왜 제외인가:** 4가지 사유 (아래 상세). 핵심 1줄: "subagent 4 fan-out + 4 MCP 동시 의존"이 anti-pattern 정량 위반(D2)이며 워크숍 5분 시연 슬롯·sandbox 자원·drift 위험을 단독으로 모두 위반. 같은 가치(전사 한 화면)를 project-brief 변형 또는 단순 cron Slack post로 달성 가능 — 이 스킬의 복잡도가 가치 대비 정당화 안 됨.
- **요약:** 매주 금요일 17:00 cron — 그 주 GitHub release + Slack 결정 + Notion 일정 + HubSpot 딜 4 부서 데이터 수집 → 부서별 4-subagent 병렬 처리 → 단일 부서 전사 공지 1건 생성
- **직군:** 전사 (cross)
- **MCP:** GitHub + Slack + Notion + HubSpot (4종)
- **Topology:** Pattern B 비대화 (skill → 4 부서 scout subagent 병렬 → synthesizer)
- **Tool 권한:** mcp__github__list_releases, mcp__slack__search_messages, mcp__notion__search, mcp__hubspot__search_deals (read 전부)
- **와우 태그:** H + X (cron 자동 + 4 MCP cross)
- **Drift 위험:** H (HubSpot drift + 4 MCP 동시 의존)
- **사내 효용:** 매주, 전사
- **워크숍 픽:** ✗ 제외
- **제외 사유 (4가지):**
  1. anti-pattern 정량 위반 (D2): subagent 4개 fan-out — Section 1 anti-pattern "parallel scout + synthesizer 패턴 또 반복 (project-brief와 차별화)" 정면 충돌. OR 조건 중 'subagent 4개 이상 fan-out scout (비대화)' 절 단독 위반.
  2. 4 MCP credential 동시 인증 — 워크숍 sandbox 부담 (Phase 1.5a 1주 + HubSpot dry-run 미완)
  3. 시연 4분+ (워크숍 5분 슬롯 초과)
  4. drift H — 4 MCP 중 1개 도구명 변경 시 시연 실패 확률 4배

### 6.7 워크숍 픽 5분 시연 흐름

```
[T+0:00] 진행자 — "Story Protocol 사내에서 실제 쓰는 plugin입니다"
         /plugin install story-toolkit  (사전 설치 — 스킵)

[T+0:30] 시연 1 (보수 워밍업, Pattern A, H+X 와우)
         (사전: Notion DB 1회 셋업 + #decisions 채널 더미 메시지 1-2건 시드)
         > /story-toolkit:decision-log-keeper --date 2026-04-26
         → 30초 후 그날 #decisions 결정 N건 → Notion DB에 N row 자동 추가
         → 진행자 Notion UI 새로고침해서 row 노출 확인
         포인트: "원래는 매일 18:00 cron이 자동. 워크숍에선 --date로 그 명령 직접 한 번 돌려봤음"

[T+1:30] 시연 2 (V+X 와우, Pattern C, 도구 권한 격리 시각화)
         (사전: 더미 #support 채널 + 메시지 1건 시드)
         > /story-toolkit:ticket-triage-router --message-url <slack-msg-url>
         → ticket-classifier subagent (도구 격리 = Slack read-only) → 카테고리 분류 결과
         → ticket-draft-writer subagent (도구 격리 = GitHub+Notion search read-only) → 답변 초안 출력
         포인트: "subagent 2개가 각자 다른 MCP read 권한만 — 자동 게시 X, 초안만"

[T+2:30] 발표자 슬라이드 전환 + Q 1차 (참가자 질문 1개 받기, MCP 인증 점검 등) — 실제 빈 시간 아님

[T+3:30] 시연 3 (X 클라이맥스, Pattern C, 명시 호출 → PR write-back 클로즈드 루프)
         (사전: 사내 더미 PR 1건 미리 open 상태로 준비)
         > /story-toolkit:smart-contract-radar --pr 42
         → static-pattern-auditor subagent 실행 (도구 격리 = read-only PR diff/files)
         → cross-module-impact subagent 실행 (도구 격리 = list_commits + grep)
         → PR comment 자동 게시 ("3건 리스크 발견 + 영향 모듈 5개")
         → 진행자 GitHub UI 새로고침해서 comment 노출 확인
         포인트: "명령 1개 → subagent 2개가 각자 다른 도구로 분석 → GitHub로 다시 써넣었다 — 클로즈드 루프"
         사내 운영 시: GitHub Actions `pull_request: opened`에서 같은 명령 외부 호출 가능 (워크숍 후 README 안내)

[T+5:00] 마무리 — "이 plugin 그대로 가져가면 여러분 회사도 됩니다. README의 /plugin install만"
```

※ 총 시연 시간 ±30% 허용 (실측 3.5-6.5분, 리허설 3회 평균 기준). 각 슬롯은 리허설 결과에 따라 조정.
※ 픽 3종 모두 Pattern A 또는 C — Pattern B(fan-out scout)는 워크숍에 노출하지 않음 (project-brief에서 이미 시연된 패턴, 와우 중복 회피).

---

## 7. 구현 단계

### Phase 1: Plugin 골격 (3일, S)
- [ ] `story-toolkit/` repo 생성 (Story 사내 GitHub) — S (1h)
- [ ] `.claude-plugin/plugin.json` + `marketplace.json` 작성 — S (2h)
- [ ] `README.md` 사내 설치 가이드 (`gh auth login` + `/plugin install`) — S (2h)
- [ ] `project-brief` 기존 파일 이식 + Slack/Notion MCP만 우선 셋업 — S (4h)
- [ ] 사내 baseline 설치 안내 추가 — `git clone piplabs/claude-code-setup` → B6 결정된 persona 선택 → `cp personas/<persona>/{settings.json,CLAUDE.md} ~/.claude/` (D5) — S (1h)
- [ ] plugin top-level hooks/hooks.json + .mcp.json 분리 패턴 실측 — agent frontmatter의 hooks/mcpServers/permissionMode 필드가 무시되는지 1회 install로 확인 (Q4 답) — S (반나절)
- **Milestone:** 빈 plugin install 후 `/story-toolkit:project-brief` 동작 확인 + `~/.claude/settings.json`이 piplabs baseline으로 교체된 환경에서 plugin 동작 확인

### Phase 1.5a: MCP 셋업 + dry-run + B6 결정 (1주, M) ← Phase 2 prerequisite
- [ ] GitHub PAT 발급 + `.mcp.json` GitHub 등록 — S (1h)
- [ ] Slack OAuth + write scope (`chat:write`) admin 결재 + `.mcp.json` 등록 — M (1-2일, admin 의존)
- [ ] Notion 통합 등록 + 페이지 권한 부여 — S (반나절)
- [ ] Google Drive Cloud Console 앱 등록 + 사내 도메인 검증 — M (1-2일)
- [ ] HubSpot MCP 1차 셋업 + 도구 ID dry-run (B1 보류 해소) — M (1일)
- [ ] B6 결정 — persona 분배 정책: 엔지니어/감사 = `l1-auditor`, 비엔지(PM/마케/세일즈/CS) = `developer`, 또는 단일 persona 통일 (D5/B6) — S (반나절, 보안팀 협의)
- **Milestone:** 5종 MCP 모두 Claude Code `/mcp` 명령에서 도구 노출 확인 + B1/B6 보류 해소

### Phase 1.5b: 외부 trigger 인프라 (별도 트랙, M) ← Phase 4 prerequisite
- [ ] launchd 등록 정책 확보 + pr-storybook / decision-log-keeper cron 스케줄 등록 — M (1일, IT 정책 의존)
- [ ] Slack Events API receiver PoC (ticket-triage-router 의존) — M (2일)
- **Milestone:** launchd 등록 정책 통과 + Slack Events API receiver 1건 PoC

### Phase 2: 워크숍 픽 3종 (1주, M)
- [ ] `decision-log-keeper` skill (Pattern A, Slack read + Notion write) — M (2일)
- [ ] `ticket-triage-router` skill + 2 subagent (ticket-classifier, ticket-draft-writer) — M (2-3일, 명시 호출 시연용으로 hook 미발화 모드 우선 구현)
- [ ] `smart-contract-radar` skill + 2 subagent (static-pattern-auditor, cross-module-impact) — M (2일, hook 인프라 무관 — 명시 호출 only)
- [ ] 워크숍용 더미 PR + 더미 #decisions/#support Slack 채널 + Notion DB 1회 시드 + sandbox MCP credential (B3 결정 후) — S (반나절)
- **Milestone:** 5분 시연 흐름 리허설 3회 통과 (B5 결정 후)

### Phase 3: 워크숍 시연 (1일)
- [ ] 사전 sandbox 인증 분배 (B3 결정) — S
- [ ] 시연 5분 + Q&A — S
- **Milestone:** 참가자 본인 PC에서 동일 명령 1건 이상 재현 = 1회 호출 성공

### Phase 4: 나머지 5개 후보 (3주, L)
- [ ] B4 결정 후 1차(코어 4개 정의에서 빠진 잔여) — M (각 2-3일)
- [ ] hook 의존 후보(pr-storybook, decision-log-keeper의 cron 모드 발화)는 Phase 1.5b의 launchd 등록 정책 통과 후 — M
- [ ] HubSpot 의존 후보(battle-card-builder, deal-context-loader)는 B1 dry-run 통과 후 — M
- [ ] 나머지(pr-storybook, spec-from-thread) 분기별 — L
- **Milestone:** 사내 8개 skill 모두 1회 이상 정상 호출 + 직군별 핵심 1건 1주 사용 통과 (= "Active" 정의)

---

## 8. 열린 질문

1. **MCP tool ID 실제 노출** — Matters because B1 dry-run 결과에 따라 1-pagers의 "Tool 권한" 필드가 바뀌고, 일부 후보는 도구 부재 시 Pattern A→B 변경 필요. Ask: 사내 워크스페이스에 MCP 5종 1차 셋업 후 Claude Code `/mcp` 명령으로 노출 도구 dump.
2. **외부 trigger 인프라 (Claude Code hook 한계)** — Claude Code hook은 PostToolUse / UserPromptSubmit / SessionStart / Stop / PreCompact 등 Claude Code 내부 이벤트만 받음. 다음은 모두 외부 인프라 별도 필요:
   - 시간 기반 cron (pr-storybook 매일 18:00, decision-log-keeper) → launchd + `claude --print "/story-toolkit:skill-name args"` 외부 호출 [`--skill` 플래그 불존재 검증됨, 슬래시 명령은 `--print` 인자로 전달 — 출처: https://code.claude.com/docs/en/cli-reference, 2026-04-27]
   - GitHub PR open / merge (smart-contract-radar 사내 운영) → `anthropics/claude-code-action@v1` + `${{ secrets.ANTHROPIC_API_KEY }}` 패턴. `pull_request_target` trigger 사용 시 prompt injection 리스크 — 사내 보안팀 검토 필요 [출처: https://code.claude.com/docs/en/github-actions, 2026-04-27]
   - Slack message arrival (ticket-triage-router) → Slack Events API 별도 receiver
   GitHub Actions runner + Claude SDK secret 분배 정책은 본 plan.md 범위 외 — 사내 인프라 RFC 별도 트랙 필요. story-toolkit Phase 1.5a milestone은 이 RFC 미통과 시에도 MCP 5종만으로 통과 가능 (smart-contract-radar는 명시 호출 only로 시연). Matters because 워크숍 픽 smart-contract-radar는 시연용 명시 호출(옵션 T1)로 우회하지만, 사내 운영(옵션 T2/T3)은 plugin 외부 인프라 의존. Ask: launchd 등록 정책 허용 여부 + GitHub Actions RFC 착수 여부.
3. **Sandbox vs 실 워크스페이스** — B3 결정 직결. Matters because 워크숍 참가자 OAuth 발급이 막히면 시연 자체가 실패. Ask: Story 사내에 read-only 더미 워크스페이스 운영 가능한지.
4. **Plugin agent 보안 제약** — Anthropic plugin 보안 정책상 plugin agent frontmatter의 `hooks` / `mcpServers` / `permissionMode` 필드는 로드 시 무시됨. Matters because smart-contract-radar의 hook + tool 권한 격리는 agent frontmatter가 아닌 plugin top-level `hooks/hooks.json`과 `.mcp.json`으로 분리해야만 동작. piplabs/claude-code-setup도 동일 분리 패턴을 채택 중 — user-level `~/.claude/settings.json`에 baseline deny+hooks 60+ 룰, plugin/persona는 `hooks/hooks.json`·`.mcp.json`으로 추가만. Story 사내가 이미 운영 중인 형태를 그대로 따르므로 plugin agent 제약 우회 경로(top-level hooks/hooks.json + .mcp.json 분리)는 사내에서 검증됨. Ask: Anthropic plugin docs 재확인 + 실측.

---

## Appendix

### A. Spec (frozen, hash 4f47b678)

```yaml
goal: "Story Protocol 사내 전사용 plugin story-toolkit 전체 설계도. plugin 컨테이너 안에 어떤 skills/subagents가 들어가야 하는지, 각 후보 1페이지 설계서, 워크숍 시연용 추천 2-3개 + 시연 흐름까지."
non_goals:
  - "이번 단계에서 실제 SKILL.md / 서브에이전트 본문 코드까지 쓰지는 않음"
  - "project-brief 재설계 X — 그대로 포함만"
  - "Story 사내 private, 외부 공개 X"
constraints:
  - "story-toolkit 플러그인 컨테이너"
  - "워크숍 시연 가능 (비개발자, macOS+Claude Pro, 3시간)"
  - "MCP 신규 셋업 자유"
  - "시연 1건당 시간 제한 없음"
success_criteria:
  - "후보 skills/subagents 8-12개"
  - "각 후보 1페이지 설계서"
  - "워크숍 시연용 추천 2-3개 + 시연 흐름"
  - "story-toolkit 디렉토리 레이아웃 + manifest 설계"
context:
  - "Story Protocol = IP L1 블록체인. 전사 — 엔지/PM/마케/세일즈/CS"
  - "사내 가용 스택: GitHub, Slack, Notion, Google Drive, HubSpot"
  - "첫 스킬 project-brief 완료 (parallel scout + synthesizer 패턴)"
  - "워크숍 = 문토 AI meetup 3시간 비개발자"
intent_items:
  - id: I1
    text: "사내에서 활용할 수 있는 skills, subagents를 플러그인화"
  - id: I2
    text: "여기서 만든 것들을 기반으로 실습에서 와우가 나올 수 있도록 세팅"
  - id: I3
    text: "사내에서 사용하기 유용한 전체 플러그인에 대한 설계도"
  - id: I4
    text: "워크숍 시연용 추천 2-3개 + 시연 흐름"
anti_patterns:
  - "단순 ChatGPT 래퍼"
  - "parallel scout + synthesizer 패턴 또 반복"
  - "5분+ 셋업 (시연용 후보 한정)"
  - "주관적 결과 (와우 검증 안 됨)"
```

### B. Research 요약 (Stage 1-5)

- Stage 1 (Plugin 아키텍처): 단일 plugin + 5 MCP 정의, plugin agent frontmatter 보안 제약 확인
- Stage 2 (5 MCP 실제 도구): GitHub 200+, Slack GA, Notion 22, GDrive 7+ Docs/Sheets 별도, HubSpot 신생 GA 2026-04-13
- Stage 3 (후보 1-pagers): 초안 13건 → 사용자 실용성 검토 후 8개 채택 + release-radio 1개 제외 분석 = 본 문서 6.6
- Stage 4 (와우 패턴): H(hook) / V(시각 산출) / X(cross-MCP) 3축 + 클로즈드 루프
- Stage 5 (참고 plugin): Anthropic 공식, ralph-wiggum, claude-code-workflows, Superpowers, Compound Engineering, Trail of Bits

### C. Cross-validation 결과 ([ISSUES:8] 해소)

| Issue | 해소 |
|---|---|
| CONFLICT:2 release-radio anti-pattern 위반 | 워크숍 픽에서 제외, decision-log-keeper / ticket-triage-router 승격 (Section 6.4) |
| WEAK_EVIDENCE:1 ticket-triage-router GitHub 사용 명세 | 6.6.8에서 search_issues 명시 |
| WEAK_EVIDENCE:2 HubSpot drift | 6.6.6 / 6.6.7에 Drift H 명시 + Section 5 B1 dry-run 보류 |
| GAP:1 OAuth 워크숍 분배 | Section 5 B3 보류 + Section 8 Q3 |
| AXIS_GAP:1 MCP drift 평가 누락 | 9축에 "추가축. MCP drift" 모든 결정 표에 반영 (D1-D4) |
| BIAS:1 워크숍 픽 전부 신규 | 보수 1건(decision-log-keeper, Pattern A flat) 포함 원칙 (D4) + 시연 순서 점층 (6.7) |
| CONNECTION:1 와우 패턴↔후보 매핑 | 1-pagers에 H/V/X 태그 명시 (6.6 전체) |
| Topology 모호 | Pattern A/B/C 3종으로 통일, 1-pager에 Pattern 표기 (6.6 + D2) |
| (2026-04-27 추가) 사용자 실용성 검토 | 토이 데모 4종(repo-radar / blog-post-forge / social-recap / runbook-finder) 제외, 8개 코어 + ★★★ 픽으로 재구성 |

### D. Sources
- Anthropic Plugin docs — https://docs.claude.com/en/docs/claude-code/plugins (accessed 2026-04-27)
- GitHub MCP server — https://github.com/github/github-mcp-server
- Slack MCP — Slack official, 2026-01-26 GA
- Notion MCP — https://github.com/makenotion/notion-mcp-server
- Google Drive MCP — claude.ai connector
- HubSpot MCP — 2026-04-13 GA, OAuth 2.1
- piplabs/claude-code-setup — https://github.com/piplabs/claude-code-setup (accessed 2026-04-27, adapted from trailofbits/claude-code-config) — Story Protocol GitHub 조직(PIP Labs) 공식 Claude Code 보안 베이스라인. `personas/developer` + `personas/l1-auditor` 채택 (D5)
- trailofbits/claude-code-config — https://github.com/trailofbits/claude-code-config (1차 근거, piplabs가 fork·adapt한 원형)
- 기존 작업 — `40-projects/munto-ai-meetup/demo/project-brief/`

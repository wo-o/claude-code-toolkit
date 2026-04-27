---
name: deal-context-loader
description: Given a deal ID or company name, fetch the HubSpot deal stage and recent activity, then search Slack for internal mentions of the same company in the last 14 days. Synthesize a 5-minute pre-call brief. Output only — no write side effect.
---

# deal-context-loader

세일즈가 진행 중 딜 30-50건 동시 들고 있을 때 "오후 3시 콜 5분 전, A사 딜 어디까지 갔지?"를 빠르게 답하기 위한 5분 브리프 합성. HubSpot 정량(스테이지·액수) + Slack 정성(엔지/PM 내부 의견)을 한 번에.

**Drift 위험 H** — HubSpot MCP GA 신생(2026-04-13). 첫 호출 전 도구 dry-run 검증 필수.

## Input

- `--deal-id <id>` 또는 `--company <name>` (둘 중 하나 필수)
- `--slack-window-days <N>` (선택, 기본 14) — Slack 멘션 검색 기간
- 둘 다 누락 → AskUserQuestion

## Output

콘솔에 5분 브리프 마크다운 (write 없음):

```
# Deal Context — <Company / Deal #ID>

## 한 줄 상태
<딜 stage·액수·last activity 1-2문장>

## HubSpot
- Deal: #<id> — <name> — <stage> — <amount> — <close date>
- 소유자: @<owner display_name>
- Last activity: <YYYY-MM-DD HH:MM> — <activity 한 줄>
- 연관 컨택트:
  - <Name> (<title>) — <email/phone>
  - ...

## 내부 Slack 의견 (최근 <window>일)
- <YYYY-MM-DD HH:MM> #<channel> — @<author>: <메시지 1-2줄>
- ...
- (총 N건 메시지에서 핵심 N개 추출)

## 종합
- 정량 진행: <stage 기준 진행률 추정>
- 내부 우려/의견:
  - <우려 1>
  - <의견 1>
- 추가 정보(Slack 한정):
  - <엔지/PM이 fit 검토하며 남긴 메모>

## 다음 액션 (제안)
- [ ] <액션 1: 콜 직전 확인 사항>
- [ ] <액션 2>

## 검토 한계
- Slack 검색은 회사명 키워드 기반 — 별칭/줄임말 미커버 가능
- HubSpot last activity는 자동 로깅 의존 — 수기 콜 메모 누락 가능
```

## 진행 순서

### 1. 입력 파싱

- `--deal-id` 우선, 없으면 `--company`로 검색
- `--slack-window-days` 기본 14

### 2. HubSpot MCP 도구 검증 (drift 방어)

첫 호출 전 도구 노출 확인:
- `mcp__hubspot__get_deal` 또는 `mcp__hubspot__search_deals`
- `mcp__hubspot__get_contact`

도구명 다르면 즉시 중단.

### 3. HubSpot deal 정보 수집

`--deal-id` 있으면:
1. `mcp__hubspot__get_deal`로 deal 객체 fetch
2. 연관 contact·company·activity 펼침 (associated objects)

`--company` 있으면:
1. `mcp__hubspot__search_companies`로 회사 매칭
2. 그 회사의 open deals 목록 fetch
3. 가장 최근 last activity인 deal 1개 자동 선택 (다중이면 안내 + 선택 요청)

### 4. Slack 내부 멘션 검색

`mcp__slack__search_messages`:
- query: 회사명 + 주요 컨택트 이름 (HubSpot에서 가져온 1-2명)
- after: 입력 날짜 - `--slack-window-days`
- before: 오늘
- 채널 필터: 내부 채널만 (외부 사용자 메시지 제외 — 별도 필터 옵션 검토)

결과 N건 수집. 0건이면 "내부 Slack 멘션 없음" 명시.

### 5. LLM 합성

LLM이 위 출력 형식에 맞춰 5분 브리프 작성:
- 정량(HubSpot)과 정성(Slack)을 분리해 보여준 뒤 마지막 "종합"에서 합성
- 내부 우려와 의견 분리 (우려는 객관 fact, 의견은 주관 해석)
- 다음 액션은 콜 5분 전 즉시 실행 가능한 형태로 (예: "owner @X에게 last activity 후속 확인", "Slack #eng-deals에서 fit 검토 결론 확인")

### 6. 출력

콘솔에 마크다운 출력. write 없음. 끝.

## 함정

- **HubSpot MCP 도구명 drift**: dry-run 통과 후 사용. 도구명 변경 시 plugin patch
- **회사명 동음이의어**: 회사명이 짧거나 일반 단어면 Slack 검색이 무관 메시지 다량 회수. 회사명 + 컨택트 이름 AND 검색으로 보정
- **외부 사용자 메시지 혼입**: Slack `#support` 등 외부 사용자가 등장하는 채널은 내부 의견과 섞임. 채널 필터 옵션 (`--exclude-channels`) 또는 내부 채널 allowlist 사용
- **HubSpot last activity 자동 로깅 누락**: 수기 콜 메모를 owner가 안 남기면 last activity가 stale. "검토 한계"에 명시
- **워크숍 시연 부적합**: HubSpot sandbox 부재. 운영 후 사례 공유용

## 톤

- 정량과 정성 분리 — 헷갈리면 안 됨
- 내부 의견 인용 시 원문 유지, 의역 금지. 작성자 display_name 그대로
- 다음 액션은 짧게 (1줄), 콜 5분 전 즉시 실행 가능
- "확신 없음"은 "검토 한계"에 명시

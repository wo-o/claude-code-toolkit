---
name: battle-card-builder
description: Given a competitor name, fetch HubSpot deals where the competitor was involved (win/loss reasons, deal sizes) and search Notion competitor comparison pages. Synthesize into a 1-page sales battle card. Output only — no write side effect.
---

# battle-card-builder

세일즈가 신규 lead 첫 미팅 직전 "이 회사 우리 경쟁사 어디랑 비교 중인가?"를 5분 만에 답하기 위한 1페이지 battle card 합성. HubSpot 단독은 과거 deal 결과만, Notion 단독은 정태적 기능 비교 — 두 출처를 LLM이 교차해 화법 추천까지 한 번에.

**Drift 위험 H** — HubSpot MCP는 GA 신생(2026-04-13). 도구명 변경 가능성 있음. 첫 호출 전 `mcp__hubspot__*` 도구 목록 dry-run 검증 필수.

## Input

- `--competitor <name>` (필수) — 경쟁사명 (예: "Avalanche", "Polygon")
- `--deal-window-days <N>` (선택, 기본 365) — HubSpot deal 검색 기간
- 입력 누락 → AskUserQuestion

## Output

콘솔에 마크다운 battle card 1페이지 (write 없음, 사용자가 미팅 노트로 직접 복사):

```
# Battle Card — vs <Competitor>

## 한 줄 요약
<우리 vs 경쟁사 핵심 차별점 1-2문장>

## 우리 강점 (3개)
1. <강점 1> — <근거: Notion 비교 페이지 또는 HubSpot 승사례>
2. <강점 2> — <근거>
3. <강점 3> — <근거>

## 경쟁사 강점 (3개, honest)
1. <강점 1> — <근거: HubSpot 패사례 또는 Notion 비교>
2. <강점 2> — <근거>
3. <강점 3> — <근거>

## 과거 승패 사유 (HubSpot)
- 승: N건 / 패: M건 / 진행중: K건 (지난 <window>일)
- 주요 패 사유 top 3:
  1. <사유>: N건
  2. ...

## 추천 화법
- 가격 이슈 시: <한 줄 화법>
- 기능 비교 시: <한 줄 화법>
- 신뢰/실적 시: <한 줄 화법>

## 인용 자료
### HubSpot deals
- Deal #<id> — <회사명> — <stage: won/lost> — <amount> — <close date>
- ...

### Notion 비교 페이지
- <페이지 제목> — <URL>
- ...

## 검토 한계
- HubSpot deal 메모는 작성자 주관. 패 사유는 작성자 해석일 수 있음
- Notion 비교 페이지가 stale일 수 있음 (last edit 일자 명시)
- 본 카드는 1차 초안 — 미팅 직전 5분 검토 후 사용
```

## 진행 순서

### 1. 입력 파싱

- `--competitor` 누락 → AskUserQuestion
- `--deal-window-days` 기본 365

### 2. HubSpot MCP 도구 검증 (drift 방어)

첫 호출 전 사용 예정 도구가 실제 노출되는지 1회 확인:
- `mcp__hubspot__search_companies` (또는 동등)
- `mcp__hubspot__get_company`
- `mcp__hubspot__list_deals` (또는 동등)

도구명이 다르면 즉시 중단 — `docs/mcp-setup.md`의 dry-run 절차로 도구명 재확인 후 `--competitor` 재호출.

### 3. HubSpot deal 수집

1. `mcp__hubspot__search_companies`로 경쟁사명 매칭 — HubSpot에 등록된 경쟁사 company 객체 1-2개
2. 그 company 객체의 associated deals fetch (최근 `--deal-window-days` 일)
3. 각 deal: stage(won/lost/open), amount, close date, lost reason 메모

deal 0건이면 "HubSpot에 해당 경쟁사 등장 deal 없음" 명시 (가짜 데이터 작성 X).

### 4. Notion 비교 페이지 검색

`mcp__notion__search`로 경쟁사명 검색:
- 키워드: 경쟁사명 + "비교" / "vs" / "competitive"
- 가장 가까운 페이지 1-3건 fetch
- last edit 일자 추출

페이지 0건이면 "Notion에 비교 페이지 없음" 명시.

### 5. Battle card 합성

LLM이 위 출력 형식에 맞춰 작성:
- 우리 강점 3 / 경쟁사 강점 3은 honest 분배 (한쪽만 적으면 신뢰 박살)
- 추천 화법은 패 사유 top에 직접 대응 (예: 가격 패 사유가 top1이면 가격 화법 1순위)
- 인용 자료에 deal #와 Notion URL 모두 명시 — 세일즈가 미팅 중 백업 자료로 열 수 있도록

### 6. 출력

콘솔에 마크다운 출력. write 없음. 끝.

## 함정

- **HubSpot MCP 도구명 drift**: 2026-04-13 GA 신생. dry-run 통과 후 사용. 도구명 변경 시 plugin patch release 필요
- **HubSpot lost reason 자유 텍스트**: 표준화 안 된 메모 — LLM이 카테고리 추정해야 함. "가격" / "기능 부족" / "타이밍" / "기타" 4-5개로 그룹핑
- **Notion 비교 페이지 stale**: last edit 1년 전이면 신뢰 낮음. "검토 한계"에 last edit 일자 명시
- **HubSpot에 등록 안 된 경쟁사**: HubSpot company 객체에 등록 안 된 경쟁사면 deal 0건. "해당 경쟁사 HubSpot 미등록" 안내 + 세일즈에게 등록 요청 권고
- **워크숍 시연 부적합**: HubSpot sandbox 부재로 본 plugin 워크숍 픽 X. 운영 후 사례 공유용

## 톤

- 우리 강점·경쟁사 강점 둘 다 honest 작성 — 가짜 self-aggrandizing 금지
- 패 사유 인용 시 작성자 해석임을 명시
- 추천 화법은 짧게 (1줄), 미팅 중 즉시 활용 가능한 형태
- "확신 없음"은 "검토 한계"에 명시 — 가짜 자신감 금지

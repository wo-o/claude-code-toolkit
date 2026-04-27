---
name: pr-storybook
description: List today's merged PRs and post a non-engineer-readable single-paragraph story to #eng-changelog. Designed to run daily via launchd cron at 18:00, but can be invoked manually with --date for backfill or workshop demo.
---

# pr-storybook

GitHub Slack 공식 통합은 PR-merge 알림에 제목·작성자만 노출 — 비-엔지(PM/마케/세일즈) 9명이 "오늘 엔지가 뭐 했는지" 알 수 있게 비-엔지 가독 한 단락 스토리로 합성 후 `#eng-changelog` Slack 자동 포스팅. 매일 자동 = hook 없으면 죽는 후보.

## Input

- `--date YYYY-MM-DD` (선택, 기본 오늘 KST) — 추출 대상 날짜
- `--repo <owner/name>` (필수, 반복 가능) — 대상 repo (다중 repo 시 반복 지정)
- `--channel #channel-name` (선택, 기본 `#eng-changelog`) — Slack 채널
- `--dry-run` (선택) — Slack post 안 하고 콘솔에만 출력

## Output

콘솔 1줄 요약 + (dry-run 아니면) Slack post URL:

```
2026-04-26: <owner>/<repo> 머지 PR N건 추출 → #eng-changelog 포스팅 1건
- Slack message: <permalink>
```

Slack 포스팅 본문 형식:

```
📦 오늘의 변경 (YYYY-MM-DD)

오늘 <owner>/<repo>에 N건 머지됐습니다.

<2-4문장 비-엔지 가독 스토리 — 엔지 식별자 없이, 사용자/팀 영향 위주>

상세:
• #PR번호 — <PR 제목 그대로> (작성자 @display_name)
• ...
```

## 진행 순서

### 1. 입력 파싱

- `--date` 누락 → 오늘 (KST 기준)
- `--repo` 누락 → AskUserQuestion (추측 금지, 다중 repo 시 반복 입력)
- `--channel` 누락 → `#eng-changelog`
- `--dry-run` 있으면 step 4 skip

### 2. 머지 PR 수집

각 repo마다 `mcp__github__list_pull_requests` 호출:
- `state`: `closed`
- `sort`: `updated`
- `direction`: `desc`
- `per_page`: 50

응답에서 `merged_at`이 입력 날짜 KST 00:00-23:59 범위인 PR만 필터.

PR 0건이면 "그날 머지 없음" 출력하고 종료 (Slack 포스팅 X).

### 3. 비-엔지 스토리 합성

PR 목록에서 다음 정보 추출:
- 제목, 본문(요약), labels, files changed 통계 (additions/deletions)
- 작성자 display name
- 영역 추정: 파일 경로 prefix (예: `contracts/` → 컨트랙트, `frontend/` → 프론트, `infra/` → 인프라)

LLM으로 한 단락(2-4문장) 스토리 작성:
- 엔지 식별자 없이 (예: 파일명/모듈명 대신 사용자 영향 한 줄로)
- 사용자/팀 영향 위주 ("외부 호출 실패 시 자동 재시도 추가" 같이)
- 형용사 자제, 사실 나열

### 4. Slack 포스팅

`--dry-run` 아니면 `mcp__slack__post_message`:
- `channel`: 입력 채널
- `text`: 위 본문 형식 마크다운

write 권한 필요 — Slack MCP의 `chat:write` scope 사전 분배.

### 5. 출력

콘솔 1줄 요약 + Slack permalink. 끝.

## 자동 운영 (cron)

매일 18:00 KST launchd cron 권장:

```
# ~/Library/LaunchAgents/<reverse-domain>.claude-code-toolkit.pr-storybook.plist
# Program: /usr/local/bin/claude
# ProgramArguments: -p "/claude-code-toolkit:pr-storybook"
# StartCalendarInterval: { Hour=18, Minute=0 }
# EnvironmentVariables:
#   CLAUDE_CODE_TOOLKIT_CRON_MODE=1
```

상세 plist 예시는 `docs/cron-setup.md`.

## 워크숍 시연 (선택)

본 plugin의 워크숍 픽 3종에 포함되지 않음 (Slack write 권한이 워크숍 sandbox 부담). 운영 후 사례 공유용으로만 노출 권장.

`--dry-run`으로 시연하면 write 권한 없이도 합성 결과만 보여줄 수 있음:

```
> /claude-code-toolkit:pr-storybook --date 2026-04-26 --dry-run
```

## 함정

- **GitHub timezone**: `merged_at`은 ISO 8601 UTC. KST 입력 날짜를 UTC로 환산 후 비교
- **PR 본문에 민감 정보 포함**: 내부 인프라 경로/시크릿 키 패턴이 PR 본문에 노출된 적 있으면 LLM 스토리에 그대로 흘러들어감. system prompt에 "환경변수/시크릿/내부 IP 스캔 + 마스킹" 강제
- **다중 repo 합성**: 여러 repo의 PR을 한 단락에 섞으면 가독성 악화. repo별로 1단락씩 분리
- **labels 신뢰 X**: `bug` / `feat` 라벨이 일관성 없는 repo에선 스토리 정확도 떨어짐. 라벨보다 PR 제목/본문 LLM 추정 우선
- **cron 모드에서 hook prompt 멈춤**: Slack MCP write scope에 plugin hook 정의 안 되어 있음 (baseline에도 없음). 향후 cron 자동 실행 사고 방지를 위해 `CLAUDE_CODE_TOOLKIT_CRON_MODE=1` 환경변수 검사하는 hook 추가 검토 — 본 릴리스에선 미포함

## 톤

- PR 제목은 원문 유지 (의역 X). 본문 요약만 비-엔지 톤으로 재작성
- 작성자 익명 처리 X — display_name 그대로
- 형용사 자제 ("멋진"/"중요한" X). 사실 위주
- "오늘 별일 없음" 도 honest — 0건이면 안 올림 (가짜 활동 만들지 않음)

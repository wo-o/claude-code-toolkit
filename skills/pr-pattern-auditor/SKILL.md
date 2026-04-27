---
name: pr-pattern-auditor
description: Scan a PR for known code-review anti-patterns and cross-module call-site impact, then post the consolidated risk report as a PR comment. Spawns static-pattern-auditor (read-only file tools) and cross-module-impact (grep + list_commits) subagents in series. Write step gated by plugin hook.
---

# pr-pattern-auditor

**워크숍 픽 #3 (클라이맥스).** PR이 올라오면 시니어 리뷰어 1명이 회당 30-60분 수동으로 하던 (a) 1차 안티패턴 스캔 + (b) cross-module 영향 분석을 자동화. 사람은 의미 있는 트레이드오프 검토에만 집중. 시연 시퀀스는 `docs/workshop-flow.md` 참조.

도구 권한은 단계별 격리(Pattern C). 패턴 스캔 = PR 파일 read만, 영향 분석 = list_commits + Bash grep만, write = 최종 합성 단계 1회만 + plugin hook PreToolUse 사용자 확인 1회.

언어 비종속(language-agnostic). 변경된 파일의 확장자/경로를 보고 그에 맞는 패턴 셋(범용 + 언어별)을 선택해 매칭한다.

## Input

- `--pr <N>` (필수) — PR 번호
- `--repo <owner/name>` (필수) — 대상 repo
- 입력 없으면 AskUserQuestion으로 받기 — 추측 금지

## Output

```
## PR Pattern Auditor — PR #<N>

### 1차 패턴 스캔 (static-pattern-auditor)
- <파일>:<line> — <안티패턴 이름>: <증상 1줄>
- ...

### Cross-module 영향 (cross-module-impact)
- 변경된 시그니처: <함수/메서드/이벤트>
- 호출처: <파일>:<line> (총 N개 모듈 영향)
- ...

### 종합 리스크 리포트
<2-4개 리스크를 [HIGH/MED/LOW] 라벨 + 1-2줄 근거로>

### 권고 사항
- [ ] <리뷰어 행동 1>
- [ ] <리뷰어 행동 2>

### 자동 검토의 한계
- <항목 1: 예) 테스트 미실행 — 정적 분석 only>
- <항목 2: 예) 빌드/타입체크 미수행>
```

위 출력은 PR comment로 자동 게시되며 동시에 콘솔에도 동일 마크다운 출력.

## 진행 순서

### 1. 입력 검증

- `--pr` 누락 → AskUserQuestion
- `--repo` 누락 → AskUserQuestion (조직 정책에 따라 외부 repo 차단을 원하면 wrapper에서 prefix 검사)
- PR 존재 확인: `mcp__github__get_pull_request` 1회 — 없으면 즉시 종료

### 2. static-pattern-auditor subagent 실행

Task 툴로 호출:

```
description: "Scan PR diff for anti-patterns"
subagent_type: "static-pattern-auditor"
prompt: |
  PR: <repo>#<N>

  변경된 파일을 mcp__github__get_pull_request_files로 나열하고
  mcp__github__get_pull_request_diff로 diff 가져와 안티패턴 매칭.

  대상 패턴 (범용):
  - secret/credential 의심 문자열 (긴 hex/base64, AWS_KEY 패턴, .env 평문 키 추가)
  - 권한 검사 누락 (admin/owner/role guard 없는 mutation 엔드포인트)
  - 에러 무시 (try/catch에서 swallow, 반환값 무체크)
  - 입력 검증 누락 (외부 입력 → DB/shell/파일 경로 직결)
  - 큰 diff (단일 파일 +500줄, 단일 함수 +200줄)
  - TODO/FIXME/XXX 누적 또는 신규 추가
  - 의존성 추가 시 lockfile 미동기화

  언어별 패턴 (확장자로 선택):
  - JS/TS: dynamic code 실행 (eval, Function 생성자), sanitize 없는 HTML 주입
  - Python: dynamic code 실행 (exec/eval), shell injection 가능 호출, 신뢰 불가 입력의 직렬화 역직렬화
  - Go: ignored errors, goroutine without recover, mutex 누락
  - Rust: unsafe block 신규 도입, unwrap 신규 도입
  - Java/Kotlin: Runtime.exec, 네트워크에서 받은 ObjectInputStream

  각 의심 위치 1줄로 보고 + 패턴명 + line 번호.
```

도구는 `mcp__github__get_pull_request_files`, `mcp__github__get_pull_request_diff`만 (read-only).

### 3. cross-module-impact subagent 실행

static-pattern-auditor 결과를 prompt에 포함해 호출:

```
description: "Find call-sites of changed signatures"
subagent_type: "cross-module-impact"
prompt: |
  PR: <repo>#<N>
  변경된 함수/메서드/이벤트/export 시그니처 목록: <auditor가 추출한 시그니처>

  각 시그니처에 대해:
  1. mcp__github__list_commits로 최근 50건 가져와 해당 시그니처의 도입/변경 이력 확인
  2. Bash grep으로 repo 내 다른 파일에서 호출처 역추적
     (테스트/스크립트/벤더 디렉토리 제외)
  3. 영향 모듈 리스트 + 호출처 file:line 보고
```

도구는 `mcp__github__list_commits` + `Bash` (grep, find — sandboxed).

### 4. 종합 리스크 리포트 작성

skill 본문에서 두 결과 합성:
- 안티패턴 위치 N개 + 영향 모듈 M개 → 리스크 [HIGH/MED/LOW] 라벨 부여
  - HIGH: 안티패턴 발견 + 영향 모듈 ≥3
  - MED: 안티패턴 발견 + 영향 모듈 1-2
  - LOW: 안티패턴 미발견, 단순 로직 변경
- 권고 사항: "이 함수의 권한 검사 추가 검토", "X 모듈 호출처 N개 동시 검토" 등

### 5. PR comment 자동 게시

`mcp__github__create_pull_request_comment` 호출:
- `owner` / `repo` / `pull_number` / `body` (위 종합 리포트 마크다운)

**hook 동작:** plugin의 `PreToolUse(mcp__github__create_pull_request_comment)` hook이 발화 — 사용자에게 "PR #<N>에 위 리포트 게시합니다. 진행?" 1회 확인. CI 모드(`CLAUDE_CODE_TOOLKIT_CI_MODE=1` 환경변수)에서는 hook bypass + audit log만 기록 (`~/.claude-code-toolkit/audit/pr-pattern-auditor-YYYY-MM-DD.jsonl`, 일자별 append).

### 6. 출력

콘솔에 동일 마크다운 + PR comment URL 1줄. 끝.

## Trigger 메커니즘 (3옵션)

| 옵션 | 발화 시점 | 인프라 | 비고 |
|---|---|---|---|
| T1 | 명시 슬래시 호출 `/claude-code-toolkit:pr-pattern-auditor --pr 42 --repo <owner>/<repo>` | Claude Code 내부 | 워크숍 시연용 — hook 무관 |
| T2 | GitHub Actions workflow `pull_request: opened` → runner에서 `claude --print "/claude-code-toolkit:pr-pattern-auditor --pr <PR_NUMBER> --repo <REPO>"` 외부 호출 | `anthropics/claude-code-action@v1` + `${{ secrets.ANTHROPIC_API_KEY }}` | 자동화 옵션 1 — 별도 RFC 트랙 |
| T3 | webhook receiver 서버 + Claude SDK 호출 | 별도 인프라 | 자동화 옵션 2 |

**주의:** Claude Code 자체 hook 시스템(PreToolUse/PostToolUse/UserPromptSubmit)은 외부 GitHub PR open 이벤트를 받지 못함. T2/T3는 plugin 외부 인프라 — 본 plugin 범위는 T1만 보장.

`pull_request_target` trigger 사용 시 prompt injection 리스크 — 보안 검토 필요. (https://code.claude.com/docs/en/github-actions, 2026-04-27)

## 워크숍 시연 (T1 명시 호출)

```
> /claude-code-toolkit:pr-pattern-auditor --pr 42 --repo <owner>/<repo>
```

사전 셋업:
- 더미 PR 1건 (의도적 권한 검사 누락 1곳 + 시그니처 변경 1건)
- 진행자 PC에 GitHub PAT (sandbox repo `pull_requests:write` scope)
- baseline sandbox 정책 활성 (keystore deny rule 등)

시연 1분 흐름:
1. 명령 입력 (5초)
2. static-pattern-auditor subagent 실행 — PR 파일만 read, 패턴 의심 위치 1건 보고 (15초)
3. cross-module-impact subagent 실행 — 영향 모듈 2-3개 grep으로 보고 (15초)
4. 종합 리포트 작성 + PreToolUse hook 발화 → "PR comment 게시?" → y (10초)
5. PR 페이지 새로고침 → comment 노출 + 진행자 설명: "subagent 2개가 도구 권한 분리 — 패턴 스캔은 read만, 영향 분석은 grep만, write는 마지막 1번만. baseline OS-수준 정책도 동시에 강제" (15초)

핵심 메시지: "Pattern C — 도구 권한별 subagent 격리. 한쪽이 망가져도 다른 쪽 결과 살아있음 + audit log에서 어느 단계가 어디까지 봤는지 추적 가능. write 권한은 마지막 합성 1회만 + hook 1회 확인"

## 함정

- **PR diff가 너무 큰 경우**: get_pull_request_diff 응답이 토큰 한도 초과. 100KB 넘으면 파일별로 쪼개서 get_pull_request_files만 사용하고 패턴 스캔을 파일 단위로 분할 호출
- **시그니처 grep false positive**: 메서드명이 흔한 단어(예: `update`, `transfer`)면 호출처 grep 결과 수백 건. cross-module-impact가 모듈/패키지 단위로 호출처 그룹핑 + 의심 컨텍스트(같은 파라미터 타입)만 하이라이트
- **테스트 파일 변경을 production 변경으로 오인**: PR 파일 목록에서 `test/`, `tests/`, `__tests__/`, `spec/`, `script/`, `scripts/`, `vendor/`, `node_modules/` 경로 자동 제외 (audit 대상 아님)
- **plugin agent frontmatter의 `hooks`/`mcpServers` 필드 무시**: Anthropic plugin 보안 정책. hook은 `hooks/hooks.json` (plugin top-level), MCP는 `.mcp.json`에만 정의. agent 파일에 적어도 로드되지 않음
- **CI 모드에서 hook prompt 멈춤**: `CLAUDE_CODE_TOOLKIT_CI_MODE=1` 환경변수로 hook bypass + audit log만 기록. 이 환경변수는 plugin hooks.json에서 검사

## 톤

- 리포트는 사실 위주, 추측 표현 금지 ("looks suspicious" X → "함수 X에서 외부 호출 후 state mutation 발생, line N")
- HIGH/MED/LOW 라벨은 위 기준 그대로 — 임의 가중치 금지
- "자동 검토의 한계" 섹션 필수 — 테스트 미실행, 빌드/타입체크 미수행 등 명시
- 가짜 자신감 ("이 PR은 안전합니다" 등) 금지 — auditor는 1차 스캔, 최종 판단은 리뷰어 사람

## 컨텍스트

이 skill은 워크숍 픽 #3 (클라이맥스). Pattern C 클로즈드 루프 (read → read → write) + 도구 권한 격리 + plugin hook 사용자 확인을 한 흐름에서 보여준다. 운영은 T2 (GitHub Actions) 권장하나 별도 RFC 필요 — 본 plugin은 T1 명시 호출만 보장.

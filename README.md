# claude-code-toolkit

팀 생산성 Claude Code plugin. 직군별(엔지·보안·PM·세일즈·CS) 자동화 skill + subagent 묶음.

- **Skills:** 8종 (project-brief, pr-storybook, pr-pattern-auditor, decision-log-keeper, spec-from-thread, battle-card-builder, deal-context-loader, ticket-triage-router)
- **Subagents:** 7종 (synthesizer, slack-scout, notion-scout, static-pattern-auditor, cross-module-impact, ticket-classifier, ticket-draft-writer)
- **MCPs:** 활성 4종 (GitHub, Slack, Notion, HubSpot) + 선택 1종 (Google Drive — 보안 검토 후 활성화)
- **보안 baseline (선택):** 조직에 기존 Claude Code baseline (sandbox + deny rule + PreToolUse 차단)이 있으면 그 위에 본 plugin을 얹는 것을 권장. 없으면 본 plugin 단독 동작도 가능.

설계 근거: `docs/workshop-flow.md`, `plan.md` (역사적 설계 문서)

---

## 사전 요구사항

1. **Claude Code 설치** — macOS / Linux. 버전 확인:
   ```
   claude --version
   ```
2. **GitHub CLI 인증** — plugin이 호스팅된 marketplace 접근:
   ```
   gh auth login
   gh auth status
   ```
3. **(선택) 보안 baseline 선행 설치** — 조직 차원 sandbox + deny rule이 있으면 그 위에 본 plugin을 얹는 것을 권장. baseline에서 다음을 강제하면 본 plugin은 그대로 안전하게 동작:
   - `sandbox.enabled: true` — macOS Seatbelt / Linux bubblewrap 격리
   - `enableAllProjectMcpServers: false` — 외부 repo `.mcp.json` 자동 활성 차단 (plugin이 정의한 .mcp.json은 자동 활성화됨)
   - 시스템·자격증명·secret 보호 deny rule
   - PreToolUse hooks: `rm -rf` 차단, `git push main` 차단

   baseline이 없으면 plugin은 단독 동작하되 위 가드들은 별도로 도입 권장.

---

## 설치

> marketplace 호스팅이 결정되기 전까지는 로컬 plugin 디렉토리 install 사용:
>
> ```
> claude /plugin install /path/to/claude-code-toolkit
> ```

```
claude /plugin marketplace add <marketplace-url>
claude /plugin install claude-code-toolkit
```

설치 후 활성화 확인:

```
claude /plugin list
# claude-code-toolkit (enabled) 표시 확인
```

---

## 1차 셋업 (MCP 4종 활성 + 1종 선택)

`.mcp.json`은 plugin이 정의하지만 인증은 사용자별로 1회 필요. 다음을 순서대로 수행.

> **보안 결정 (2026-04-27):** Google Drive MCP는 현재 Composio 브로커 경유로 OAuth 토큰이 외부 인프라를 거치므로 `.mcp.json`에서 분리되어 있다 (`.mcp.google-drive.optional.json`). 보안 검토 후에만 수동 머지하여 활성화. 본 1차 셋업은 4종(GitHub/Slack/Notion/HubSpot)만 다룬다.

### 1. GitHub PAT

`CLAUDE_CODE_TOOLKIT_GITHUB_PAT` 환경변수에 GitHub Fine-grained PAT 등록:

- Repository access: 사용 repo만 명시 (전체 접근 X)
- Permissions (최소권):
  - `Contents: read` — PR diff/files 조회
  - `Pull requests: read+write` — pr-pattern-auditor PR comment 게시 only
  - `Issues: read` — pr-storybook의 PR 메타 fetch (write 불요)
  - `Metadata: read` — repo 메타정보

PAT 저장 — 다음 중 1개 (위에서 아래로 보안 강도 순):

**Option A — 1Password CLI (권장):**
```
# ~/.zshrc
export CLAUDE_CODE_TOOLKIT_GITHUB_PAT="$(op read 'op://Employee/claude-code-toolkit-github-pat/credential' 2>/dev/null)"
```
PAT는 1Password Vault에만 저장. shell startup마다 lazy fetch. `op signin` 미인증 상태면 빈 문자열 → skill 호출 시 명시적 실패.

**Option B — macOS Keychain (차선):**
```
# 1회 등록 (대화형으로 PAT 입력)
security add-generic-password -a "$USER" -s claude-code-toolkit-github-pat -w

# ~/.zshrc
export CLAUDE_CODE_TOOLKIT_GITHUB_PAT="$(security find-generic-password -s claude-code-toolkit-github-pat -w 2>/dev/null)"
```
PAT는 Keychain에만 저장. Time Machine 백업 시 암호화된 keychain 파일만 백업.

**Option C — `~/.zshrc` 평문 export (비권장 — 폴백):**
```
export CLAUDE_CODE_TOOLKIT_GITHUB_PAT="github_pat_..."
```
보안 risk: PAT가 모든 child process 환경변수에 노출 (`printenv`, `/proc/<pid>/environ`, shell history backup, dotfile sync 도구). dev machine 개인 사용 한정 + 다른 도구 없을 때만.

**보안 가이드:**
- PAT 만료 90일 권장 (Fine-grained PAT). 만료 알림 캘린더 등록 필수
- `gh auth token`(gh CLI 토큰)과 CLAUDE_CODE_TOOLKIT_GITHUB_PAT를 같은 값으로 재사용 금지 — scope 다름
- 분실/유출 시: GitHub Settings → Developer settings → Fine-grained tokens → Revoke 즉시

### 2. Slack OAuth

`/mcp slack` 명령으로 OAuth 흐름 트리거. Slack 워크스페이스 admin이 `chat:write`, `channels:history`, `channels:read`, `search:read`, `users:read` scope 결재 후 사용자별 OAuth 완료.

### 3. Notion 통합

`/mcp notion` 명령으로 OAuth 트리거. Notion workspace 관리자가 통합 등록 + plugin이 접근할 페이지/DB 권한 부여 (Decision Log DB 등).

### 4. HubSpot

`/mcp hubspot` 명령으로 OAuth 트리거. HubSpot 계정에 plugin OAuth 앱 승인.

> **Drift H 주의:** HubSpot remote MCP는 2026-04 GA 직후 — 도구 ID 안정성 dry-run 1회 후 첫 호출 권장 (`docs/mcp-setup.md` 참조).

### 5. Google Drive (선택, 보안 검토 후만 활성화)

기본 `.mcp.json`에 **포함되지 않음**. 활성화 절차:

1. **보안 검토** — Composio 브로커 (`mcp.composio.dev`) 경유로 OAuth refresh token이 외부 인프라에 위탁된다는 사실 검토 + 승인 (Anthropic-native GDrive MCP가 GA되면 즉시 교체).
2. 결재 후 `.mcp.google-drive.optional.json`의 `mcpServers.google-drive` 블록을 `.mcp.json`의 `mcpServers`에 머지.
3. `claude /plugin` 재시작 → `/mcp google-drive` OAuth.
4. 사용 종료 시 https://composio.dev → connected accounts → revoke 필수.

활성화 전까지 google-drive 의존 워크플로우는 Notion/Slack 우선 사용.

---

활성 4종(또는 5종) 인증 후 도구 노출 검증:

```
claude /mcp
# mcp__github__*, mcp__slack__*, mcp__notion__*, mcp__hubspot__* 노출 확인
# google-drive 활성 시 mcp__google-drive__* 추가
```

---

## 사용

각 skill은 `/claude-code-toolkit:<skill-name>`으로 호출.

```
# 프로젝트 1쪽 브리프 (Slack + Notion 정찰)
/claude-code-toolkit:project-brief --project <project-name>

# 그날 #decisions 결정 → Notion DB 자동 정리
/claude-code-toolkit:decision-log-keeper --date 2026-04-26

# Slack 메시지 → 카테고리 분류 + 답변 초안
/claude-code-toolkit:ticket-triage-router --message-url https://...

# PR diff → 안티패턴 + 영향 모듈 분석 → PR comment write-back
/claude-code-toolkit:pr-pattern-auditor --pr 42 --repo <owner>/<repo>

# 그 외 4종
/claude-code-toolkit:pr-storybook
/claude-code-toolkit:spec-from-thread
/claude-code-toolkit:battle-card-builder
/claude-code-toolkit:deal-context-loader
```

직군별 권장 skill — `docs/workshop-flow.md` 참조.

---

## Cron / 외부 trigger 등록 (선택)

`decision-log-keeper`, `pr-storybook`은 매일 자동 실행이 가치가 큰 후보. Claude Code hook은 외부 cron을 발화하지 않으므로, macOS는 `launchd`, Linux는 `systemd timer` / `cron`으로 등록.

예시 — macOS launchd:

```
# ~/Library/LaunchAgents/<reverse-domain>.claude-code-toolkit.decision-log-keeper.plist
# StartCalendarInterval Hour=18 Minute=0 → claude -p "/claude-code-toolkit:decision-log-keeper --date $(date +%Y-%m-%d)"
```

상세는 `docs/cron-setup.md`.

---

## 트러블슈팅

| 증상 | 원인 | 해결 |
|---|---|---|
| `/plugin install` 실패 — 404 | gh auth 미인증 또는 marketplace repo 권한 부족 | `gh auth refresh -h github.com -s repo` |
| `/mcp` 명령에 server 미노출 | baseline `enableAllProjectMcpServers: false` 의도 동작. plugin이 정의한 .mcp.json은 자동 활성화됨 | `claude /plugin list`로 enable 확인 |
| GitHub MCP 도구 ID 누락 | `CLAUDE_CODE_TOOLKIT_GITHUB_PAT` 미설정 또는 token 만료 | 환경변수 재설정, shell 재시작 |
| Slack `chat:write` 거부 | 워크스페이스 admin 미승인 | admin에게 `docs/mcp-setup.md` 공유 후 재OAuth |
| `Bash(rm -rf *)` 차단 | baseline 정상 동작 (재정의 X) | 의도된 동작 — `trash` 등 안전 도구 사용 |
| pr-pattern-auditor PR comment 직전 사용자 확인 prompt | plugin 고유 hook (`PreToolUse(mcp__github__create_pull_request_comment)`) | y로 승인 또는 hook 비활성화: `--allowedTools "mcp__github__create_pull_request_comment"` |
| CI/cron 환경에서 hook prompt 멈춤 (`read -r _ </dev/tty` 대기) | plugin hook이 인터랙티브 확인 요구 | pr-pattern-auditor는 `CLAUDE_CODE_TOOLKIT_CI_MODE=1` (audit: `pr-pattern-auditor-YYYY-MM-DD.jsonl`), decision-log-keeper는 `CLAUDE_CODE_TOOLKIT_CRON_MODE=1` (audit: `decision-log-YYYY-MM-DD.jsonl`). spec-from-thread는 본 릴리스 인터랙티브 전용 (audit log 미분리 — §보안 모델 참조) |
| audit log 파일 누적 | 일자별 JSONL append, rotation 없음 | 운영 정책에 따라 주/월 단위 cleanup (`find ~/.claude-code-toolkit/audit -mtime +30 -delete` 등) |

---

## 보안 모델

- 조직 baseline의 deny rule + sandbox + PreToolUse `rm -rf`/`git push main` 차단은 **재정의 안 함** — 그대로 동작.
- plugin 고유 hook (`hooks/hooks.json`)은 baseline 미정의 영역만 추가:
  - `PreToolUse(mcp__github__create_pull_request_comment)` — PR write-back 직전 1회 사용자 확인
  - `PreToolUse(mcp__notion__create_page)` — Notion DB row 추가 직전 1회 사용자 확인 (cron 모드에서는 audit log only)
- 시연 픽 3종(decision-log-keeper, ticket-triage-router, pr-pattern-auditor) subagent는 **도구 권한 격리** — read-only / write 분리 (Pattern C). 각 skill의 `agents/` 정의 + frontmatter `tools` allowlist 참조.
- **제3자 trust hop (Google Drive MCP) — 기본 비활성화:** Composio 브로커(`mcp.composio.dev`) 경유로 OAuth refresh token이 외부 인프라에 위탁되는 risk가 있어 `.mcp.json`에서 분리 — `.mcp.google-drive.optional.json`에만 정의됨. 보안 검토 후 수동 머지로만 활성화. Anthropic-native GDrive MCP가 GA되면 즉시 교체 + Composio 측 token revoke. 그 전까지 중요 문서 워크플로우는 Notion/Slack 우선.
- **Secret 저장 정책:** PAT/DB ID는 macOS Keychain 또는 1Password CLI에서만 fetch (README §1차 셋업 참조). `~/.zshrc` 평문 export는 폴백 옵션. cron plist `EnvironmentVariables`에 secret 직접 기재 **금지** — wrapper script + Keychain 패턴 (docs/cron-setup.md §secret 저장 정책 참조).
- **bypass 환경변수 사용 시점:**
  - `CLAUDE_CODE_TOOLKIT_CI_MODE=1` — pr-pattern-auditor PR comment용. audit log: `~/.claude-code-toolkit/audit/pr-pattern-auditor-YYYY-MM-DD.jsonl`
  - `CLAUDE_CODE_TOOLKIT_CRON_MODE=1` — decision-log-keeper Notion row 추가용. audit log: `~/.claude-code-toolkit/audit/decision-log-YYYY-MM-DD.jsonl`
  - spec-from-thread는 본 릴리스에서 **인터랙티브 confirm 전용** (현 hook이 caller별 audit log를 분리하지 않으므로 bypass 사용 시 audit trail이 decision-log-keeper와 섞임). caller-aware audit log 분리 후 cron 활성화 검토.
  - pr-storybook은 Slack `chat:write`에 plugin hook이 정의되지 않음 (의도) — cron 운영 시 Slack post가 confirm 없이 발화되며 audit row 없음. 워크숍 시연에서는 `--dry-run` 사용 권장. 향후 hook 추가 시 별도 RFC.
  - bypass 환경변수는 shell 평소 환경에 export 금지 — cron plist 또는 GitHub Actions secrets에서만 주입.

---

## 워크숍

5분 시연 흐름: `docs/workshop-flow.md`

```
[T+0:30] /claude-code-toolkit:decision-log-keeper --date 2026-04-26       (보수 워밍업, H+X)
[T+1:30] /claude-code-toolkit:ticket-triage-router --message-url <slack>  (V+X 와우)
[T+3:30] /claude-code-toolkit:pr-pattern-auditor --pr 42 --repo <repo>    (X 클라이맥스, PR write-back)
```

---

## 라이선스 / 기여

- 라이선스: MIT (`LICENSE` 참조).
- 이슈/PR: https://github.com/wo-o/claude-code-toolkit
- 설계 변경 제안: 이슈로 먼저 논의 후 PR.

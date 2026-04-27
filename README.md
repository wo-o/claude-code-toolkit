# claude-code-toolkit

팀 생산성 Claude Code plugin. 직군별(엔지·보안·PM·세일즈·CS) 자동화 skill + subagent 묶음.

- **Skills:** 6종 (project-brief, decision-log-keeper, spec-from-thread, ticket-triage-router, battle-card-builder, deal-context-loader)
- **Subagents:** 5종 (synthesizer, slack-scout, notion-scout, ticket-classifier, ticket-draft-writer)
- **외부 통합:** Slack / Notion / HubSpot — Anthropic 공식 Connector를 `claude.ai/customize/connectors` UI에서 토글. plugin은 Connector 등록을 관리하지 않음
- **보안 baseline (선택):** 조직에 기존 Claude Code baseline (sandbox + deny rule + PreToolUse 차단)이 있으면 그 위에 본 plugin을 얹는 것을 권장. 없으면 본 plugin 단독 동작도 가능.

설계 근거: `docs/workshop-flow.md`

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

## Connectors 활성 (Slack / Notion / HubSpot)

본 plugin은 외부 통합을 직접 등록하지 않는다. Anthropic 공식 Connector를 사용한다 — 이유:

- claude.ai 계정에 토글 1회 → CLI / Desktop / Mobile 모두에 자동 노출
- OAuth 인증을 Anthropic 인프라가 처리, 사용자 PC에 토큰 저장 안 됨
- skill 코드는 그대로 — 도구 ID(`mcp__slack__*`, `mcp__notion__*`, `mcp__hubspot__*`)가 동일

활성 절차:

1. https://claude.ai/customize/connectors 접속
2. Slack / Notion / HubSpot 각각 토글 → OAuth 흐름 따라가기 (워크스페이스 admin 결재 필요할 수 있음)
3. CLI에서 노출 검증:
   ```
   claude /mcp
   # mcp__slack__*, mcp__notion__*, mcp__hubspot__* 노출 확인
   ```

> **GitHub은 의도적으로 미포함.** Anthropic 공식 GitHub Connector는 read-only(파일·브랜치 조회만 — PR diff/comment, Issue write 미지원)이며, 본 toolkit이 PR write까지 수행했던 데모 skill(pr-pattern-auditor / pr-storybook)은 v1에서 제거했다. GitHub 자동화 워크플로우는 별도 trial repo로 분리 예정.

---

## 사용

각 skill은 `/claude-code-toolkit:<skill-name>`으로 호출.

```
# 프로젝트 1쪽 브리프 (Slack + Notion 정찰)
/claude-code-toolkit:project-brief --project <project-name>

# 그날 #decisions 결정 → Notion DB 자동 정리
/claude-code-toolkit:decision-log-keeper --date 2026-04-26

# Slack thread → Notion 스펙 페이지 자동 생성
/claude-code-toolkit:spec-from-thread --thread-url https://...

# Slack #support 메시지 → 분류 + 답변 초안 (Pattern C 격리)
/claude-code-toolkit:ticket-triage-router --message-url https://...

# 그 외 2종
/claude-code-toolkit:battle-card-builder
/claude-code-toolkit:deal-context-loader
```

직군별 권장 skill — `docs/workshop-flow.md` 참조.

---

## Cron / 외부 trigger 등록 (선택)

`decision-log-keeper`는 매일 자동 실행이 가치가 큰 후보. Claude Code hook은 외부 cron을 발화하지 않으므로, macOS는 `launchd`, Linux는 `systemd timer` / `cron`으로 등록.

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
| `/mcp` 명령에 server 미노출 | Connector 토글 누락 또는 claude.ai 미인증 | https://claude.ai/customize/connectors 에서 활성 후 Claude Code 재시작 |
| Slack `chat:write` 거부 | 워크스페이스 admin 미승인 | admin에게 Connector 결재 요청 후 재OAuth |
| `Bash(rm -rf *)` 차단 | baseline 정상 동작 (재정의 X) | 의도된 동작 — `trash` 등 안전 도구 사용 |
| decision-log-keeper Notion 쓰기 직전 사용자 확인 prompt | plugin 고유 hook (`PreToolUse(mcp__notion__create_page)`) | y로 승인 또는 hook 비활성화: `--allowedTools "mcp__notion__create_page"` |
| CI/cron 환경에서 hook prompt 멈춤 (`read -r _ </dev/tty` 대기) | plugin hook이 인터랙티브 확인 요구 | decision-log-keeper는 `CLAUDE_CODE_TOOLKIT_CRON_MODE=1` (audit: `decision-log-YYYY-MM-DD.jsonl`). spec-from-thread는 본 릴리스 인터랙티브 전용 (audit log 미분리 — §보안 모델 참조) |
| audit log 파일 누적 | 일자별 JSONL append, rotation 없음 | 운영 정책에 따라 주/월 단위 cleanup (`find ~/.claude-code-toolkit/audit -mtime +30 -delete` 등) |

---

## 보안 모델

- 조직 baseline의 deny rule + sandbox + PreToolUse `rm -rf`/`git push main` 차단은 **재정의 안 함** — 그대로 동작.
- plugin 고유 hook (`hooks/hooks.json`)은 baseline 미정의 영역만 추가:
  - `PreToolUse(mcp__notion__create_page)` — Notion DB row 추가 직전 1회 사용자 확인 (cron 모드에서는 audit log only)
- 시연 픽 3종(decision-log-keeper, ticket-triage-router, spec-from-thread) subagent는 **도구 권한 격리** — read-only / write 분리 (Pattern C). 각 skill의 `agents/` 정의 + frontmatter `tools` allowlist 참조.
- **외부 통합은 Anthropic 공식 Connector만 사용** — Slack/Notion/HubSpot 모두 OAuth 토큰을 Anthropic 인프라가 관리하며 사용자 디스크에 secret이 저장되지 않는다. plugin이 자체 `.mcp.json`을 갖지 않으므로 환경변수 PAT 주입 표면이 없다.
- **Composio 등 제3자 broker 미사용:** v0에 포함됐던 Google Drive Composio 통합은 v1에서 제거됨 (외부 인프라에 OAuth refresh token 위탁되는 trust hop 회피). Anthropic-native GDrive Connector가 GA되면 검토 후 도입.
- **bypass 환경변수 사용 시점:**
  - `CLAUDE_CODE_TOOLKIT_CRON_MODE=1` — decision-log-keeper Notion row 추가용. audit log: `~/.claude-code-toolkit/audit/decision-log-YYYY-MM-DD.jsonl`
  - spec-from-thread는 본 릴리스에서 **인터랙티브 confirm 전용** (현 hook이 caller별 audit log를 분리하지 않으므로 bypass 사용 시 audit trail이 decision-log-keeper와 섞인다). caller-aware audit log 분리 후 cron 활성화 검토.
  - bypass 환경변수는 shell 평소 환경에 export 금지 — cron plist 또는 GitHub Actions secrets에서만 주입.

---

## 워크숍

5분 시연 흐름: `docs/workshop-flow.md`

```
[T+0:30] /claude-code-toolkit:decision-log-keeper --date 2026-04-26       (보수 워밍업, H+X)
[T+1:30] /claude-code-toolkit:ticket-triage-router --message-url <slack>  (V+X 와우)
[T+3:30] /claude-code-toolkit:spec-from-thread --thread-url <slack>       (X 클라이맥스, raw → spec 변환)
```

---

## 라이선스 / 기여

- 라이선스: MIT (`LICENSE` 참조).
- 이슈/PR: https://github.com/wo-o/claude-code-toolkit
- 설계 변경 제안: 이슈로 먼저 논의 후 PR.

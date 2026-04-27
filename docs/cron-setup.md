# Cron Setup

`decision-log-keeper`, `pr-storybook`은 자동 실행 가치가 큰 후보. Claude Code 자체 hook은 외부 시각 trigger를 받지 못하므로 OS scheduler를 사용한다. spec-from-thread는 본 릴리스에서 cron 비권장 (아래 §spec-from-thread 절 참조).

## 공통 원칙

- Claude Code CLI를 `-p` (print mode) 호출 — 비대화형
- bypass 환경변수는 plist/cron 환경에서만 주입 (shell 평소 환경 X)
- 실패 시 stderr를 별도 log 파일로 redirect — Claude Code stdout이 cron 메일 노이즈 발생
- 실행 user는 plugin이 설치된 일반 user (root 금지)

### secret 저장 정책 (HARD)

PAT, Notion DB ID, OAuth refresh token은 **plist `EnvironmentVariables` 직접 기재 금지**. 이유:
- `~/Library/LaunchAgents/*.plist`는 평문 XML. Time Machine 백업 + iCloud 동기화 + macOS Backup tool 전부 평문으로 가져감
- `launchctl print` / `launchctl list -x` 출력에 환경변수 값이 노출됨
- spotlight indexing 대상

대신 **wrapper script가 Keychain/1Password에서 읽어와 export 후 claude 호출**:

```bash
#!/bin/bash
# ~/.claude-code-toolkit/bin/decision-log-keeper-wrapper.sh
set -euo pipefail

# Keychain 또는 1Password CLI에서 secret 로드 (둘 중 1)
export CLAUDE_CODE_TOOLKIT_GITHUB_PAT="$(security find-generic-password -s claude-code-toolkit-github-pat -w 2>/dev/null)"
export CLAUDE_CODE_TOOLKIT_DECISION_DB_ID="$(security find-generic-password -s claude-code-toolkit-decision-db-id -w 2>/dev/null)"
# 또는 1Password:
# export CLAUDE_CODE_TOOLKIT_GITHUB_PAT="$(op read 'op://Employee/claude-code-toolkit-github-pat/credential')"

# bypass 환경변수만 평문 (값 자체가 secret 아님)
export CLAUDE_CODE_TOOLKIT_CRON_MODE=1

# 실패 fast-fail — secret 로드 실패 시 cron 발화 회피
[ -z "${CLAUDE_CODE_TOOLKIT_GITHUB_PAT}" ] && { echo "[cron] PAT 로드 실패" >&2; exit 1; }

exec /usr/local/bin/claude -p "/claude-code-toolkit:decision-log-keeper"
```

권한 강화:
```
chmod 700 ~/.claude-code-toolkit/bin/decision-log-keeper-wrapper.sh
chmod 700 ~/.claude-code-toolkit/bin/
```

plist는 wrapper만 호출 (아래 예시 참조).

## decision-log-keeper (매일 18:00 KST)

### macOS launchd

`~/Library/LaunchAgents/<reverse-domain>.claude-code-toolkit.decision-log-keeper.plist`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key>
  <string><reverse-domain>.claude-code-toolkit.decision-log-keeper</string>

  <key>ProgramArguments</key>
  <array>
    <string>/bin/bash</string>
    <string>-c</string>
    <string>$HOME/.claude-code-toolkit/bin/decision-log-keeper-wrapper.sh</string>
  </array>

  <key>StartCalendarInterval</key>
  <dict>
    <key>Hour</key>
    <integer>18</integer>
    <key>Minute</key>
    <integer>0</integer>
  </dict>

  <!-- secret은 plist에 직접 적지 않음. wrapper script가 Keychain에서 fetch.
       비-secret bypass flag는 wrapper 안에서 export. -->

  <key>StandardOutPath</key>
  <string>/tmp/claude-code-toolkit-decision-log.out</string>
  <key>StandardErrorPath</key>
  <string>/tmp/claude-code-toolkit-decision-log.err</string>
</dict>
</plist>
```

**plist 권한:** `chmod 600 ~/Library/LaunchAgents/<reverse-domain>.claude-code-toolkit.decision-log-keeper.plist` — 다른 user가 plist를 읽을 수 없게. (secret이 plist에 없더라도 wrapper 경로 자체가 정찰 정보)

등록:

```
launchctl load ~/Library/LaunchAgents/<reverse-domain>.claude-code-toolkit.decision-log-keeper.plist
launchctl list | grep claude-code-toolkit
```

수동 trigger 테스트:

```
launchctl start <reverse-domain>.claude-code-toolkit.decision-log-keeper
```

### Linux systemd timer

`~/.config/systemd/user/claude-code-toolkit-decision-log.service`:

```ini
[Unit]
Description=Story Toolkit decision-log-keeper

[Service]
Type=oneshot
Environment=CLAUDE_CODE_TOOLKIT_CRON_MODE=1
EnvironmentFile=%h/.claude-code-toolkit/env  # PAT, DB ID 등 모든 secret 분리 보관
ExecStart=/usr/local/bin/claude -p /claude-code-toolkit:decision-log-keeper
StandardOutput=append:%h/.claude-code-toolkit/log/decision-log.out
StandardError=append:%h/.claude-code-toolkit/log/decision-log.err
```

`%h/.claude-code-toolkit/env` 형식:
```
CLAUDE_CODE_TOOLKIT_GITHUB_PAT=github_pat_...
CLAUDE_CODE_TOOLKIT_DECISION_DB_ID=...
```

**HARD: 권한 600 필수** — systemd는 600이 아니면 거부:
```
chmod 600 ~/.claude-code-toolkit/env
chmod 700 ~/.claude-code-toolkit/
```

systemd는 macOS Keychain 같은 system secret store가 없으므로 EnvironmentFile + 600 권한 + 디스크 암호화(LUKS)가 사실상의 baseline. 조직 표준에 HashiCorp Vault / AWS Secrets Manager가 있으면 wrapper script로 fetch.

`~/.config/systemd/user/claude-code-toolkit-decision-log.timer`:

```ini
[Unit]
Description=Daily 18:00 KST run

[Timer]
OnCalendar=*-*-* 09:00:00 UTC
Persistent=true

[Install]
WantedBy=timers.target
```

활성화:

```
systemctl --user daemon-reload
systemctl --user enable --now claude-code-toolkit-decision-log.timer
systemctl --user list-timers | grep claude-code-toolkit
```

## pr-storybook (매일 18:00 KST)

decision-log-keeper와 동일 패턴. 차이점:
- bypass 환경변수 **없음** — Slack `chat:write`는 현 release에 plugin hook 미정의 (README §보안 모델 참조). 향후 hook 추가 시 `CLAUDE_CODE_TOOLKIT_CRON_MODE=1` 적용
- ProgramArguments: `claude -p "/claude-code-toolkit:pr-storybook"`
- 추가 env: `CLAUDE_CODE_TOOLKIT_GITHUB_PAT`만

## spec-from-thread (수동 우선, cron 비권장)

thread URL을 입력으로 받기 때문에 cron 자동화 부적합. on-demand 호출만 권장.

## 실패 시 알림

cron 실패는 silent — alert 채널 별도 구성:

1. plist/timer ExecStart에 wrapper script 사용:
   ```bash
   #!/bin/bash
   /usr/local/bin/claude -p "/claude-code-toolkit:decision-log-keeper" || \
     curl -X POST "$SLACK_WEBHOOK" -d '{"text":"decision-log-keeper failed at '$(date)'"}'
   ```
2. audit log 비정상 종료 감시: `~/.claude-code-toolkit/audit/decision-log-YYYY-MM-DD.jsonl` 라인 수가 채널 메시지 수와 비교했을 때 0이면 alert

## 운영 체크리스트

- [ ] plist/timer 등록 후 1주 실제 실행 모니터링 (audit log 라인 누적 확인)
- [ ] PAT 만료 알림 (1년 주기) — 만료 시 cron silent fail 위험
- [ ] OAuth 토큰 만료 — Slack/Notion은 OAuth refresh 자동, 실패 시 audit log에 401 패턴 검색
- [ ] `~/.claude-code-toolkit/audit/` 디렉토리 cleanup 정책 (월 단위 archive 등)

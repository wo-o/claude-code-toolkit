# Cron Setup

`decision-log-keeper`는 자동 실행 가치가 큰 후보. Claude Code 자체 hook은 외부 시각 trigger를 받지 못하므로 OS scheduler를 사용한다. spec-from-thread는 본 릴리스에서 cron 비권장 (아래 §spec-from-thread 절 참조).

## 공통 원칙

- Claude Code CLI를 `-p` (print mode) 호출 — 비대화형
- bypass 환경변수는 plist/cron 환경에서만 주입 (shell 평소 환경 X)
- 실패 시 stderr를 별도 log 파일로 redirect — Claude Code stdout이 cron 메일 노이즈 발생
- 실행 user는 plugin이 설치된 일반 user (root 금지)

### 외부 통합 secret 정책

본 plugin은 자체 secret을 요구하지 않는다 — Slack/Notion/HubSpot은 Anthropic 공식 Connector(`claude.ai/customize/connectors`)를 통해 OAuth 인증되며, 토큰은 Anthropic 인프라에서 관리된다. 사용자 디스크에 PAT/refresh token이 저장되지 않으므로 plist `EnvironmentVariables` / `EnvironmentFile`에 secret을 적을 일이 없다.

bypass flag는 secret이 아니므로 wrapper script에서 평문 export 가능:

```bash
#!/bin/bash
# ~/.claude-code-toolkit/bin/decision-log-keeper-wrapper.sh
set -euo pipefail

# bypass 환경변수만 (값 자체가 secret 아님)
export CLAUDE_CODE_TOOLKIT_CRON_MODE=1

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

  <!-- Connector OAuth는 Anthropic 인프라가 관리. plist에 secret 없음.
       비-secret bypass flag는 wrapper 안에서 export. -->

  <key>StandardOutPath</key>
  <string>/tmp/claude-code-toolkit-decision-log.out</string>
  <key>StandardErrorPath</key>
  <string>/tmp/claude-code-toolkit-decision-log.err</string>
</dict>
</plist>
```

**plist 권한:** `chmod 600 ~/Library/LaunchAgents/<reverse-domain>.claude-code-toolkit.decision-log-keeper.plist` — 다른 user가 plist를 읽을 수 없게.

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
Description=claude-code-toolkit decision-log-keeper

[Service]
Type=oneshot
Environment=CLAUDE_CODE_TOOLKIT_CRON_MODE=1
ExecStart=/usr/local/bin/claude -p /claude-code-toolkit:decision-log-keeper
StandardOutput=append:%h/.claude-code-toolkit/log/decision-log.out
StandardError=append:%h/.claude-code-toolkit/log/decision-log.err
```

Connector 인증은 Anthropic 측에서 관리되므로 `EnvironmentFile`에 secret을 둘 필요가 없다. claude.ai에 사전 OAuth 완료 + 해당 user 세션이 활성 상태이기만 하면 cron 환경에서도 MCP 도구가 노출된다.

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

## spec-from-thread (수동 우선, cron 비권장)

thread URL을 입력으로 받기 때문에 cron 자동화 부적합. on-demand 호출만 권장. 또한 본 릴리스의 PreToolUse hook은 caller별 audit log를 분리하지 않으므로 `CLAUDE_CODE_TOOLKIT_CRON_MODE=1` bypass를 spec-from-thread에 사용하면 audit trail이 decision-log-keeper와 섞인다 (README §보안 모델 참조).

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
- [ ] Connector OAuth 만료 — Slack/Notion은 OAuth refresh 자동, 실패 시 audit log에 401 패턴 검색
- [ ] `~/.claude-code-toolkit/audit/` 디렉토리 cleanup 정책 (월 단위 archive 등)

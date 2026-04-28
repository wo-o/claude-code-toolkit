# Cron Setup

Eight of the fourteen skills are designed for automated execution. Claude Code's own hooks cannot receive external time triggers, so we lean on the OS scheduler (launchd on macOS, systemd timers on Linux). spec-from-thread and meeting-prep-pull are not recommended for cron in this release (see the §spec-from-thread section below — meeting-prep-pull is intentionally on-demand because it is invoked just-in-time before each meeting).

## Cron-enabled skills overview

| Skill | Cadence | Bypass env vars |
|---|---|---|
| decision-log-keeper | Daily 18:00 KST | `CLAUDE_CODE_TOOLKIT_CRON_MODE=1` |
| weekly-shipped-from-noise | Friday 17:00 KST | `CLAUDE_CODE_TOOLKIT_CRON_MODE=1` |
| blocker-radar | Weekdays 09:00 KST | `CLAUDE_CODE_TOOLKIT_CRON_MODE=1` |
| knowledge-graph-builder | Daily 18:00 KST (start `--review-mode`) | `CLAUDE_CODE_TOOLKIT_CRON_MODE=1` (Notion only — Slack post never bypassed) |
| okr-sync | Monday 09:00 KST | `CLAUDE_CODE_TOOLKIT_CRON_MODE=1` |
| thread-zombie-killer | Weekdays 09:00 KST | `CLAUDE_CODE_TOOLKIT_CRON_MODE=1` |
| inbox-zero-triage | Weekdays 08:00 KST | `CLAUDE_CODE_TOOLKIT_CRON_MODE=1` |
| follow-up-radar | Weekdays 09:00 KST | `CLAUDE_CODE_TOOLKIT_CRON_MODE=1` (only relevant when `--notion-mirror-page-id` is used) |

## Common principles

- Invoke the Claude Code CLI in `-p` (print mode) — non-interactive
- Inject bypass environment variables only inside the plist/cron environment (never in your normal shell)
- On failure, redirect stderr to a separate log file — Claude Code's stdout creates noise in cron mail
- Run as a regular user with the plugin installed (never root)

### External integration secret policy

The plugin requires no secrets of its own — Slack/Notion/HubSpot are authenticated through Anthropic's official Connectors (`claude.ai/customize/connectors`), and tokens are managed by Anthropic infrastructure. No PAT or refresh token lands on the user's disk, so there is never anything to put in plist `EnvironmentVariables` / `EnvironmentFile`.

The bypass flag is not a secret, so it is fine to export it in plain text from a wrapper script:

```bash
#!/bin/bash
# ~/.claude-code-toolkit/bin/decision-log-keeper-wrapper.sh
set -euo pipefail

# Bypass env var only (the value itself is not a secret)
export CLAUDE_CODE_TOOLKIT_CRON_MODE=1

exec /usr/local/bin/claude -p "/claude-code-toolkit:decision-log-keeper"
```

Tighten permissions:
```
chmod 700 ~/.claude-code-toolkit/bin/decision-log-keeper-wrapper.sh
chmod 700 ~/.claude-code-toolkit/bin/
```

The plist only invokes the wrapper (see example below).

## decision-log-keeper (daily, 18:00 KST)

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

  <!-- Connector OAuth is managed by Anthropic infrastructure. No secrets in this plist.
       The non-secret bypass flag is exported inside the wrapper. -->

  <key>StandardOutPath</key>
  <string>/tmp/claude-code-toolkit-decision-log.out</string>
  <key>StandardErrorPath</key>
  <string>/tmp/claude-code-toolkit-decision-log.err</string>
</dict>
</plist>
```

**plist permissions:** `chmod 600 ~/Library/LaunchAgents/<reverse-domain>.claude-code-toolkit.decision-log-keeper.plist` — so other users cannot read the plist.

Register:

```
launchctl load ~/Library/LaunchAgents/<reverse-domain>.claude-code-toolkit.decision-log-keeper.plist
launchctl list | grep claude-code-toolkit
```

Manual trigger to test:

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

Connector authentication is managed on Anthropic's side, so there is no need to put secrets in `EnvironmentFile`. As long as OAuth has been completed in claude.ai in advance and that user's session is active, MCP tools are exposed even in the cron environment.

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

Activate:

```
systemctl --user daemon-reload
systemctl --user enable --now claude-code-toolkit-decision-log.timer
systemctl --user list-timers | grep claude-code-toolkit
```

## weekly-shipped-from-noise (Friday, 17:00 KST)

### macOS launchd

`~/Library/LaunchAgents/<reverse-domain>.claude-code-toolkit.weekly-shipped.plist`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key>
  <string><reverse-domain>.claude-code-toolkit.weekly-shipped</string>

  <key>ProgramArguments</key>
  <array>
    <string>/bin/bash</string>
    <string>-c</string>
    <string>$HOME/.claude-code-toolkit/bin/weekly-shipped-wrapper.sh</string>
  </array>

  <key>StartCalendarInterval</key>
  <dict>
    <key>Weekday</key>
    <integer>5</integer>
    <key>Hour</key>
    <integer>17</integer>
    <key>Minute</key>
    <integer>0</integer>
  </dict>

  <key>StandardOutPath</key>
  <string>/tmp/claude-code-toolkit-weekly-shipped.out</string>
  <key>StandardErrorPath</key>
  <string>/tmp/claude-code-toolkit-weekly-shipped.err</string>
</dict>
</plist>
```

Wrapper (`~/.claude-code-toolkit/bin/weekly-shipped-wrapper.sh`):

```bash
#!/bin/bash
set -euo pipefail
export CLAUDE_CODE_TOOLKIT_CRON_MODE=1
export CLAUDE_CODE_TOOLKIT_SHIPPED_CHANNELS="#wins,#shipped,#launches,#product-updates"
export CLAUDE_CODE_TOOLKIT_SHIPPED_PARENT_ID="<notion-parent-page-id>"
WEEK=$(date +%G-W%V)
exec /usr/local/bin/claude -p "/claude-code-toolkit:weekly-shipped-from-noise --week ${WEEK}"
```

### Linux systemd timer

`~/.config/systemd/user/claude-code-toolkit-weekly-shipped.service`:

```ini
[Unit]
Description=claude-code-toolkit weekly-shipped-from-noise

[Service]
Type=oneshot
Environment=CLAUDE_CODE_TOOLKIT_CRON_MODE=1
Environment=CLAUDE_CODE_TOOLKIT_SHIPPED_CHANNELS=#wins,#shipped,#launches,#product-updates
Environment=CLAUDE_CODE_TOOLKIT_SHIPPED_PARENT_ID=<notion-parent-page-id>
ExecStart=/usr/local/bin/claude -p /claude-code-toolkit:weekly-shipped-from-noise
StandardOutput=append:%h/.claude-code-toolkit/log/weekly-shipped.out
StandardError=append:%h/.claude-code-toolkit/log/weekly-shipped.err
```

`~/.config/systemd/user/claude-code-toolkit-weekly-shipped.timer`:

```ini
[Unit]
Description=Friday 17:00 KST run

[Timer]
OnCalendar=Fri *-*-* 08:00:00 UTC
Persistent=true

[Install]
WantedBy=timers.target
```

## blocker-radar (weekdays, 09:00 KST)

### macOS launchd

`~/Library/LaunchAgents/<reverse-domain>.claude-code-toolkit.blocker-radar.plist`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key>
  <string><reverse-domain>.claude-code-toolkit.blocker-radar</string>

  <key>ProgramArguments</key>
  <array>
    <string>/bin/bash</string>
    <string>-c</string>
    <string>$HOME/.claude-code-toolkit/bin/blocker-radar-wrapper.sh</string>
  </array>

  <key>StartCalendarInterval</key>
  <array>
    <dict><key>Weekday</key><integer>1</integer><key>Hour</key><integer>9</integer><key>Minute</key><integer>0</integer></dict>
    <dict><key>Weekday</key><integer>2</integer><key>Hour</key><integer>9</integer><key>Minute</key><integer>0</integer></dict>
    <dict><key>Weekday</key><integer>3</integer><key>Hour</key><integer>9</integer><key>Minute</key><integer>0</integer></dict>
    <dict><key>Weekday</key><integer>4</integer><key>Hour</key><integer>9</integer><key>Minute</key><integer>0</integer></dict>
    <dict><key>Weekday</key><integer>5</integer><key>Hour</key><integer>9</integer><key>Minute</key><integer>0</integer></dict>
  </array>

  <key>StandardOutPath</key>
  <string>/tmp/claude-code-toolkit-blocker-radar.out</string>
  <key>StandardErrorPath</key>
  <string>/tmp/claude-code-toolkit-blocker-radar.err</string>
</dict>
</plist>
```

Wrapper (`~/.claude-code-toolkit/bin/blocker-radar-wrapper.sh`):

```bash
#!/bin/bash
set -euo pipefail
export CLAUDE_CODE_TOOLKIT_CRON_MODE=1
export CLAUDE_CODE_TOOLKIT_BLOCKER_CHANNELS="#eng,#product,#design,#ops"
export CLAUDE_CODE_TOOLKIT_BLOCKER_DB_ID="<notion-db-id>"
SINCE=$(date -v-1d +%Y-%m-%d 2>/dev/null || date -d "yesterday" +%Y-%m-%d)
exec /usr/local/bin/claude -p "/claude-code-toolkit:blocker-radar --since ${SINCE}"
```

### Linux systemd timer

`~/.config/systemd/user/claude-code-toolkit-blocker-radar.service`:

```ini
[Unit]
Description=claude-code-toolkit blocker-radar

[Service]
Type=oneshot
Environment=CLAUDE_CODE_TOOLKIT_CRON_MODE=1
Environment=CLAUDE_CODE_TOOLKIT_BLOCKER_CHANNELS=#eng,#product,#design,#ops
Environment=CLAUDE_CODE_TOOLKIT_BLOCKER_DB_ID=<notion-db-id>
ExecStart=/usr/local/bin/claude -p /claude-code-toolkit:blocker-radar
StandardOutput=append:%h/.claude-code-toolkit/log/blocker-radar.out
StandardError=append:%h/.claude-code-toolkit/log/blocker-radar.err
```

`~/.config/systemd/user/claude-code-toolkit-blocker-radar.timer`:

```ini
[Unit]
Description=Weekdays 09:00 KST run

[Timer]
OnCalendar=Mon..Fri *-*-* 00:00:00 UTC
Persistent=true

[Install]
WantedBy=timers.target
```

## knowledge-graph-builder (daily, 18:00 KST)

Run with `--review-mode` for the first 1-2 weeks before enabling auto-link.

### macOS launchd

`~/Library/LaunchAgents/<reverse-domain>.claude-code-toolkit.knowledge-graph.plist`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key>
  <string><reverse-domain>.claude-code-toolkit.knowledge-graph</string>

  <key>ProgramArguments</key>
  <array>
    <string>/bin/bash</string>
    <string>-c</string>
    <string>$HOME/.claude-code-toolkit/bin/knowledge-graph-wrapper.sh</string>
  </array>

  <key>StartCalendarInterval</key>
  <dict>
    <key>Hour</key>
    <integer>18</integer>
    <key>Minute</key>
    <integer>0</integer>
  </dict>

  <key>StandardOutPath</key>
  <string>/tmp/claude-code-toolkit-knowledge-graph.out</string>
  <key>StandardErrorPath</key>
  <string>/tmp/claude-code-toolkit-knowledge-graph.err</string>
</dict>
</plist>
```

Wrapper (`~/.claude-code-toolkit/bin/knowledge-graph-wrapper.sh`):

```bash
#!/bin/bash
set -euo pipefail
# CRON_MODE only bypasses Notion update_page hook. Slack post_message hook is never bypassed.
export CLAUDE_CODE_TOOLKIT_CRON_MODE=1
export CLAUDE_CODE_TOOLKIT_KG_CHANNELS="#eng,#product,#design,#research"
export CLAUDE_CODE_TOOLKIT_KG_WORKSPACE="<notion-workspace-id>"
# For the first 1-2 weeks, use --review-mode to validate match quality:
# exec /usr/local/bin/claude -p "/claude-code-toolkit:knowledge-graph-builder --review-mode"
exec /usr/local/bin/claude -p "/claude-code-toolkit:knowledge-graph-builder"
```

### Linux systemd timer

`~/.config/systemd/user/claude-code-toolkit-knowledge-graph.service`:

```ini
[Unit]
Description=claude-code-toolkit knowledge-graph-builder

[Service]
Type=oneshot
Environment=CLAUDE_CODE_TOOLKIT_CRON_MODE=1
Environment=CLAUDE_CODE_TOOLKIT_KG_CHANNELS=#eng,#product,#design,#research
Environment=CLAUDE_CODE_TOOLKIT_KG_WORKSPACE=<notion-workspace-id>
ExecStart=/usr/local/bin/claude -p /claude-code-toolkit:knowledge-graph-builder
StandardOutput=append:%h/.claude-code-toolkit/log/knowledge-graph.out
StandardError=append:%h/.claude-code-toolkit/log/knowledge-graph.err
```

`~/.config/systemd/user/claude-code-toolkit-knowledge-graph.timer`:

```ini
[Unit]
Description=Daily 18:00 KST run

[Timer]
OnCalendar=*-*-* 09:00:00 UTC
Persistent=true

[Install]
WantedBy=timers.target
```

## okr-sync (Monday, 09:00 KST)

### macOS launchd

`~/Library/LaunchAgents/<reverse-domain>.claude-code-toolkit.okr-sync.plist`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key>
  <string><reverse-domain>.claude-code-toolkit.okr-sync</string>

  <key>ProgramArguments</key>
  <array>
    <string>/bin/bash</string>
    <string>-c</string>
    <string>$HOME/.claude-code-toolkit/bin/okr-sync-wrapper.sh</string>
  </array>

  <key>StartCalendarInterval</key>
  <dict>
    <key>Weekday</key>
    <integer>1</integer>
    <key>Hour</key>
    <integer>9</integer>
    <key>Minute</key>
    <integer>0</integer>
  </dict>

  <key>StandardOutPath</key>
  <string>/tmp/claude-code-toolkit-okr-sync.out</string>
  <key>StandardErrorPath</key>
  <string>/tmp/claude-code-toolkit-okr-sync.err</string>
</dict>
</plist>
```

Wrapper (`~/.claude-code-toolkit/bin/okr-sync-wrapper.sh`):

```bash
#!/bin/bash
set -euo pipefail
export CLAUDE_CODE_TOOLKIT_CRON_MODE=1
export CLAUDE_CODE_TOOLKIT_OKR_DB_ID="<notion-okr-db-id>"
export CLAUDE_CODE_TOOLKIT_OKR_CHANNELS="#wins,#shipped,#launches,#product-updates"
exec /usr/local/bin/claude -p "/claude-code-toolkit:okr-sync --window-weeks 1"
```

### Linux systemd timer

`~/.config/systemd/user/claude-code-toolkit-okr-sync.service`:

```ini
[Unit]
Description=claude-code-toolkit okr-sync

[Service]
Type=oneshot
Environment=CLAUDE_CODE_TOOLKIT_CRON_MODE=1
Environment=CLAUDE_CODE_TOOLKIT_OKR_DB_ID=<notion-okr-db-id>
Environment=CLAUDE_CODE_TOOLKIT_OKR_CHANNELS=#wins,#shipped,#launches,#product-updates
ExecStart=/usr/local/bin/claude -p /claude-code-toolkit:okr-sync --window-weeks 1
StandardOutput=append:%h/.claude-code-toolkit/log/okr-sync.out
StandardError=append:%h/.claude-code-toolkit/log/okr-sync.err
```

`~/.config/systemd/user/claude-code-toolkit-okr-sync.timer`:

```ini
[Unit]
Description=Monday 09:00 KST run

[Timer]
OnCalendar=Mon *-*-* 00:00:00 UTC
Persistent=true

[Install]
WantedBy=timers.target
```

## thread-zombie-killer (weekdays, 09:00 KST)

### macOS launchd

`~/Library/LaunchAgents/<reverse-domain>.claude-code-toolkit.zombie-killer.plist`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key>
  <string><reverse-domain>.claude-code-toolkit.zombie-killer</string>

  <key>ProgramArguments</key>
  <array>
    <string>/bin/bash</string>
    <string>-c</string>
    <string>$HOME/.claude-code-toolkit/bin/zombie-killer-wrapper.sh</string>
  </array>

  <key>StartCalendarInterval</key>
  <array>
    <dict><key>Weekday</key><integer>1</integer><key>Hour</key><integer>9</integer><key>Minute</key><integer>0</integer></dict>
    <dict><key>Weekday</key><integer>2</integer><key>Hour</key><integer>9</integer><key>Minute</key><integer>0</integer></dict>
    <dict><key>Weekday</key><integer>3</integer><key>Hour</key><integer>9</integer><key>Minute</key><integer>0</integer></dict>
    <dict><key>Weekday</key><integer>4</integer><key>Hour</key><integer>9</integer><key>Minute</key><integer>0</integer></dict>
    <dict><key>Weekday</key><integer>5</integer><key>Hour</key><integer>9</integer><key>Minute</key><integer>0</integer></dict>
  </array>

  <key>StandardOutPath</key>
  <string>/tmp/claude-code-toolkit-zombie-killer.out</string>
  <key>StandardErrorPath</key>
  <string>/tmp/claude-code-toolkit-zombie-killer.err</string>
</dict>
</plist>
```

Wrapper (`~/.claude-code-toolkit/bin/zombie-killer-wrapper.sh`):

```bash
#!/bin/bash
set -euo pipefail
export CLAUDE_CODE_TOOLKIT_CRON_MODE=1
export CLAUDE_CODE_TOOLKIT_ZOMBIE_CHANNELS="#eng,#product,#design,#ops,#general"
export CLAUDE_CODE_TOOLKIT_ZOMBIE_DB_ID="<notion-zombie-db-id>"
exec /usr/local/bin/claude -p "/claude-code-toolkit:thread-zombie-killer"
```

### Linux systemd timer

`~/.config/systemd/user/claude-code-toolkit-zombie-killer.service`:

```ini
[Unit]
Description=claude-code-toolkit thread-zombie-killer

[Service]
Type=oneshot
Environment=CLAUDE_CODE_TOOLKIT_CRON_MODE=1
Environment=CLAUDE_CODE_TOOLKIT_ZOMBIE_CHANNELS=#eng,#product,#design,#ops,#general
Environment=CLAUDE_CODE_TOOLKIT_ZOMBIE_DB_ID=<notion-zombie-db-id>
ExecStart=/usr/local/bin/claude -p /claude-code-toolkit:thread-zombie-killer
StandardOutput=append:%h/.claude-code-toolkit/log/zombie-killer.out
StandardError=append:%h/.claude-code-toolkit/log/zombie-killer.err
```

`~/.config/systemd/user/claude-code-toolkit-zombie-killer.timer`:

```ini
[Unit]
Description=Weekdays 09:00 KST run

[Timer]
OnCalendar=Mon..Fri *-*-* 00:00:00 UTC
Persistent=true

[Install]
WantedBy=timers.target
```

## inbox-zero-triage (weekdays, 08:00 KST)

### macOS launchd

`~/Library/LaunchAgents/<reverse-domain>.claude-code-toolkit.inbox-triage.plist`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key>
  <string><reverse-domain>.claude-code-toolkit.inbox-triage</string>

  <key>ProgramArguments</key>
  <array>
    <string>/bin/bash</string>
    <string>-c</string>
    <string>$HOME/.claude-code-toolkit/bin/inbox-triage-wrapper.sh</string>
  </array>

  <key>StartCalendarInterval</key>
  <array>
    <dict><key>Weekday</key><integer>1</integer><key>Hour</key><integer>8</integer><key>Minute</key><integer>0</integer></dict>
    <dict><key>Weekday</key><integer>2</integer><key>Hour</key><integer>8</integer><key>Minute</key><integer>0</integer></dict>
    <dict><key>Weekday</key><integer>3</integer><key>Hour</key><integer>8</integer><key>Minute</key><integer>0</integer></dict>
    <dict><key>Weekday</key><integer>4</integer><key>Hour</key><integer>8</integer><key>Minute</key><integer>0</integer></dict>
    <dict><key>Weekday</key><integer>5</integer><key>Hour</key><integer>8</integer><key>Minute</key><integer>0</integer></dict>
  </array>

  <key>StandardOutPath</key>
  <string>/tmp/claude-code-toolkit-inbox-triage.out</string>
  <key>StandardErrorPath</key>
  <string>/tmp/claude-code-toolkit-inbox-triage.err</string>
</dict>
</plist>
```

Wrapper (`~/.claude-code-toolkit/bin/inbox-triage-wrapper.sh`):

```bash
#!/bin/bash
set -euo pipefail
export CLAUDE_CODE_TOOLKIT_CRON_MODE=1
export CLAUDE_CODE_TOOLKIT_INBOX_DB_ID="<notion-inbox-triage-db-id>"
exec /usr/local/bin/claude -p "/claude-code-toolkit:inbox-zero-triage --since-hours 24"
```

### Linux systemd timer

`~/.config/systemd/user/claude-code-toolkit-inbox-triage.service`:

```ini
[Unit]
Description=claude-code-toolkit inbox-zero-triage

[Service]
Type=oneshot
Environment=CLAUDE_CODE_TOOLKIT_CRON_MODE=1
Environment=CLAUDE_CODE_TOOLKIT_INBOX_DB_ID=<notion-inbox-triage-db-id>
ExecStart=/usr/local/bin/claude -p "/claude-code-toolkit:inbox-zero-triage --since-hours 24"
StandardOutput=append:%h/.claude-code-toolkit/log/inbox-triage.out
StandardError=append:%h/.claude-code-toolkit/log/inbox-triage.err
```

`~/.config/systemd/user/claude-code-toolkit-inbox-triage.timer`:

```ini
[Unit]
Description=Weekdays 08:00 KST run

[Timer]
OnCalendar=Mon..Fri *-*-* 23:00:00 UTC
Persistent=true

[Install]
WantedBy=timers.target
```

## follow-up-radar (weekdays, 09:00 KST)

By default this skill is read-only and prints to stdout. Cron is most useful when paired with `--notion-mirror-page-id` so the personal dashboard is also pushed to a Notion page each morning.

### macOS launchd

`~/Library/LaunchAgents/<reverse-domain>.claude-code-toolkit.follow-up-radar.plist`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key>
  <string><reverse-domain>.claude-code-toolkit.follow-up-radar</string>

  <key>ProgramArguments</key>
  <array>
    <string>/bin/bash</string>
    <string>-c</string>
    <string>$HOME/.claude-code-toolkit/bin/follow-up-radar-wrapper.sh</string>
  </array>

  <key>StartCalendarInterval</key>
  <array>
    <dict><key>Weekday</key><integer>1</integer><key>Hour</key><integer>9</integer><key>Minute</key><integer>0</integer></dict>
    <dict><key>Weekday</key><integer>2</integer><key>Hour</key><integer>9</integer><key>Minute</key><integer>0</integer></dict>
    <dict><key>Weekday</key><integer>3</integer><key>Hour</key><integer>9</integer><key>Minute</key><integer>0</integer></dict>
    <dict><key>Weekday</key><integer>4</integer><key>Hour</key><integer>9</integer><key>Minute</key><integer>0</integer></dict>
    <dict><key>Weekday</key><integer>5</integer><key>Hour</key><integer>9</integer><key>Minute</key><integer>0</integer></dict>
  </array>

  <key>StandardOutPath</key>
  <string>/tmp/claude-code-toolkit-follow-up-radar.out</string>
  <key>StandardErrorPath</key>
  <string>/tmp/claude-code-toolkit-follow-up-radar.err</string>
</dict>
</plist>
```

Wrapper (`~/.claude-code-toolkit/bin/follow-up-radar-wrapper.sh`):

```bash
#!/bin/bash
set -euo pipefail
export CLAUDE_CODE_TOOLKIT_CRON_MODE=1
export CLAUDE_CODE_TOOLKIT_FOLLOWUP_MY_EMAIL="<your-gmail-address>"
# Optional: mirror the report to a personal Notion page (overwrites each run)
# CLAUDE_CODE_TOOLKIT_FOLLOWUP_NOTION_PAGE_ID="<page-id>"
exec /usr/local/bin/claude -p "/claude-code-toolkit:follow-up-radar ${CLAUDE_CODE_TOOLKIT_FOLLOWUP_NOTION_PAGE_ID:+--notion-mirror-page-id $CLAUDE_CODE_TOOLKIT_FOLLOWUP_NOTION_PAGE_ID}"
```

### Linux systemd timer

`~/.config/systemd/user/claude-code-toolkit-follow-up-radar.service`:

```ini
[Unit]
Description=claude-code-toolkit follow-up-radar

[Service]
Type=oneshot
Environment=CLAUDE_CODE_TOOLKIT_CRON_MODE=1
Environment=CLAUDE_CODE_TOOLKIT_FOLLOWUP_MY_EMAIL=<your-gmail-address>
ExecStart=/usr/local/bin/claude -p /claude-code-toolkit:follow-up-radar
StandardOutput=append:%h/.claude-code-toolkit/log/follow-up-radar.out
StandardError=append:%h/.claude-code-toolkit/log/follow-up-radar.err
```

`~/.config/systemd/user/claude-code-toolkit-follow-up-radar.timer`:

```ini
[Unit]
Description=Weekdays 09:00 KST run

[Timer]
OnCalendar=Mon..Fri *-*-* 00:00:00 UTC
Persistent=true

[Install]
WantedBy=timers.target
```

## spec-from-thread (manual-first, cron not recommended)

It takes a thread URL as input, which makes cron automation a poor fit. Recommend on-demand invocation only. In addition, this release's PreToolUse hook does not separate audit logs by caller, so using `CLAUDE_CODE_TOOLKIT_CRON_MODE=1` for spec-from-thread would mix its audit trail with decision-log-keeper's (see README §Security model).

## meeting-prep-pull (on-demand, cron not recommended)

This skill runs just-in-time before each meeting — the operator typically invokes it 5-15 minutes ahead of a calendar event. A daily cron would either pre-bake stale pre-reads (the email/Notion context drifts within hours of generation) or fire at the wrong granularity. Keep it on-demand.

## Failure notifications

cron failures are silent — set up a separate alert channel for every cron-enabled skill:

1. Wrap the claude invocation with a fallback `curl` to a Slack webhook. Example for `weekly-shipped-from-noise`:
   ```bash
   #!/bin/bash
   set -euo pipefail
   export CLAUDE_CODE_TOOLKIT_CRON_MODE=1
   /usr/local/bin/claude -p "/claude-code-toolkit:weekly-shipped-from-noise" || \
     curl -X POST "$SLACK_WEBHOOK" \
       -H 'Content-Type: application/json' \
       -d "{\"text\":\":warning: weekly-shipped-from-noise failed at $(date)\"}"
   ```
   The `$SLACK_WEBHOOK` value is treated as a secret — store in keychain (macOS `security add-generic-password` + read in the wrapper) or systemd `LoadCredential=`, never in plain `EnvironmentVariables` of the plist
2. Watch for abnormal audit log termination — alert when the day's audit log has zero lines but the source channels show activity. Per-skill audit log path:
   - decision-log-keeper: `~/.claude-code-toolkit/audit/decision-log-YYYY-MM-DD.jsonl`
   - weekly-shipped-from-noise: `~/.claude-code-toolkit/audit/weekly-shipped-YYYY-Www.jsonl`
   - blocker-radar: `~/.claude-code-toolkit/audit/blocker-radar-YYYY-MM-DD.jsonl`
   - knowledge-graph-builder: `~/.claude-code-toolkit/audit/knowledge-graph-YYYY-MM-DD.jsonl`
   - okr-sync: `~/.claude-code-toolkit/audit/okr-sync-YYYY-Www.jsonl`
   - thread-zombie-killer: `~/.claude-code-toolkit/audit/zombie-killer-YYYY-MM-DD.jsonl`
   - inbox-zero-triage: `~/.claude-code-toolkit/audit/inbox-triage-YYYY-MM-DD.jsonl`
   - follow-up-radar: `~/.claude-code-toolkit/audit/follow-up-radar-YYYY-MM-DD.jsonl` (only when `--notion-mirror-page-id` is set)

## Operations checklist

- [ ] After registering each plist/timer, monitor real execution for one week (confirm per-skill audit log lines accumulating)
- [ ] Connector OAuth expiry — Slack/Notion auto-refresh OAuth; on failure, search the audit log for 401 patterns
- [ ] `~/.claude-code-toolkit/audit/` directory cleanup policy (recommended: `find ~/.claude-code-toolkit/audit -mtime +30 -delete` weekly)
- [ ] `knowledge-graph-builder`: run with `--review-mode` for the first 1-2 weeks; only switch to direct write after the precision is verified
- [ ] `okr-sync`: reset `Evidence count (this quarter)` at quarter boundaries (manual step, not part of the cron)
- [ ] `thread-zombie-killer`: frame as a personal dashboard, not a team scoreboard — never broadcast the `Top responders with backlog` line publicly
- [ ] `inbox-zero-triage`: tune the ticket-classifier prompt with 2-3 organization-specific examples after the first week (newsletter false negatives are the most common failure)
- [ ] `follow-up-radar`: keep the report private — never copy into a shared Notion page or Slack channel. Default cron uses stdout only; `--notion-mirror-page-id` should point to a personal page
- [ ] `meeting-prep-pull`: confirm `CLAUDE_CODE_TOOLKIT_PREP_INTERNAL_DOMAIN` covers all of the operator's organization domains (comma-separated for multiple)

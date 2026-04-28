---
name: decision-log-keeper
description: Extract decision messages from a Slack channel for a given date and append them as rows to a Notion "Decision Log" database. Designed to run daily via launchd cron, but can be invoked manually with --date for backfill or workshop demo.
---

# decision-log-keeper

**Workshop pick #1 (H+X warm-up).** Each day, automatically moves decisions dropped into the Slack #decisions channel during PM meetings into a Notion DB. Permanently preserved so that, at quarterly retros, you can trace "Why did we decide it that way back then?". See `docs/workshop-flow.md` for the demo sequence.

## Input

- `--date YYYY-MM-DD` (optional, defaults to today) — the target date to extract
- `--channel #channel-name` (optional, default `#decisions`) — Slack channel
- `--notion-db-id <id>` (optional, falls back to env `CLAUDE_CODE_TOOLKIT_DECISION_DB_ID`) — Notion DB ID

## Output

Prints the number of rows added + the Notion URL of each row. One-line console summary:

```
2026-04-26: extracted N decisions from #decisions → added N rows to Notion DB
- <Notion row URL 1>
- <Notion row URL 2>
- ...
```

## Procedure

### 1. Parse input

- `--date` missing → today (timezone Asia/Seoul, KST)
- `--channel` missing → `#decisions`
- `--notion-db-id` missing AND `CLAUDE_CODE_TOOLKIT_DECISION_DB_ID` env missing → AskUserQuestion

### 2. Collect Slack messages

Call `mcp__claude_ai_Slack__slack_read_channel`:
- `channel`: the input channel's ID (resolve name → ID via `mcp__claude_ai_Slack__slack_search_channels`)
- `oldest`: Unix timestamp for 00:00 KST on the input date
- `latest`: Unix timestamp for 23:59 KST on the input date
- `limit`: 200

N messages returned. If 0, print "No messages on that date" and exit.

### 3. Filter decision messages (LLM)

Classify the N collected messages with the LLM:
- **decision**: declarative decisions like "going with X", "decided on Y", "finalized Z". Prefer messages tagged with approval reactions like `:white_check_mark:` or `:approved:`.
- **chatter**: greetings, weather, lunch, etc.
- **question**: ends in `?` or seeks an answer

Keep decisions only — drop chatter and questions.

Attach the following metadata to each decision:
- `Author`: resolve user_id → display_name via `mcp__claude_ai_Slack__slack_read_user_profile`
- `Time`: ISO 8601 message timestamp
- `Context (one line)`: a one-line summary of the decision's background, drawn from 1-2 prior messages in the same thread or 1 prior message in the channel
- `Slack permalink`: format `https://<workspace>.slack.com/archives/<channel-id>/p<ts>`

### 4. Add Notion DB rows

For each decision, call `mcp__claude_ai_Notion__notion-create-pages`:
- `parent`: `database_id` = the input DB ID
- `properties`:
  - `Title` (title): one-line summary of the decision (≤ 50 chars)
  - `Date` (date): decision time
  - `Author` (rich_text): Slack display_name
  - `Context` (rich_text): one-line context
  - `Slack Link` (url): Slack permalink
- `children`: full text of the decision (paragraph block)

**Hook behavior:** the plugin's `PreToolUse(mcp__claude_ai_Notion__notion-create-pages)` hook fires on the first row creation — asks the user once "About to add N rows to the Notion DB. Proceed?". In cron mode, set `CLAUDE_CODE_TOOLKIT_CRON_MODE=1` to bypass the hook and only record an audit log entry (audit log path = `~/.claude-code-toolkit/audit/decision-log-YYYY-MM-DD.jsonl`).

### 5. Output

Print rows added + each row's Notion URL in the one-line summary format. Done.

## Automated operation (cron)

Recommend launchd cron at 18:00 KST daily:

```
# ~/Library/LaunchAgents/<reverse-domain>.claude-code-toolkit.decision-log-keeper.plist
# Program: /usr/local/bin/claude
# ProgramArguments: -p "/claude-code-toolkit:decision-log-keeper"
# StartCalendarInterval: { Hour=18, Minute=0 }
# EnvironmentVariables:
#   CLAUDE_CODE_TOOLKIT_CRON_MODE=1
#   CLAUDE_CODE_TOOLKIT_DECISION_DB_ID=<id>
```

See `docs/cron-setup.md` for the full plist example.

## Workshop demo (T1 explicit invocation)

```
> /claude-code-toolkit:decision-log-keeper --date 2026-04-26
```

Pre-demo setup:
- One-time creation of the Notion "Decision Log" DB (Title / Date / Author / Context / Slack Link columns)
- Seed 1-2 dummy decision messages into `#decisions` (e.g. "Decided to change token vesting from 6 months → 12 months")
- Presenter PC has `CLAUDE_CODE_TOOLKIT_DECISION_DB_ID` env var set + Slack/Notion MCP authenticated

30-second demo flow:
1. Type the command (5s)
2. Slack message fetch + LLM decision extraction (10s)
3. Notion DB row creation + user confirmation prompt → y (5s)
4. Presenter refreshes Notion UI to show the row appear (10s)

Key talking point: "Normally a daily 18:00 cron does this automatically. For the workshop we ran the same command manually with `--date`."

## Pitfalls

- **Slack timezone confusion**: `oldest`/`latest` are Unix timestamps (UTC). For `--date 2026-04-26`, convert KST 00:00-23:59 to UTC before passing.
- **Decision/chatter LLM classification too loose**: false positives (chatter classified as decisions) flood the Notion DB with noise. Force this in the system prompt: "A decision = an explicit decision verb (`finalize/decide/confirm/pick`) or an approval reaction (:white_check_mark: :approved:)".
- **Notion DB schema drift**: if the DB columns or types change, `create_page` fails. On the first call, run `mcp__claude_ai_Notion__notion-fetch <db-id>` once to validate the schema before proceeding.
- **In cron mode, the hook prompt blocks**: use `CLAUDE_CODE_TOOLKIT_CRON_MODE=1` to bypass the hook and only record the audit log. The plugin checks this env var in hooks.json.

## Tone

- Quote the decision text verbatim — no paraphrasing
- Do not anonymize the author — keep display_name as-is
- The one-line context is "why this decision came up" in one sentence — no speculation

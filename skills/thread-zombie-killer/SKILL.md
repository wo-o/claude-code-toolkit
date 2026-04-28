---
name: thread-zombie-killer
description: Find Slack mention threads that received no reply within configurable age tiers (3 / 7 / 14 days), classify each by who-owes-whom, and upsert rows into a Notion "Open Mentions" DB grouped by responder. Designed for daily 09:00 launchd cron, surfacing dormant requests before they become escalations.
---

# thread-zombie-killer

A teammate `@mentions` you on Tuesday. You see it, plan to reply that evening, and forget. By Friday it's buried. By the next Monday it's lost entirely. Multiply by every team member, every channel — these dormant zombies are the largest source of trust erosion in remote teams. People mostly don't notice they are being ignored; they just stop trusting that messages get answered. This skill scans the workspace daily, finds mentions that have crossed the 3-day / 7-day / 14-day silence threshold, and surfaces them in Notion grouped by the person who owes the reply.

Zero-config: this skill auto-resolves the Notion DB "Open Mentions" on first run and caches the ID — see docs/auto-resolve.md. The channel set is auto-derived by first searching for mentions of the watched users (or the authenticated user) workspace-wide, then narrowing scans to the channels those mentions appeared in — never a blind scan of every public channel.

## Input

- `--scan-channels <comma-separated>` (override only — defaults to channels auto-derived from `slack_search_public` for `@<watched-user>` mentions in the last 21 days. Env override: `CLAUDE_CODE_TOOLKIT_ZOMBIE_CHANNELS`)
- `--watch-users <comma-separated handles>` (optional) — narrow surveillance to specific responders (e.g. team leads). When omitted, every mentioned user is tracked
- `--notion-db-id <id>` (override only — defaults to auto-resolve Notion DB "Open Mentions". Env override: `CLAUDE_CODE_TOOLKIT_ZOMBIE_DB_ID`)
- `--exclude-bots` (optional flag, default true) — skip mentions from/to bots

## Output

```
2026-04-27 (cron 09:00): scanned C channels, M mentions tracked → tier-3d X · tier-7d Y · tier-14d Z (resolved-since-last-run R)
- Open Mentions DB: <URL>
- Top responders with backlog: @user1 (N) · @user2 (M) · @user3 (P)
```

The Notion DB schema (one-time setup; the skill validates with `mcp__claude_ai_Notion__notion-fetch` before writing):

| Column | Type | Content |
|---|---|---|
| Title | title | One-line summary of the original ask |
| Asker | rich_text | Slack display_name of who posted the mention |
| Responder | rich_text | Slack display_name of who was mentioned |
| Channel | rich_text | Channel name |
| Posted at | date | Original mention timestamp |
| Age (days) | number | Auto-computed days since posted |
| Tier | select | 3d / 7d / 14d / Resolved |
| Tone | select | Question / Action request / FYI / Unclear |
| Source | url | Slack permalink |
| Last sync | date | Run timestamp |

## Procedure

### 1. Parse input

- Slack channels: when `--scan-channels` and env are missing, run `mcp__claude_ai_Slack__slack_search_public` for the last 21 days with the query `@<watched-user>` (one search per watched user, or the authenticated user when `--watch-users` is omitted). Take the unique set of channel IDs from the results and use that as the scan set. This avoids the "every public channel" rate-limit blast — the skill only re-reads channels where a relevant mention has actually been observed recently.
- Notion DB ID: auto-resolve "Open Mentions" via the algorithm in docs/auto-resolve.md (cache-first, fall through to notion-search). Override: `--notion-db-id <id>` or env `CLAUDE_CODE_TOOLKIT_ZOMBIE_DB_ID`.
- Validate the resolved Notion DB schema with `mcp__claude_ai_Notion__notion-fetch` once

### 2. Find mentions

For each channel, call `mcp__claude_ai_Slack__slack_read_channel` over the last 21 days (longest tier + buffer). Filter messages whose body contains an `@user` reference. Resolve user_ids → display_names via `mcp__claude_ai_Slack__slack_read_user_profile` (cached).

For thread parents with at least one mention, also fetch thread replies (so the reply check is accurate).

### 3. Determine reply state per mention

For each mention, check the thread (or, if not a thread, the next 24 hours of channel messages) for any reply by the mentioned user. A "reply" counts as:

- A message in the thread by the mentioned user
- A reaction by the mentioned user (`:eyes:` / `:white_check_mark:` / `:thumbsup:` count as acknowledgement)
- A new top-level message by the mentioned user explicitly referencing the asker (`@asker`) within the same channel within 6 hours

If any reply exists, mark `Tier: Resolved` and skip ahead to step 6 (DB upsert closes the row).

If no reply: compute age in days. Assign tier:
- 1 ≤ age < 3 days → not yet a zombie, **drop**
- 3 ≤ age < 7 days → `Tier: 3d`
- 7 ≤ age < 14 days → `Tier: 7d`
- age ≥ 14 days → `Tier: 14d`

### 4. Classify tone (LLM)

For each surviving mention, invoke the LLM with the body + a small classifier prompt:

```
Slack message: <body>
Mention: @<responder>

Classify the tone as exactly one of [Question / Action request / FYI / Unclear].
- Question: ends in `?`, "how", "what", "where", "when", "why", "did you"
- Action request: imperative verb directed at @responder (e.g. "please review", "check this", "approve")
- FYI: information-only with the @ as a heads-up (no expected reply)
- Unclear: cannot determine

Output: classification + 1-line rationale.
```

Mentions classified as `FYI` are still surfaced (some FYIs were actually action requests in disguise) but with that label, so the responder can quickly drop them.

### 5. Group by responder

Aggregate mentions per responder. The responder dashboard counts feed the output's "Top responders with backlog" line.

### 6. Upsert into Notion DB

For each mention:

- Search the DB for an existing row with the same `Source` URL
- If found → `update_page` (Tier may have advanced from 3d → 7d, Age advances daily, Last sync updates). If `Tier: Resolved`, set the Resolved status and stop touching the row in future runs (operator can archive)
- If not found and `Tier ≠ Resolved` → `create_page`
- If not found and `Tier: Resolved` → skip (no point creating a closed row)

Call `mcp__claude_ai_Notion__notion-create-pages` or `mcp__claude_ai_Notion__notion-update-page` accordingly.

**Hook behavior:** the plugin's `PreToolUse(mcp__claude_ai_Notion__notion-create-pages)` and `PreToolUse(mcp__claude_ai_Notion__notion-update-page)` hooks fire on the first call per run — asks once "About to upsert N mention rows. Proceed?". In cron mode set `CLAUDE_CODE_TOOLKIT_CRON_MODE=1` to bypass + write only an audit log entry (`~/.claude-code-toolkit/audit/zombie-killer-YYYY-MM-DD.jsonl`).

### 7. Output

Print the one-line console summary + DB URL + top-responder backlog counts. Done.

## Automated operation (cron)

Recommended launchd cron at weekday 09:00 KST:

```
# ~/Library/LaunchAgents/<reverse-domain>.claude-code-toolkit.zombie-killer.plist
# Program: /usr/local/bin/claude
# ProgramArguments: -p "/claude-code-toolkit:thread-zombie-killer"
# StartCalendarInterval: [{Weekday=1..5, Hour=9, Minute=0}]
# EnvironmentVariables:
#   CLAUDE_CODE_TOOLKIT_CRON_MODE=1
#   # Optional — only set if you want to override auto-resolve
#   CLAUDE_CODE_TOOLKIT_ZOMBIE_CHANNELS=#eng,#product,#design,#ops,#general
#   CLAUDE_CODE_TOOLKIT_ZOMBIE_DB_ID=<id>
```

See `docs/cron-setup.md` for the full plist example.

## Pitfalls

- **Reactions as "reply"**: a `:eyes:` reaction means "I see it" but not "I will act". Allow it as a reply to avoid false positives, but consider whether your team's culture treats `:eyes:` as a true acknowledgement before deploying
- **DM scope**: this skill scans channels only. DMs and group DMs would require separate scopes and are higher risk — explicitly out of scope for v1
- **Bot mentions**: bots get @-mentioned constantly (CI bots, deployment bots). Default `--exclude-bots=true` is correct; setting it to false floods the DB
- **Vacation / OOO**: someone on a known vacation should not appear in tier-7d. There is no calendar integration in v1 — operators must manually mark Resolved or filter the Notion view by responder availability
- **Thread of 1 self-reply**: when the asker replies to themselves and tags the responder again, it counts as the same outstanding ask. Group mentions by `(asker, responder, channel, day)` before tier assignment
- **Thread that resolves out of band**: the responder may answer over DM or in a meeting. The DB will still show it as open. Provide an obvious "mark Resolved" button in the Notion view (manual)
- **Surveillance optics**: explicit framing matters. This skill helps responders catch dropped balls — frame it as a personal dashboard, not a team scoreboard. Avoid posting the "Top responders with backlog" line publicly

## Tone

- Quote the original mention verbatim — no rephrasing
- Tier is a neutral fact, never editorial ("very late", "ignored", etc.)
- The output's "Top responders with backlog" line is for the operator, not for broadcast
- Resolved rows are kept in the DB for audit, not deleted — operators can archive at quarter boundaries

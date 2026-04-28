---
name: follow-up-radar
description: Scan Sent mail for threads where the operator's last message has gone unanswered for 3 / 7 / 14+ days, group by recipient, and produce a personal follow-up dashboard. Pure read-only by default — no Notion / Slack writes. Designed for daily 09:00 launchd cron, optional Notion mirror via flag.
---

# follow-up-radar

The personal mirror of `thread-zombie-killer` for Gmail. Email follow-ups die in two ways: the recipient forgets, or the operator forgets they sent it. This skill surfaces the operator's own outbound threads that have gone quiet, tiered by how many days they have been waiting. Like thread-zombie-killer, this is a **personal dashboard** — never broadcast, never published in a shared place by default.

The skill is **read-only by design**. It does not draft replies, does not post to Slack, and does not write to Notion unless `--notion-mirror-page-id` is explicitly passed.

## Input

- `--my-email <email>` (optional, falls back to env `CLAUDE_CODE_TOOLKIT_FOLLOWUP_MY_EMAIL`) — the authenticated Gmail address (used to detect "the operator was the last sender")
- `--tiers 3,7,14` (optional, default `3,7,14`) — tier days for `Soft`, `Hard`, `Cold` follow-up buckets
- `--ignore-labels <list>` (optional, default `Newsletter,Automated,No-reply`) — skip threads with any of these labels (most newsletters are one-way and never expected to reply)
- `--ignore-domains <list>` (optional) — skip recipient domains (e.g. `noreply.github.com`)
- `--notion-mirror-page-id <id>` (optional) — if set, mirror the report into a Notion page (overwrites the page each run)
- Missing required input → AskUserQuestion (never guess)

## Output

```
2026-04-28: scanned N sent threads → S still-waiting (Soft 3d a / Hard 7d b / Cold 14d c)
- Top recipient backlog: <name1> × x, <name2> × y, <name3> × z
- Notion mirror: <page URL> (only if --notion-mirror-page-id is set)
```

Console report format (markdown to stdout):

```
# Follow-up Radar — YYYY-MM-DD

## Soft (3-6 days, n)
- <recipient name> <recipient email> — "<subject>" (sent <YYYY-MM-DD>, <thread URL>)
- ...

## Hard (7-13 days, n)
- ...

## Cold (14+ days, n)
- ...

## By recipient
- <recipient> × <count> threads pending
- ...
```

## Procedure

### 1. Parse input

- `--my-email` missing AND env missing → AskUserQuestion (the skill cannot detect "still waiting" without knowing the operator's address)
- `--tiers` validate: 3 ascending integers
- `--ignore-labels` defaults to a sensible newsletter-shaped list

### 2. Pull Sent mail

Call `mcp__gmail__search_threads` with the query:

```
in:sent newer_than:<max-tier + 7>d -label:<ignore-label-1> -label:<ignore-label-2> ...
```

`max-tier + 7` provides a buffer so threads that *just* passed the Cold tier still show up. Limit 200.

For each returned thread, call `mcp__gmail__get_thread` to inspect the messages list.

### 3. Detect "still waiting" threads

A thread is `still_waiting` iff:

- The latest message author is `--my-email`
- AND the latest message is older than `tiers[0]` days
- AND none of the recipients replied after the operator's latest message
- AND the thread is not auto-generated (`From: noreply@...`, `List-Unsubscribe` header present → drop)

For each surviving thread, extract:
- `recipients`: To + Cc minus self
- `subject`: latest message subject
- `last_sent_ts`: the operator's latest message timestamp
- `age_days`: (now - last_sent_ts).days
- `tier`: `Soft` if 3 ≤ age < 7, `Hard` if 7 ≤ age < 14, `Cold` if age ≥ 14

### 4. Group + render

- Sort each tier by `age_days` descending (most-cold first)
- Compute `By recipient` aggregate: count of pending threads per top-level recipient (To address)
- Render the markdown report

### 5. Optional Notion mirror

If `--notion-mirror-page-id` is set:

- Fetch the existing page contents via `mcp__notion__fetch`
- Replace the body with the new markdown report (single-page overwrite — there is no DB upsert)
- Call `mcp__notion__update_page`

**Hook behavior:** when `--notion-mirror-page-id` is passed, the plugin's `PreToolUse(mcp__notion__update_page)` hook fires once per run — asks "About to overwrite the follow-up radar page. Proceed?". In cron mode set `CLAUDE_CODE_TOOLKIT_CRON_MODE=1` to bypass + audit log (`~/.claude-code-toolkit/audit/follow-up-radar-YYYY-MM-DD.jsonl`).

### 6. Output

Print the one-line console summary + (optional) Notion URL. Done.

## Automated operation (cron)

Recommended launchd cron at every weekday 09:00 KST:

```
# ~/Library/LaunchAgents/<reverse-domain>.claude-code-toolkit.follow-up-radar.plist
# Program: /usr/local/bin/claude
# ProgramArguments: -p "/claude-code-toolkit:follow-up-radar"
# StartCalendarInterval: [{Weekday=1..5, Hour=9, Minute=0}]
# EnvironmentVariables:
#   CLAUDE_CODE_TOOLKIT_CRON_MODE=1
#   CLAUDE_CODE_TOOLKIT_FOLLOWUP_MY_EMAIL=<your-gmail-address>
```

See `docs/cron-setup.md` for the full plist example.

## Pitfalls

- **Auto-reply false-positives**: vacation autoresponders count as "the recipient replied" and silently drop a thread that is actually still waiting. Detect autoresponders via headers (`Auto-Submitted: auto-replied`, `X-Autoreply: yes`) and ignore them when computing "did anyone reply"
- **Long-running negotiations**: contracts, vendor onboarding — sometimes a 14-day silence is the expected cadence. The skill cannot know this. Provide a `--ignore-thread-id` repeat flag for permanent suppression, or label the thread `Triage/Cold-Expected` and add it to `--ignore-labels`
- **Group emails**: when the operator emails an alias like `team@example.com`, "did anyone reply" still answers correctly, but the `recipients` field may show only the alias. Acceptable — the operator knows which group it is
- **Personal dashboard discipline**: do NOT post the report into a public Slack channel or shared Notion page. The output is private to the operator. Mirror only into a personal Notion page when used at all
- **Self-reply across aliases**: if the operator uses multiple addresses (`me@work.com` + `me@personal.com`), set `--my-email` to the primary one and add the others to a recipient-domain ignore list

## Tone

- Subject quoted verbatim, no summarization
- Recipient name + email shown side by side (the operator may not remember the email)
- Tier names (`Soft / Hard / Cold`) describe age, not judgment — never imply the recipient is rude
- "Currently zero pending follow-ups" is welcome — do not invent

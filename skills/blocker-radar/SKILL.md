---
name: blocker-radar
description: Scan the last N days of Slack across configured channels for help/blocked patterns, classify each candidate as a real ask vs noise via the ticket-classifier subagent, then upsert rows into a Notion "Blockers" DB with owner / waiting-on / age metadata. Designed for daily 09:00 launchd cron.
---

# blocker-radar

A teammate types "blocked — I don't have access" at 14:00 Tuesday in `#eng`. Nobody replies. By Friday's 1:1 the manager finds out — and is annoyed that nobody flagged it, even though the teammate did. The cost of a blocker grows non-linearly with age (2→4 days is much cheaper to fix than 4→8 days, because attention has already moved elsewhere). This skill scans configurable channels every morning, recognizes the SOS signals a manager can't possibly follow in real time, and surfaces them as a Notion DB the manager can scan in 60 seconds.

## Input

- `--since YYYY-MM-DD` (optional, defaults to yesterday 00:00 KST) — earliest message to consider
- `--channels <comma-separated>` (optional, falls back to env `CLAUDE_CODE_TOOLKIT_BLOCKER_CHANNELS`) — channels to scan (e.g. `#eng,#product,#ops,#design`)
- `--notion-db-id <id>` (optional, falls back to env `CLAUDE_CODE_TOOLKIT_BLOCKER_DB_ID`) — Notion Blockers DB ID
- `--min-age-hours N` (optional, default 24) — only surface a blocker if it has gone unanswered for at least N hours
- Missing input → AskUserQuestion (never guess)

## Output

```
2026-04-27 (since 2026-04-26): scanned N messages → K blocker candidates → C classified as real → U upserted (new N / aged M)
- Notion DB: <DB URL>
```

The Notion DB schema (one-time setup; the skill validates with `mcp__claude_ai_Notion__notion-fetch` before writing):

| Column | Type | Content |
|---|---|---|
| Title | title | One-line blocker summary |
| Person | rich_text | Slack display_name of who is stuck |
| Topic | select | Inferred area (auth / infra / design / data / external / unknown) |
| Waiting on | rich_text | Person/team or system the blocker depends on (if mentioned) |
| First seen | date | The earliest matching message timestamp |
| Last update | date | The most recent matching message timestamp |
| Age (hours) | number | Hours between First seen and now |
| Status | select | New / Aging / Stale (auto-bumped per thresholds) |
| Source | url | Slack permalink to the earliest matching message |

## Procedure

### 1. Parse input

- `--since` missing → yesterday 00:00 KST
- `--channels` missing AND env missing → AskUserQuestion
- `--notion-db-id` missing AND env missing → AskUserQuestion
- Validate the Notion DB schema with `mcp__claude_ai_Notion__notion-fetch` once. If columns are missing, exit and print the expected schema for the operator to add manually

### 2. Run slack-scout for each channel

Invoke slack-scout via the Task tool, **once per channel**, with a window from `--since` to now. Parallelize up to 5.

slack-scout returns its standard structured summary; the SKILL keeps the raw "Key messages" + "Unresolved questions" lines as blocker candidates.

### 3. Apply pattern filter (cheap pre-filter)

Drop candidates that lack any of these signals (case-insensitive):

- `blocked`, `stuck`, `waiting`, `can't`, `cannot`, `need help`, `anyone know`, `permission`, `access`, `no access`, `pending`

This pre-filter cuts 90% of noise before the LLM call. Operators with non-English-speaking teams should extend the signal list with localized equivalents (e.g. add Korean / Japanese / Spanish blocker phrases).

### 4. Run ticket-classifier on survivors

For each candidate, invoke the ticket-classifier subagent with a custom prompt:

```
description: "Classify blocker candidate"
subagent_type: "ticket-classifier"
prompt: |
  Slack message: <body>
  Channel: <name>
  Author: <display_name>

  Classify as exactly one of [help_request / status_update / venting / not_blocker].
  - help_request: explicit ask for unblock or pointer
  - status_update: announcing being blocked but not asking (still counts — surface it)
  - venting: emotional but no ask, no actionable detail
  - not_blocker: neither

  Output: classification + confidence + 1-line rationale + inferred topic (auth/infra/design/data/external/unknown) + inferred waiting-on (person or team or "(unknown)").
```

Keep `help_request` and `status_update`. Drop `venting` and `not_blocker`.

### 5. Group same-person same-topic candidates

A teammate often re-mentions the same blocker over multiple days. Group by:

- Same `display_name` + ≥50% topic overlap → same blocker

For each group:
- `First seen` = earliest message
- `Last update` = latest message
- `Age (hours)` = (now - First seen) in hours
- `Status` = `New` if age < 48h, `Aging` if 48 ≤ age < 168h, `Stale` if ≥ 168h

### 6. Apply min-age threshold

Drop blockers whose age is below `--min-age-hours`. Default 24 hours — anything younger is still in the "give the team a chance to reply naturally" window.

### 7. Upsert into Notion DB

For each surviving blocker:

- Search the DB for existing rows with the same `Person` + `Topic` (open status). If found → `update_page` (Last update / Age / Status). If not → `create_page`

Call `mcp__claude_ai_Notion__notion-create-pages` or `mcp__claude_ai_Notion__notion-update-page` accordingly.

**Hook behavior:** the plugin's `PreToolUse(mcp__claude_ai_Notion__notion-create-pages)` hook fires on the first row creation per run — asks once "About to upsert N blocker rows to Notion. Proceed?". In cron mode set `CLAUDE_CODE_TOOLKIT_CRON_MODE=1` to bypass and write only an audit log entry (`~/.claude-code-toolkit/audit/blocker-radar-YYYY-MM-DD.jsonl`).

### 8. Output

Print the one-line console summary + DB URL. Done.

## Automated operation (cron)

Recommended launchd cron at every weekday 09:00 KST:

```
# ~/Library/LaunchAgents/<reverse-domain>.claude-code-toolkit.blocker-radar.plist
# Program: /usr/local/bin/claude
# ProgramArguments: -p "/claude-code-toolkit:blocker-radar"
# StartCalendarInterval: [{Weekday=1..5, Hour=9, Minute=0}]
# EnvironmentVariables:
#   CLAUDE_CODE_TOOLKIT_CRON_MODE=1
#   CLAUDE_CODE_TOOLKIT_BLOCKER_CHANNELS=#eng,#product,#ops,#design
#   CLAUDE_CODE_TOOLKIT_BLOCKER_DB_ID=<id>
```

See `docs/cron-setup.md` for the full plist example.

## Pitfalls

- **Pattern filter false negatives**: subtle blockers ("this isn't quite working…") slip past. Accept this — false-negative is preferable to drowning the manager in false-positive
- **Same person, multiple topics in one day**: grouping by topic-overlap is fragile. When in doubt, create separate rows (over-surfacing is acceptable; merging too aggressively is not)
- **Ticket-classifier mis-tagging venting as help_request**: tune the classifier prompt with explicit examples. Venting + an emoji is still venting unless there is a question mark or explicit ask
- **Notion DB schema drift**: if columns are renamed, every run fails. The pre-flight `mcp__claude_ai_Notion__notion-fetch` validation catches this — don't skip it
- **Stale rows accumulate**: rows whose blockers have been resolved silently are not auto-closed. Consider a separate `blocker-radar --close-resolved` pass that scans recent thread replies for resolution language and bumps Status → Resolved (out of scope for v1)

## Tone

- Quote the original ask verbatim in the row body — no rephrasing
- The Title is a 1-line manager-readable summary, not a copy of the message
- "Waiting on (unknown)" is welcome — do not fabricate dependents
- Topic `unknown` is welcome — do not force a category guess

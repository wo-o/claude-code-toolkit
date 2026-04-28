---
name: weekly-shipped-from-noise
description: Scan a configurable set of Slack channels for one week of shipping signals (`shipped` / `merged` / `launched` / `released` / `deployed` text + ship-it emoji), dedupe events, then write a Notion "Weekly Shipped" page categorized by Engineering / Product / Operations with source links. Designed for Friday 17:00 launchd cron.
---

# weekly-shipped-from-noise

Every Friday afternoon a manager opens five Slack channels, 30 PR notifications, and eight Notion pages to assemble a weekly report — and still misses things. This skill collapses the same hour-and-a-half job into a single command. With one week as the window, the skill scans every shipping signal scattered across Slack, deduplicates the same event mentioned in multiple channels, and writes a categorized Notion page with every source linked.

## Input

- `--week YYYY-Www` (optional, defaults to the current ISO week in KST) — target week (e.g. `2026-W17`)
- `--channels <comma-separated>` (optional, falls back to env `CLAUDE_CODE_TOOLKIT_SHIPPED_CHANNELS`) — channels to scan (e.g. `#general,#eng,#product,#wins,#shipped,#launches`)
- `--notion-parent-id <id>` (optional, falls back to env `CLAUDE_CODE_TOOLKIT_SHIPPED_PARENT_ID`) — Notion parent page ID under which the weekly page is created
- Missing input → AskUserQuestion (never guess)

## Output

```
2026-W17 (2026-04-21 ~ 2026-04-27): scanned N channels, M messages → K shipped events (dedup D)
- Engineering: a / Product: b / Operations: c
- Notion: <weekly page URL>
```

Format of the Notion page body:

```
# Weekly Shipped — YYYY-Www (YYYY-MM-DD ~ YYYY-MM-DD)

## Source
- Channels: #channel-1, #channel-2, ...
- Messages scanned: N
- Events extracted: K (dedup: D)
- Generated at: YYYY-MM-DD HH:MM KST

## Engineering
- <event 1> — owner: @<display_name>, <YYYY-MM-DD>, source: <permalink>
- ...

## Product
- ...

## Operations
- ...

## Notion pages flipped to Done this week
- <page title> — last edit: YYYY-MM-DD, <Notion URL>

## Caveats
- Same event mentioned in multiple channels was kept only once (earliest message wins)
- Messages with `:no_entry_sign:` reaction by the author are excluded as retracted
```

## Procedure

### 1. Parse input

- `--week` missing → current ISO week (KST). Compute the Monday 00:00 ~ Sunday 23:59 boundary
- `--channels` missing AND env missing → AskUserQuestion with the default 6-channel list as a recommendation
- `--notion-parent-id` missing AND env missing → AskUserQuestion

### 2. Run slack-scout for each channel

Invoke slack-scout via the Task tool, **once per channel**, with a 7-day window. slack-scout returns its standard structured summary; the SKILL collects raw decisions/announcements lines for downstream classification.

Parallelize up to 5 slack-scout invocations — Slack API rate limits make more aggressive parallelism risky.

### 3. Extract shipping signals (LLM)

For each message returned by the scouts, classify as `shipped` / `not-shipped`. A message is `shipped` only if it contains either:

- A shipping verb in the present/past tense — `shipped`, `merged`, `launched`, `released`, `deployed`, `made live`, `rolled out`, `pushed to prod` (operators with non-English-speaking teams should extend this list with localized verbs)
- A shipping emoji as a reaction or in body — `:shipit:`, `:ship:`, `:tada:`, `:rocket:`, `:white_check_mark:`
- A canonical PR/release URL pattern (e.g. `github.com/.../pull/N` followed by "merged")

When the signal is ambiguous (`will ship`, `planning to ship`), classify as `not-shipped`. Future-tense never counts.

### 4. Deduplicate

The same event often appears in multiple channels (e.g. `#eng` and `#wins`). Group `shipped` messages by similarity:

- Same author + same day + ≥60% noun-phrase overlap → same event
- Or message body contains the same canonical URL (PR/release/Linear) → same event

Keep the earliest message; record the rest as `also mentioned in: #channel` metadata.

### 5. Categorize (LLM)

Tag each unique event as `Engineering` / `Product` / `Operations`:

| Category | Signal |
|---|---|
| Engineering | merged PR, released library, deployed service, hotfix, infra change |
| Product | feature flag enabled, public-facing launch, copy/UX update, marketing push |
| Operations | incident closed, runbook published, migration finished, DB rotation, backup verified |

When ambiguous, prefer Operations over Engineering, Engineering over Product (more specific wins).

### 6. Pull Notion "Done" pages

Call notion-scout once with the same week window:

- Search the Notion workspace for pages whose status flipped to `Done` between Monday and Sunday
- Take the top 10 most-recently-edited

Add them as a separate section — these are confirmation signals, not authoritative shipping events.

### 7. Create the weekly Notion page

Call `mcp__notion__create_page`:
- `parent`: `page_id` = the input parent page ID
- `properties`: `title` = `Weekly Shipped — YYYY-Www`
- `children`: heading_2 + bulleted_list_item blocks per the body format above

**Hook behavior:** the plugin's `PreToolUse(mcp__notion__create_page)` hook fires once. In cron mode set `CLAUDE_CODE_TOOLKIT_CRON_MODE=1` to bypass and write only an audit log entry (`~/.claude-code-toolkit/audit/weekly-shipped-YYYY-Www.jsonl`).

### 8. Output

Print the one-line console summary + Notion URL. Done.

## Automated operation (cron)

Recommended launchd cron at Friday 17:00 KST:

```
# ~/Library/LaunchAgents/<reverse-domain>.claude-code-toolkit.weekly-shipped.plist
# Program: /usr/local/bin/claude
# ProgramArguments: -p "/claude-code-toolkit:weekly-shipped-from-noise"
# StartCalendarInterval: { Weekday=5, Hour=17, Minute=0 }
# EnvironmentVariables:
#   CLAUDE_CODE_TOOLKIT_CRON_MODE=1
#   CLAUDE_CODE_TOOLKIT_SHIPPED_CHANNELS=#general,#eng,#product,#wins,#shipped,#launches
#   CLAUDE_CODE_TOOLKIT_SHIPPED_PARENT_ID=<id>
```

See `docs/cron-setup.md` for the full plist example.

## Pitfalls

- **Channel scope too wide**: scanning #random brings in noise that survives dedup. Curate the channel list — defaulting to "every channel" produces a worse report than no report
- **Future-tense leak**: "we'll ship X next week" sometimes survives the LLM filter. Add a hard regex deny on `will / next week / scheduled / planned / upcoming` in the post-classification pass (extend with localized future-tense markers as needed)
- **Author retraction**: when an author reacts to their own message with `:no_entry_sign:` or `:rewind:`, treat it as retracted and exclude
- **Re-running the same week creates a duplicate page**: there is no built-in dedupe by week — operator must delete the prior page or use an `--update` mode (TBD)
- **Dedup over-merges**: short event names ("v2.3 out") collide with unrelated short events. When noun-phrase overlap is the only signal, require ≥80% rather than 60%

## Tone

- Quote shipping language verbatim — no rephrasing into corporate-speak
- Author handles stay as Slack `display_name`
- Mark inferred categories as "estimated" only when the LLM's confidence was low
- "Currently no shipping signals detected" is an acceptable output for a quiet week — do not invent events

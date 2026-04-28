---
name: meeting-prep-pull
description: Given a Google Calendar event (defaults to the next meeting on the primary calendar), pull recent email threads with each external attendee + relevant Notion pages, and write a one-page Notion pre-read with overview / email context / related pages / talking points. Three-source synthesis (Calendar + Gmail + Notion) via Pattern C subagents.
---

# meeting-prep-pull

The 10-minute pre-meeting cram, automated. Before a 14:00 partner call, the operator usually opens five email threads, scrolls Slack, and skims two Notion pages. This skill collapses that scramble into a single command — given a calendar event, it gathers Gmail threads with each external attendee from the last 14 days, finds related Notion pages, and writes a one-page pre-read in Notion the operator can read in 90 seconds.

Zero-config: this skill auto-resolves the Notion parent page "Meeting Prep" on first run and caches the ID — see docs/auto-resolve.md. The internal domain is inferred from the authenticated Gmail account when not specified.

The skill is on-demand only (not a scheduled cron) — it pulls just-in-time before each meeting.

## Input

- `--event-id <id>` (optional) — Google Calendar event ID. If omitted, the skill calls `mcp__claude_ai_Google_Calendar__list_events` for the next 24 hours on the primary calendar and picks the soonest one. If multiple within the next 2h → AskUserQuestion
- `--lookback-days N` (optional, default 14) — Gmail and Notion lookback window
- `--notion-parent-id <id>` (override only — defaults to auto-resolve Notion parent page "Meeting Prep". Env override: `CLAUDE_CODE_TOOLKIT_PREP_PARENT_ID`)
- `--internal-domain <domain>` (override only — defaults to the domain of the authenticated Gmail account. Env override: `CLAUDE_CODE_TOOLKIT_PREP_INTERNAL_DOMAIN`)

## Output

```
Meeting: <event title> @ <YYYY-MM-DD HH:MM>
Attendees: N (E external / T internal)
- Gmail: scanned X threads → top 3 per external attendee
- Notion: matched Y pages from the last <lookback-days> days
- Pre-read: <Notion page URL>
```

Pre-read page format:

```
# Meeting Pre-Read — <event title>

## Overview
- When: YYYY-MM-DD HH:MM ~ HH:MM (TZ)
- Where: <location or video link>
- Attendees: <name1 (email1)>, <name2 (email2)>, ...
- Agenda: <event description, verbatim if present, otherwise "(none provided)">

## Email context
### <external attendee 1 — name + email>
- "<thread subject 1>" — last activity YYYY-MM-DD, <gmail URL>
  - 1-line summary of the latest message
- "<thread subject 2>" — ...

### <external attendee 2 — name + email>
- ...

## Related Notion pages
- "<page title 1>" — last edit YYYY-MM-DD, <Notion URL>
  - 1-line summary
- "<page title 2>" — ...

## Talking points and open questions
- <bullet 1, derived from the latest email/Notion context — phrased as a question or decision the operator needs to make>
- <bullet 2>
- ...

## Caveats
- Email scope: last <lookback-days> days, external attendees only
- Notion scope: top 5 most-relevant pages by keyword match against event title + description
- Generated at YYYY-MM-DD HH:MM (TZ)
```

## Procedure

### 1. Parse input + pick the event

- Notion parent page ID: auto-resolve "Meeting Prep" via the algorithm in docs/auto-resolve.md (cache-first, fall through to notion-search with page filter). Override: `--notion-parent-id <id>` or env `CLAUDE_CODE_TOOLKIT_PREP_PARENT_ID`.
- Internal domain: when missing, derive from the authenticated Gmail account (e.g. `me@example.com` → `example.com`). Override: `--internal-domain <domain>` or env `CLAUDE_CODE_TOOLKIT_PREP_INTERNAL_DOMAIN`.
- If `--event-id` missing → call `mcp__claude_ai_Google_Calendar__list_events` for `primary` calendar, `time_min=now`, `time_max=now+24h`
- If 0 events → exit with "No meetings in the next 24 hours"
- If 1 event → use it
- If 2+ events with the soonest within 2h → use the soonest
- Otherwise → AskUserQuestion with the list of upcoming events

Call `mcp__claude_ai_Google_Calendar__get_event` for the chosen event_id to fetch full details: summary, description, start, end, location, attendees.

### 2. Classify attendees

Split attendees into:
- `internal`: email domain == `--internal-domain`
- `external`: everyone else

If 0 external attendees → exit with "Internal-only meeting — pre-read is not generated" (the operator presumably already has the Slack/Notion context for internal meetings).

### 3. Gmail context per external attendee

For each external attendee, invoke `mcp__claude_ai_Gmail__search_threads`:

```
from:<attendee_email> OR to:<attendee_email>  newer_than:<lookback-days>d
```

Limit 20 per attendee. Parallelize up to 5 attendees in a single Task batch.

For each thread, call `mcp__claude_ai_Gmail__get_thread` and capture:
- `subject`, `last_message_ts`, `last_message_snippet`, `permalink`

Sort by `last_message_ts` descending and keep top 3 per attendee.

### 4. Notion context

Run notion-scout once:

```
description: "Find Notion pages related to the meeting"
subagent_type: "notion-scout"
prompt: |
  Meeting title: <event summary>
  Description: <event description>
  Attendees: <list>

  Find up to 5 Notion pages from the last <lookback-days> days that are most relevant.
  Match by:
  - Page title overlap with the meeting title
  - Body keyword overlap with the description
  - @mentions of any external attendee
  Return: title, last edit, URL, 1-line summary, match reason.
```

### 5. Synthesize via the synthesizer subagent

Invoke synthesizer with the gathered Calendar + Gmail + Notion context:

```
description: "Synthesize meeting pre-read"
subagent_type: "synthesizer"
prompt: |
  Calendar event: <full event JSON>
  Gmail context per attendee: <per-attendee top-3 thread summaries>
  Notion pages: <top-5 page summaries>
  Internal domain: <internal-domain>

  Produce the pre-read in the format above. Talking points should be:
  - Concrete decisions the operator needs to make in the meeting
  - Open questions surfaced by the email/Notion context
  - 3-5 bullets max — fewer is better than padded
```

The synthesizer is read-only — it never writes to Notion / Gmail / Slack. Pattern C subagent permission isolation is preserved (gmail/notion read-only allowlist + synthesizer no-tools allowlist).

### 6. Create the Notion pre-read page

Call `mcp__claude_ai_Notion__notion-create-pages`:
- `parent`: `page_id` = the resolved parent page ID
- `properties`: `title` = `Pre-Read — <event title> — YYYY-MM-DD HH:MM`
- `children`: heading_2 + bulleted_list_item blocks per the body format above

**Hook behavior:** the plugin's `PreToolUse(mcp__claude_ai_Notion__notion-create-pages)` hook fires once per run — asks "About to create a meeting pre-read page in Notion. Proceed?". In cron mode (rare for this on-demand skill) set `CLAUDE_CODE_TOOLKIT_CRON_MODE=1` to bypass + audit log (`~/.claude-code-toolkit/audit/meeting-prep-YYYY-MM-DD.jsonl`).

### 7. Output

Print the one-line console summary + Notion URL. Done.

## Demo (T+0:30, on-demand)

```
> /claude-code-toolkit:meeting-prep-pull
```

Pre-demo setup:
- One Google Calendar event in the next 2h with 1-2 external attendees
- 2-3 demo email threads with each attendee from the last 14 days
- 1-2 Notion pages whose titles overlap with the event title
- Presenter env: `CLAUDE_CODE_TOOLKIT_PREP_PARENT_ID` + `CLAUDE_CODE_TOOLKIT_PREP_INTERNAL_DOMAIN` set, Gmail + Calendar + Notion Connectors authenticated

90-second flow:
1. Type the command (5s)
2. Calendar lookup + attendee classification (5s)
3. Parallel Gmail searches per external attendee (15s)
4. Notion scout (10s)
5. Synthesizer + Notion page creation + hook prompt → y (15s)
6. Open the Notion page in browser to show the pre-read (40s)

## Pitfalls

- **No upcoming meeting**: when the operator runs the skill before 09:00 with no events scheduled today, output is empty. Provide a `--event-id` flag for explicit override
- **Internal-domain mis-set**: if the operator's company uses multiple domains (`@example.com` + `@example.io`), the secondary domain attendees get classified as external. Make `--internal-domain` accept comma-separated multiple values
- **Email noise**: a chatty attendee may have 50 threads in 14 days. Capping top-3-per-attendee is correct; do not raise this default — the pre-read should fit on one page
- **Notion-scout false positives**: keyword matching over-surfaces unrelated pages with common words (e.g. "Q1 Review" matches every Q1 page). Tune the scout prompt: prefer pages where ≥ 2 keywords overlap, not 1
- **Description with personal info**: if the calendar event description contains a personal phone number or home address, do NOT redact — the operator chose to put it there, and rephrasing breaks "verbatim" trust. The pre-read inherits the same privacy boundary as the calendar event itself

## Tone

- Subject and event summary are quoted verbatim
- Talking points framed as questions or decisions, not commands
- Empty sections are acceptable ("No related Notion pages found in the lookback window")
- The pre-read is a reading aid, not a decision — never tell the operator what to say or decide

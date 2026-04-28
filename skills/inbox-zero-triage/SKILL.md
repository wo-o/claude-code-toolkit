---
name: inbox-zero-triage
description: Scan the inbox for unreplied threads older than N hours, classify each as `action_required` / `FYI` / `newsletter` / `spam-like` via the ticket-classifier subagent, apply Gmail labels under a configurable prefix, and upsert the action_required ones into a Notion "Inbox Triage" DB. Designed for daily 08:00 launchd cron.
---

# inbox-zero-triage

By 08:30 the inbox already has 40 new threads. Half are newsletters, a quarter are FYI, and the small remainder is what actually needs a reply today — but to find them you scroll for fifteen minutes. This skill runs the scroll for you: it labels every thread inside Gmail (so the inbox is sorted at a glance) and upserts the action_required threads into a Notion DB the operator can scan in 60 seconds.

Zero-config: this skill auto-resolves the Notion DB "Inbox Triage" on first run and caches the ID — see docs/auto-resolve.md. Gmail is the only inbound source; no Slack channel resolution needed.

Outbound is **draft-only** by Connector design (Anthropic's Gmail Connector exposes `create_draft` but not `send_message`) — this skill never drafts replies, only labels and indexes. Reply drafting is a separate skill (out of scope for v1).

## Input

- `--since-hours N` (optional, default 24) — only consider threads with messages in the last N hours
- `--label-prefix S` (optional, default `Triage/`) — Gmail label prefix (`Triage/Action`, `Triage/FYI`, `Triage/Newsletter`, `Triage/Spam`)
- `--notion-db-id <id>` (override only — defaults to auto-resolve Notion DB "Inbox Triage". Env override: `CLAUDE_CODE_TOOLKIT_INBOX_DB_ID`)
- `--exclude-labels <list>` (optional, default `Triage/Action,Triage/FYI,Triage/Newsletter,Triage/Spam`) — skip threads already triaged

## Output

```
2026-04-28 (since-hours 24): scanned N threads → A action_required / F FYI / W newsletter / S spam-like → U upserted to Notion (new N / aged M)
- Gmail labels applied: <label-prefix>Action × A, <label-prefix>FYI × F, <label-prefix>Newsletter × W, <label-prefix>Spam × S
- Notion DB: <DB URL>
```

The Notion DB schema (one-time setup; the skill validates with `mcp__claude_ai_Notion__notion-fetch` before writing):

| Column | Type | Content |
|---|---|---|
| Title | title | One-line subject summary |
| Sender | rich_text | From address (display name + email) |
| Subject | rich_text | Original subject line, verbatim |
| Snippet | rich_text | First 240 chars of the latest message body |
| Age (hours) | number | Hours since the latest message arrived |
| Priority | select | High / Medium / Low (LLM-inferred) |
| Status | select | Open / Replied / Archived (auto-bumped per follow-up runs) |
| Source | url | Gmail thread permalink |

## Procedure

### 1. Parse input

- `--since-hours` missing → 24
- `--label-prefix` missing → `Triage/`
- Notion DB ID: auto-resolve "Inbox Triage" via the algorithm in docs/auto-resolve.md (cache-first, fall through to notion-search). Override: `--notion-db-id <id>` or env `CLAUDE_CODE_TOOLKIT_INBOX_DB_ID`.
- Validate the resolved Notion DB schema with `mcp__claude_ai_Notion__notion-fetch` once. If columns are missing, exit and print the expected schema for the operator to add manually
- Ensure the four labels exist via `mcp__claude_ai_Gmail__list_labels` — create any missing ones with `mcp__claude_ai_Gmail__create_label`

### 2. Collect candidate threads

Call `mcp__claude_ai_Gmail__search_threads` with the query:

```
in:inbox is:unread newer_than:<N>h -label:<label-prefix>Action -label:<label-prefix>FYI -label:<label-prefix>Newsletter -label:<label-prefix>Spam
```

Limit 100. For each thread returned, call `mcp__claude_ai_Gmail__get_thread` to fetch headers + the latest message body. Build a candidate list with:

- `thread_id`, `from`, `subject`, `snippet`, `last_message_ts`, `is_self_reply` (whether the most recent message was from the authenticated user)

Drop `is_self_reply == true` threads — those are awaiting the recipient, not the operator.

### 3. Classify with ticket-classifier (LLM)

For each candidate, invoke the ticket-classifier subagent:

```
description: "Classify inbox thread"
subagent_type: "ticket-classifier"
prompt: |
  From: <from>
  Subject: <subject>
  Snippet: <snippet (≤ 500 chars)>

  Classify as exactly one of [action_required / FYI / newsletter / spam-like].
  - action_required: the sender is asking for a decision, reply, signature, or scheduling
  - FYI: informational update, no ask, but human-relevant
  - newsletter: bulk-sent content (digests, marketing, product updates), no personal ask
  - spam-like: cold outreach, generic sales templates, low signal

  Output: classification + confidence + 1-line rationale + inferred priority (high/medium/low) for action_required only.
```

Default to `FYI` when confidence < 0.6 — when unsure, surface (do not drop) but mark Priority=Low.

### 4. Apply Gmail labels

For each classified thread, call `mcp__claude_ai_Gmail__label_thread`:
- action_required → `<label-prefix>Action`
- FYI → `<label-prefix>FYI`
- newsletter → `<label-prefix>Newsletter`
- spam-like → `<label-prefix>Spam`

Labels are visible-only to the operator's Gmail account — no recipient-visible side effects.

### 5. Upsert action_required to Notion DB

For each `action_required` thread:

- Search the DB for a row with the same `Source` (Gmail thread permalink)
- If found → `mcp__claude_ai_Notion__notion-update-page` (Last update / Age / Priority / Snippet)
- If not → `mcp__claude_ai_Notion__notion-create-pages` with the schema above

The Gmail thread permalink format: `https://mail.google.com/mail/u/0/#inbox/<thread_id>`.

**Hook behavior:** the plugin's `PreToolUse(mcp__claude_ai_Notion__notion-create-pages)` hook fires on the first row creation per run — asks once "About to upsert N inbox rows. Proceed?". In cron mode set `CLAUDE_CODE_TOOLKIT_CRON_MODE=1` to bypass + write only an audit log entry (`~/.claude-code-toolkit/audit/inbox-triage-YYYY-MM-DD.jsonl`).

### 6. Output

Print the one-line console summary + DB URL. Done.

## Automated operation (cron)

Recommended launchd cron at every weekday 08:00 KST:

```
# ~/Library/LaunchAgents/<reverse-domain>.claude-code-toolkit.inbox-zero-triage.plist
# Program: /usr/local/bin/claude
# ProgramArguments: -p "/claude-code-toolkit:inbox-zero-triage"
# StartCalendarInterval: [{Weekday=1..5, Hour=8, Minute=0}]
# EnvironmentVariables:
#   CLAUDE_CODE_TOOLKIT_CRON_MODE=1
#   # Optional — only set if you want to override auto-resolve
#   CLAUDE_CODE_TOOLKIT_INBOX_DB_ID=<id>
```

See `docs/cron-setup.md` for the full plist example.

## Pitfalls

- **Newsletter false-negative blast**: a misclassified newsletter inflates the Notion DB. Tune the classifier with explicit examples — anything with `unsubscribe` in the footer should default to newsletter unless the body has a personal salutation
- **Cold-outreach false-positive as action_required**: aggressive cold sales templates often phrase a CTA as a question. Add a hard demote rule: when sender domain has appeared > 3× in the last 30 days with the same subject pattern, downgrade to spam-like
- **Self-reply detection drift**: if the operator uses a separate sender alias (e.g. `me+notes@gmail.com`), `is_self_reply` may miss. Treat any sender matching the authenticated address book as self
- **Label limit**: Gmail caps labels at 10,000 per account — practically not an issue but the create_label call should be idempotent (skip if exists). The pre-flight `list_labels` covers this
- **Notion DB schema drift**: same as blocker-radar — pre-flight `mcp__claude_ai_Notion__notion-fetch` validation is mandatory

## Tone

- Subject and snippet are quoted verbatim — no summarization or rephrasing
- Sender display name preserved as-is (no first-name extraction)
- Priority `Low` is welcome — do not inflate priority to look productive
- "Currently no action_required threads" is an acceptable output for a quiet morning — do not invent

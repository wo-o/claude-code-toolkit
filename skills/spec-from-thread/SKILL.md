---
name: spec-from-thread
description: Take a Slack thread URL, classify each message into [decision / requirement / open question / chatter], and create a Notion spec page from the non-chatter messages with the original thread permalink attached.
---

# spec-from-thread

When a PM debates a new feature in a Slack thread, decisions, requirements, and open questions get scattered across 50-100 messages. Hand-converting them into a Notion spec page takes 1-2 hours — so it never happens. With a single URL as input, this skill fetches + classifies + writes to Notion in one command.

## Input

- `--thread-url <slack-permalink>` (required) — Slack thread permalink (`https://<workspace>.slack.com/archives/<channel>/p<ts>` format)
- `--notion-parent-id <id>` (optional, falls back to env `CLAUDE_CODE_TOOLKIT_SPEC_PARENT_ID`) — Notion parent page ID
- `--title <text>` (optional) — title for the spec page being created. If missing, the LLM extracts one from the thread's first message.
- Missing input → AskUserQuestion (never guess)

## Output

One-line console summary + Notion page URL:

```
Thread <permalink> → classified N messages (decisions N / requirements N / questions N / chatter N) → Notion spec page created
- Notion: <page URL>
```

Format of the Notion page body:

```
# <Title>

## Source
- Slack thread: <permalink>
- Extracted at: YYYY-MM-DD HH:MM KST
- Message count: N (decisions N / requirements N / questions N / chatter excluded)

## Purpose
<thread's first message, or LLM-inferred 1-paragraph summary>

## Decisions
- <decision 1> (author: @<display_name>, <time>)
- ...

## Requirements
- <requirement 1> (author: ...)
- ...

## Open issues
- <question 1> (author: ...)
- ...

## Next actions (suggested)
- [ ] <action 1>
- [ ] <action 2>

## Caveats
- N messages classified as chatter were excluded from the body (originals available in the Slack thread)
- When decision/requirement is ambiguous, decision wins
```

## Procedure

### 1. Parse input

- Parse `--thread-url`: extract workspace · channel-id · ts. Reject if not in this format.
- `--notion-parent-id` missing AND env missing → AskUserQuestion
- `--title` missing → LLM-infer in step 3

### 2. Collect thread messages

Call `mcp__claude_ai_Slack__slack_read_channel`:
- `channel`: extracted channel-id
- `oldest`: thread_ts (parent message time)
- `latest`: thread_ts + 30 days (to cover all thread replies)
- `limit`: 200

Or use a thread-dedicated tool if exposed (check the Slack Connector's tool name in `claude /mcp` output).

For each message, resolve user_id → display_name with `mcp__claude_ai_Slack__slack_read_user_profile` (cached).

### 3. LLM classification

Classify each message into one of [decision / requirement / question / chatter]:

| Category | Signal |
|---|---|
| decision | "going with X", "decided on Y", "finalized Z", approval reactions |
| requirement | "need Y", "must have X", "definitely", spec-shaped statements |
| question | ends in `?`, "how to", "is it possible", "what about exceptions?" |
| chatter | greetings, agreement ("ok"), thanks, unrelated info |

When confidence is unclear, decision wins (if requirement/question contains decision-like phrasing, classify as decision).

### 4. Title inference (if needed)

If `--title` is missing, generate a title (≤ 50 chars) using the LLM with the thread's first message + 1-2 decisions as input.

### 5. Create Notion page

Call `mcp__claude_ai_Notion__notion-create-pages`:
- `parent`: `page_id` = the input parent page ID
- `properties`: `title` = title
- `children`: combination of paragraph / heading_2 / bulleted_list_item / to_do blocks per the body format above

**Hook behavior:** the plugin's `PreToolUse(mcp__claude_ai_Notion__notion-create-pages)` hook fires — asks the user once "About to create a new Notion page. Proceed?". **In this release this is interactive-confirm only — bypass not allowed** (the current hook only writes audit logs to `decision-log-YYYY-MM-DD.jsonl`, so a spec-from-thread bypass would mix into another skill's audit trail). Cron automation will be considered after caller-aware audit log separation lands.

### 6. Output

One-line console summary + Notion page URL. Done.

## Pitfalls

- **Thread parent message is a #channel root with almost no replies**: the "thread" may actually be a single message rather than an actual thread. If only the parent exists, print "No spec to extract" and exit.
- **Loose classification prompt**: when decisions and requirements blend in one sentence, the LLM tags both. Force in the system prompt: "one category per message, decisions win".
- **LLM drops factual info from chatter messages**: chatter-classified messages may carry real facts — mention this explicitly in the "Caveats" section.
- **Re-running on the same thread duplicates pages**: every call creates a new Notion page. No dedupe logic — the user must delete the old page or use an `--update` mode (TBD).
- **No thread-dedicated tool from the Slack Connector**: confirm the exposed tool names with `claude /mcp` (e.g. whether `get_thread_replies` is exposed separately). If absent, fall back to `get_channel_history` + thread_ts.

## Tone

- Quote decisions/requirements verbatim — no paraphrasing
- Do not anonymize the author — keep display_name as-is
- "Speculative" wording is welcome (when chatter classification confidence is low)
- No fake confidence ("these are all the decisions", etc.)

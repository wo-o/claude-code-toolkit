---
name: ticket-triage-router
description: Classify a Slack #support message and produce an answer draft. Spawns ticket-classifier (Slack read-only) and ticket-draft-writer (Notion search read-only) subagents in series. No auto-post — draft only, for human review.
---

# ticket-triage-router

**Workshop pick #2 (V+X wow moment).** Automates the CS team's job of classifying external user messages in Slack #support as [bug / feature request / usage question] and drafting a reply. Per-tool permission isolation (Pattern C) means the classification and drafting steps are separable in the audit trace. See `docs/workshop-flow.md` for the demo sequence.

## Input

- `--message-url <slack-permalink>` (required) — Slack message permalink (`https://<workspace>.slack.com/archives/<channel>/p<ts>` format)
- If missing, take it via AskUserQuestion — never guess.

## Output

```
## Ticket #<auto-id>

### Classification (ticket-classifier)
- Category: bug | feature_request | usage_question
- Confidence: high / medium / low
- Rationale: <1-2 sentences>

### Draft reply (ticket-draft-writer)
<200-400 char draft based on category-specific search results>

### Related material
- Notion page: <title> — <URL>

### Items requiring manual review
- <item 1>
- <item 2>
```

## Procedure

### 1. Validate input

- `--message-url` missing → AskUserQuestion
- Reject if the URL is not in Slack permalink format

### 2. Run the ticket-classifier subagent

Invoke ticket-classifier via the Task tool:

```
description: "Classify support ticket"
subagent_type: "ticket-classifier"
prompt: |
  Slack message URL: <message-url>

  Fetch the message body and classify as exactly one of [bug / feature_request / usage_question].
  Include confidence and a 1-2 sentence rationale.
```

ticket-classifier holds only `mcp__claude_ai_Slack__slack_get_message` (or equivalent) read-only access. Returns one classification + confidence + rationale.

### 3. Run the ticket-draft-writer subagent

Pass the classifier's result into ticket-draft-writer:

```
description: "Draft support answer"
subagent_type: "ticket-draft-writer"
prompt: |
  Message body: <text fetched by classifier>
  Category: <classifier result>

  Search Notion (mcp__claude_ai_Notion__notion-search) per category:
  - bug: search the Bugs DB or incident pages for the same symptom. If found, link + cite status. If not, prompt for repro info.
  - feature_request: search the Roadmap/Backlog page. If already on the roadmap, cite the schedule. If not, reply "Will review and respond".
  - usage_question: search FAQ/Docs pages, cite the 1-2 closest pages.

  Output a 200-400 char draft + related Notion page links.
  No auto-posting — draft only.
```

ticket-draft-writer holds only `mcp__claude_ai_Notion__notion-search` read-only access. **Zero write tools.**

### 4. Combine results

Combine ticket-classifier + ticket-draft-writer results into the SKILL output format. Auto-posting to Slack is **forbidden** — the CS owner reviews and posts manually.

## Automation (optional)

A separate Slack Events API receiver can listen for new messages on `#support`, then have its webhook call out to `claude --print "/claude-code-toolkit:ticket-triage-router --message-url <url>"` and attach the result to the thread as a bot preview (a human still does the actual post).

Claude Code's internal hooks cannot receive external Slack events, so a separate receiver is required.

## Workshop demo (T1 explicit invocation)

```
> /claude-code-toolkit:ticket-triage-router --message-url https://<workspace>.slack.com/archives/C123/p1234567890
```

Pre-demo setup:
- Dummy `#support` channel + 1 seeded message (e.g. "I get an error after doing Y in feature X. How do I fix this?")
- 1 row in a Notion Bugs DB (same symptom, prior report) + 1 FAQ page

1-minute demo flow:
1. Type the command (5s)
2. ticket-classifier subagent runs — classification: bug, confidence high (15s)
3. ticket-draft-writer subagent runs — cites Notion Bugs DB row + draft reply (30s)
4. Result rendered + presenter narration: "Two subagents with split tool permissions — Slack read only / Notion search only. No auto-posting." (10s)

Key talking point: "Each subagent runs in isolation with only one MCP read permission. A human reviews and then posts the reply — there are zero write tools."

## Pitfalls

- **Classifier labels everything as usage_question**: LLM default bias. Force in the system prompt: "Emotion / opinion expressions are not classification signals — keywords (crash, error, doesn't work = bug; "suggest", "would be nice" = feature) take priority".
- **draft-writer searches without category awareness**: keywords and the target Notion area must vary by category. Branch explicitly in the prompt based on the classifier result.
- **Tempted to auto-post replies**: granting write tools risks the bot publishing wrong info. Never grant write.
- **Message URL points to a thread reply**: the thread parent may carry more important context. The classifier should detect thread_ts and fetch the parent as well.

## Tone

- Draft replies are polite and helpful. Ignore the user message's emotional tone.
- Honestly word unresolved cases as "Currently under review. Track progress at <Notion link>".
- "Not certain" wording is welcome — no fake confidence.

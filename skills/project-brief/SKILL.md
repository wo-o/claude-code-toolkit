---
name: project-brief
description: Generate a one-page project brief by gathering recent activity from Slack and project status from Notion. Spawns slack-scout and notion-scout subagents in parallel, then synthesizer to merge. Input - project name (string).
---

# project-brief

Collects a project's recent Slack activity and Notion progress, then condenses it into a one-page brief.

Zero-config: when the user opts to save the brief to Notion, this skill auto-resolves the Notion parent page "Project Briefs" on first run and caches the ID — see docs/auto-resolve.md. Saving to Notion remains opt-in; the default output is console only.

## Input

- Project name (required, free-form string). Examples: `mobile-relaunch`, `payments-v2`, `Q2 OKR`
- If missing, take it via AskUserQuestion — never guess.

## Output

A one-page brief in this format:

```
# <Project Name> Brief (YYYY-MM-DD)

## One-line summary
<1-2 sentences on current state>

## Recent Slack activity
- <YYYY-MM-DD HH:MM> #channel — speaker: key message
- (up to 5, newest first)

## Notion progress
- Page/DB: <title>
- Milestones: <achieved / in progress / delayed>
- Key decisions: <list>

## Issues found
- <issue 1>
- <issue 2>

## Next actions (suggested)
- [ ] <action 1>
- [ ] <action 2>
```

## Procedure

### 1. Validate input
- If the project name is empty, take it via AskUserQuestion
- If too abstract (e.g. "the company"), ask for something more concrete

### 2. Parallel scouts (spawn 2 subagents simultaneously)

Invoke both subagents in a single message via the Task tool.

**slack-scout invocation:**
```
description: "Scout Slack for <project name>"
subagent_type: "slack-scout"
prompt: "Project name: <project name>. Search related channels/messages and summarize the key activity from the last 7 days. Format the response per SKILL.md output section's 'Recent Slack activity'."
```

**notion-scout invocation:**
```
description: "Scout Notion for <project name>"
subagent_type: "notion-scout"
prompt: "Project name: <project name>. Search related pages/DBs and summarize progress. Format the response per SKILL.md output section's 'Notion progress'."
```

Issue both Task tool calls in one message in parallel. Never run them sequentially — that defeats the whole point of Pattern B (fan-out scouts).

### 3. Merge (subagent synthesizer)

Pass slack-scout / notion-scout results into the synthesizer subagent:

```
description: "Synthesize project brief"
subagent_type: "synthesizer"
prompt: |
  Project name: <project name>

  [Slack scout result]
  <slack-scout response>

  [Notion scout result]
  <notion-scout response>

  Merge the two inputs into a one-page brief in the exact full output format from SKILL.md.
  Especially highlight:
  - Points where Slack discussion and Notion docs disagree (issue candidates)
  - Things decided in Slack but not yet reflected in Notion (next-action candidates)
```

### 4. Output

Show the synthesizer's result to the user as-is. Do not post-process — the synthesizer already enforces the format.

(Optional) If the user asks to "save to Notion", call `mcp__claude_ai_Notion__notion-create-pages`. Resolve the parent page by auto-resolving "Project Briefs" via the algorithm in docs/auto-resolve.md (cache-first, fall through to notion-search with page filter) — do NOT ask the user for the parent page. This path triggers the plugin hook (`PreToolUse(mcp__claude_ai_Notion__notion-create-pages)`) which prompts once for confirmation before proceeding.

## Pitfalls

- **Calling the two scout subagents sequentially**: defeats the point of Pattern B fan-out. Always issue both Task calls in a single message.
- **Bypassing the synthesizer and merging in main**: both scout results would land back in the main context, defeating context isolation. The merge step also runs in an isolated context.
- **Main calls MCP directly**: if main accesses Slack/Notion directly, subagents lose their purpose. MCP calls happen only inside the scout subagents.
- **Output format breaks**: if the synthesizer prompt does not enforce the format, the result varies every time. Specify the exact output section format inside the synthesizer's input prompt.

## Tone

- Stick to facts; no commentary
- Milestone status uses one of three labels only: "achieved / in progress / delayed"
- Quoted Slack messages must include speaker + date

---
name: deal-context-loader
description: Given a deal ID or company name, fetch the HubSpot deal stage and recent activity, then search Slack for internal mentions of the same company in the last 14 days. Synthesize a 5-minute pre-call brief. Output only — no write side effect.
---

# deal-context-loader

A 5-minute brief synthesized for a salesperson juggling 30-50 active deals at once and trying to answer "5 minutes before my 3pm call, where are we on Account A?". HubSpot quantitative data (stage, amount) + Slack qualitative data (engineering / PM internal opinions) in one shot.

**Drift risk H** — HubSpot MCP is brand new GA (2026-04-13). Mandatory: dry-run-verify the tools before the first call.

## Input

- `--deal-id <id>` or `--company <name>` (one is required)
- `--slack-window-days <N>` (optional, default 14) — Slack mention search window
- Both missing → AskUserQuestion

## Output

Prints a 5-minute brief in markdown (no writes):

```
# Deal Context — <Company / Deal #ID>

## One-line status
<1-2 sentences on the deal stage, amount, and last activity>

## HubSpot
- Deal: #<id> — <name> — <stage> — <amount> — <close date>
- Owner: @<owner display_name>
- Last activity: <YYYY-MM-DD HH:MM> — <one-line activity description>
- Associated contacts:
  - <Name> (<title>) — <email/phone>
  - ...

## Internal Slack mentions (last <window> days)
- <YYYY-MM-DD HH:MM> #<channel> — @<author>: <1-2 line message>
- ...
- (top N extracted from N total messages)

## Synthesis
- Quantitative progress: <progress estimate based on stage>
- Internal concerns / opinions:
  - <concern 1>
  - <opinion 1>
- Slack-only intel:
  - <notes left by eng/PM while assessing fit>

## Next actions (suggested)
- [ ] <action 1: pre-call check>
- [ ] <action 2>

## Caveats
- Slack search is keyword-based on the company name — aliases / abbreviations may be missed
- HubSpot last activity depends on auto-logging — manual call notes may be missing
```

## Procedure

### 1. Parse input

- `--deal-id` takes priority; otherwise search by `--company`
- `--slack-window-days` defaults to 14

### 2. Verify HubSpot MCP tools (drift defense)

Before the first call, confirm tool exposure:
- `mcp__claude_ai_HubSpot__get_deal` or `mcp__claude_ai_HubSpot__search_deals`
- `mcp__claude_ai_HubSpot__get_contact`

Abort immediately if tool names differ.

### 3. Collect HubSpot deal info

If `--deal-id`:
1. `mcp__claude_ai_HubSpot__get_deal` to fetch the deal object
2. Expand associated contacts / company / activities (associated objects)

If `--company`:
1. `mcp__claude_ai_HubSpot__search_companies` to match the company
2. Fetch the company's open deals
3. Auto-select the deal with the most recent last activity (if multiple, prompt with the list)

### 4. Search internal Slack mentions

`mcp__claude_ai_Slack__slack_search_public_and_private`:
- query: company name + the 1-2 main contact names (pulled from HubSpot)
- after: input date - `--slack-window-days`
- before: today
- channel filter: internal channels only (exclude external-user messages — consider a separate filter option)

Collect N results. If 0, print "No internal Slack mentions".

### 5. LLM synthesis

The LLM writes the 5-minute brief in the format above:
- Show quantitative (HubSpot) and qualitative (Slack) data separately, then synthesize them in the final "Synthesis" section
- Separate internal concerns from opinions (concerns are objective fact, opinions are subjective interpretation)
- Phrase next actions so they are immediately executable 5 minutes before the call (e.g. "Ping owner @X for a follow-up after last activity", "Check Slack #eng-deals for the fit-review conclusion")

### 6. Output

Print markdown to the console. No writes. Done.

## Pitfalls

- **HubSpot MCP tool name drift**: use only after dry-run passes. Tool name changes require a plugin patch.
- **Ambiguous company names**: short or generic names trigger heavy noise in Slack search. Mitigate with an AND query of company name + contact name.
- **External-user messages mixed in**: channels like Slack `#support` host external users alongside internal opinions. Use a channel filter option (`--exclude-channels`) or an internal-channel allowlist.
- **HubSpot last activity auto-log gaps**: if the owner does not log manual call notes, last activity goes stale. Note in "Caveats".
- **Not suited for workshop demo**: no HubSpot sandbox available. Reserved for post-deployment case sharing.

## Tone

- Keep quantitative and qualitative data separated — readers must not confuse them.
- When quoting internal opinions, preserve the original wording — no paraphrasing. Use the writer's display_name as-is.
- Keep next actions short (one line) and executable 5 minutes before the call.
- "Not certain" goes in "Caveats".

---
name: battle-card-builder
description: Given a competitor name, fetch HubSpot deals where the competitor was involved (win/loss reasons, deal sizes) and search Notion competitor comparison pages. Synthesize into a 1-page sales battle card. Output only — no write side effect.
---

# battle-card-builder

A one-page battle card synthesized for a salesperson trying to answer "Which of our competitors is this account weighing us against?" in the 5 minutes before a first meeting with a new lead. HubSpot alone gives only past deal outcomes; Notion alone gives only static feature comparisons — the LLM cross-references both sources and even recommends talking points in a single pass.

**Drift risk H** — the HubSpot MCP is brand new GA (2026-04-13). Tool names may change. Mandatory: dry-run-verify the `mcp__hubspot__*` tool list before the first call.

## Input

- `--competitor <name>` (required) — competitor name (e.g. "Avalanche", "Polygon")
- `--deal-window-days <N>` (optional, default 365) — HubSpot deal search window
- Missing input → AskUserQuestion

## Output

Prints a one-page markdown battle card to the console (no writes; the salesperson copies it directly into meeting notes):

```
# Battle Card — vs <Competitor>

## One-line summary
<1-2 sentences on our core differentiator vs the competitor>

## Our strengths (3)
1. <Strength 1> — <evidence: Notion comparison page or HubSpot win case>
2. <Strength 2> — <evidence>
3. <Strength 3> — <evidence>

## Competitor strengths (3, honest)
1. <Strength 1> — <evidence: HubSpot loss case or Notion comparison>
2. <Strength 2> — <evidence>
3. <Strength 3> — <evidence>

## Past win/loss reasons (HubSpot)
- Wins: N / Losses: M / In progress: K (last <window> days)
- Top 3 loss reasons:
  1. <reason>: N
  2. ...

## Recommended talking points
- On price: <one-line line>
- On feature comparison: <one-line line>
- On trust/track record: <one-line line>

## Citations
### HubSpot deals
- Deal #<id> — <company> — <stage: won/lost> — <amount> — <close date>
- ...

### Notion comparison pages
- <page title> — <URL>
- ...

## Caveats
- HubSpot deal notes are subjective to the writer. Loss reasons may reflect the writer's interpretation.
- Notion comparison pages may be stale (last edit date is shown).
- This card is a first draft — review for 5 minutes before the meeting.
```

## Procedure

### 1. Parse input

- `--competitor` missing → AskUserQuestion
- `--deal-window-days` defaults to 365

### 2. Verify HubSpot MCP tools (drift defense)

Before the first call, confirm the tools you intend to use are actually exposed:
- `mcp__hubspot__search_companies` (or equivalent)
- `mcp__hubspot__get_company`
- `mcp__hubspot__list_deals` (or equivalent)

If a tool name differs, abort immediately — re-confirm tool names with the dry-run procedure in `docs/mcp-setup.md`, then re-run with `--competitor`.

### 3. Collect HubSpot deals

1. `mcp__hubspot__search_companies` to match the competitor name — 1-2 competitor company objects registered in HubSpot
2. Fetch deals associated with that company (within the last `--deal-window-days`)
3. For each deal, capture: stage (won/lost/open), amount, close date, lost reason notes

If 0 deals, state explicitly "No HubSpot deals featuring this competitor" (do not fabricate data).

### 4. Search Notion comparison pages

Use `mcp__notion__search` with the competitor name:
- Keywords: competitor name + "comparison" / "vs" / "competitive"
- Fetch the 1-3 closest pages
- Extract last edit date

If 0 pages, state "No comparison page in Notion".

### 5. Synthesize the battle card

The LLM writes per the output format above:
- Distribute the 3 our-strengths / 3 competitor-strengths honestly (lopsided distribution destroys trust)
- Tie recommended talking points directly to the top loss reasons (e.g. if price is the #1 loss reason, lead with the price talking point)
- Include both HubSpot deal #s and Notion URLs in citations — so the salesperson can pull them up as backup material mid-meeting

### 6. Output

Print markdown to the console. No writes. Done.

## Pitfalls

- **HubSpot MCP tool name drift**: brand new at 2026-04-13 GA. Use only after dry-run passes. Tool name changes require a plugin patch release.
- **HubSpot lost reason as free text**: notes are not standardized — the LLM has to infer categories. Group into 4-5 buckets like "price" / "missing feature" / "timing" / "other".
- **Stale Notion comparison pages**: if the last edit was a year ago, trust is low. List the last edit date in "Caveats".
- **Competitor not registered in HubSpot**: if the competitor has no HubSpot company object, you get 0 deals. Print "Competitor not registered in HubSpot" and recommend that sales register them.
- **Not suited for workshop demo**: with no HubSpot sandbox available, this plugin is not on the workshop pick list. Reserved for post-deployment case sharing.

## Tone

- Write our-strengths and competitor-strengths both honestly — no fake self-aggrandizing.
- When citing loss reasons, note that they reflect the writer's interpretation.
- Keep recommended talking points short (one line) and immediately usable mid-meeting.
- "Not certain" goes in "Caveats" — no fake confidence.

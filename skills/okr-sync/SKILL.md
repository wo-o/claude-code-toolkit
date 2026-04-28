---
name: okr-sync
description: Match Notion OKR Key Results to Slack #wins / #shipped / #launches activity over a configurable window, infer per-KR progress signals, and update each KR row with last-evidence and a derived progress pulse. Flags KRs with no matching evidence over N weeks. Designed for weekly Monday 09:00 launchd cron.
---

# okr-sync

Quarterly OKR rituals collapse for a reliable reason: nobody updates progress until the day before review. By that point three things have happened — half the KRs have moved without anybody updating them, the other half have not moved and nobody noticed, and the leadership review becomes an exercise in fiction. This skill closes the gap by matching evidence (Slack shipping signals + Notion Done pages) to KRs every Monday morning, writing the matches into each KR row, and surfacing KRs with zero evidence over N weeks as candidates for re-scoping or removal.

## Input

- `--window-weeks N` (optional, default 1) — evidence window (1 = last week)
- `--okr-db-id <id>` (optional, falls back to env `CLAUDE_CODE_TOOLKIT_OKR_DB_ID`) — Notion OKR DB ID
- `--evidence-channels <comma-separated>` (optional, falls back to env `CLAUDE_CODE_TOOLKIT_OKR_CHANNELS`) — channels to scan for shipping evidence (e.g. `#wins,#shipped,#launches,#product-updates`)
- `--stale-threshold-weeks N` (optional, default 4) — flag KRs with no evidence over this many weeks
- Missing input → AskUserQuestion (never guess)

## Output

```
2026-W17 OKR sync: K KRs scanned · M evidence events · matched-this-week W · stale (≥ S weeks) X
- OKR DB: <URL>
- Stale KRs flagged: <count>
```

The Notion OKR DB schema (one-time setup; the skill validates with `mcp__claude_ai_Notion__notion-fetch` before writing):

| Column | Type | Content |
|---|---|---|
| Key Result | title | KR statement (manually authored) |
| Objective | relation | Parent objective |
| Target | rich_text | Numeric target if any |
| Owner | people | KR owner |
| Last evidence | rich_text | Auto-updated: latest matching event summary + date |
| Last evidence link | url | Auto-updated: Slack permalink or Notion page URL |
| Evidence count (this quarter) | number | Auto-incremented counter |
| Pulse | select | Auto-set: Active / Quiet / Stale |
| Last sync | date | Auto-set to run timestamp |

## Procedure

### 1. Parse input

- `--window-weeks` clamped to [1, 13] (one-week minimum, one-quarter maximum)
- `--okr-db-id` missing AND env missing → AskUserQuestion
- `--evidence-channels` missing AND env missing → AskUserQuestion
- Validate the OKR DB schema with `mcp__claude_ai_Notion__notion-fetch` once — fail loudly if columns are missing

### 2. Fetch all KRs

Call `mcp__claude_ai_Notion__notion-fetch` against the OKR DB. Fetch every row's Key Result text + Objective + Owner + the existing auto-fields. Skip rows whose Status is `Closed` or `Archived` (custom column in some setups — best-effort detection).

### 3. Run slack-scout for evidence channels

Parallelize up to 5 slack-scout invocations, each with a `--window-weeks * 7` day window. Keep raw "Decisions / announcements" + "Key messages" lines as evidence candidates.

Then pull Notion pages whose status flipped to `Done` in the same window (re-using the weekly-shipped-from-noise step 6 pattern via notion-scout).

### 4. Match evidence to KRs (LLM-assisted)

For each `(KR, evidence)` candidate pair where the cheap keyword overlap is ≥30%, call the LLM:

```
Score 0.0~1.0 how directly this evidence advances this Key Result.

Score 1.0 = explicit completion of a milestone named in the KR
Score 0.7~0.9 = clear progress toward the KR's metric or scope
Score 0.4~0.6 = related work that may indirectly help
Below 0.4 = unrelated

Be conservative — when in doubt, score lower.

Key Result: <text>
Evidence: <message body / page title + excerpt>
```

Keep matches with score ≥ 0.7. Drop the rest.

### 5. Update each KR row

For each KR with at least one match this window:

- `Last evidence` ← top-scored match summary + date
- `Last evidence link` ← top-scored match URL
- `Evidence count (this quarter)` += number of new matches
- `Pulse` ← `Active`
- `Last sync` ← now

For each KR with zero matches this window:

- Compute weeks since `Last evidence` (use the existing `Last evidence` date if present)
- If weeks < `--stale-threshold-weeks` → `Pulse` ← `Quiet`
- If weeks ≥ `--stale-threshold-weeks` → `Pulse` ← `Stale`
- `Last sync` ← now

Call `mcp__claude_ai_Notion__notion-update-page` for each KR.

**Hook behavior:** the plugin's `PreToolUse(mcp__claude_ai_Notion__notion-update-page)` hook fires on the first update — asks once "About to update K OKR rows. Proceed?". In cron mode set `CLAUDE_CODE_TOOLKIT_CRON_MODE=1` to bypass + write only an audit log entry (`~/.claude-code-toolkit/audit/okr-sync-YYYY-Www.jsonl`).

### 6. Generate the stale-KR digest

Build a separate Notion page (or update an existing one named `OKR Stale Digest — YYYY-Www`) listing every KR whose new Pulse is `Stale`. Each row:

- KR text + Owner + weeks since last evidence + suggested action (`re-scope` / `remove` / `confirm still relevant`)

The suggested action is LLM-generated and explicitly marked `suggestion only — leadership decides`.

### 7. Output

Print the one-line console summary + DB URL + stale digest URL. Done.

## Automated operation (cron)

Recommended launchd cron at every Monday 09:00 KST:

```
# ~/Library/LaunchAgents/<reverse-domain>.claude-code-toolkit.okr-sync.plist
# Program: /usr/local/bin/claude
# ProgramArguments: -p "/claude-code-toolkit:okr-sync"
# StartCalendarInterval: { Weekday=1, Hour=9, Minute=0 }
# EnvironmentVariables:
#   CLAUDE_CODE_TOOLKIT_CRON_MODE=1
#   CLAUDE_CODE_TOOLKIT_OKR_DB_ID=<id>
#   CLAUDE_CODE_TOOLKIT_OKR_CHANNELS=#wins,#shipped,#launches,#product-updates
```

See `docs/cron-setup.md` for the full plist example.

## Pitfalls

- **KR text too vague to match anything**: KRs phrased as "Improve user satisfaction" will rarely score above 0.4 against any evidence. The skill correctly marks them `Stale` after the threshold — that is a feature (forces re-scoping), not a bug. Communicate this expectation up-front
- **Owner change between quarters**: when a KR's owner changes mid-quarter, the `Last evidence` rolls forward but the historical evidence count includes prior-quarter activity. Reset `Evidence count (this quarter)` at quarter boundaries (separate manual step)
- **Stale flagging too aggressive**: `--stale-threshold-weeks=4` is reasonable for engineering KRs with weekly shipping cadence. For research / sales / hiring KRs, use 8 or higher (per-KR threshold override is out of scope for v1)
- **Multiple KRs match the same evidence**: a single shipping event can advance 2-3 KRs. Allow this — record the same evidence link on each matching KR. Do not de-duplicate across KRs
- **Notion DB schema drift**: column renames break every run. The pre-flight `mcp__claude_ai_Notion__notion-fetch` validation catches this — exit with the expected schema printed
- **Slack window mismatch with Notion quarter**: the skill's window is N weeks, not aligned to the OKR cycle. The `Evidence count (this quarter)` column requires the operator to manually reset at quarter boundaries

## Tone

- Quote evidence text verbatim in the `Last evidence` field
- "Pulse: Stale" is a neutral status — never editorialize as "failing" or "behind"
- Stale-digest suggestions are explicitly marked `suggestion only — leadership decides`. Never strip this
- KRs without auto-detectable progress are not assumed to be unworked — say so in the digest preamble

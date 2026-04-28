---
name: knowledge-graph-builder
description: Match yesterday's Slack threads against yesterday's Notion page edits, score similarity, and bidirectionally link strong matches — Slack thread gets a bot reply citing the related Notion page; Notion page gets a "Related discussions" section with the Slack permalink. Designed for daily 18:00 launchd cron with a strict precision threshold to avoid noise.
---

# knowledge-graph-builder

Same topic, two homes — and the homes do not know each other. The Notion page "API auth design" sits next to a separate Slack thread debating the same decision; new hires see only one side. Hand-linking is a willpower problem nobody wins. This skill closes the loop automatically: every evening it scans the day's Slack threads and Notion edits, finds high-confidence matches, and inserts cross-references on both sides. False positives are the entire risk model — so the skill defaults to high-precision matching and never auto-links low-confidence pairs.

## Input

- `--since YYYY-MM-DD` (optional, defaults to yesterday 00:00 KST) — earliest message/edit to consider
- `--slack-channels <comma-separated>` (optional, falls back to env `CLAUDE_CODE_TOOLKIT_KG_CHANNELS`) — channels to scan for threads
- `--notion-workspace <id>` (optional, falls back to env `CLAUDE_CODE_TOOLKIT_KG_WORKSPACE`) — Notion workspace scope (omit for whole-account search)
- `--threshold N` (optional, default 0.75) — minimum similarity score (0.0~1.0) to auto-link
- `--review-mode` (optional flag) — output match candidates to a Notion review page instead of writing cross-references directly. Recommended for the first 1-2 weeks
- Missing input → AskUserQuestion (never guess)

## Output

```
2026-04-27 (since 2026-04-26): scanned T threads · P pages → M strong matches (≥0.75) → L bidirectional links written
- Auto-linked: <count>
- Below threshold (skipped): <count>
- Review-mode page: <Notion URL> (when --review-mode is set)
```

## Procedure

### 1. Parse input

- `--since` missing → yesterday 00:00 KST
- Channel list missing → AskUserQuestion (default to a curated 4-5 channel set — do not scan the whole workspace)
- `--threshold` clamped to [0.6, 1.0]; below 0.6 the false-positive rate becomes unacceptable

### 2. Collect Slack threads

Invoke slack-scout once per channel with the `--since` window. Then, for each thread surfaced (≥3 messages, ≥2 distinct authors), call `mcp__slack__get_channel_history` with the thread `ts` to fetch full replies.

Skip threads with <3 messages or single-author monologues — those rarely produce decision content worth cross-linking.

### 3. Collect Notion page edits

Invoke notion-scout with a custom prompt:

```
description: "Recent Notion edits scout"
subagent_type: "notion-scout"
prompt: |
  Search Notion for pages whose last_edited_time is within the window 2026-04-26 ~ 2026-04-27.
  Limit to the top 30 most-recently-edited.
  For each page, fetch title + the first 500 characters of the body.
  Output the standard format.
```

### 4. Build matching features

For each Slack thread:
- Extract noun phrases, project codenames, ALL_CAPS abbreviations, URLs
- Compute a bag-of-features signature

For each Notion page:
- Extract the same features from title + body excerpt
- Compute the same signature

### 5. Score matches (LLM-assisted)

For each `(thread, page)` candidate pair where the bag-of-features overlap is ≥30%, call the LLM with a focused prompt:

```
Compare this Slack thread to this Notion page. Output a similarity score 0.0~1.0 plus 1-line rationale.

Score 1.0 = same topic, same scope, same decision being recorded
Score 0.7~0.9 = same topic, partial overlap (e.g. thread is a debate, page is the decision)
Score 0.4~0.6 = related but different scope (thread mentions the topic in passing)
Score below 0.4 = coincidence

Be conservative — when in doubt, score lower.

Slack thread: <thread summary>
Notion page: <title> + <body excerpt>
```

This LLM step is the core of precision — do not skip it. Pure embedding distance over-merges.

### 6. Filter by threshold

Keep only matches with score ≥ `--threshold`. Discard the rest.

### 7. Write cross-references (or review-mode output)

**Default mode** — for each surviving match, write both directions:

- Slack: post a thread reply via `mcp__slack__post_message` (in-thread, as a bot) with text:
  ```
  📚 Related Notion: <page title> — <URL>
  Reply with :x: to remove this suggestion.
  ```
- Notion: update the page via `mcp__notion__update_page`, appending to a "Related discussions" section a bullet:
  ```
  - <YYYY-MM-DD> #<channel> thread — <permalink>
  ```
  If the section does not exist, create it as a heading_2 + bulleted_list_item

**`--review-mode`** — instead of writing, append match candidates to a Notion review page (`Knowledge Graph Review — YYYY-MM-DD`) with score + rationale + both URLs. The operator manually applies (or rejects) the links. Use this for the first 1-2 weeks to calibrate the threshold.

**Hook behavior:** the plugin's `PreToolUse(mcp__notion__update_page)` and `PreToolUse(mcp__slack__post_message)` hooks fire — Slack post is **never bypassed** even in cron mode (a wrong public bot reply is not silently recoverable). Notion updates can be bypassed with `CLAUDE_CODE_TOOLKIT_CRON_MODE=1` after the first successful run signs off the threshold; audit log lives at `~/.claude-code-toolkit/audit/knowledge-graph-YYYY-MM-DD.jsonl`.

### 8. Output

Print the one-line console summary + (if review-mode) the review page URL. Done.

## Automated operation (cron)

Recommended launchd cron at daily 18:00 KST:

```
# ~/Library/LaunchAgents/<reverse-domain>.claude-code-toolkit.knowledge-graph.plist
# Program: /usr/local/bin/claude
# ProgramArguments: -p "/claude-code-toolkit:knowledge-graph-builder"
# StartCalendarInterval: { Hour=18, Minute=0 }
# EnvironmentVariables:
#   CLAUDE_CODE_TOOLKIT_CRON_MODE=1
#   CLAUDE_CODE_TOOLKIT_KG_CHANNELS=#eng,#product,#design,#research
#   CLAUDE_CODE_TOOLKIT_KG_WORKSPACE=<id>
```

For the first 1-2 weeks, run with `--review-mode` (interactive only) to validate match quality. Switch to direct write only after the precision is verified.

## Pitfalls

- **False positives are catastrophic**: a wrong "Related Notion" reply on a public Slack thread is visible to everyone. Never lower the threshold below 0.6, and prefer `--review-mode` for the first runs
- **LLM scoring drift**: the same pair scored at different times can vary by ±0.1. When near the threshold, prefer not-linking
- **Notion edits storm**: API rate limits trigger on >20 update_page calls in a minute. Cap at 15 links per run; queue the rest for the next day
- **Slack bot post permission**: `chat:write` scope must be approved by the workspace admin. Without it, only the Notion side can be written — print this asymmetry in the output
- **Thread author dislikes the bot reply**: respect a `:x:` reaction by the thread author or the OP — track via a dedupe ledger to avoid re-posting the same suggestion the next day
- **Page → page linking is out of scope**: Notion-to-Notion knowledge graph is a separate skill (see roadmap). This skill only crosses Slack ↔ Notion

## Tone

- The Slack reply is a single line + footer. No rationale, no scoring exposed to readers
- The Notion bullet is a permalink with a date — no editorial summary
- "Speculative" wording is welcome in the review-mode page (when score is borderline)
- Never claim "this is the official decision page" — the skill links related material, it does not adjudicate authority

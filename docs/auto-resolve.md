# Auto-resolution of connector resources

The toolkit aims for zero-config setup. If you have already linked Slack / Notion / Gmail / Calendar via claude.ai/customize/connectors, no skill should ever ask you for a workspace ID, database ID, channel ID, or parent page ID.

This document defines the shared algorithm every skill follows when it needs to find a Notion DB, a Notion parent page, or a Slack channel.

## The cache

All resolved IDs are cached at `~/.claude-code-toolkit/auto-resolve.json`:

```json
{
  "notion": {
    "Blockers": "<db-id>",
    "Decision Log": "<db-id>",
    "Specs": "<page-id>"
  },
  "slack": {
    "#decisions": "C0XXXXXXX",
    "#wins": "C0XXXXXXX"
  }
}
```

Each skill writes the entries it resolves; subsequent runs read from the cache first. The file is plain JSON — operators can hand-edit it to override.

## Notion DB resolution (per skill)

Each skill that writes rows into a Notion DB has a **canonical DB name**. The skill resolves it like this:

1. **Cache hit** — read `auto-resolve.json` → `notion.<canonical>`. If present, validate with `mcp__claude_ai_Notion__notion-fetch`. If 200 → use it. If 404 → invalidate and fall through.
2. **Search** — call `mcp__claude_ai_Notion__notion-search` with `query=<canonical>` and a data-source / database type filter.
3. **Exact match (1 result)** — write to cache, use it.
4. **Exact match (multiple)** — pick the most recently edited; write to cache; print a one-line note `[auto-resolve] multiple "<canonical>" DBs found, using most recently edited (<id>). Edit ~/.claude-code-toolkit/auto-resolve.json to override.`
5. **No exact match** — fail with a clear setup hint:

   ```
   [auto-resolve] No Notion DB named "<canonical>" found.
   Create one in Notion with the schema below, then re-run.
   <expected schema>
   ```

The skill **never** invokes AskUserQuestion for the DB ID. The override path is `--notion-db-id <id>` — only power users editing the cache file or passing the flag should ever need to think about IDs.

## Notion parent page resolution (per skill)

Skills that create a child page (not a DB row) have a **canonical parent page name**. Same algorithm as DB resolution, except:

- Step 2 filter = `page` type
- Step 5 hint = "Create a Notion page named '<canonical>' and re-run"

## Slack channel resolution (per skill)

Each skill that scans Slack has a **canonical channel-set descriptor** — either a single name (e.g. `#decisions`) or a list / pattern (e.g. `["#wins", "#shipped", "#launches"]`).

1. **Cache hit** — read `auto-resolve.json` → `slack.<canonical>`. If present, use it.
2. **Resolve** — call `mcp__claude_ai_Slack__slack_search_channels` with the canonical name (one call per channel for a list).
3. **Match** — write the channel ID(s) to cache, use them.
4. **No match** — fail with `[auto-resolve] Slack channel "<name>" not found in your workspace. Create or join it, then re-run.`

The skill **never** invokes AskUserQuestion for a channel that matches a known canonical name. Override flag is `--channels <comma-separated>`.

## Canonical names registry

| Skill | Canonical Notion target | Canonical Slack target |
|---|---|---|
| blocker-radar | DB "Blockers" | `#help-*` | (any channel matching pattern) |
| decision-log-keeper | DB "Decision Log" | `#decisions` |
| inbox-zero-triage | DB "Inbox Triage" | (Gmail only) |
| thread-zombie-killer | DB "Open Mentions" | (auto-derived from mentions) |
| follow-up-radar | page "Follow-up Radar" (single-page mirror, optional) | (Gmail-only) |
| knowledge-graph-builder | parent page "Knowledge Graph Review" (only used in `--review-mode`) | (auto-derived from mention activity) |
| okr-sync | DB "OKRs" | `["#wins", "#shipped", "#launches"]` |
| spec-from-thread | parent page "Specs" | (caller passes thread URL) |
| meeting-prep-pull | parent page "Meeting Prep" | (Calendar only) |
| weekly-shipped-from-noise | parent page "Weekly Shipped" | `["#wins", "#shipped", "#launches"]` |
| project-brief | parent page "Project Briefs" | (caller passes content) |

## When auto-resolve is wrong

The cache is plain JSON. Edit `~/.claude-code-toolkit/auto-resolve.json` to point a skill at a different DB / page / channel. The next run picks up the new value without re-resolving.

If you want to force a re-resolve, delete the relevant key from the cache file and re-run.

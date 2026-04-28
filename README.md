# claude-code-toolkit

Team productivity Claude Code plugin. A role-aware (eng / security / PM / sales / CS) bundle of automation skills + subagents.

- **Skills:** 14
  - On-demand: project-brief, decision-log-keeper, spec-from-thread, ticket-triage-router, battle-card-builder, deal-context-loader, meeting-prep-pull
  - Scheduled (designed for cron): weekly-shipped-from-noise, blocker-radar, knowledge-graph-builder, okr-sync, thread-zombie-killer, inbox-zero-triage, follow-up-radar
- **Subagents:** 5 (synthesizer, slack-scout, notion-scout, ticket-classifier, ticket-draft-writer)
- **External integrations:** Slack / Notion / HubSpot / Gmail / Google Calendar — toggle Anthropic's official Connectors in the `claude.ai/customize/connectors` UI. The plugin does not manage Connector registration.
- **Security baseline (optional):** if your organization already has a Claude Code baseline (sandbox + deny rules + PreToolUse blocking), we recommend layering this plugin on top of it. The plugin also runs standalone if no baseline exists.

Design rationale: `docs/workshop-flow.md`

---

## Prerequisites

1. **Install Claude Code** — macOS / Linux. Verify the version:
   ```
   claude --version
   ```
2. **GitHub CLI authentication** — required to access the marketplace where the plugin is hosted:
   ```
   gh auth login
   gh auth status
   ```
3. **(Optional) Install the security baseline first** — if an org-wide sandbox + deny rule baseline exists, we recommend layering this plugin on top. With the baseline enforcing the following, the plugin runs safely as-is:
   - `sandbox.enabled: true` — macOS Seatbelt / Linux bubblewrap isolation
   - Deny rules protecting system files, credentials, and secrets
   - PreToolUse hooks: blocking `rm -rf` and `git push main`

   Without a baseline, the plugin still runs standalone, but we recommend adopting the guards above separately.

---

## Installation

> Until marketplace hosting is finalized, install from a local plugin directory:
>
> ```
> claude /plugin install /path/to/claude-code-toolkit
> ```

```
claude /plugin marketplace add <marketplace-url>
claude /plugin install claude-code-toolkit
```

After install, verify activation:

```
claude /plugin list
# Confirm claude-code-toolkit (enabled) is shown
```

---

## Enabling Connectors (Slack / Notion / HubSpot / Gmail / Google Calendar)

This plugin does not register external integrations directly. We use Anthropic's official Connectors instead — reasons:

- One toggle in your claude.ai account → automatically exposed across CLI / Desktop / Mobile
- OAuth is handled by Anthropic infrastructure, so no tokens are stored on the user's machine
- Skill code stays unchanged — the tool IDs (`mcp__slack__*`, `mcp__notion__*`, `mcp__hubspot__*`, `mcp__gmail__*`, `mcp__google_calendar__*`) remain identical
- **Gmail outbound is draft-only by design** — the Connector exposes `create_draft` but not `send_message`. Drafts land in the operator's Gmail Drafts folder and are never sent until the operator clicks send in the Gmail UI. This eliminates the "wrong message went out" risk that exists for Slack `post_message`.

Per-skill Connector requirements:

| Skill | Connectors |
|---|---|
| project-brief, decision-log-keeper, spec-from-thread, ticket-triage-router | Slack, Notion |
| battle-card-builder, deal-context-loader | HubSpot, Notion |
| weekly-shipped-from-noise, blocker-radar, knowledge-graph-builder, okr-sync, thread-zombie-killer | Slack, Notion |
| inbox-zero-triage, follow-up-radar | Gmail, Notion |
| meeting-prep-pull | Gmail, Google Calendar, Notion |

Enablement steps:

1. Visit https://claude.ai/customize/connectors
2. Toggle the Connectors required by the skills you plan to use, then walk through OAuth (workspace admin approval may be required)
3. Verify exposure from the CLI:
   ```
   claude /mcp
   # Confirm mcp__slack__*, mcp__notion__*, mcp__hubspot__*, mcp__gmail__*, mcp__google_calendar__* are exposed
   ```

> **GitHub is intentionally excluded.** Anthropic's official GitHub Connector is read-only (file/branch lookup only — no PR diff/comment, no Issue write), and the demo skills that performed PR writes (pr-pattern-auditor / pr-storybook) have been removed in v1. The GitHub automation workflow will be split out into a separate trial repo.

---

## Usage

Each skill is invoked via `/claude-code-toolkit:<skill-name>`.

### On-demand skills (manual invocation)

```
# One-page project brief (Slack + Notion scouts)
/claude-code-toolkit:project-brief --project <project-name>

# Auto-organize that day's #decisions into a Notion DB
/claude-code-toolkit:decision-log-keeper --date 2026-04-26

# Slack thread → auto-generated Notion spec page
/claude-code-toolkit:spec-from-thread --thread-url https://...

# Slack #support message → classification + draft reply (Pattern C isolation)
/claude-code-toolkit:ticket-triage-router --message-url https://...

# Sales-oriented two
/claude-code-toolkit:battle-card-builder
/claude-code-toolkit:deal-context-loader

# Meeting pre-read from Calendar + Gmail + Notion (next event by default)
/claude-code-toolkit:meeting-prep-pull
```

### Scheduled skills (designed for cron, also invokable on-demand)

```
# Weekly Shipped page from Slack shipping signals (Friday 17:00 KST)
/claude-code-toolkit:weekly-shipped-from-noise --week 2026-W17

# Daily blocker radar across configured channels (weekdays 09:00 KST)
/claude-code-toolkit:blocker-radar --since 2026-04-26

# Slack thread ↔ Notion page bidirectional cross-references (daily 18:00 KST)
/claude-code-toolkit:knowledge-graph-builder --review-mode

# Match Notion OKR KRs to Slack shipping evidence, mark stale KRs (Monday 09:00 KST)
/claude-code-toolkit:okr-sync --window-weeks 1

# Surface dormant @mentions in 3d / 7d / 14d tiers (weekdays 09:00 KST)
/claude-code-toolkit:thread-zombie-killer

# Daily inbox triage — classify + label + Notion DB upsert (weekdays 08:00 KST)
/claude-code-toolkit:inbox-zero-triage --since-hours 24

# Personal Gmail follow-up dashboard (3d / 7d / 14d, weekdays 09:00 KST)
/claude-code-toolkit:follow-up-radar
```

Recommended skills by role — see `docs/workshop-flow.md`. Per-skill cron setup — see `docs/cron-setup.md`.

---

## Cron / external trigger registration (optional)

Eight of the fourteen skills can be cron-scheduled (seven scheduled-by-design + decision-log-keeper as a daily-cron-friendly on-demand skill). Claude Code hooks cannot fire external cron, so use `launchd` on macOS and `systemd timer` / `cron` on Linux.

| Skill | Recommended cadence | Bypass env var |
|---|---|---|
| decision-log-keeper | Daily 18:00 KST | `CLAUDE_CODE_TOOLKIT_CRON_MODE=1` |
| weekly-shipped-from-noise | Friday 17:00 KST | `CLAUDE_CODE_TOOLKIT_CRON_MODE=1` |
| blocker-radar | Weekdays 09:00 KST | `CLAUDE_CODE_TOOLKIT_CRON_MODE=1` |
| knowledge-graph-builder | Daily 18:00 KST (start with `--review-mode`) | `CLAUDE_CODE_TOOLKIT_CRON_MODE=1` (Notion only — Slack post never bypassed) |
| okr-sync | Monday 09:00 KST | `CLAUDE_CODE_TOOLKIT_CRON_MODE=1` |
| thread-zombie-killer | Weekdays 09:00 KST | `CLAUDE_CODE_TOOLKIT_CRON_MODE=1` |
| inbox-zero-triage | Weekdays 08:00 KST | `CLAUDE_CODE_TOOLKIT_CRON_MODE=1` |
| follow-up-radar | Weekdays 09:00 KST | `CLAUDE_CODE_TOOLKIT_CRON_MODE=1` (only relevant when `--notion-mirror-page-id` is used) |

Example — macOS launchd:

```
# ~/Library/LaunchAgents/<reverse-domain>.claude-code-toolkit.decision-log-keeper.plist
# StartCalendarInterval Hour=18 Minute=0 → claude -p "/claude-code-toolkit:decision-log-keeper --date $(date +%Y-%m-%d)"
```

Per-skill plist + systemd timer examples in `docs/cron-setup.md`.

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `/plugin install` fails — 404 | gh not authenticated, or insufficient permission on the marketplace repo | `gh auth refresh -h github.com -s repo` |
| `/mcp` does not list the server | Connector toggle missing, or claude.ai not authenticated | Enable at https://claude.ai/customize/connectors and restart Claude Code |
| Slack `chat:write` denied | Workspace admin has not approved | Request Connector approval from admin and re-run OAuth |
| `Bash(rm -rf *)` blocked | Baseline working as intended (not overridden) | Intended behavior — use safe tools like `trash` |
| User confirmation prompt right before decision-log-keeper Notion write | Plugin-specific hook (`PreToolUse(mcp__notion__create_page)`) | Approve with y, or disable the hook: `--allowedTools "mcp__notion__create_page"` |
| In CI/cron, hook prompt blocks (`read -r _ </dev/tty` waiting) | Plugin hook is requesting interactive confirmation | All scheduled skills support `CLAUDE_CODE_TOOLKIT_CRON_MODE=1` (per-skill audit logs: `decision-log-YYYY-MM-DD.jsonl`, `weekly-shipped-YYYY-Www.jsonl`, `blocker-radar-YYYY-MM-DD.jsonl`, `knowledge-graph-YYYY-MM-DD.jsonl`, `okr-sync-YYYY-Www.jsonl`, `zombie-killer-YYYY-MM-DD.jsonl`, `inbox-triage-YYYY-MM-DD.jsonl`, `follow-up-radar-YYYY-MM-DD.jsonl`). spec-from-thread and meeting-prep-pull are interactive-only in this release (audit log not separated — see §Security model) |
| `knowledge-graph-builder` over-linking | Threshold too low, false positives | Raise `--threshold` (default 0.75 — never below 0.6) or run `--review-mode` for 1-2 weeks before enabling auto-link |
| `inbox-zero-triage` over-classifies as `action_required` | LLM confidence too low default to action | Raise the classifier confidence floor in the skill (currently 0.6 → FYI fallback). Add domain-specific examples to the ticket-classifier subagent |
| `meeting-prep-pull` returns "No meetings" | No upcoming events in the next 24h, or `--internal-domain` mismatch | Pass `--event-id` explicitly, or set `CLAUDE_CODE_TOOLKIT_PREP_INTERNAL_DOMAIN` correctly (multiple domains comma-separated) |
| Audit log files piling up | Per-day JSONL append, no rotation | Clean up weekly/monthly per your operations policy (`find ~/.claude-code-toolkit/audit -mtime +30 -delete`, etc.) |

---

## Security model

- The org baseline's deny rules + sandbox + PreToolUse blocks on `rm -rf` / `git push main` are **never overridden** — they keep working as-is.
- Plugin-specific hooks (`hooks/hooks.json`) only add coverage in areas the baseline does not define:
  - `PreToolUse(mcp__notion__create_page)` — one user confirmation right before adding a Notion DB row (in cron mode, audit log only)
  - `PreToolUse(mcp__notion__update_page)` — same shape, fires for upserts (blocker-radar, okr-sync, knowledge-graph-builder, thread-zombie-killer)
  - `PreToolUse(mcp__slack__post_message)` — **never bypassed by `CRON_MODE`**. A wrong public Slack message is not silently recoverable. knowledge-graph-builder gates Slack posts here
  - `mcp__gmail__create_draft` is **NOT gated** — Gmail drafts land in the operator's Drafts folder and never leave the account until the operator presses send in the Gmail UI. The Connector intentionally has no `send_message` API, so the "wrong message went out" risk for Gmail is zero by construction
  - `mcp__gmail__label_thread` / `mcp__gmail__label_message` are **NOT gated** — labels are visible only to the authenticated Gmail account and can be removed with one click
- The three demo picks (decision-log-keeper, ticket-triage-router, spec-from-thread) use **per-tool permission isolation** for their subagents — read-only / write are split (Pattern C). See each skill's `agents/` definition + the frontmatter `tools` allowlist. The seven scheduled skills + on-demand meeting-prep-pull reuse the same subagents (slack-scout, notion-scout, ticket-classifier, synthesizer) under the same Pattern C model — no new subagents introduced.
- **External integrations only use Anthropic's official Connectors** — for Slack/Notion/HubSpot, OAuth tokens are managed by Anthropic infrastructure and no secret is stored on the user's disk. The plugin ships no `.mcp.json` of its own, so there is no environment-variable PAT injection surface.
- **No third-party broker (Composio, etc.):** the Google Drive Composio integration that shipped in v0 has been removed in v1 (avoiding the trust hop where OAuth refresh tokens get delegated to external infrastructure). Will reconsider once Anthropic's native GDrive Connector reaches GA.
- **When to use bypass environment variables:**
  - `CLAUDE_CODE_TOOLKIT_CRON_MODE=1` — for decision-log-keeper Notion row creation. Audit log: `~/.claude-code-toolkit/audit/decision-log-YYYY-MM-DD.jsonl`
  - In this release spec-from-thread is **interactive-confirm only** (the current hook does not split audit logs by caller, so using bypass would mix its audit trail with decision-log-keeper's). Cron enablement will be revisited once caller-aware audit log separation is in place.
  - Never `export` bypass environment variables in your normal shell environment — only inject them via cron plist or GitHub Actions secrets.

---

## Workshop

The 14-skill catalog is too much for a single 5-minute demo, so the workshop runs a 3-skill core lineup. Beyond that core, the extended lineup in `docs/workshop-flow.md` adds longer demos (`weekly-shipped-from-noise`, `blocker-radar`, `knowledge-graph-builder`, `meeting-prep-pull`, etc.) for follow-up office hours.

Core 5-minute demo flow:

```
[T+0:30] /claude-code-toolkit:decision-log-keeper --date 2026-04-26       (conservative warm-up, H+X)
[T+1:30] /claude-code-toolkit:ticket-triage-router --message-url <slack>  (V+X wow moment)
[T+3:30] /claude-code-toolkit:spec-from-thread --thread-url <slack>       (X climax — raw → spec transformation)
```

Extended demos and audience-specific picks: `docs/workshop-flow.md`. Per-skill cron registration: `docs/cron-setup.md`.

---

## License / contributing

- License: MIT (see `LICENSE`).
- Issues / PRs: https://github.com/wo-o/claude-code-toolkit
- For design changes: discuss in an issue first, then open a PR.

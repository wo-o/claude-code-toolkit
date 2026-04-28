# Workshop Flow

5-minute workshop demo guide. The three picks (conservative → V → X climax) run in a fixed order.

## Demo sequence

| Time | Command | Intent | Tone |
|---|---|---|---|
| T+0:30 | `/claude-code-toolkit:decision-log-keeper --date 2026-04-26` | Auto-organize decisions Slack→Notion | H+X warm-up — "Today's decisions are scattered, right? Done in one line." |
| T+1:30 | `/claude-code-toolkit:ticket-triage-router --message-url <slack-permalink>` | CS message classification + draft reply (Pattern C isolation) | V+X wow moment — visualize subagent permission isolation |
| T+3:30 | `/claude-code-toolkit:spec-from-thread --thread-url <slack-permalink>` | Slack thread raw conversation → auto-generated Notion spec page | X climax — hook prompt fires right before Notion write |

5 minutes total. Allow a 30-second handoff line between each demo.

## Pre-demo prep

- Prepare a demo Slack workspace: 3-5 demo messages in `#decisions`, 1 user message in `#support` (classification target), and one separate thread of 10-20 messages with decisions/requirements (for spec-from-thread)
- Prepare a demo Notion workspace: one "Decision Log" DB (Title/Date/Channel/Owner/Decision columns) + one parent page named "Specs" (spec-from-thread will add child pages under it)
- Presenter environment: complete Slack/Notion OAuth in claude.ai/customize/connectors in advance. **Do not export `CLAUDE_CODE_TOOLKIT_CRON_MODE` during the demo** — showing the hook prompt is the whole point.

## Presenter notes

- decision-log-keeper: "Normally runs daily at 18:00 via launchd cron — for the workshop we invoke it manually."
- ticket-triage-router: emphasize permission isolation between classifier (Slack read-only) and draft-writer (Notion search read-only). "If one agent holds every permission, your audit trail is broken."
- spec-from-thread: deliberately show the moment the hook prompt fires. "This is what the plugin does — forces user confirmation right before a Notion write. It is also the most visceral way to show raw conversation being transformed into a structured spec doc."

## Skills not suited for the 5-minute demo

| Skill | Why it does not fit the core slot |
|---|---|
| project-brief | 2 scouts + synthesizer takes the demo over budget (3min+) |
| battle-card-builder | No HubSpot sandbox available (Drift H, GA newcomer) |
| deal-context-loader | Same reason (HubSpot Drift H) |

## Extended demo lineup (longer sessions / office hours)

The 7 scheduled + 1 on-demand "extended" skills do not fit the 5-minute slot but earn loud reactions in 15-30 minute follow-up sessions. Pick by audience.

### Engineering audience (15-min slot)

| Time | Command | Intent | Tone |
|---|---|---|---|
| T+0:30 | `/claude-code-toolkit:knowledge-graph-builder --review-mode` | Match yesterday's Slack threads to yesterday's Notion edits, score similarity, surface candidate cross-references | "The same decision lives in two places — Slack and Notion — and the homes do not know each other. This skill closes the loop, but with a precision threshold you can tune." |
| T+5:30 | `/claude-code-toolkit:blocker-radar --since 2026-04-27` | Daily blocker scan across configured channels, upsert into Notion DB with Tier=New/Aging/Stale | "Blockers buried in Slack die quietly. Tier shifts as days pass — operators see the same blocker once and stop seeing it after it is resolved." |
| T+10:30 | `/claude-code-toolkit:meeting-prep-pull` | Calendar + Gmail + Notion three-source synthesis → one-page Notion pre-read for the next meeting | "The 10-minute pre-meeting cram, automated. The skill picks the next event, gathers 14 days of email per attendee, finds related Notion pages, and writes a one-pager." |

### Product / PM audience (15-min slot)

| Time | Command | Intent | Tone |
|---|---|---|---|
| T+0:30 | `/claude-code-toolkit:weekly-shipped-from-noise --week 2026-W17` | Friday 17:00 — gather the week's shipping signals from `#wins` / `#shipped` / `#launches` plus Notion Done flips, write a single Shipped page | "PM does this manually on Friday afternoon every week. This skill replaces 90 minutes of scrolling with a 60-second run." |
| T+5:30 | `/claude-code-toolkit:okr-sync --window-weeks 1` | Monday 09:00 — match each KR to evidence over the last week, mark KRs with no evidence as Stale | "OKR rituals collapse because nobody updates progress until review day. This runs every Monday so the leadership review is grounded in evidence, not fiction." |
| T+10:30 | `/claude-code-toolkit:thread-zombie-killer` | Surface dormant `@mentions` in 3d / 7d / 14d tiers, grouped by responder | "A teammate `@mention`s you Tuesday. By Friday it is buried. This is the largest source of trust erosion in remote teams — and it is a personal dashboard, not a team scoreboard." |

### Sales / CS audience (10-min slot)

| Time | Command | Intent | Tone |
|---|---|---|---|
| T+0:30 | `/claude-code-toolkit:battle-card-builder` | Build a one-pager battle card for a prospect (HubSpot + public web data) | "Pre-call prep, in one command. Sales reps stop digging through HubSpot history mid-call." |
| T+4:30 | `/claude-code-toolkit:deal-context-loader` | Load deal context into the current conversation — recent emails, calls, support tickets | "Hand-off between AE and CSM stops being a 30-minute meeting. Context preloaded." |

### Inbox-heavy audience (10-min slot)

| Time | Command | Intent | Tone |
|---|---|---|---|
| T+0:30 | `/claude-code-toolkit:inbox-zero-triage --since-hours 24` | Daily 08:00 — classify unreplied threads (action_required / FYI / newsletter / spam-like), label in Gmail, upsert action_required to Notion | "By 08:30 the inbox is already 40 threads deep. The skill pre-sorts them so the operator sees only the threads that actually need a reply today." |
| T+4:30 | `/claude-code-toolkit:follow-up-radar` | Personal Gmail follow-up dashboard — emails the operator sent that have gone unanswered, tiered by 3d / 7d / 14d | "The personal mirror of thread-zombie-killer. The skill is read-only by default. It is a private dashboard, never broadcast." |

### Why these are not in the core 5-minute slot

- **Latency**: scheduled skills aggregate over 24h-7d windows. Even a manual invocation needs 60-180 seconds. The core demo budget is 90 seconds per skill
- **Hook prompts**: each scheduled skill triggers `PreToolUse(mcp__notion__create_page)` or `PreToolUse(mcp__notion__update_page)`. Showing one hook is the X climax; showing four is fatigue
- **Audience match**: `blocker-radar` resonates with eng managers but bores PMs; `okr-sync` is the reverse. The core 3 picks were chosen for cross-role appeal

## Presenter prep for extended demos

- **`weekly-shipped-from-noise`**: 4-6 demo messages across `#wins` / `#shipped` / `#launches`, plus 1 Notion page whose status flipped to Done in the same window
- **`okr-sync`**: a Notion OKR DB with at least 3 KRs (one with clear shipping evidence, one borderline, one with nothing) — this lets you show all three Pulse states (Active / Quiet / Stale)
- **`thread-zombie-killer`**: 2-3 unanswered `@mentions` from 4-5 days ago, plus 1 from 8 days ago, to populate both the 3d and 7d tiers
- **`knowledge-graph-builder`**: run `--review-mode` only during the demo. Auto-link mode is too risky for a live audience — one wrong public bot reply is a memorable disaster
- **`blocker-radar`**: 3-4 messages with explicit blocker language (`blocked by`, `waiting on`, `can't proceed`) plus 1 mention from earlier this week (so Tier=Aging shows up)
- **`inbox-zero-triage`**: 8-12 demo emails arriving in the last 24h — mix of action_required (a customer asking a question), FYI (an internal status email), newsletter (one with `unsubscribe` in the footer), and spam-like (cold sales template). The Notion "Inbox Triage" DB must be created with the schema in `skills/inbox-zero-triage/SKILL.md`
- **`follow-up-radar`**: 2-3 sent emails from the operator's account that have been unanswered for 4-5 days, plus 1 from 8 days ago. Set `CLAUDE_CODE_TOOLKIT_FOLLOWUP_MY_EMAIL` to the demo Gmail address
- **`meeting-prep-pull`**: 1 Google Calendar event in the next 1-2 hours with 1-2 external attendees, 2-3 demo email threads with each attendee from the last 14 days, and 1-2 Notion pages whose titles overlap with the event title. Set `CLAUDE_CODE_TOOLKIT_PREP_PARENT_ID` and `CLAUDE_CODE_TOOLKIT_PREP_INTERNAL_DOMAIN`

## Post-demo operations

- After the demo, share the install guide with workshop attendees (`README.md` §Installation + §Enabling Connectors)
- Open a Slack `#claude-toolkit` channel for Q&A and bug reports
- A week later, collect usage stats (Notion DB row count, audit log line count) and review operations

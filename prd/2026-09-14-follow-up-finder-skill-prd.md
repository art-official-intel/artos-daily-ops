---
document_type: feature-prd
feature_name: follow-up-finder
component: artos-followup-slack (adapter — see core PRD for shared logic)
track: Glean agent port (personal productivity, not tied to a GTM workstation)
status: draft-for-iteration
version: "0.3"
created: 2026-09-14
updated: 2026-09-14 (embellishments added, see core PRD v1.1)
author: Art Hernandez
project: none yet — see Open Items
source_agent: Glean "A1 - Follow Up Finder (Slack)" (https://app.glean.com/chat/agents/a0c54df18ac84a84a1e57ac4e193031c)
source_plan: Glean-authored portability plan, reviewed 2026-09-14 (https://docs.google.com/document/d/1D9xg1cWUh8CYB5RrlXQBJ7_S0YSYkZ2ieOvx3LVCQeM)
precedent: artos-meeting-prep (Glean "A1 - Meeting Prep" port, shipped 2026-09-10)
core_prd: 2026-09-14-follow-up-finder-core-prd.md
sibling_adapter: 2026-09-14-follow-up-finder-gmail-prd.md
supersedes: v0.1 of this same file (pre-split; shared logic has moved to the core PRD, nothing here contradicts v0.1, it's just been trimmed)
depends_on:
  - Slack connector (connected, available now)
  - Google Drive connector (connected, active v1 source)
  - Google Calendar connector (connected, active v1 source)
  - Atlassian/Jira connector (NOT yet authorized — blocks task-system status checks)
  - Salesforce connector (NOT yet authorized — optional status source, not required for v1)
tags: [glean-port, slack, follow-ups, personal-productivity, adapter]
---

> _Note: names and company references in this document (e.g., "Northbridge Systems," "Meridian," "Summit Ridge," "Riley Chen," "Dana," "Jordan") have been substituted with fictional stand-ins for version control. The design decisions and logic below are unchanged and real._

# Feature PRD: Follow-Up Finder — Slack Adapter (`artos-followup-slack`) — v0.3

> **Read the core PRD first** (`2026-09-14-follow-up-finder-core-prd.md`) — it has the extraction rules, status taxonomy, safety bar, output shape, and test method, all shared with the Gmail adapter. This document covers only what's specific to Slack: which tools, which table columns, and this adapter's own test history. As of v0.2, this file no longer restates logic that lives in the core PRD — see the core PRD's change log for what moved.

---

## Why this adapter

Art asked Glean to draft a plan for porting its "Follow Up Finder (Slack)" agent off Glean. That plan is genuinely useful for its business logic, but written as a coded microservice — not how skills get built here. See the core PRD for the architecture decision (single Claude skill, no microservice) and why the shared logic is single-sourced there rather than repeated per adapter.

---

## What This Adapter Does (Plain English)

On demand, scans Art's Slack activity over a date range, pulls out things he committed to or was asked to do, checks whether each has since been resolved (across Slack, Drive, and Calendar), and produces one Markdown digest per the core PRD's output shape.

---

## Non-Goals (Slack-specific — see core PRD for the shared ones)

- **Doesn't do task-system status checks yet.** Atlassian (Jira) isn't authorized in this environment. v1 evidence sources are Slack, Drive, and Calendar (expanded after Run #1 — see findings below).

---

## Packaging & Triggers

*Resolved 2026-09-14 — see core PRD's Triggers & Invocation for the shared rules (umbrella phrase, date-range default disclosure).*

- **Skill name:** `artos-followup-slack`. Standalone skill, no workstation — same pattern as `artos-meeting-prep`. Art may want to distribute this beyond his own account.
- **Source-specific trigger phrases:** "check my Slack follow-ups," "Slack follow-up finder," "what am I on the hook for in Slack," or similar — any clear ask for a Slack-scoped follow-up digest.
- **Umbrella phrase:** also responds (alongside the Gmail adapter) to a combined ask like "check my follow-ups" — renders its own separate digest, run back-to-back with Gmail's, never merged.
- **Date range:** optional; defaults to yesterday → now per Runtime Configuration below. If the user doesn't specify one, the digest opens with a line noting the default was used (per core PRD).

---

## Tool Mapping

| Abstract capability (source plan) | Concrete tool here | Status |
|---|---|---|
| Slack search + full thread retrieval | Slack connector (`slack_search_public`, `slack_search_public_and_private`, `slack_read_channel`, `slack_read_thread`) | Connected, usable now |
| Status adapter: Slack | Same Slack tools, re-queried per follow-up item | Connected |
| Status adapter: task system (Jira/Linear) | Atlassian connector | **Not authorized** — needs connecting before task-system status checks can be included |
| Status adapter: Salesforce | Salesforce connector | Not authorized — optional, not required for v1 |
| Status adapter: Drive/Notion | Google Drive connector | **Active v1 source**, confirmed load-bearing by Run #1 (a real item's true status lived in Drive, not Slack). Notion excluded — not connected. |
| Status/calendar adapter | Google Calendar connector | **Active v1 source**, added after Run #1 (control items depended on meeting/RSVP data Slack doesn't carry). |
| Nudge drafting | Slack connector (`slack_send_message_draft`) | Added 2026-09-14 (core PRD's Actionability embellishment). Drafts only — never sends. |

---

## Runtime Configuration

- `user_email`: ahernandez@siftscience.com
- Identity resolution: the Slack connector self-identifies the acting user — no separate handle needed.
- `local_timezone`: America/New_York
- `start_date` / `end_date`: defaults per core PRD (yesterday → now) unless stated
- `enabled_status_sources`: Slack, Google Drive, Google Calendar

**Test window used so far:** `start_date` = 2026-09-07, `end_date` = 2026-09-14 — matches Art's real Glean Follow-Up Finder control output for the same range.

---

## Slack-Specific Steps

Steps 1-3 are Slack-specific mechanics. Steps 4-7 apply the core PRD's rules directly — restated briefly here only to note which Slack tools/columns they use.

**Step 1 — Resolve date range & identity.** Confirm start/end date; identity resolves automatically via the Slack connector.

**Step 2 — Slack retrieval.** Search Slack (channels + DMs Art has access to) within the date range. Pull complete parent + reply threads, not just matching messages — partial threads produce false follow-ups and false completions. Retain: message/thread URL, channel or DM name, author, timestamp, thread ID.

**Step 3 — Normalize & dedupe.** Merge messages from the same Slack thread describing the same work into one candidate item before extraction.

**Step 4 — Extract follow-up items.** Apply the core PRD's extraction rules to the normalized Slack threads.

**Step 5 — Gather status evidence.** Apply the core PRD's evidence-gathering rules, checking Slack, Drive (actual current content of any linked doc, not just the message that shared it), and Calendar (invite existence, RSVP state, whether a meeting's already happened). Jira/Linear folds in here once Atlassian is authorized.

**Step 6 — Classify status.** Apply the core PRD's status taxonomy.

**Step 7 — Render the digest.** Apply the core PRD's output shape (Need Action split into Waiting on You / Waiting on Others, aging notes, cross-source flag, confidence tags where applicable). Slack-specific summary table columns: **Conversation, Topic, Status.**

Voice: per core PRD (light assistant persona) — no Slack-specific override.

---

## Embellishments (added 2026-09-14, per core PRD v1.1)

- **Nudges:** text-only for now, per Art's explicit call (2026-09-14) — display suggested nudge text inline under each Waiting-on-Others item. `slack_send_message_draft` stays mapped in the Tool Mapping above for later, but must not be called yet.
- **Cross-source overlap flag:** if the Gmail adapter's most recent digest for an overlapping window is readable, check for plausible matches and add a one-line pointer. Concrete case already observed: Northbridge MNDA shows up in both.
- **Aging & run history:** persist state at `output/follow-up-finder-slack-state.json` (or the account-appropriate output path once packaged), keyed by Slack thread_id. Records first-seen, last-seen, appearance count, nudge count per item.
- **Confidence tagging:** not yet applicable — this adapter has no inference-based classification rule of its own (unlike Gmail's ownership/suspect-solicitation heuristics). Revisit if one gets added later.
- **Scheduling:** explicitly not adopted yet, per core PRD.

---

## Key Decisions (Slack-adapter-specific)

Architecture, career-item handling, and voice are decided once in the core PRD and inherited here. This table tracks only calls specific to this adapter.

| Decision | Choice | Confirmed |
|---|---|---|
| v1 status sources | Slack + Drive + Calendar; Jira added once Atlassian is authorized | 2026-09-14, expanded after Run #1 findings |
| Cadence | On-demand only for now | 2026-09-14, matching meeting-prep's rollout |
| Build sequence | PRD → manual test → revise → retest → package as SKILL.md | 2026-09-14; Run #1 and Run #2 both complete |

---

## Open Items

- [x] **Slack handle** — moot; the connector self-identifies the acting user.
- [x] **Date-range default** — resolved: keep "yesterday through now," with the digest always disclosing when that default was used (see core PRD's Triggers & Invocation).
- [x] **Where this lives** — resolved: standalone skill (`artos-followup-slack`), no workstation, same as meeting-prep. Art may want to distribute it.
- [x] **Voice** — confirmed via core PRD: light assistant persona.
- [x] **Jira timing** — resolved by running without it. Confirmed workable as a first build.
- [x] **Calendar as a status-evidence source** — confirmed.
- [x] **Drive as an active v1 source** — confirmed.
- [x] **Personal-commitment handling** — resolved via core PRD: separate "Art Only" section, not excluded.

---

## Test & Iteration Plan

Method is the core PRD's shared Test Method (diff against a real Glean control for the same window). This adapter's own history:

**Run #1 (2026-09-14): done.** See `resources/2026-09-14-control-test-report.md`. Extraction recall was clean; one specific citation (quote + timestamp) verified exactly against the real thread. Every status miss traced to Slack-only scope (Drive/Calendar not yet in scope), not a logic error — plus one gap on personal-commitment filtering, since resolved via the core PRD's carve-out.

**Run #2 (2026-09-14): done.** See `resources/2026-09-14-control-test-report-run2.md`. With Calendar + Drive added: the Sept 15 review session confirmed real, but with a corrected time (10 AM ET, not 9 AM as circulated) and a sharper attendee-confirmation read than the Glean control gave. More significantly: **the Glean control's own "Completed" call on the Meridian deck was wrong** — the real Drive file showed the requested revisions were never applied. Verified, not assumed. Confirms the core PRD's rule that status classification must check a linked resource's actual current state.

**Run #3 (2026-09-14): done.** See `resources/2026-09-14-run3-embellishment-validation-report.md`. No new extraction — exercised all four core-PRD embellishments end to end against the same window: Waiting on You/Others split, text-only nudges (5 items, no draft/send tool called), the cross-source pointer to the Gmail digest on Northbridge, and aging notes sourced from a new state file (`output/2026-09-14-follow-up-finder-slack-state.json`). No PRD changes triggered — existing rules covered every judgment call.

**Sample output rendered:** `output/2026-09-14-follow-up-finder-digest-sample.md` — now reflects all three runs' corrections plus the confirmed "Art Only" section, voice, and full embellishment set.

Next: package as a SKILL.md that reads the core rules resource (`resources/2026-09-14-follow-up-finder-core-rules.md`) plus this adapter's Slack-specific tool mapping and table shape. All Open Items above are resolved.

---

## Conventions Applied

- Dated filename per naming standard; this file kept its original name across the v0.1 → v0.2 split (no rename) to avoid breaking existing links to it.
- Shared logic lives in the core PRD as of v0.2 — see its change log for what moved out of this file.
- Draft-for-iteration status — not yet a build-ready SKILL.md.

---

*Follow-Up Finder — Slack Adapter PRD (`artos-followup-slack`) v0.3 · 2026-09-14*

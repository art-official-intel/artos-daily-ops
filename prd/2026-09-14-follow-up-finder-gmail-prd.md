---
document_type: feature-prd
feature_name: follow-up-finder
component: artos-followup-gmail (adapter — see core PRD for shared logic)
track: Glean agent port (personal productivity, not tied to a GTM workstation)
status: dry-run-complete-proceeding-without-control
version: "0.3"
created: 2026-09-14
updated: 2026-09-14 (embellishments added, see core PRD v1.1)
author: Art Hernandez
project: none yet — see Open Items
source_agent: Glean "A1 - Follow Up Finder (GMAIL)" (https://app.glean.com/chat/agents/598ad1a092ea4fa890e660e75c260d47)
source_plan: Glean-authored portability instructions, reviewed 2026-09-14 (https://docs.google.com/document/d/1wy4r2I_Of2FW6g1bg1U6NpGmWj6MNfcTJAFrN6DAltg)
precedent: artos-followup-slack (2026-09-14-follow-up-finder-skill-prd.md — built, tested, 2 runs complete)
core_prd: 2026-09-14-follow-up-finder-core-prd.md
sibling_adapter: 2026-09-14-follow-up-finder-skill-prd.md (Slack)
decision_trail: 2026-09-14-gmail-consolidation-decision.md
supersedes: none — net new
depends_on:
  - Gmail connector (connected, available now)
  - Slack / Drive / Calendar connectors (optional, for corroborating status evidence only — see Non-Goals)
tags: [glean-port, gmail, follow-ups, personal-productivity, adapter]
---

> _Note: names and company references in this document (e.g., "Northbridge Systems," "Meridian," "Summit Ridge," "Riley Chen," "Dana," "Jordan") have been substituted with fictional stand-ins for version control. The design decisions and logic below are unchanged and real._

# Feature PRD: Follow-Up Finder — Gmail Adapter (`artos-followup-gmail`) — v0.3

> **Read the core PRD first** (`2026-09-14-follow-up-finder-core-prd.md`, now v1.1). This document covers only what's specific to Gmail. Unlike the Slack adapter, **this one has never been checked against a real Glean control** — no such output exists, and Art has confirmed (2026-09-14) to proceed without one rather than wait. Dry Run #1 (see Test & Iteration Plan) validated the logic directionally and surfaced two real corrections. Treat this adapter as best-effort and less-validated than Slack going forward, not just for the dry run — there is no plan to backfill a real control later.

---

## Why this adapter

Art has a separate Glean agent, "A1 - Follow Up Finder (GMAIL)," running the same kind of daily digest against Gmail instead of Slack. Comparing its portability instructions against the Slack agent's showed ~90% identical logic (status taxonomy, extraction principles, safety bar, output shape) and ~10% real difference (retrieval mechanics, table columns, a message-count disclosure requirement, mail-specific filtering). Per the consolidation decision, this ships as its own skill built on the same core logic — not merged into the Slack adapter, not a rebuild of shared rules from scratch.

---

## What This Adapter Does (Plain English)

On demand, scans Art's Gmail activity over a date range, pulls out things he committed to or was asked to do by email, checks whether each has since been resolved, and produces one Markdown digest per the core PRD's output shape — plus an upfront disclosure of how many raw messages were found versus how many distinct threads actually got evaluated.

---

## Non-Goals (Gmail-specific — see core PRD for the shared ones)

- **No cross-source item merging.** If a Gmail thread and a Slack thread concern the same real-world follow-up, they'll appear as two separate items in two separate digests (Slack adapter's and this one's). Deliberately deferred — see the consolidation decision memo.
- **Cross-source status *evidence* is allowed, cross-source *extraction* is not.** This adapter only ever creates follow-up items from Gmail messages. When checking whether an email-originated item has been resolved, it's fine (per the core PRD's evidence-gathering rule) to also check Slack, Drive, or Calendar for corroborating evidence — that's not the same as merging items across sources, and doesn't require solving the hard dedup problem the decision memo flagged.
- **No real Glean control, by decision, not by delay.** Art confirmed 2026-09-14 to proceed without one — this adapter's validation ceiling is a dry run diffed against his own judgment, not a real control diff like Slack's. Treat status calls here as best-effort.

---

## Packaging & Triggers

*Resolved 2026-09-14 — see core PRD's Triggers & Invocation for the shared rules (umbrella phrase, date-range default disclosure).*

- **Skill name:** `artos-followup-gmail`. Standalone skill, no workstation — same pattern as `artos-meeting-prep`. Art may want to distribute this beyond his own account.
- **Source-specific trigger phrases:** "check my email follow-ups," "Gmail follow-up finder," "what am I on the hook for in email," or similar — any clear ask for a Gmail-scoped follow-up digest.
- **Umbrella phrase:** also responds (alongside the Slack adapter) to a combined ask like "check my follow-ups" — renders its own separate digest, run back-to-back with Slack's, never merged.
- **Date range:** optional; defaults to yesterday → now per Runtime Configuration below. If the user doesn't specify one, the digest opens with a line noting the default was used (per core PRD).

---

## Tool Mapping

| Abstract capability (source plan) | Concrete tool here | Status |
|---|---|---|
| Gmail search | Gmail connector (`search_threads`) | Connected, usable now |
| Gmail thread retrieval (parent + replies) | Gmail connector (`get_thread`, `get_message`) | Connected |
| Status evidence: Gmail | Same Gmail tools, re-queried per follow-up | Connected |
| Status evidence: Slack / Drive / Calendar (optional, corroborating only) | Same connectors as the Slack adapter | Connected, available if useful — not required |
| Status adapter: task system (Jira/Linear) | Atlassian connector | Not authorized, same as the Slack adapter |
| Nudge drafting | Gmail connector (`create_draft`) | Added 2026-09-14 (core PRD's Actionability embellishment). Drafts only — never sends. |

If the Gmail connector itself is ever unavailable, stop and report that Gmail access is required — there's no fallback source for extraction (per the source plan; this is Gmail-only at the extraction stage, unlike status evidence which can pull from elsewhere).

---

## Runtime Configuration

- `user_email`: ahernandez@siftscience.com
- Identity resolution: the Gmail connector is scoped to the authenticated user — no separate handle needed.
- `local_timezone`: America/New_York
- `start_date` / `end_date`: defaults per core PRD (yesterday → now) unless stated
- `enabled_status_sources`: Gmail (required), Slack/Drive/Calendar (optional, corroborating)

**Test window:** not yet chosen — needs a real Glean Gmail control output first (see Open Items).

---

## Gmail-Specific Rules (on top of the core PRD)

**Collection-stage filtering (hard exclude).** Exclude messages with high confidence that they're advertisements, generic newsletters, automated marketing, or other non-actionable bulk mail. Do **not** exclude a message just because it's automated if it contains a specific actionable request or operational commitment (e.g., an automated Jira-to-email notification asking for a decision still counts).

**Ownership heuristic (classify Informational, don't hard-exclude).** Being a named recipient — even the sole "To:" — isn't the same as being the person who actually acts on a message. Found in Dry Run #1: "New Partner Application" notifications go to Art and Riley Chen both, contain "confirm or deny" language, but Art doesn't personally process them — someone else in the workflow does, or they're genuinely FYI. Infer this (no maintained list — heuristics only) from signals like: multiple named recipients on an identical templated notification, a workflow/queue-style sender address (e.g., "notifications@," "no-reply@," "support@...program"), boilerplate body language, and a recurring subject-line pattern. When these line up, classify as **Informational**, not Pending — it still appears in the summary table (that row is its "one-line mention"), it just doesn't get full Need Action treatment.

**Suspect-solicitation heuristic (classify Informational, don't hard-exclude).** Found in Dry Run #1: an email from a personal/free email domain (gmail, yahoo, etc.) signing as if representing a company, with no prior relationship and repeated unanswered follow-ups from the same sender, reads as cold-outreach/lead-gen rather than a genuine ask — even though it's personalized and specific. Lean Informational, not Pending. This is a soft signal, not a hard exclude like true newsletters — the item still gets a table row rather than vanishing, in case the read is ever wrong. Contrast case, extracted normally as Pending: the Summit Ridge MNDA thread — real corporate domain (LinkSquares-powered), procedurally specific, no reason for suspicion. Domain-and-relationship pattern matters more than tone or personalization.

**Count disclosure.** Before the digest table, report: total raw Gmail messages found in the date range, the number of unique threads actually evaluated for follow-ups, and a brief note on the gap between the two (filtering, thread grouping, dedup, excluded bulk mail, and items downgraded to Informational per the two heuristics above). Track these as two distinct counts — never conflate them.

---

## Gmail-Specific Steps

Steps 1-3 are Gmail-specific mechanics. Steps 4-6 apply the core PRD's rules directly. Step 7 uses Gmail-specific table columns.

**Step 1 — Resolve date range & identity.** Confirm start/end date; identity resolves via the Gmail connector.

**Step 2 — Gmail retrieval.** Search Gmail for messages where the user is sender, recipient, CC, or BCC (when available), within the date range. Retrieve complete parent + reply threads, not just matching messages. Preserve a stable Gmail URL or message/thread identifier for every item. Apply the Gmail-specific collection-stage filtering above during this step, not after.

**Step 3 — Normalize & dedupe.** Merge messages from the same thread describing the same work into one candidate item. Record `raw_message_count` and `evaluation_thread_count` here for the later disclosure.

**Step 4 — Extract follow-up items.** Apply the core PRD's extraction rules to the normalized threads.

**Step 5 — Gather status evidence.** Apply the core PRD's evidence-gathering rules. Primary source: Gmail. Optionally check Slack, Drive, or Calendar for corroborating evidence of resolution (see Non-Goals — this is evidence-gathering, not cross-source item merging).

**Step 6 — Classify status.** Apply the core PRD's status taxonomy.

**Step 7 — Render the digest.** Open with the count disclosure (above), then apply the core PRD's output shape (Need Action split into Waiting on You / Waiting on Others, aging notes, cross-source flag, confidence tags on the two heuristics below). Gmail-specific summary table columns, in this order: **Status, Email thread subject line, Key topic, Key Contact.** (Note the order differs from the Slack adapter's table — Status leads here, per the source agent's own design; not worth forcing alignment between the two adapters' column order.)

Voice: per core PRD (light assistant persona) — no Gmail-specific override, though the source plan notes any humorous closing must come only after the substantive digest, never before or mixed into it.

---

## Embellishments (added 2026-09-14, per core PRD v1.1)

- **Nudges:** text-only for now, per Art's explicit call (2026-09-14) — display suggested nudge text inline under each Waiting-on-Others item. `create_draft` stays mapped in the Tool Mapping above for later, but must not be called yet.
- **Cross-source overlap flag:** if the Slack adapter's most recent digest for an overlapping window is readable, check for plausible matches and add a one-line pointer. Concrete case already observed: Northbridge MNDA shows up in both, with email carrying more resolution detail.
- **Aging & run history:** persist state at `output/follow-up-finder-gmail-state.json` (or the account-appropriate output path once packaged), keyed by Gmail thread_id. Records first-seen, last-seen, appearance count, nudge count per item.
- **Confidence tagging:** applies directly to this adapter's two existing heuristics. Any Informational row produced by the Ownership heuristic or the Suspect-solicitation heuristic renders with an explicit tag, e.g. "(inferred — flag if wrong)," distinguishing it from an Informational call backed by direct evidence (if one ever occurs). This formalizes exactly what happened with Art's Dry Run #1 feedback — the goal is to make future corrections that easy every time, not just this once.
- **Scheduling:** explicitly not adopted yet, per core PRD.

---

## Key Decisions

| Decision | Choice | Confirmed |
|---|---|---|
| Ownership heuristic mechanism | Inferred from signals each run — no maintained sender/pattern list | 2026-09-14, after Dry Run #1 feedback |
| Suspect-solicitation visibility | Classify Informational (one-line table mention), not a hard exclude | 2026-09-14, after Dry Run #1 feedback |
| Rule scope | Both heuristics are Gmail-adapter-specific, not promoted to the core PRD (Slack hasn't shown the same pattern yet) | 2026-09-14 |

---

## Open Items

- [x] **Real Glean Gmail-agent control output** — resolved 2026-09-14: proceeding without one, per Art's explicit call. This adapter stays best-effort/less-validated than Slack indefinitely, not just until a control shows up. Dry Run #1 (diffed against Art's own review, not a control) is the validation ceiling here.
- [x] **Date range for testing** — resolved: continue using Sept 7-14 (matches the Slack adapter's window, and Dry Run #1 already used it) for any further runs, including Run #3.
- [ ] **Confirm the ad/newsletter filtering threshold** — "high confidence" is the source plan's own phrasing but isn't further defined. Worth watching in future runs for both false exclusions (a real ask that reads as bulk mail) and false inclusions (a newsletter that slips through).
- [x] **Naming, packaging, triggers** — resolved: see Packaging & Triggers above.

---

## Test & Iteration Plan

**Dry Run #1 (2026-09-14): done, no control.** See `resources/2026-09-14-gmail-dry-run-report.md`. Ran the logic against real Gmail activity, Sept 7-14 (same window as the Slack adapter's control runs) since a real Glean Gmail control still isn't available. Extraction cleanly separated 5 real follow-ups from a large volume of noise, correctly excluded two out-of-window threads Gmail's search leaked in, and surfaced one follow-up (Northbridge MNDA) that's the same real-world item already in the Slack digest — with more resolution detail than Slack carried, a live example of the core PRD's accepted "no cross-source merging" tradeoff. **This is directional only — not yet checked against a real Glean output**, so treat status calls and formatting as a first draft, not a validated result.

**Art's review of Dry Run #1 (2026-09-14):** two miscalls, both corrected — see the Ownership heuristic and Suspect-solicitation heuristic added above. "New Partner Application" was wrongly flagged Pending (Art doesn't personally action these, despite being a named recipient); the Apex Horizon Summit outreach was wrongly flagged Pending (personal-Gmail-domain cold outreach Art has never replied to, correctly read by him as likely junk). Summit Ridge's MNDA was confirmed as a correct, legitimate Pending call — kept in the PRD as the calibration example for what a real ask looks like. Saved to memory as standing guidance beyond just this skill.

**Run #3 (2026-09-14): done.** See `resources/2026-09-14-run3-embellishment-validation-report.md`. No new extraction — exercised the full embellishment set against the same window: Waiting on You/Others split, text-only nudges (2 items, no `create_draft` call made), the cross-source pointer to the Slack digest on Northbridge, aging notes sourced from a new state file (`output/2026-09-14-follow-up-finder-gmail-state.json`), and explicit `(inferred — flag if wrong)` tags on both heuristic-driven Informational rows. No PRD changes triggered.

**Sample output rendered:** `output/2026-09-14-follow-up-finder-gmail-digest-sample.md` — now reflects the Dry Run #1 corrections plus the full embellishment set from Run #3.

Method is otherwise the core PRD's shared Test Method, minus the real-control diff step — by decision, this adapter validates against Art's own review instead. Next: package as a SKILL.md that reads the shared core rules resource plus this adapter's Gmail-specific tool mapping, filtering rule, count disclosure, and table shape. All Open Items above are resolved.

---

## Conventions Applied

- Dated filename per naming standard.
- Shared logic lives in the core PRD, not restated here.
- Status field reflects that this adapter is proceeding without a real control, by decision — not a temporary gap to close later.

---

*Follow-Up Finder — Gmail Adapter PRD (`artos-followup-gmail`) v0.3 · 2026-09-14*

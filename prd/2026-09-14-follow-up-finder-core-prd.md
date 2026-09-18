---
document_type: core-prd
feature_name: follow-up-finder
component: follow-up-finder-core (shared logic — not a skill itself)
status: draft-for-iteration
version: "1.1"
created: 2026-09-14
updated: 2026-09-14
author: Art Hernandez
consumed_by:
  - artos-followup-slack (see 2026-09-14-follow-up-finder-skill-prd.md)
  - artos-followup-gmail (see 2026-09-14-follow-up-finder-gmail-prd.md)
runtime_artifact: resources/2026-09-14-follow-up-finder-core-rules.md
decision_trail: 2026-09-14-gmail-consolidation-decision.md
tags: [glean-port, follow-ups, core-logic, shared]
---

> _Note: names and company references in this document (e.g., "Northbridge Systems," "Meridian," "Summit Ridge," "Riley Chen," "Dana," "Jordan") have been substituted with fictional stand-ins for version control. The design decisions and logic below are unchanged and real._

# Core PRD: Follow-Up Finder — Shared Logic

> This document is not a skill and won't become a SKILL.md on its own. It's the single-sourced ~90% that both `follow-up-finder-slack` and `follow-up-finder-gmail` build on, so changing a rule here means changing it once, not twice. Each adapter PRD covers only its own 10%: source-specific tools, table columns, and its own test history.

---

## Why this exists

Art asked Glean to draft portability plans for two of its Follow-Up Finder agents — one for Slack, one for Gmail (see the decision memo for the comparison). They're close cousins: same status taxonomy, same extraction principles, same safety bar, same output shape. Building each as a fully independent skill would mean maintaining that shared 90% by hand in two places, with real risk of quiet drift (fix a rule in one, forget the other).

The decision (`2026-09-14-gmail-consolidation-decision.md`): keep them as separate, independently-testable skills — not one consolidated cross-source skill — but single-source the shared logic so it can't drift. This document is that single source.

---

## Architecture

Single Claude skill per source (no coded microservice — no adapters/*.py, no schema-validation layer, no model gateway, no orchestrator process; Claude reasons directly against connected tools each run), same pattern established by `artos-meeting-prep`. This core document plus its runtime resource (`resources/2026-09-14-follow-up-finder-core-rules.md`) gets read by both skills; each skill's own SKILL.md only adds its source-specific retrieval and rendering.

---

## Non-Goals (applies to every adapter)

- Not multi-user — Art's own activity only, in whichever source.
- Not scheduled by default — on-demand, until proven out. Reconfirmed 2026-09-14 when a scheduled morning-brief version was floated alongside the other embellishments below; Art's call was explicitly not yet.
- Read-only — no replies, no sends, no writes to any connected system.
- No cross-source deduplication in v1 — a Slack-side and Gmail-side follow-up about the same real-world thing will show up as two separate items in two separate digests for now. This is a known, accepted limitation (see decision memo), not an oversight.

---

## Extraction Principles

Select an item when at least one holds:
- The user authored a message/email and made an explicit or implied commitment, promise, request, TODO, or deadline.
- Someone else authored it and made a direct ask, question, assignment, or request addressed to the user.

Rules:
- Merge multiple messages/threads about the same piece of work into one follow-up item — never split one real task into several table rows.
- Preserve the exact source link and timestamp on every item. Never fabricate a link, timestamp, owner, or participant.
- An empty result for a given window is a valid, complete answer — not a failure to report as if something went wrong.
- **Exclude personal, non-work commitments by default** — except career/internal-development items that are organization-related but personal to the user (e.g., an internal role application). Those aren't dropped; they go in a separate "[Name] Only" section (see Output Shape), kept out of the standard work-tracking sections but still tracked. This was a real gap found in testing — the extraction rule as originally drafted would have surfaced a personal item into the main digest without this explicit carve-out.

---

## Evidence & Status Classification

For each extracted item, gather evidence of resolution from every connected, relevant source — not just the one it originated in. **No matching evidence is evidence of nothing; never infer completion from silence, from an empty search result, or from the mere passage of time.**

A real, verified example from testing: a follow-up's origin message can read as unresolved while the actual current state of a linked file (a deck, a doc) tells a different — sometimes contradictory — story. Status classification must check the linked resource's real current content/state, not just the tone of the last message that mentioned it. This caught a genuine wrong "Completed" call in a real Glean control output during Run #2 of the Slack adapter — treat this as a standing behavior, not an edge case.

Status taxonomy (exactly these four, plus one internal-only value):
1. **Pending**
2. **Partially Complete**
3. **Completed**
4. **Informational**
5. *(Internal only)* **Unknown** — never rendered as-is; always maps to Pending with an explicit "evidence insufficient" note. Never render Unknown as Completed.

For each item, capture and be ready to render: status, owner (or "TBD" — never guessed), actions taken, outcome, recommended next step, suggested timing, dependencies/blockers, and the evidence links behind the call. If a status source wasn't checked (not connected, not authorized), say so explicitly rather than treating it as evidence of anything.

---

## Output Shape

Single Markdown digest per run, per adapter. Every adapter's digest includes, at minimum:

- A summary table (adapter-specific columns — see each adapter's PRD), sorted Pending → Partially Complete → Completed → Informational.
- A **Need Action** section, split into two subsections (added 2026-09-14 — see Actionability below):
  - **Waiting on You** — items where the user is the next actor.
  - **Waiting on Others** — items where someone else owes the next move. Each gets a drafted, ready-to-send nudge (see Actionability).
  - Both present even if empty.
- A **Completed Follow-Ups** section: one to three concise bullets per item — who did what, when, and the outcome. Present even if empty.
- A **"[Name] Only" section** (e.g., "Art Only"): career/internal-development items per the extraction-principles carve-out above. Not present if nothing qualifies that run.
- Every material claim carries a real, working inline link to its source. No raw search dumps — synthesize only what supports each item.
- Explicit uncertainty wherever ownership, timing, or evidence is thin — never smoothed over into false confidence.
- Any status/category derived from inference rather than direct evidence carries a visible confidence tag (see Confidence Tagging below) — never presented with the same unqualified confidence as an evidence-backed call.
- Where available, an aging note per open item (see Aging & Run History below) and a cross-source pointer (see Cross-Source Overlap Flag below).

---

## Triggers & Invocation

*Added 2026-09-14, resolving naming, packaging, and invocation questions Art answered directly.*

**Packaging:** Both adapters ship as standalone skills, not tied to any workstation — same pattern as `artos-meeting-prep`. Reason (Art's own): he may want to distribute these beyond his own account, so they shouldn't be locked to an Art OS-specific folder structure.

**Names:** `artos-followup-slack` and `artos-followup-gmail`.

**Trigger phrases:**
- Each adapter responds to its own source-specific phrases (see that adapter's PRD) — e.g., "check my Slack follow-ups."
- An **umbrella phrase** (e.g., "check my follow-ups," "run follow-up finder") runs both adapters back-to-back in one turn and renders **two separate digests, one per source — never merged.** This is just convenient sequencing, not a new consolidation path; the Non-Goals above (no cross-source dedup) still hold.

**Date range:** Optional. The user may state one explicitly (e.g., "since Monday," "Sept 7 to Sept 14"). If none is given, default to yesterday → now per each adapter's Runtime Configuration — **and say so explicitly in the digest.** Every digest opens with a one-line note when the default was used (e.g., "No date range given, so this covers yesterday through now — say the word if you want a wider window"). Never apply the default silently; the user should always know what window they're looking at without having to ask.

---

## Actionability: Ready-to-Send Nudges

*Added 2026-09-14, in response to Art's own daily-review need — the biggest single time-save isn't more information, it's less composing. Scoped down 2026-09-14, same day: text-only for now, per Art's explicit call to build confidence before writing into live accounts.*

For every item in **Waiting on Others**, compose a short nudge and **display its text inline in the digest** — do not create an actual Slack or Gmail draft yet. The nudge text:
- References the original ask/commitment and how long it's been open.
- Is short, plain, and in the user's own voice register — not a form letter.
- Renders directly under the item in the digest, ready to copy, not linked to an external draft object.

**Do not call `create_draft`, `slack_send_message_draft`, or any other write-capable tool for this feature.** This is a deliberate, temporary scope-down — the mechanism to actually create drafts is already mapped in each adapter's Tool Mapping for when it's time to turn this on, but it stays inert until Art explicitly says to flip it on. Revisit once confidence has built up across enough runs (see each adapter's Open Items for the graduation criterion once set).

---

## Cross-Source Overlap Flag

*Added 2026-09-14, in response to a real observed case (Northbridge MNDA appeared in both the Slack and Gmail digests for the same week).*

This is **not** the cross-source item merging the core PRD's Non-Goals rules out — no shared retrieval, no single merged item, no new extraction logic. It's a lightweight, best-effort pointer added after an adapter's own digest is otherwise complete:

- If a companion adapter's most recent digest output (for an overlapping date window) is available to read, compare this adapter's extracted items against it on rough similarity — shared participant(s), close subject/topic match, same external party name.
- On a plausible match, add a one-line note to the item: "Also tracked in the [Slack/Gmail] digest — see there for [what the other source adds]."
- Never let this flag suppress, merge, or change the status of an item. It's informational only. A false-positive match (flagging two genuinely unrelated items as the same) is a lower-severity mistake than silently merging two items would be — err toward flagging when unsure.
- If no companion digest is available, skip this step silently — it's additive, never a blocker.

---

## Aging & Run History

*Added 2026-09-14, in response to Art's day-to-day review need — a stateless snapshot can't show what's actually stalling.*

Each adapter persists a small state record between runs (adapter-specific file location — see each adapter's PRD), keyed by a stable per-source identifier (thread or message ID, not a regenerated title/summary, since those can vary run to run). Per tracked item, record: first-seen date, last-seen date, number of times it's appeared across runs, and number of times a nudge was drafted for it.

On each run, before rendering: match newly-extracted items against the state record by identifier, update the record, and surface an aging note on any item that isn't new (e.g., "open since Sept 10, appeared in 3 runs, nudged twice"). New items get no aging note — a first appearance isn't "aging" yet. An item absent from the current run that was previously open should not be assumed resolved (see the core evidence rule) — just don't carry an aging note forward for something no longer extracted; if evidence later shows it resolved, it moves to Completed Follow-Ups normally.

State is per-adapter, not shared — an item's Slack-side age and Gmail-side age (if it appears in both, per the Cross-Source Overlap Flag) are tracked independently.

State tracks open (Need Action) items only. Once an item resolves to Completed, drop it from the state record rather than carrying it forward — its history from here on lives in that run's Completed Follow-Ups section, not in the aging mechanism. Informational items aren't aged either; aging exists to show what's stalling, and neither a resolved nor an informational item is stalling. Confirmed by Run #3 (2026-09-14), the first run to actually exercise this.

---

## Confidence Tagging

*Added 2026-09-14, generalizing the pattern from the Gmail adapter's ownership and suspect-solicitation heuristics.*

Any classification reached by inference/heuristic — rather than direct stated evidence — must carry a visible, brief tag distinguishing it from an evidence-backed call (e.g., "(inferred — flag if wrong)"). This applies wherever an adapter's own PRD defines a heuristic-based rule; the core PRD doesn't mandate which heuristics exist, only that inferred calls are never presented with the same unqualified confidence as a directly-evidenced one. When the user corrects an inferred call, log it the same way as any other feedback — capture it in project memory as standing guidance, not just a one-off fix to that day's digest.

### Voice and persona

Light assistant persona, full restore of the Glean-style voice, same override as `artos-meeting-prep`: brief banter to open, a short pop-culture reference to close, both kept subordinate to the actual content. This is the one place the user's "match my voice" writing rule is intentionally suspended — the digest reads as an assistant reporting to the user, not the user speaking to someone else. Applies to both adapters; only the open/close carry personality; the body (Need Action detail) stays plain and factual.

---

## Safety & Quality Bar

- Never infer completion from an absence of evidence.
- Never expose a message the user doesn't have permission to see in the source system (should be a non-issue since retrieval runs as the authenticated user, but stated explicitly).
- Never fabricate a link, owner, deadline, participant, or outcome.
- Label inferred ownership/timing as inferred, never stated as fact.
- No raw search dumps in the final digest.
- High-severity failures to actively test against: marking something completed with no evidence, dropping a source link, missing an obvious direct ask, inventing a deadline/owner/outcome, conflating a status source's absence with proof of non-completion.

---

## Test Method (shared across adapters)

Each adapter is validated independently, the same way, before it's packaged as a SKILL.md:

1. Get a real Glean output from the source agent being ported, for a specific, known date window.
2. Run the adapter's logic (by hand, in a session, not yet as a packaged skill) against that same window.
3. Diff item-by-item: same items found? same statuses? same owners/evidence? Spot-check at least one specific quote or citation directly against the live source rather than trusting it at face value.
4. Where the port's output and the Glean control disagree, don't assume the control is right — check the actual evidence. (Run #2 of the Slack adapter found a case where the real Glean control's own "Completed" call was wrong.)
5. Log findings as a dated test report, feed anything systemic back into this core PRD (if shared) or the adapter's own PRD (if source-specific).
6. Only package as a SKILL.md once a couple of runs hold up.

No cross-adapter validation exists or is planned for v1 (see Non-Goals) — each adapter's control comes from that same source's Glean agent.

---

## Change Log

- 2026-09-14 — v1.0. Extracted from the Slack adapter PRD after the decision to keep Slack and Gmail as separate skills built on shared logic. Original content and both Slack test-run findings live in `2026-09-14-follow-up-finder-skill-prd.md`.
- 2026-09-14 — v1.1. Added four embellishments requested after reviewing both adapters' sample output: ready-to-send nudges (with the Need Action → Waiting on You / Waiting on Others split), the cross-source overlap flag, aging & run history, and confidence tagging for inferred classifications. Scheduling was floated and explicitly declined for now.
- 2026-09-14 — v1.1 (cont.). Resolved naming (`artos-followup-slack` / `artos-followup-gmail`), packaging (standalone, no workstation — Art may want to distribute), and invocation (source-specific + umbrella trigger phrases, optional date range, mandatory disclosure when the default window is used). See new Triggers & Invocation section.

---

*Follow-Up Finder core PRD · v1.1 · 2026-09-14*

---
name: follow-up-finder-core-rules
description: Shared logic loaded by every Follow-Up Finder adapter skill (Slack, Gmail, and any future source). Source-specific tool mapping and table columns live in each adapter's own SKILL.md — this file is everything else.
version: "1.1"
last_updated: 2026-09-14
derived_from: ../prd/2026-09-14-follow-up-finder-core-prd.md
---

## Extraction rules

Select an item when at least one holds:
- The user authored a message and made an explicit or implied commitment, promise, request, TODO, or deadline.
- Someone else authored it and made a direct ask, question, assignment, or request addressed to the user.

- Merge everything about the same piece of work into one item. Never split one real task into multiple rows.
- Preserve the exact source link and timestamp on every item. Never fabricate a link, timestamp, owner, or participant.
- An empty result for the window is a valid, complete answer.
- Exclude personal, non-work commitments — **except** career/internal-development items that are organization-related but personal (e.g., an internal role application). Those go in a separate "[Name] Only" section, not the standard sections, and are not dropped.

## Evidence gathering

- Check every connected, relevant source for resolution evidence — not just the item's source of origin.
- No matching evidence is evidence of nothing. Never infer completion from silence, an empty search, or elapsed time.
- When a follow-up references a linked resource (a doc, deck, ticket), check that resource's actual current state — not just the tone of the last message that mentioned it. A message can sound resolved while the linked file itself shows otherwise.
- If a status source wasn't checked (not connected/authorized), say so explicitly. Don't treat "not checked" as evidence of non-completion.

## Status taxonomy

Exactly four rendered values, sorted in this order everywhere:
1. Pending
2. Partially Complete
3. Completed
4. Informational

Internal-only fifth value, **Unknown** — never rendered as-is. Always maps to Pending with an explicit "evidence insufficient" note. Never render Unknown as Completed.

For every item, be ready to state: status, owner (or "TBD" — never guessed), actions taken, outcome, recommended next step, suggested timing, dependencies/blockers, evidence links.

## Output shape

Every digest includes, in this order:
1. Summary table (adapter defines its own columns), sorted by the status order above.
2. **Need Action**, split into:
   - **Waiting on You** — user is the next actor.
   - **Waiting on Others** — someone else owes the next move; each gets a drafted, ready-to-send nudge (see Nudges below). Both present even if empty.
3. **Completed Follow-Ups** — 1-3 bullets per item: who did what, when, outcome. Present section even if empty.
4. **"[Name] Only"** — career/internal items per the extraction carve-out above. Omit the section entirely if nothing qualifies.
5. **Informational** (adapter-defined) — items classified via inference rather than direct evidence render here with a confidence tag, not silently dropped.

Every material claim carries a real, working inline link. No raw search dumps. Any inferred (non-evidence-backed) classification carries a visible tag, e.g. "(inferred — flag if wrong)."

## Nudges (Waiting on Others)

Compose a short nudge per Waiting-on-Others item and **display the text inline in the digest**. Reference the original ask and how long it's been open; keep it short and in the user's own register, not a form letter. **Text-only for now — do not call any draft-creation or send tool** (`create_draft`, `slack_send_message_draft`, or equivalent). This is temporary, by explicit instruction, until confidence is built up; do not re-enable actual draft creation without being told to.

## Cross-source overlap flag

If a companion adapter's recent output for an overlapping window is readable, check for plausible matches (shared participant, close subject/topic, same external party) and add a one-line pointer ("Also tracked in the [X] digest"). Never merge, suppress, or change status based on this — informational only, skip silently if no companion output exists. When unsure whether two items match, flag it anyway — a false-positive pointer is a smaller mistake than a silent merge would be.

## Aging & run history

Persist a small per-adapter state record between runs, keyed by a stable source ID (thread/message ID — never a regenerated title). Track first-seen date, last-seen date, times appeared, times nudged. Surface an aging note on non-new items ("open since [date], appeared in N runs, nudged Nx"). New items get no aging note. Don't assume an item is resolved just because it didn't get extracted this run — only evidence moves it to Completed.

## Voice

Light assistant persona: brief banter to open, a short pop-culture reference to close, both subordinate to the content. This is the one place the user's "match my voice" rule is suspended — the digest is the assistant reporting to the user. Keep the body (Need Action detail) plain and factual; only the open/close carry personality.

## Safety bar (non-negotiable)

- Never infer completion from absence of evidence.
- Never expose something the user doesn't have permission to see in the source system.
- Never fabricate a link, owner, deadline, participant, or outcome.
- Label inferred ownership/timing as inferred.
- No raw search dumps.
- Treat as high-severity and actively guard against: false "Completed" with no evidence, a dropped source link, a missed direct ask, an invented deadline/owner/outcome, treating "source unavailable" as proof of non-completion.

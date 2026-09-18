---
document_type: decision-memo
feature_name: follow-up-finder
question: consolidate Slack + Gmail follow-up finders into one skill, or keep them separate
created: 2026-09-14
author: Art Hernandez
inputs:
  - Glean "A1 - Follow Up Finder (Slack)" port — this PRD, 2 test runs complete
  - Glean "A1 - Follow Up Finder (GMAIL)" — https://docs.google.com/document/d/1wy4r2I_Of2FW6g1bg1U6NpGmWj6MNfcTJAFrN6DAltg
status: awaiting decision
---

> _Note: names and company references in this document (e.g., "Northbridge Systems," "Meridian," "Summit Ridge," "Riley Chen," "Dana," "Jordan") have been substituted with fictional stand-ins for version control. The design decisions and logic below are unchanged and real._

# Decision Memo: Consolidate or Separate — Slack + Gmail Follow-Up Finders

## The two source agents, compared

They're close cousins, not the same agent. Same bones, different skin:

| | Slack agent | Gmail agent |
|---|---|---|
| Status taxonomy | Pending / Partially Complete / Completed / Informational | Same four, plus an explicit "Unknown → renders as Pending" rule |
| Extraction rule | User's commitments + direct asks to user | Identical logic, applied to email instead of messages |
| Output sections | Summary table, Need Action, Completed Follow-Ups | Same two sections, plus a required raw-message-count vs. evaluated-thread-count disclosure up front |
| Table columns | Conversation, Topic, Status | Status, Subject, Topic, **Key Contact** — different shape, Status leads instead of trailing |
| Source-specific filtering | (none specified) | Explicit ad/newsletter/bulk-mail exclusion at collection |
| Safety bar | Near-identical: no inferring completion from silence, no fabricated links/owners/dates | Same, word for word in spirit |

**Bottom line: ~90% shared DNA (taxonomy, safety rules, section shape), ~10% real divergence (table columns, count disclosure, mail-specific filtering).**

---

## Option A: Consolidate into one skill

**Pros**

- One digest instead of two — Art runs one thing, reads one thing, instead of mentally merging two documents.
- Fixes a real gap we already saw: the Northbridge MNDA follow-up lives partly in Slack ("any response from northbridge?") and partly in email (Jordan: "I've chased Dana again on email"). Split across two skills, that's two disconnected, half-informed items. Merged, it's one item with real cross-source evidence — closer to what Glean's original multi-source agents were actually built for.
- One evidence-gathering pass naturally checks both sources before calling anything "no evidence found," which is strictly safer against the "silence ≠ completion" rule than two skills that each only see half the picture.
- Avoids long-term drift — the shared 90% (taxonomy, safety bar, rendering) only has to be written and maintained once instead of kept in sync by hand across two files.

**Cons**

- The 10% that's genuinely different (table columns, Gmail's count-disclosure requirement, mail-specific ad filtering) has to be reconciled or conditionally branched inside one skill — real complexity for a moderate payoff.
- No single Glean control to validate against. Our whole method so far has been "diff against a real Glean output for the same window" — that exists separately for Slack and for Gmail, but there's no real "combined" Glean output to check a merged skill's behavior against. We'd be building the one part of this (cross-source dedup) with no ground truth at all.
- Cross-source deduplication (deciding a Slack thread and an email thread are "the same" follow-up) is a new problem neither original Glean agent had to solve. Doing it carelessly risks false merges (two unrelated things collapsed into one) or missed merges (same thing shown twice) — arguably worse than the problem it's fixing.
- One skill, one blast radius — a bug in Gmail retrieval (bad date filter, auth hiccup) can now degrade the Slack side of the output too, instead of failing in isolation.
- Slower every run — always pays for two full retrieval passes, even on a day Art only cares about one source.
- Needs new graceful-degradation logic for "Gmail's authorized but Slack isn't" (or vice versa) — neither source agent had to handle partial availability, because each assumed its one source always existed.

---

## Option B: Keep them separate (two skills)

**Pros**

- Matches the proven build method exactly — each skill stays a faithful, independently testable port, each with its own real Glean control to diff against (already done for Slack, same method available for Gmail).
- Smaller, more debuggable, same shape as `artos-meeting-prep` and the Slack finder — one retrieval, one render, easy to reason about and fix in isolation.
- A connector outage or auth gap in one source doesn't touch the other.
- Faster when Art only wants one source's answer on a given day.
- Defers the hard cross-source dedup problem instead of solving it under time pressure — it can still be tackled later, deliberately, once both skills are individually trusted.

**Cons**

- Two things to run, two digests to read — real risk of "forgot to run the other one" or only checking one and missing something.
- Same-topic items spanning both sources (Northbridge is the concrete example already in hand) show up as two disconnected half-items instead of one fuller one.
- The shared 90% logic lives in two separate SKILL.md files, and has to be kept in sync by hand as it evolves — improve the Slack version's Drive-checking behavior, for instance, and you have to remember to port the same fix to the Gmail version.

---

## Recommendation

**Keep them separate for now, built from the same underlying rules so they don't drift, with cross-source merging deliberately deferred rather than skipped.**

Reasoning: the biggest risk in Option A isn't the table-format differences (that's just work) — it's that consolidating means inventing and shipping a new, unvalidated cross-source dedup logic with no real Glean baseline to check it against, on top of the CoM/deal-deck team's very concrete recent lesson (see the `com-gong-transcript-ingress-snowflake` PRD in Sales Deal Intel: coverage claims that look right on paper turn out to be account-dependent until actually benchmarked). The Northbridge example is real evidence that overlap exists, but it's one example, not proof the overlap is common enough to justify solving the hard problem before either skill is even out of testing.

Practical middle path: build the Gmail finder as its own skill next, run the same two-test-run validation method against it, and write both from one shared "rules" reference (taxonomy, safety bar, section shape) so they can't quietly diverge. If overlap turns out to be frequent once both are running for real, a thin third step — one that reads both finished digests and flags likely duplicates, rather than merging retrieval itself — is a much safer version of "one skill" to build later, because by then there's real usage data instead of one anecdote to design it against.

---

*Follow-Up Finder consolidation decision · 2026-09-14 · awaiting Art's call*

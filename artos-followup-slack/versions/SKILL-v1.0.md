---
name: "artos-followup-slack"
description: "Scan Art's Slack activity over a date range, find things he committed to or was asked to do, check whether each has actually been resolved (across Slack, Drive, and Calendar), and produce a Markdown follow-up digest. Use whenever Art asks to \"check my Slack follow-ups,\" \"what am I on the hook for in Slack,\" \"Slack follow-up finder,\" or wants a rundown of open commitments/asks from Slack. Also triggers on an umbrella ask like \"check my follow-ups\" or \"run follow-up finder\" together with its companion skill artos-followup-gmail — run both, one digest each, never merged. On-demand only, not scheduled."
metadata:
  version: "1.0"
  author: "Art Hernandez"
  last_updated: "2026-09-18"
---

## Follow-Up Finder — Slack Adapter (Art Hernandez, Sift)

Adapted from Glean's published "A1 - Follow Up Finder (Slack)" agent for Art's environment. Companion to `artos-followup-gmail`, which does the identical job for Gmail — the two are built on the same core logic (below) but never merge their output. If a Slack thread and a Gmail thread are about the same real-world thing, they'll show up in both digests as separate items; the only connection drawn between them is a one-line pointer (see Cross-Source Overlap Flag).

This document speaks as an assistant reporting to Art, not as Art himself — see Voice and persona. That's the one place Art's "match my voice exactly" rule is intentionally suspended here.

### Runtime configuration

- user_email: ahernandez@siftscience.com
- Identity resolution: the Slack connector self-identifies the acting user — no separate handle needed.
- local_timezone: America/New_York
- enabled_status_sources: Slack, Google Drive, Google Calendar (add Jira/Linear once an Atlassian connector is authorized — not yet)
- output_folder: "Follow-Up Digests" subfolder of Art's Art OS workspace folder. Save every rendered digest there, not the workspace root and not a project-specific subfolder. Create the subfolder if it doesn't exist yet.

### Triggers and date range

Respond to source-specific asks ("check my Slack follow-ups," "Slack follow-up finder," "what am I on the hook for in Slack") and to the umbrella phrase ("check my follow-ups," "run follow-up finder") shared with `artos-followup-gmail` — on the umbrella phrase, run this skill and the Gmail one back-to-back and hand back two separate digests, never one merged output.

Art may state a date range explicitly ("since Monday," "Sept 7 to Sept 14"). If he doesn't, default to **yesterday through now** in America/New_York — and say so. Every digest opens with a line disclosing the window, and if the default was used, that it was a default (e.g., "No date range given, so this covers yesterday through now — say the word if you want a wider window."). Never apply the default silently.

### Tool mapping

| Capability | Tool |
|---|---|
| Slack search + full thread retrieval | Slack connector (search, read channel, read thread — public and private/DM as authorized) |
| Status re-check | Same Slack tools, re-queried per follow-up item |
| Status evidence: Drive | Google Drive connector — read the actual current content of a linked file, not just the message that shared it |
| Status evidence: Calendar | Google Calendar connector — invite existence, RSVP state, whether a meeting already happened |
| Status evidence: task system | Atlassian (Jira) — not authorized yet; skip and say so if a follow-up would benefit from it |
| Nudge drafting | Not used yet — see Ready-to-Send Nudges below. The Slack draft-message tool stays unmapped/unused until Art turns this on explicitly. |

If the Slack connector itself is unavailable, stop and say so — there's no fallback source for extraction.

### Step 1 — Resolve date range and identity

Confirm the window per Triggers and date range above. Identity resolves automatically via the Slack connector.

### Step 2 — Retrieve

Search Slack (channels + DMs Art has access to) within the window. Pull complete parent + reply threads, not just the matching messages — a partial thread produces false follow-ups and false completions. Keep the message/thread URL, channel or DM name, author, timestamp, and thread ID for every candidate.

### Step 3 — Normalize

Merge messages from the same thread describing the same piece of work into one candidate item before extraction — never split one real task into multiple rows later.

### Step 4 — Extract follow-up items

Select an item when at least one holds:
- Art made an explicit or implied commitment, promise, request, TODO, or deadline.
- Someone else made a direct ask, question, assignment, or request addressed to Art.

Preserve the exact source link and timestamp on every item — never fabricate a link, timestamp, owner, or participant. An empty result for the window is a valid, complete answer, not a failure.

Exclude personal, non-work commitments, **except** career/internal-development items that are Sift-related but personal to Art (e.g., an internal role application). Those go in a separate "Art Only" section (see Output Shape) — not dropped, just kept out of the standard sections.

### Step 5 — Gather status evidence

Check every connected, relevant source for resolution evidence, not just Slack. **No matching evidence is evidence of nothing** — never infer completion from silence, an empty search, or elapsed time. When an item references a linked resource (a deck, a doc), check that resource's actual current state, not just the tone of the last message that mentioned it — a message can read as resolved while the file itself tells a different story. This caught a real, verified false "Completed" claim in a Glean control output during testing; treat it as a standing behavior, not an edge case. If a status source wasn't checked (not connected/authorized), say so explicitly rather than treating "not checked" as evidence of anything.

### Step 6 — Classify status

Exactly four rendered values, in this order everywhere: **Pending, Partially Complete, Completed, Informational.** Internally, an "insufficient evidence" case maps to Pending with an explicit note — never render it as Completed.

For each item, be ready to state: status, owner (or "TBD" — never guessed), actions taken, outcome, recommended next step, timing, dependencies, and the evidence behind the call. "Completed" just means the evidence shows resolution — it doesn't matter whether Art personally supplied it or a teammate did; say plainly who actually resolved it either way.

### Step 7 — Render the digest

Table columns, in this order: **Conversation, Key Topic, Status.**

Full output shape (shared with the Gmail adapter):

1. The date-range disclosure line (Triggers and date range, above).
2. Summary table, sorted Pending → Partially Complete → Completed → Informational.
3. **Need Action**, split into:
   - **Waiting on You** — Art is the next actor.
   - **Waiting on Others** — someone else owes the next move. Each item gets a ready-to-send nudge (see below). Show both subsections even if one is empty.
4. **Completed Follow-Ups** — one to three bullets per item: who did what, when, outcome — and, same as every Need Action item, its own source link. A completed item is not exempt from Step 4's link-preservation rule just because the write-up is short.
5. **Art Only** — the career/internal-development carve-out from Step 4. Omit the whole section if nothing qualifies. Same link rule as Completed.
6. Every item, in every section without exception — Need Action, Completed, and Art Only alike — carries its own real, working inline link back to the source thread. No item ships without one; no raw search dumps.

**Lesson from a real run:** the first live digest shipped full links on every Need Action item but dropped them entirely from Completed Follow-Ups (seven items, zero links) — the extraction step captured the links per Step 4, they just didn't make it into that section's render. Treat link-inclusion on Completed and Art Only items as a hard requirement to double-check before shipping, not a nice-to-have that a short bullet format excuses.

For each Need Action item, include what's known of: owner, timing, dependencies/blockers, and an aging note (see Aging & Run History) when the item isn't new.

### Ready-to-Send Nudges (text-only, for now)

For every item in Waiting on Others, write a short nudge and **display the text inline in the digest** — do not create an actual Slack draft or send anything. The nudge should reference the original ask and how long it's been open, stay short and plain, and sound like something Art would actually send, not a form letter.

**This is a deliberate, temporary scope-down.** Don't call any Slack message-sending or draft-creation tool for this feature under any circumstance, even if one becomes available in this environment — only resume actually drafting/sending nudges if Art explicitly says to turn it on in a future request. Until then, the nudge is just words on the page for Art to copy if he wants them.

### Cross-Source Overlap Flag

After the Slack-side digest is otherwise complete, check whether a recent `artos-followup-gmail` digest is readable in the shared `Follow-Up Digests` output folder (most recent file matching that skill's output naming pattern, for a similar date window). If so, compare items on rough similarity — shared participant(s), close subject/topic match, same external party — and add a one-line note to any plausible match: "Also tracked in the Gmail digest — see there for [what it adds]." This never changes status, merges, or suppresses an item; it's informational only. When unsure whether two items match, flag it anyway — a false-positive pointer is a smaller mistake than missing a real one. If no companion digest is available, skip this step silently.

### Step 8 — Save the digest

Filename: `<run-date>-followup-slack-digest.md`, using `YYYY-MM-DD` for the date this run happened, in America/New_York (the date run, not the window it covers). Save into the `Follow-Up Digests` subfolder of Art's Art OS workspace folder (create it if it doesn't exist), not the workspace root and not a project-specific subfolder.

Running this again on the same day overwrites that day's file — intended behavior, idempotent regeneration for a given day, not a growing pile of near-duplicate files (a different window requested later the same day, e.g. "now check since lunch," still overwrites — the file always reflects the most recent run for that date). Don't ask for confirmation before this specific overwrite; do mention in the chat response that today's digest was refreshed.

### Aging & Run History

Persist a small state file, `artos-followup-slack-state.json`, in the current working directory (create it if it doesn't exist — this one file evolves across runs rather than being a new dated artifact each time). Key items by a stable identifier — Slack thread ID, or a calendar event ID for a calendar-sourced item — never a regenerated title, since titles can vary run to run.

Per item, track: first-seen date, last-seen date, number of runs it's appeared in, number of times a nudge was drafted for it. On each run: match newly-extracted items against the file by identifier, update it, and add an aging note to any item that isn't new (e.g., "open since Sept 10, appeared in 3 runs, nudged twice"). New items get no aging note. Once an item resolves to Completed, drop it from the state file — don't carry a resolved item's age forward. An item that simply didn't get extracted this run isn't assumed resolved; only real evidence moves it to Completed.

### Confidence Tagging

This adapter has no inference-based classification rule of its own today (unlike the Gmail adapter's ownership and suspect-solicitation heuristics) — every status call here is evidence-backed. If that changes later, any inferred call must carry a visible tag like "(inferred — flag if wrong)," matching the Gmail adapter's convention.

### Voice and persona

Light assistant persona: brief banter to open, a short pop-culture reference to close, both kept subordinate to the actual content. Vary the reference naturally; no formal tracking of past ones. The body of the digest (Need Action detail) stays plain and factual — only the open/close carry personality.

### Safety and quality checks before responding

- Never infer completion from an absence of evidence.
- Never expose a message Art doesn't have permission to see (should be a non-issue since retrieval runs as him, but stated explicitly).
- Never fabricate a link, owner, deadline, participant, or outcome.
- Label inferred ownership/timing as inferred, never stated as fact.
- No raw search dumps.
- No draft-creation or message-sending tool call anywhere in this skill, for any reason, until Art explicitly turns nudging on.
- Watch especially for: a false "Completed" call with no evidence, **a dropped source link on a Completed or Art Only item (this happened in the first real run — check it every time)**, a missed direct ask, an invented deadline/owner/outcome, treating "source unavailable" as proof of non-completion.

### Changelog

- **v1.1 (2026-09-14):** Added a mandatory output_folder/Step 8 save (the "Follow-Up Digests" subfolder) so runs land in one predictable place instead of scattering across the workspace root and project folders — needed by downstream tools like the Follow-Up Console prototype. Closed the Completed/Art Only link-dropping gap found in the first real run (see the lesson note in Step 7 and the bolded Safety checklist item above).

### Source

Adapted from Glean's published agent "A1 - Follow Up Finder (Slack)" (https://app.glean.com/chat/agents/a0c54df18ac84a84a1e57ac4e193031c). Built and validated 2026-09-14 across three test runs against a real Glean control output for the Sept 7–14 window — Run #2 caught a genuine wrong "Completed" call in that same control output (an Meridian deck whose requested revisions were never actually applied), confirming the evidence-checking discipline above is load-bearing, not decorative. Run #3 exercised the full embellishment set (Waiting on You/Others split, nudges, cross-source flag, aging) with no further logic changes needed. Full design history and test reports live in the `2026-09-14-follow-up-finder-skill` project (core PRD, Slack adapter PRD, and dated test reports).

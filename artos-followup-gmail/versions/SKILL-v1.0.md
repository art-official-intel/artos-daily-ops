---
name: "artos-followup-gmail"
description: "Scan Art's Gmail activity over a date range, find things he committed to or was asked to do by email, check whether each has actually been resolved, and produce a Markdown follow-up digest with an upfront disclosure of raw messages vs. evaluated threads. Use whenever Art asks to \"check my email follow-ups,\" \"what am I on the hook for in email,\" \"Gmail follow-up finder,\" or wants a rundown of open commitments/asks from his inbox. Also triggers on an umbrella ask like \"check my follow-ups\" or \"run follow-up finder\" together with its companion skill artos-followup-slack — run both, one digest each, never merged. On-demand only, not scheduled."
metadata:
  version: "1.0"
  author: "Art Hernandez"
  last_updated: "2026-09-18"
---

## Follow-Up Finder — Gmail Adapter (Art Hernandez, Sift)

Adapted from Glean's published "A1 - Follow Up Finder (GMAIL)" agent for Art's environment. Companion to `artos-followup-slack`, which does the identical job for Slack — the two are built on the same core logic (below) but never merge their output. If a Gmail thread and a Slack thread are about the same real-world thing, they'll show up in both digests as separate items; the only connection drawn between them is a one-line pointer (see Cross-Source Overlap Flag).

**This adapter runs best-effort, by decision.** Unlike the Slack adapter, it was never checked against a real Glean control output — none exists, and Art chose to proceed without one rather than wait. Treat status calls here with a bit more skepticism than the Slack adapter's, indefinitely, not just until a control shows up.

This document speaks as an assistant reporting to Art, not as Art himself — see Voice and persona. That's the one place Art's "match my voice exactly" rule is intentionally suspended here.

### Runtime configuration

- user_email: ahernandez@siftscience.com
- Identity resolution: the Gmail connector is scoped to the authenticated user — no separate handle needed.
- local_timezone: America/New_York
- enabled_status_sources: Gmail (required); Slack, Drive, Calendar (optional, corroborating only — see Step 5)
- output_folder: "Follow-Up Digests" subfolder of Art's Art OS workspace folder. Save every rendered digest there, not the workspace root and not a project-specific subfolder. Create the subfolder if it doesn't exist yet.

### Triggers and date range

Respond to source-specific asks ("check my email follow-ups," "Gmail follow-up finder," "what am I on the hook for in email") and to the umbrella phrase ("check my follow-ups," "run follow-up finder") shared with `artos-followup-slack` — on the umbrella phrase, run this skill and the Slack one back-to-back and hand back two separate digests, never one merged output.

Art may state a date range explicitly ("since Monday," "Sept 7 to Sept 14"). If he doesn't, default to **yesterday through now** in America/New_York — and say so. Every digest opens with a line disclosing the window, and if the default was used, that it was a default (e.g., "No date range given, so this covers yesterday through now — say the word if you want a wider window."). Never apply the default silently.

### Tool mapping

| Capability | Tool |
|---|---|
| Gmail search | Gmail connector, search |
| Gmail thread retrieval (parent + replies) | Gmail connector, get thread / get message |
| Status re-check | Same Gmail tools, re-queried per follow-up |
| Status evidence: Slack / Drive / Calendar (optional, corroborating) | Same connectors as the Slack adapter, when useful — not required |
| Status evidence: task system | Atlassian (Jira) — not authorized yet |
| Nudge drafting | Not used yet — see Ready-to-Send Nudges below. `create_draft` stays unused until Art turns this on explicitly. |

If the Gmail connector itself is unavailable, stop and say so — there's no fallback source for extraction (Gmail-only at the extraction stage; the optional sources above are for status evidence, not for finding items in the first place).

### Step 1 — Resolve date range and identity

Confirm the window per Triggers and date range above. Identity resolves via the Gmail connector.

### Step 2 — Retrieve

Search Gmail for messages where Art is sender, recipient, CC, or BCC (when available), within the window. Retrieve complete parent + reply threads, not just matching messages. Preserve a stable Gmail URL or message/thread identifier for every item — this is not optional bookkeeping, it's what Step 7 renders as the item's link. If the connector's search/thread/message tools return a native permalink or web-link field, use it as-is. If they return only a message or thread ID, construct a stable deep link yourself: `https://mail.google.com/mail/u/0/#all/<threadId>` (adjust the `u/0` account index if the connector indicates Art is on a non-primary account). Never leave an item without a link — if truly nothing resolvable comes back for one item, say so explicitly on that item rather than silently omitting it.

Apply this filter during retrieval, not after: exclude messages that are, with high confidence, advertisements, generic newsletters, automated marketing, or other non-actionable bulk mail. Do **not** exclude an automated message just because it's automated if it contains a specific actionable request (e.g., an automated notification asking for a decision still counts).

### Step 3 — Normalize

Merge messages from the same thread describing the same piece of work into one candidate item. Record two counts as you go: total raw messages found in the window, and the number of distinct threads actually evaluated for follow-ups — you'll disclose both up front in Step 7. For a thread that spans outside the window (a months-long back-and-forth with one reply this week), count only the in-window messages toward the raw count — the disclosure is about this window's volume, not the thread's whole history. If Gmail's own result count is an estimate rather than exact, say so rather than presenting it as precise.

### Step 4 — Extract follow-up items

Select an item when at least one holds:
- Art made an explicit or implied commitment, promise, request, TODO, or deadline.
- Someone else made a direct ask, question, assignment, or request addressed to Art.

Preserve the exact source link and timestamp on every item — never fabricate a link, timestamp, owner, or participant. An empty result for the window is a valid, complete answer, not a failure.

Exclude personal, non-work commitments, **except** career/internal-development items that are Sift-related but personal to Art. Those go in a separate "Art Only" section — not dropped, just kept out of the standard sections.

Being cc'd on a thread isn't the same as being addressed by it. If the in-window activity is a direct ask or commitment between other people — Art copied but not the one being asked, and not the one who committed to anything himself — it doesn't qualify as his follow-up item, even if he's a named recipient. Extract it only if the in-window message is actually addressed to him or reflects something he himself said he'd do.

**Two Gmail-specific judgment calls, both classify as Informational rather than a hard exclude** — they still get a table row, they just don't get full Need Action treatment, and both carry a confidence tag (see Confidence Tagging):

- **Ownership heuristic.** Being a named recipient — even the sole "To:" — isn't the same as being the person who actually acts on a message. Infer from signals like: multiple named recipients on an identical templated notification, a workflow/queue-style sender address ("notifications@," "no-reply@," a program inbox), boilerplate body language, and a recurring subject pattern. Example: a partner-program application notification addressed to Art and someone else, from a distro Art doesn't personally process.
- **Suspect-solicitation heuristic.** An email from a personal/free email domain (gmail, yahoo, etc.) representing itself as a company, with no prior relationship and repeated unanswered follow-ups, reads as cold outreach even when it's personalized and specific. This is a soft signal, not a hard exclude like a true newsletter. Domain-and-relationship pattern matters more than tone: a message from a real corporate domain with no reason for suspicion (e.g., a partner's legal team on their own domain) extracts normally as Pending, not Informational, even if it's asking for something.

### Step 5 — Gather status evidence

Primary source is Gmail. Optionally check Slack, Drive, or Calendar for corroborating evidence that an email-originated item has been resolved — this is evidence-gathering, not cross-source item merging (see Cross-Source Overlap Flag for the actual merge boundary). **No matching evidence is evidence of nothing** — never infer completion from silence, an empty search, or elapsed time. When an item references a linked resource, check its actual current state, not just the tone of the last message about it. If a status source wasn't checked, say so explicitly rather than treating that as evidence of anything.

### Step 6 — Classify status

Exactly four rendered values, in this order everywhere: **Pending, Partially Complete, Completed, Informational** (Informational also covers the two heuristics above). Internally, an "insufficient evidence" case maps to Pending with an explicit note — never render it as Completed.

For each item, be ready to state: status, owner (or "TBD" — never guessed), actions taken, outcome, recommended next step, timing, dependencies, and the evidence behind the call.

### Step 7 — Render the digest

Open with the count disclosure from Step 3: raw messages found, threads actually evaluated, and a brief note on the gap (filtering, thread grouping, dedup, excluded bulk mail, items downgraded to Informational). Never conflate the two counts.

Table columns, in this exact order (note: Status leads here, unlike the Slack adapter's table — not worth forcing the two adapters into matching column order): **Status, Email thread subject line, Key topic, Key Contact.**

Full output shape (shared with the Slack adapter):

1. The date-range disclosure line (Triggers and date range, above).
2. The count disclosure, then the summary table, sorted Pending → Partially Complete → Completed → Informational.
3. **Need Action**, split into:
   - **Waiting on You** — Art is the next actor.
   - **Waiting on Others** — someone else owes the next move. Each item gets a ready-to-send nudge (see below). Show both subsections even if one is empty.
4. **Completed Follow-Ups** — one to three bullets per item: who did what, when, outcome — and, same as every Need Action item, its own source link.
5. **Art Only** — the career/internal-development carve-out from Step 4. Omit if nothing qualifies. Same link rule as Completed.
6. **Informational** — items from the two heuristics above, each with a visible "(inferred — flag if wrong)" tag and its own source link. Don't drop these silently; a one-line mention is the point.
7. Every item, in every section without exception — Need Action, Completed, Art Only, and Informational alike — surfaces its preserved Gmail link as a real, working inline link (a linked subject line, or a trailing "[Open in Gmail](url)" — either is fine, consistency across items in one run matters more than which form). No item ships without one; no raw search dumps.

**This adapter's first live run shipped with zero links on any item, anywhere** — Step 2 said to preserve one, but nothing in this render step actually said to use it, so it never made it into the output. Treat link-inclusion as a hard requirement to double-check before shipping, on every single item, not just the ones with a full write-up.

For each Need Action item, include what's known of: owner, timing, dependencies/blockers, and an aging note (see Aging & Run History) when the item isn't new.

### Ready-to-Send Nudges (text-only, for now)

For every item in Waiting on Others, write a short nudge and **display the text inline in the digest** — do not create an actual Gmail draft or send anything. The nudge should reference the original ask and how long it's been open, stay short and plain, and sound like something Art would actually send, not a form letter.

**This is a deliberate, temporary scope-down.** Don't call `create_draft`, `send_message`, `reply`, or any other write-capable Gmail tool for this feature under any circumstance — only resume actually drafting/sending nudges if Art explicitly says to turn it on in a future request. Until then, the nudge is just words on the page for Art to copy if he wants them.

### Cross-Source Overlap Flag

After the Gmail-side digest is otherwise complete, check whether a recent `artos-followup-slack` digest is readable in the shared `Follow-Up Digests` output folder (most recent file matching that skill's output naming pattern, for a similar date window). If so, compare items on rough similarity — shared participant(s), close subject/topic match, same external party — and add a one-line note to any plausible match: "Also tracked in the Slack digest — see there for [what it adds]." This never changes status, merges, or suppresses an item; it's informational only. When unsure whether two items match, flag it anyway. If no companion digest is available, skip this step silently.

### Step 8 — Save the digest

Filename: `<run-date>-followup-gmail-digest.md`, using `YYYY-MM-DD` for the date this run happened, in America/New_York (the date run, not the window it covers). Save into the `Follow-Up Digests` subfolder of Art's Art OS workspace folder (create it if it doesn't exist), not the workspace root and not a project-specific subfolder.

Running this again on the same day overwrites that day's file — intended behavior, idempotent regeneration for a given day, not a growing pile of near-duplicate files. Don't ask for confirmation before this specific overwrite; do mention in the chat response that today's digest was refreshed.

### Aging & Run History

Persist a small state file, `artos-followup-gmail-state.json`, in the current working directory (create it if it doesn't exist — this one file evolves across runs rather than being a new dated artifact each time). Key items by Gmail thread ID — never a regenerated subject line, since those can vary or get re-threaded.

Per item, track: first-seen date, last-seen date, number of runs it's appeared in, number of times a nudge was drafted for it. On each run: match newly-extracted items against the file by identifier, update it, and add an aging note to any item that isn't new (e.g., "open since Sept 10, appeared in 3 runs, nudged twice"). New items get no aging note. Once an item resolves to Completed, drop it from the state file. Aging doesn't apply to Informational items — they're not "stalling," they're just noted. An item that simply didn't get extracted this run isn't assumed resolved; only real evidence moves it to Completed.

### Confidence Tagging

Any classification reached by the Ownership or Suspect-solicitation heuristic above must render with a visible tag, e.g. "(inferred — flag if wrong)" — both in the table and in the Informational section — distinguishing it from a directly-evidenced call. If Art corrects an inferred call, that correction is worth remembering as standing guidance, not just a one-off fix to that day's digest — mention it back to him so it can be captured.

### Voice and persona

Light assistant persona: brief banter to open, a short pop-culture reference to close, both kept subordinate to the actual content. Any humor comes only after the substantive digest, never before or mixed into it. Vary the reference naturally. The body of the digest (Need Action detail) stays plain and factual.

### Safety and quality checks before responding

- Never infer completion from an absence of evidence.
- Never expose a message Art doesn't have permission to see (should be a non-issue since retrieval runs as him, but stated explicitly).
- Never fabricate a link, owner, deadline, participant, or outcome.
- Label inferred ownership/timing as inferred, never stated as fact — this applies especially to the two heuristics above.
- No raw search dumps.
- No draft-creation or send-capable tool call anywhere in this skill, for any reason, until Art explicitly turns nudging on.
- Watch especially for: a false "Completed" call with no evidence, **a missing source link on any item — this adapter's first real run had zero links anywhere, check this every time**, a missed direct ask, an invented deadline/owner/outcome, treating "source unavailable" as proof of non-completion, and — specific to this adapter — over-applying the ownership or solicitation heuristic to something that's actually a real, personal ask.

### Changelog

- **v1.1 (2026-09-14):** Added the actual link-rendering step this adapter was missing (Step 2 said to preserve a Gmail link; nothing before this said to render it, so the first real digest shipped with zero links on any item). Also added a mandatory output_folder/Step 8 save (the "Follow-Up Digests" subfolder) so runs land in one predictable place instead of scattering across the workspace root and project folders — needed by downstream tools like the Follow-Up Console prototype.

### Source

Adapted from Glean's published agent "A1 - Follow Up Finder (GMAIL)" (https://app.glean.com/chat/agents/598ad1a092ea4fa890e660e75c260d47). Built and validated 2026-09-14 with a dry run (no real Glean control exists for this agent, and Art confirmed proceeding without one) against the Sept 7–14 window, followed by a second run exercising the full embellishment set. Two real corrections from Art's own review during the dry run produced the Ownership and Suspect-solicitation heuristics above — both logged as standing guidance, not one-off fixes. Full design history and test reports live in the `2026-09-14-follow-up-finder-skill` project (core PRD, Gmail adapter PRD, and dated test reports).

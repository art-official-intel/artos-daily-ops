# artos-daily-ops skills

Experimental version control for Art's Daily Ops skill set (calendar nav,
meeting prep, and Slack/Gmail follow-up finders). Goal: run this logic from
Claude Code, independent of any one laptop, with real version history.

## Status

Early experiment. Not yet the source of truth for these skills — that's
still the Cowork skill cache, reached through the `save_skill` tool. This
repo is where changes get tested and reviewed before they're trusted to
sync back. See the workstation's `CLAUDE.md` (in the connected Art OS
folder, not copied into this repo) for the full versioning and publishing
protocol: disk leads, live follows, and publishing to the live skill is
always a separate, deliberate step, never an implicit side effect of
editing here.

## What lives here

Each skill's `SKILL.md`, `CHANGELOG.md`, and `versions/` (prior snapshots —
the actual rollback mechanism, since `save_skill` itself has no undo). Also
`resources/2026-09-14-follow-up-finder-core-rules.md`, the shared extraction/
status logic used by both follow-up adapters, and `prd/`, the 4 Follow-Up
Finder PRDs and the Slack/Gmail consolidation decision memo.

The PRDs originally named real colleagues and real partner deals (an MNDA
follow-up, a partner deck review) as concrete examples grounding the design
decisions. Rather than leave those out entirely, the names were substituted
with fictional stand-ins ("Northbridge Systems," "Meridian," "Summit Ridge,"
"Riley Chen," "Dana," "Jordan") — each PRD says so in a note right under its
frontmatter. The reasoning and decisions themselves are real and unchanged;
only the identifying names are fictional. No generated output, no live
follow-up-tracking state, and only the one resource file above — see
`.gitignore` for why: rendered digests, test/validation reports, and early
prototypes all quote real, real-world Slack/Gmail/Calendar activity
directly (real messages, real timestamps), which is a different bar than a
PRD's occasional named example and wasn't sanitized for this pass.

## Skills in this set

- `artos-daily-nav` — renders one day as an interactive HTML calendar-nav
  brief, leaning on `artos-meeting-prep` for content.
- `artos-meeting-prep` — turns a day's Google Calendar events into an
  evidence-based meeting-prep Markdown doc.
- `artos-followup-slack` — scans Slack activity for open commitments/asks,
  checks real resolution status, produces a Markdown digest.
- `artos-followup-gmail` — same job as the Slack adapter, for Gmail.
- `artos-followup-console` — renders both follow-up digests as one
  interactive Kanban board.

## Baseline

All 5 skills are at v1.0 as of 2026-09-18 (see each skill's `CHANGELOG.md`).
This was a disk-and-live sync pass, not a behavior change: two skills
(`artos-daily-nav`, `artos-meeting-prep`) had drifted from what was actually
live, and a `metadata` block (version/author/last_updated) was added to all
five for the first time. Confirmed live-synced across all 5 via `save_skill`
before this repo was created.

## Left out of this migration, deliberately

- `resources/` (all but `2026-09-14-follow-up-finder-core-rules.md`) — dry
  run and control-test reports quoting real Slack/Gmail/Calendar content,
  real colleagues, and real partners by name.
- `resources/prototypes/` — early HTML prototypes, same issue.
- `skills-packaged/` — the installable `.skill` zips are a build artifact
  (SKILL.md + CHANGELOG.md), regenerable from what's already in this repo.
- `output/`, `state/` — Art's real rendered briefs/digests and live
  follow-up tracking state. Never copied out of the workstation folder.

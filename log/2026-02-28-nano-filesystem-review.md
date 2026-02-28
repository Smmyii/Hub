---
date: 2026-02-28
type: review
summary: "Reviewed Nano filesystem design doc, identified 4 Hub improvements, implemented all"
projects_touched: [nano, hub]
source: hub-session
---

## What Was Done

Reviewed Nano's `docs/plans/2026-02-28-better-filesystem-design.md` (5 sections, 6 decisions) against the existing Hub architecture. Identified what overlaps (Hub already has append-only history, thin context, structured handoffs, ownership model) and what's worth importing.

Implemented 4 improvements:
1. **State freshness** — `last_updated` ISO timestamps standardized across all work item state.yaml files
2. **Active session registry** — `Hub/.active/` directory with per-session YAML files for concurrent instance visibility
3. **Structured session logging** — `Hub/log/` directory with frontmatter schema adapted for Hub context
4. **Bob profile freshness** — `last_mail_date` and stale contact flagging in bob profiles

Explicitly rejected 6 Nano concepts that don't fit Hub (ops department, career tracking, position-aware CLAUDE.md, Nano-specific skills, derived views, hierarchical orchestration).

## Decisions Made

- Hub log schema uses `type: review | dispatch | planning | bob-deploy | mail` (not Nano's `work | decision | blocker | milestone | reflection`)
- No `skills_demonstrated` or `difficulty` fields — Hub doesn't build things, those are Nano-specific
- Active session files go in `Hub/.active/` (top-level dot directory) not inside `.planning/active/` — Hub isn't a department
- Bob staleness threshold set at 7 days with no contact

## What's Next

- Hub architecture docs need updating to reflect the 4 new conventions
- Future sessions should write to `Hub/log/` and register in `Hub/.active/`
- Consider a hub-scribe skill to enforce session logging (like Nano's nano-scribe)

# BMO Conventions — Formats & Procedures

## Mail Format

All mail is markdown files in `~/Documents/Hub/mail/`.

**Filename:** `YYYY-MM-DD-<direction>-<agent>-<subject>.md`
- Inbound: `2026-03-10-from-larry-licom-status-update.md`
- Outbound: `2026-03-10-to-larry-desktop-widget-brief.md`

**Structure:**
```markdown
# Subject Line

**From:** BMO | Larry:<Project>
**To:** Larry:<Project> | BMO | Sammy
**Date:** YYYY-MM-DD
**Type:** status_update | task | request | question | context_share
**Action Needed:** Yes/No — brief description if yes

---

Body content here.

---

*<sender> — YYYY-MM-DD*
```

## Status Reports (from Larrys)

Larrys push status updates to Hub mail. BMO distills these into the dashboard.

## Dashboard Format (`bmo/dashboard.md`)

```markdown
# Project Dashboard — YYYY-MM-DD

| Project | Larry | Status | Current Focus | Blocked |
|---------|-------|--------|---------------|---------|
| Licom | Larry:Licom | Active | v1.1 planning | No |
| Desktop | Larry:Desktop | Active | Widget Phase 1 | No |

## Notes
- One-liner per notable item.
```

Update the dashboard when: a Larry reports status, a project changes phase, or a blocker appears.

## Larry Brief Format

When routing work to a Larry, write a brief:

```markdown
# Brief: <title>

**Work Item:** WI-XXX
**Priority:** high | medium | low
**Context:** What BMO knows about this task
**Goal:** What the Larry should achieve
**Constraints:** Deadlines, dependencies, blockers
**Report back:** What BMO needs to know when done
```

## Creating a New Larry

1. Create profile: `larrys/profiles/<name>.yaml`
   ```yaml
   name: "Larry:<Project>"
   project: <project-name>
   created: YYYY-MM-DD
   status: active
   repo: ~/Documents/<Project>/
   ```
2. Copy template to project: `larrys/template/` -> project's `.planning/larry/`
3. Write initial brief with project context and first task
4. Log creation in `bmo/decisions.log`
5. Update `bmo/dashboard.md`

## Project Configs (`projects/*.yaml`)

```yaml
name: <project>
repo: ~/Documents/<Project>/
larry: Larry:<Name>
status: active | paused | completed
created: YYYY-MM-DD
notes: "one-liner"
```

## Station Workflow (Work Items)

Work items on the board move through stations:
`Proposed -> Research -> Planning -> Ready -> Executing -> Review -> Done`

BMO owns state transitions on the board. Larrys own execution within their project.

## decisions.log (append-only)

```
YYYY-MM-DD | DECISION | <what> | <why>
YYYY-MM-DD | OVERRIDE | <what changed> | <why>
```

# Larry Conventions — Communication & File Formats

## Mail Format

Messages to BMO go in `~/Documents/Hub/mail/inbox/` as markdown files.

**Filename:** `YYYY-MM-DD-from-larry-<project>-<subject>.md`

**Structure:**
```markdown
# Subject Line

**From:** Larry:<Project>
**To:** BMO
**Date:** YYYY-MM-DD
**Type:** status_update | request | question | context_share
**Action Needed:** Yes/No — brief description if yes

---

Body content here.

---

*Larry:<Project> — YYYY-MM-DD*
```

**Types:**
- `status_update` — work completed, phase changed, milestone hit
- `request` — need something from BMO or another Larry
- `question` — need a decision or clarification
- `context_share` — sharing information BMO should know

## Status Files

### state.yaml (Larry's current position)
```yaml
project: <name>
session_date: YYYY-MM-DD
current_focus: "brief description"
departments:
  core: "one-line status"
  mobile: "one-line status"
active_work_package: "<filename in active-work/>"
blocked: false
blocked_reason: ""
```

### decisions.log (append-only)
```
YYYY-MM-DD | DECISION | <what was decided> | <why>
YYYY-MM-DD | OVERRIDE | <what changed> | <why it changed>
```

## Boot Sequence

1. Read `identity.md` — who am I, rules
2. Read `state.yaml` — where is everything
3. Read `departments.yaml` — department summaries
4. Check `needs/*.md` — pending requests
5. Check BMO mail — anything from BMO
6. Read `active-work/<focus>.md` — based on session direction
7. Grep `decisions.log` — when past context needed

Steps 1-5 are mandatory (~100 lines). Steps 6-7 are on demand.

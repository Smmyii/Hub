# Bob Conventions — Communication & File Formats

## Mail Format

Messages to Hub go in `~/Documents/Hub/mail/inbox/` as markdown files.

**Filename:** `YYYY-MM-DD-from-bob-<project>-<subject>.md`

**Structure:**
```markdown
# Subject Line

**From:** Bob:<Project>
**To:** Hub Claude
**Date:** YYYY-MM-DD
**Type:** status_update | request | question | context_share
**Action Needed:** Yes/No — brief description if yes

---

Body content here.

---

*Bob:<Project> — YYYY-MM-DD*
```

**Types:**
- `status_update` — work completed, phase changed, milestone hit
- `request` — need something from Hub or another Bob
- `question` — need a decision or clarification
- `context_share` — sharing information Hub should know

## Status Files

### state.yaml (Bob's current position)
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
5. Check Hub mail — anything from Hub
6. Read `active-work/<focus>.md` — based on session direction
7. Grep `decisions.log` — when past context needed

Steps 1-5 are mandatory (~100 lines). Steps 6-7 are on demand.

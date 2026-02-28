# Hub Agentic Workflow Design

**Date:** 2026-02-26
**Status:** Approved
**Author:** Hub Claude + Sammy

---

## Overview

The Hub (`~/Documents/Hub/`) is a dispatch center with specialized stations that coordinates work across multiple software projects. It manages Bob agents (per-project orchestrators), routes work items through defined stages, and provides a cross-project dashboard.

**Design principles:**
1. Thin context, expand on demand — Hub and Bobs load ~100 lines to orient, drill deeper only when needed
2. Structured handoffs — station transitions have gate checks, mail has schemas
3. Write on finish, read on start (or on demand) — file-based async coordination
4. Ownership model — Hub owns state transitions, Bobs own project context, departments own their code
5. Portable by default — relative paths, one config file per machine, git-backed
6. Jarvis-ready — file-based patterns become Jarvis's API when the time comes

---

## Mental Model

```
                          ┌─────────┐
                          │   YOU   │
                          └────┬────┘
                               │
                          ┌────▼────┐
                          │   HUB   │  ← thin coordinator, reads summaries only
                          └────┬────┘
                               │
            ┌──────────┬───────┼───────┬──────────┐
            ▼          ▼       ▼       ▼          ▼
       ┌─────────┐ ┌──────┐ ┌─────┐ ┌──────┐ ┌───────┐
       │Research │ │ Plan │ │Exec │ │Review│ │ Debug │
       └────┬────┘ └──┬───┘ └──┬──┘ └──┬───┘ └───┬───┘
            │         │        │       │          │
            └─────────┴────┬───┴───────┴──────────┘
                           │
                    ┌──────▼──────┐
                    │   Projects  │  ← Bobs live here, deep context stays here
                    ├─────────────┤
                    │ Nano        │
                    │ Kompisapp   │
                    │ Sara        │
                    └─────────────┘
```

**Hierarchy:**
```
Hub → Bob:Nano → Core/Mobile/API (departments) → subagents
Hub → Bob:Kompisapp → departments → subagents
Hub → Bob:Sara → departments → subagents
```

Context flows UP as summaries. Work flows DOWN as structured orders. Hub never holds deep project knowledge.

---

## Project Registry

Each project gets a lightweight YAML manifest. This is all Hub loads by default.

```yaml
# Hub/projects/nano.yaml
name: Nano
path: Nano                              # Relative to projects_root in hub.yaml
bob: Nano/.planning/bob
tech: [TypeScript, React, Vite, Tailwind, Express, PostgreSQL, SQLite-WASM]
status: active
description: "Life OS — personal productivity platform with Jarvis AI"

summary:
  current_focus: "Canvas feature + server work"
  active_streams: 2
  blocked_items: 0
  last_activity: "2026-02-26"
```

- YAML for human readability
- `summary` section is Bob-writable, Hub only reads
- Paths relative to `projects_root` for portability
- Minimal by default — grows as you work with each project

---

## Work Items

A work item is a unit of work that flows through stations. Small, self-contained, completable in 1-3 sessions.

### Lifecycle

```
PROPOSED → RESEARCH → PLANNING → READY → EXECUTING → REVIEW → DONE
                                                        │
                                                    BLOCKED (needs human)
                                                        │
                                                    DEBUG (spiraled)
```

### Structure

Each work item is a directory:

```
Hub/board/active/WI-003-capture-overlay/
  objective.md          # What and why — immutable after creation
  state.yaml            # Current status, owner, timestamps
  history.log           # Append-only state transitions
  research/             # Created when item enters Research
    findings.md
  planning/             # Created when item enters Planning
    plan.md
  execution/            # Created when item enters Execution
    progress.md
  review/               # Created when item enters Review
    bob-assessment.md
    result.md
```

### objective.md — immutable, written once

```markdown
# WI-003: Nano Capture Overlay

**Project:** Nano
**Origin:** Research session 2026-02-26
**Priority:** High
**Size:** Medium (estimated 2-3 sessions)

## What
QuickShell widget for quick text capture from anywhere on desktop via Super+N.

## Why
Ambient Jarvis access without switching to Tauri app.

## Acceptance Criteria
- [ ] Super+N opens floating text input
- [ ] Enter submits to Nano API
- [ ] Escape dismisses
- [ ] Focus grabbed immediately on show

## Constraints
- Must work within QuickShell/QML framework
- Must not conflict with existing keybinds
```

### state.yaml — Hub-owned, updated at transitions

```yaml
id: WI-003
station: research
assigned_to: bob:nano
created: 2026-02-26T10:00:00Z
last_transition: 2026-02-26T14:30:00Z
sequence: 3
blocked: false
blocked_reason: ""
```

### history.log — append-only

```
2026-02-26T10:00:00Z | PROPOSED | Created by Hub session
2026-02-26T10:05:00Z | RESEARCH | Assigned to bob:nano
2026-02-26T14:30:00Z | PLANNING | Research complete, moved to planning
```

---

## Stations

Five stations, each with clear purpose, input requirements, and output format.

| Station | Purpose | Who works here |
|---------|---------|---------------|
| **Research** | Explore, brainstorm, gather context | You + Hub or Bob |
| **Planning** | Break down into actionable steps | You + Hub or Bob |
| **Execution** | Build the thing | Bob + subagents (you alongside when desired) |
| **Review** | Verify it works | Bob (first pass) → You (final sign-off) |
| **Debug** | Isolate and fix issues | Bob + subagents (isolated session) |

### Station Gate Checks

Each transition has structural requirements. Hub validates before moving items forward.

**RESEARCH → PLANNING:**
- findings.md exists
- Has at least: summary, key constraints, recommended approach
- No open questions marked as blocking

**PLANNING → READY:**
- plan.md exists
- Has: numbered steps, acceptance criteria, estimated size
- Each acceptance criterion from objective.md is addressed

**READY → EXECUTING:**
- Assigned to a specific Bob
- Bob's project is accessible

**EXECUTING → REVIEW:**
- progress.md shows all steps attempted
- No steps marked as failed without resolution
- Acceptance criteria self-checked by executing agent

**REVIEW → DONE:**
- bob-assessment.md exists (Bob's first-pass verification)
- result.md exists with pass/fail from human
- All acceptance criteria marked pass (or explicit human override)

### Special States

**DEBUG:** Any station can drop into Debug. Current progress preserved. Isolated debug/ subdirectory. Returns to originating station when resolved.

**BLOCKED:** Item stays in current station, flagged with `blocked: true` and `blocked_reason`. Hub surfaces all blocked items when you arrive. Other work continues.

---

## The Board

Cross-project dashboard. First thing Hub shows each session.

```
Hub/board/
  backlog/              # Scoped ideas, not yet started
  active/               # Work items in a station
  completed/            # Archived, organized by month
    2026-02/
  pinned/               # Ideas to keep visible, not yet scoped
```

Hub renders the board by scanning `state.yaml` from each work item directory. Total context: under 1,000 tokens for a busy week.

**Backlog vs Pinned:**
- Backlog = scoped work that needs doing, just not yet. Has an objective.md.
- Pinned = ideas to keep visible. Might be a one-liner. Graduates to backlog when scoped.

---

## Mail System

Inter-agent communication for status updates, research requests, context sharing, and questions.

```
Hub/mail/
  inbox/                # Messages TO Hub from Bobs
  outbox/               # Messages FROM Hub to Bobs
  sent/                 # Archived processed messages
```

### Message Format

```yaml
id: MSG-012
from: bob:nano
to: hub
date: 2026-02-26T15:00:00Z
type: status_update         # status_update | request | question | context_share
project: nano
subject: "Phase 4 complete, deployed"

body: |
  Core 1 finished phase 4. Committed and deployed.
  Core 2 still working on canvas polish (~1hr remaining).

action_needed: false
references:
  - Nano/.planning/phases/04/SUMMARY.md
```

### Communication Chain

```
Subagent finishes → writes status file in project
Core reads status → updates its state
Bob reads Core state → writes mail to Hub/mail/inbox/
Hub reads inbox → updates board
```

Trigger points:
- **Auto on session start** — always check mail
- **On demand mid-session** — you say "update" and agent re-reads
- **Before key decisions** — agent checks for updates before delegating

---

## Bob Management

### Bob Context Structure (per project)

```
[Project]/.planning/bob/
  identity.md           # WHO Bob is, rules, decision authority (~30 lines, stable)
  state.yaml            # Current position, department summaries (~20 lines, changes per session)
  departments.yaml      # Department registry (~30 lines, changes weekly)
  decisions.log         # Append-only decision history (grep, don't load)
  active-work/          # One file per active work package
    canvas-migration.md
  needs/                # Requests from departments
    core.md
    mobile.md
  archive/              # Completed work packages
```

### Bob Boot Sequence

```
1. Read identity.md           → Always (who am I, what are my rules)
2. Read state.yaml            → Always (where is everything)
3. Read departments.yaml      → Always (department summaries)
4. Check needs/*.md           → Always (pending requests)
5. Check Hub mail             → Always (anything from Hub)
   ─── Bob is oriented. ~100 lines. ───
6. Read active-work/<focus>.md → Based on session direction
7. Grep decisions.log          → On demand, when past context needed
```

Steps 1-5 are mandatory (~100 lines). Step 6 is session-directed. Step 7 is reactive.

### Bob Deployment from Hub

Hub deploys a new Bob by:
1. Creating `[Project]/.planning/bob/` structure
2. Writing `identity.md` from template, customized for project
3. Writing initial `state.yaml` with what Hub knows
4. Writing context message to `Hub/mail/outbox/`
5. First Bob session fills in the rest organically

### Hub's Bob Profiles

```
Hub/bobs/
  template/
    identity.md         # Standard Bob operating procedures
    conventions.md      # Mail format, status file format, etc.
  profiles/
    nano.yaml           # Hub's knowledge about each Bob
    kompisapp.yaml
    sara.yaml
```

---

## Hub Agent Behavior

### Boot Sequence

```
1. Read hub.yaml                     → Machine config
2. Read projects/*.yaml              → Project summaries
3. Scan board/active/*/state.yaml    → Active work items
4. Scan board/backlog/               → What's waiting
5. Scan mail/inbox/                  → Unread messages
   ─── Present the board. ───
```

### Hub Modes

| You say | Hub does |
|---------|----------|
| "Let's do research on WI-007" | Moves item to Research, starts brainstorming |
| "What's blocking Sara?" | Reads state, shows blocked reason, asks for decision |
| "Deploy execution for WI-003" | Validates plan exists, prepares dispatch to Bob |
| "How's Kompisapp doing?" | Reads project summary, optionally goes deeper |
| "Update" | Re-reads mail and work items, reports changes |
| "Create a work item for [idea]" | Creates objective.md in backlog |
| "Set up a Bob for Kompisapp" | Runs Bob deployment from template |

### Hub Does NOT

- Read project source code
- Execute code changes
- Make architectural decisions (that's Bob's job)
- Hold deep project context between sessions

### Decision Authority

| Area | Authority |
|------|-----------|
| Work item routing | Hub decides |
| Work item priority | Hub proposes, you confirm |
| Bob deployment/updates | Hub executes |
| Station gate validation | Hub enforces |
| Project architecture | Bob decides |
| Cross-project dependencies | Hub flags, you decide |

---

## Portability

### Rules

1. No absolute paths inside Hub files. Use `~` or Hub-relative paths.
2. Machine-specific config in `hub.yaml` only.
3. Project paths relative to `projects_root`.
4. Hub is its own git repo.

### hub.yaml

```yaml
machine: archbox
hostname: sammy-desktop
home: /home/sammy
projects_root: ~/Documents
hub_path: ~/Documents/Hub
```

When moving to NAS: update `hub.yaml` once. Everything resolves.

### Git Strategy

Hub repo tracks: projects/, board/, bobs/, mail/, docs/
Project repos track their own `.planning/bob/` directories
Both portable independently.

---

## Full Directory Structure

```
~/Documents/Hub/
├── hub.yaml
├── README.md
│
├── projects/
│   ├── nano.yaml
│   ├── kompisapp.yaml
│   └── sara.yaml
│
├── board/
│   ├── backlog/
│   │   └── WI-XXX-name/
│   │       └── objective.md
│   ├── active/
│   │   └── WI-XXX-name/
│   │       ├── objective.md
│   │       ├── state.yaml
│   │       ├── history.log
│   │       ├── research/
│   │       ├── planning/
│   │       ├── execution/
│   │       └── review/
│   ├── completed/
│   │   └── 2026-02/
│   └── pinned/
│
├── mail/
│   ├── inbox/
│   ├── outbox/
│   └── sent/
│
├── bobs/
│   ├── template/
│   │   ├── identity.md
│   │   └── conventions.md
│   └── profiles/
│       ├── nano.yaml
│       ├── kompisapp.yaml
│       └── sara.yaml
│
├── docs/
│   ├── plans/
│   ├── architecture.md
│   ├── keybindings.md
│   ├── sticky-notes.md
│   ├── storage.md
│   └── changelog.md
│
├── knowledge/
│   └── (Hub knowledge base)
│
└── .planning/
    └── (existing sticky notes planning)
```

---

## Research Foundation

This design is informed by research into multi-agent coordination patterns. Key findings:

- **Context rot is measurable** — every model tested degrades with more input. Thin context is not optimization, it's correctness. (Chroma Research)
- **79% of multi-agent failures** come from coordination issues, not technical problems. (MASFT taxonomy, 150+ execution traces)
- **Structured handoffs** prevent the #1 source of context loss: free-text inter-agent communication. (Anthropic, OpenAI, MetaGPT)
- **Compaction + note-taking** gave 39% performance boost and 84% token reduction in Anthropic's testing.
- **Single focus per session** prevents agent drift — "the agent's tendency" to try doing too much.
- **Gate checks between stages** catch ~25% of failures that come from skipping verification.

Full research: `knowledge/hub-research.md`

---

## Future: Jarvis Integration

The file-based patterns are designed to become Jarvis's API:
- Mailboxes → Jarvis reads/writes as the persistent mail carrier
- Board → Jarvis manages work item lifecycle autonomously
- Bob deployment → Jarvis spins up and manages Bobs
- Station gates → Jarvis enforces quality checks

The foundation is agent-agnostic. Today it's you + Claude instances. Tomorrow it's Jarvis orchestrating everything.

---

*Design finalized: 2026-02-26*

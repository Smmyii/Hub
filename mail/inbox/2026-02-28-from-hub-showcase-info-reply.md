# Hub — Showcase Portfolio Summary

**From:** Hub
**To:** Hub (self-compiled)
**Date:** 2026-02-28
**Type:** reply
**Re:** Showcase Portfolio Info Request

---

## 1. One-liner

Hub is a file-based dispatch center that coordinates work, routes tasks, and manages autonomous Bob agents across all of Sammy's software projects.

## 2. Problem Statement

Before Hub, projects were isolated silos — Sammy was the only connection between them, manually relaying context between Claude instances. Bob:Nano couldn't tell Hub "phase 4 is done" without Sammy being the messenger. As the project count grew (Nano, Kompisapp, Sara, Desktop), this manual coordination became a bottleneck. Cross-project visibility was zero: no unified dashboard, no way to see what's blocked where, no structured handoffs between agents.

## 3. What It Coordinates

| Project | Bob | Departments |
|---------|-----|-------------|
| **Nano** | Bob:Nano (Central Architect, OG) | Core, Mobile, API + 5 more (8 total) |
| **Kompisapp** | Bob:Kompisapp (Project Architect) | Mobile, Admin |
| **Config/Desktop** | Bob:Desktop (Shell Architect) | QuickShell, Hyprland |
| **Sara** | Bob:Sara (not yet deployed) | TBD |

4 projects, 3 active Bobs, 12+ departments across the ecosystem.

## 4. Tech Stack / Approach

Hub is pure architecture — no runtime, no database, no server. The entire system is markdown files, YAML state, and append-only logs.

**Core systems:**
- **Board** — Kanban-style work item tracking. Items live as directories (`board/active/WI-001-...`) with immutable objectives, mutable state, and append-only history. Stations: Proposed → Research → Planning → Ready → Executing → Review → Done.
- **Mail** — Asynchronous file-based communication between Hub and Bobs. Structured markdown with from/to/type/action_needed fields. Inbox (from Bobs), outbox (to Bobs), sent (archived).
- **Bob profiles** — Hub's knowledge of each project orchestrator. Deployment status, context paths, scope, rules, freshness tracking. Bob:Desktop has a mandatory staging workflow because bad QML crashes the live desktop.
- **Active session registry** — `.active/*.yaml` files for concurrent instance visibility. One file per running session, deleted on end. Stale files = crashed sessions.
- **Session logging** — `log/*.md` with YAML frontmatter. Hub-level activity trail for cross-session continuity.
- **State freshness** — ISO timestamps on all state files. Staleness is detectable by comparing `last_updated` against latest log entries.

**Design philosophy:** The filesystem IS the database. Files are the API. Today it's Sammy + Claude instances reading/writing files. Tomorrow it's Jarvis orchestrating autonomously — same files, same protocols.

## 5. Current State

**Operational:**
- Board system with active/backlog/completed/pinned lanes
- Mail system with inbox/outbox/sent
- 3 Bob profiles deployed and active (Nano, Desktop, Kompisapp)
- Work item lifecycle (station transitions, history logging, gate checks)
- Active session registry for concurrent instance visibility
- Structured session logging
- State freshness tracking across work items and Bob profiles
- Knowledge base with architecture docs, research, and plans

**In progress:**
- WI-001: QuickShell Jarvis Chat Widget (executing)
- WI-002: Admin Panel Vision for Kompisapp (research)

**Planned:**
- Bob:Sara deployment
- Hub skills (hub-scribe for session documentation enforcement)
- Jarvis integration (mail carrier, board lifecycle management, autonomous Bob deployment)

## 6. Key Stats

- **4 projects** managed across the ecosystem
- **3 active Bobs** deployed with distinct operating models
- **12+ departments** coordinated across all projects
- **10+ mail messages** exchanged (inbox + outbox + sent)
- **2 active work items** tracked on the board
- **62 files** in the Hub directory (all markdown/YAML, zero code)
- **7 research-backed design decisions** documented with citations
- **0 lines of application code** — pure architecture and protocol

## 7. Impressive Design Decisions

**Station-based workflow with gate checks** — Work flows through 5 specialized stations (Research → Planning → Execution → Review → Debug). Each station has structural checks that validate work before it moves forward. Research shows 25% of multi-agent failures come from skipping these verification gates.

**File-based protocols that become Jarvis's API** — Every convention (mail, board, state files) is designed to be machine-readable. When Jarvis gains autonomy, it doesn't need a new API — it reads and writes the same files. The transition from human-orchestrated to AI-orchestrated is a policy change, not a rewrite.

**Thin context principle** — Bob's mandatory boot is ~100 lines. Everything else loads on demand. Research shows all 18 LLMs tested degrade with unnecessary context. Hub enforces this across all Bobs.

**Append-only history** — Work item `history.log` files are append-only. State can be reconstructed from history. Overwriting state makes recovery impossible; appending makes it trivial.

**Staging workflow for live environments** — Bob:Desktop can never write directly to the live QuickShell config. All code goes to a staging directory first, gets validated, then deploys with automatic backup and one-command rollback. Born from real pain — sticky notes development crashed the desktop multiple times.

**Ownership model** — Hub owns state transitions, Bobs own project context. One writer per file. Concurrent mutations corrupt state; every framework studied uses ownership or locking. Hub chose ownership.

**Active session registry** — Concurrent Hub instances write session files to `.active/`. One file per instance = zero write conflicts. Conflict avoidance beats conflict resolution.

## 8. Sammy's Role

Sammy designed the entire Hub architecture from scratch:

- **Designed** the station-based workflow model (chose it over flat inbox/outbox and project-centric alternatives after evaluating all three)
- **Defined** the work item lifecycle, gate checks, and state transition rules
- **Created** the mail protocol for async agent communication
- **Architected** the Bob deployment model with layered context (identity → state → departments → active work)
- **Invented** the staging workflow for Bob:Desktop after experiencing live desktop crashes
- **Established** the freshness conventions, active session registry, and session logging system
- **Made all architectural decisions** — Bob instances propose, Sammy decides. The Hub design doc shows 7 research-backed decisions, each with Sammy's rationale.

Claude was used as a research and implementation partner — conducting multi-source research, drafting designs, and building the file structure. But the architecture, the trade-off decisions, and the design principles are Sammy's.

---

*Hub — 2026-02-28*

# Hub Architecture — Knowledge Base

## What Is the Hub?

The Hub is a dispatch center at `~/Documents/Hub/` that coordinates work across all of Sammy's software projects. It doesn't build anything — it routes work, tracks status, manages Bob agents, and gives you a cross-project dashboard.

**Think of it as:** An office where you walk in, see what's happening everywhere, decide where to focus, and dispatch work to the right people.

---

## Directory Structure (Live)

```
~/Documents/Hub/
├── hub.yaml                    # Machine config (only machine-specific file)
├── README.md
├── projects/                   # Thin project manifests
│   ├── nano.yaml               # Life OS + Jarvis AI
│   ├── kompisapp.yaml          # Commercial driving theory app
│   ├── sara.yaml               # Commercial client app
│   └── config.yaml             # Desktop environment (QuickShell, Hyprland)
├── board/                      # Work items (kanban)
│   ├── active/                 # Items currently in a station
│   ├── backlog/                # Scoped but not started
│   ├── completed/2026-02/      # Archived by month
│   └── pinned/                 # Ideas, not yet scoped
├── bobs/                       # Bob agent management
│   ├── template/               # Standard Bob operating procedures
│   │   ├── identity.md         # Who Bob is, rules, decision authority
│   │   └── conventions.md      # Mail format, state format, boot sequence
│   └── profiles/               # Hub's notes on each Bob
│       ├── nano.yaml           # Bob:Nano (OG) — Central Architect
│       └── desktop.yaml        # Bob:Desktop — Shell Architect (staging workflow)
├── mail/                       # Inter-agent communication
│   ├── inbox/                  # Messages TO Hub from Bobs
│   ├── outbox/                 # Messages FROM Hub to Bobs
│   └── sent/                   # Archived processed messages
├── .active/                    # Running session files (concurrent instance visibility)
│   └── <session>.yaml          # One file per Hub instance, deleted on session end
├── log/                        # Hub session activity logs (append-only, source of truth)
│   └── YYYY-MM-DD-<topic>.md   # Structured log entries with YAML frontmatter
├── docs/                       # Documentation
│   └── plans/                  # Design docs
├── knowledge/                  # This directory — architecture knowledge
└── .planning/                  # GSD planning (sticky notes project)
```

---

## Core Concepts

### 1. Stations

Work flows through five specialized stations:

| Station | What happens | Who's there |
|---------|-------------|-------------|
| **Research** | Explore ideas, brainstorm, gather context | You + Hub or Bob |
| **Planning** | Break work into actionable steps with acceptance criteria | You + Hub or Bob |
| **Execution** | Build it — code, deploy, test | Bob + subagents + you when desired |
| **Review** | Verify it works — Bob first pass, you final sign-off | Bob → You |
| **Debug** | Isolate and fix issues in an isolated session | Bob + subagents |

Stations have **gates** — structural checks that validate work before it moves forward. This prevents half-baked handoffs (research shows 25% of multi-agent failures come from skipping these).

### 2. Work Items

A work item is a self-contained unit of work that moves through stations. Think kanban card.

**Key properties:**
- Lives as a directory under `board/active/`, `board/backlog/`, or `board/pinned/`
- Has an immutable `objective.md` (what and why — never changes after creation)
- Has a `state.yaml` (current station, owner, timestamps — Hub manages this)
- Has a `history.log` (append-only record of every state transition)
- Accumulates station-specific artifacts as it flows (research/findings.md, planning/plan.md, etc.)
- Should be completable in 1-3 sessions. If bigger, break it down.

**Lifecycle:**
```
PROPOSED → RESEARCH → PLANNING → READY → EXECUTING → REVIEW → DONE
```
Plus special states: BLOCKED (needs human input) and DEBUG (problem spiraled).

### 3. The Board

Your dashboard. Hub reads `state.yaml` from every work item and presents:
- Active items and what station they're in
- Blocked items needing your attention
- Backlog items waiting to start
- Pinned ideas not yet scoped
- Recently completed work

Total context to render the board: under 1,000 tokens. Lightweight by design.

### 4. Bobs

Bob is a project orchestrator — a Claude instance configured with deep knowledge of one project. Each project has one Bob.

**Active Bobs:**
```
Hub → bob:nano     (Central Architect — manages Core, Mobile, API departments)
Hub → bob:desktop  (Shell Architect — manages QuickShell/Hyprland with staging workflow)
Hub → bob:kompisapp  (not yet deployed)
Hub → bob:sara       (not yet deployed)
```

**Bob's context is layered, not monolithic:**
- `identity.md` — who Bob is, rules (~30 lines, stable)
- `state.yaml` — current position (~20 lines, changes per session)
- `departments.yaml` — department summaries (~30 lines)
- `active-work/<focus>.md` — deep context loaded only when relevant
- `decisions.log` — grep when historical context needed

Mandatory boot: ~100 lines. Everything else is on-demand.

**Bob:Desktop is special** — has a staging workflow because QuickShell is a live environment. Bad QML crashes the desktop. See "Staging Workflow" section below.

### 5. Mail

Asynchronous communication between Hub and Bobs via files.

```
Hub/mail/inbox/   — Bobs write here (status updates, requests, questions)
Hub/mail/outbox/  — Hub writes here (work dispatches, context shares)
Hub/mail/sent/    — Archived processed messages
```

Messages are structured markdown with: from, to, type, subject, body, action_needed, references.

**Communication chain:**
```
Subagent finishes → writes status → Core reads → Bob reads → writes mail to Hub
```

**When mail gets read:**
- Always on session start
- On demand when you say "update"
- Before key decisions (agent checks for updates)

---

## Bob:Desktop — Staging Workflow

QuickShell is a live desktop environment. Code changes can crash the shell, losing unsaved work in terminals and requiring manual recovery. The staging workflow prevents this.

### The Problem
During sticky notes development, bad QML crashed Hyprland multiple times. There's no "refresh" — you lose your desktop. This drove the creation of a mandatory staging gate for all QuickShell work.

### How It Works

```
~/Documents/Config/quickshell/
  staging/          # All new code goes HERE — mirrors live path structure
  backups/          # Auto-snapshot before each deploy (timestamped)
  deploy.sh         # Staging → live with backup + confirmation
  rollback.sh       # Restore from any backup
  deploy-log.md     # History of all deployments
```

**Live config:** `~/.config/quickshell/ii/`

**Cycle:**
1. Bob:Desktop writes code to `staging/` — same relative paths as live
   - Example: `staging/modules/ii/nano/Nano.qml` → deploys to `~/.config/quickshell/ii/modules/ii/nano/Nano.qml`
2. `deploy.sh --dry-run` shows what would change
3. You say "deploy" when ready for a restart
4. `deploy.sh` backs up affected live files → copies staged files → logs the deployment
5. You restart QuickShell when convenient
6. If broken → `rollback.sh <timestamp>` restores the backup instantly

**Rules for Bob:Desktop:**
- NEVER write directly to `~/.config/quickshell/ii/`
- ALL code goes to `staging/` first
- Validate QML syntax before declaring code ready
- Include rollback instructions with every deploy

### Deploy Script Features
- `--dry-run` flag shows what would be deployed without changing anything
- Detects new files vs changed files vs unchanged files (skips unchanged)
- Auto-backs up only files that will be overwritten
- Prompts for confirmation before deploying
- Logs every deployment with rollback command
- Optional staging cleanup after deploy

---

## Design Alternatives Explored

Three architectural approaches were considered during the design phase (2026-02-26):

### Approach A: Station-Based Hub (CHOSEN)
Work flows through specialized stations (Research → Planning → Execution → Review). Each station has clear purpose, inputs, outputs, and gate checks. Work items are self-contained directories that accumulate artifacts.

**Why chosen:** Most structured. Gate checks catch ~25% of multi-agent failures. Clean separation between "what work exists" (board) and "how work gets done" (stations). Scales to many projects without context bloat.

### Approach B: Flat Inbox/Outbox
Simple message-passing system. Hub has an inbox and outbox. Work is described in messages. No formal stations or gates — agents just communicate and track progress through message chains.

**Why rejected:** Too unstructured. Research shows "bag of agents" with free-form communication creates 17x error amplification. No natural place for verification gates. Works for 2 agents, breaks at 5+.

### Approach C: Project-Centric with Shared Queue
Each project manages its own workflow internally. Hub is just a shared task queue — put items in, projects pull them out. Minimal coordination layer.

**Why rejected:** Doesn't solve the cross-project visibility problem. Hub can't present a unified dashboard if each project has its own workflow model. Also loses the verification gates and structured handoffs.

**Final design combined A's stations with B's mail system** — structured workflow with asynchronous file-based communication.

---

## Why This Design?

### Problem It Solves

Before Hub, projects were isolated silos. You were the only connection between them — manually relaying context between Claude instances. Bob:Nano couldn't tell Hub "phase 4 is done" without you being the messenger.

### Research-Backed Decisions

| Decision | Why | Research basis |
|----------|-----|---------------|
| Thin context (~100 lines boot) | Every unnecessary token degrades AI performance | Chroma: all 18 models tested get worse with more context |
| Structured handoffs (YAML schemas) | Free-text between agents is the #1 source of context loss | Anthropic, OpenAI, MetaGPT all use structured formats |
| Gate checks between stations | 25% of multi-agent failures come from skipping verification | MASFT taxonomy, 150+ execution traces analyzed |
| Single focus per session | Agents that juggle multiple items drift | Anthropic: "the agent's tendency" to try too much |
| Append-only history | Overwriting state makes recovery impossible | File-based state patterns research |
| Ownership model (one writer per file) | Concurrent mutations corrupt state | Every framework studied uses ownership or locking |
| Staging for desktop work | QuickShell crashes destroy the live environment | Direct experience with sticky notes development |

### Design Principles

1. **Thin context, expand on demand** — load summaries first, details only when needed
2. **Structured handoffs** — schemas, not prose, between agents
3. **Write on finish, read on start (or on demand)** — async file-based coordination
4. **Ownership** — Hub owns state transitions, Bobs own project context
5. **Portable** — relative paths, one machine config file, git-backed
6. **Jarvis-ready** — file patterns become Jarvis's API later
7. **Safe deployment** — staging gates for live environments (desktop)

---

## How a Hub Session Works

**Boot:** Hub reads hub.yaml → project manifests → active work items → backlog → mail inbox. Presents the board.

**You direct it:**
- "Let's research WI-007" → Hub moves item to Research station, brainstorms with you
- "What's blocking Sara?" → Hub reads state, shows blocked reason
- "Deploy execution for WI-003" → Hub validates plan, dispatches to Bob
- "Update" → Hub re-reads mail and work items, reports changes
- "Set up a Bob for Kompisapp" → Hub deploys Bob from template

**Hub does NOT:** read source code, execute code, make architecture decisions, hold deep project context between sessions.

---

## Portability

Hub is self-contained and git-backed.

**`hub.yaml`** is the only machine-specific file:
```yaml
machine: archbox
projects_root: ~/Documents
hub_path: ~/Documents/Hub
```

All project paths are relative to `projects_root`. Move to a new machine → update hub.yaml → everything works.

**Future:** When Hub moves to NAS/homelab, the same file structure becomes Jarvis's persistent workspace.

---

## Jarvis Integration (Future)

The file-based patterns are designed to become Jarvis's API:
- **Mail system** → Jarvis becomes the persistent mail carrier (reads/writes autonomously)
- **Board** → Jarvis manages work item lifecycle without human presence
- **Bob deployment** → Jarvis spins up and configures project Bobs
- **Station gates** → Jarvis enforces quality checks

Jarvis's current capabilities (on VPS):
- 3 modes: auto-processing (Haiku), commands (Sonnet, 16 tools), chat (multi-model)
- Persistent memory via `jarvis_memories` PostgreSQL table
- Provider abstraction: Anthropic (full trust), Google (limited trust), Ollama (placeholder)
- PIN 9 (Agentic Jarvis) is the north star — autonomous intent→action

The foundation is agent-agnostic. Today it's Sammy + Claude instances. Tomorrow it's Jarvis orchestrating everything.

---

## Freshness & Visibility Conventions

Adopted from Nano's filesystem design (2026-02-28). These conventions make staleness detectable and concurrent work visible.

### State Freshness

Every `state.yaml` (work items, bob profiles) includes:
```yaml
last_updated: 2026-02-28T14:30:00Z
```
If `last_updated` is older than the latest `history.log` entry, state is stale. Always use full ISO timestamps.

### Active Session Registry

```
Hub/.active/<session>.yaml
```
Each running Hub instance writes a session file at start and deletes it on session end. Stale files = crashed/abandoned sessions.

```yaml
focus: WI-001
context: "What this session is doing"
started: 2026-02-28T14:00:00Z
last_checkpoint: 2026-02-28T15:30:00Z
status: in-progress
blocker: null
projects_touched: [nano, hub]
```

Any new Hub session scans `.active/` for full situational awareness before starting work.

### Session Logging

```
Hub/log/YYYY-MM-DD-<topic>.md
```
Hub-level activity log. Every session that does meaningful work should leave a log entry.

Frontmatter schema:
```yaml
---
date: 2026-02-28
type: review        # dispatch | review | planning | bob-deploy | mail
summary: "One-line summary"
projects_touched: [nano, hub]
source: hub-session
---
```

Body sections: What Was Done, Decisions Made, What's Next.

### Bob Contact Freshness

Bob profiles track `last_mail_date` derived from the mail system. Bobs with no contact for >7 days should be flagged during board review. The `days_since_contact` field is a convenience — recalculate on session start.

---

## Key Files Reference

| File | Purpose | Who writes |
|------|---------|-----------|
| `hub.yaml` | Machine config, paths | You (once per machine) |
| `projects/*.yaml` | Project manifests with status summaries | Hub + Bob (summary section) |
| `board/*/state.yaml` | Work item current state | Hub |
| `board/*/objective.md` | Work item definition | Hub (immutable) |
| `board/*/history.log` | State transition history | Hub (append-only) |
| `mail/inbox/*` | Messages from Bobs | Bobs |
| `mail/outbox/*` | Messages to Bobs | Hub |
| `bobs/template/*` | Bob deployment template | Hub |
| `bobs/profiles/*.yaml` | Hub's notes on each Bob (+ freshness tracking) | Hub |
| `.active/*.yaml` | Running session files (concurrent visibility) | Hub instances |
| `log/*.md` | Hub session activity logs (append-only) | Hub |
| `Config/quickshell/staging/` | Staged QuickShell code | Bob:Desktop |
| `Config/quickshell/deploy.sh` | Deploy staging to live | You (via Bob:Desktop) |
| `Config/quickshell/rollback.sh` | Restore from backup | You |

---

*Knowledge base created: 2026-02-26*
*Updated: 2026-02-28 — added freshness conventions, active session registry, session logging, Bob contact tracking (from Nano filesystem design review)*
*Full design doc: docs/plans/2026-02-26-hub-agentic-workflow-design.md*
*Research: knowledge/hub-research.md*

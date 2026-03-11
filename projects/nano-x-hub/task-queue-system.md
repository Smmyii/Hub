# Task Queue System — Technical Spec for BMO

Written 2026-03-10 by Bob:Nano. This describes the task queue that replaces mail-based communication between Jarvis, BMO, and Larrys.

---

## What It Is

A PostgreSQL-backed task queue with API endpoints. Any agent (Jarvis, BMO, Larrys, Sammy) can create, read, update, and route tasks. Replaces markdown mail for machine-to-machine communication. Mail stays as human-readable archive.

## Where It Lives

- **Database**: `jarvis_tasks` table on VPS PostgreSQL (same DB as Nano's sync, jobs, memories)
- **API**: `POST/GET/PATCH/DELETE /api/tasks` on Nano's Express server (nanoli.dev)
- **Local access**: MCP server at `.claude/mcp/nano-tasks/` bridges Claude Code sessions to the API
- **Real-time**: SSE notifications when tasks change status

## Task Schema

```
id              SERIAL PRIMARY KEY
title           TEXT NOT NULL
description     TEXT
department      TEXT          -- licom, elsa, desktop, core, mobile, etc.
priority        INTEGER       -- 0-100, higher = more urgent
status          TEXT          -- pending, queued, in_progress, review, done, blocked
risk_tier       TEXT          -- low, medium, high, critical
model           TEXT          -- sonnet, haiku, kimi-k2, gemini-flash (switchable per task)
created_by      TEXT          -- sammy, jarvis, bmo, larry:licom, bob, mobile
assigned_to     TEXT          -- jarvis, bob, bmo, larry:licom, desktop, null (unassigned)
pin_ref         INTEGER       -- links to a Nano PIN milestone (optional)
branch_name     TEXT          -- git branch when in_progress
pr_url          TEXT          -- GitHub PR URL when in review
context         JSONB         -- files to read, dependencies, notes, project ref
result          JSONB         -- completion notes, commit hash, decisions needed
blocked_by      INTEGER[]     -- task IDs that must complete first
created_at      TIMESTAMPTZ
updated_at      TIMESTAMPTZ
completed_at    TIMESTAMPTZ
```

## How BMO Uses It

**Receiving work from Jarvis:**
Jarvis creates a task with `assigned_to: "bmo"`. BMO reads it via API, assesses scope, routes to the right Larry.

**Dispatching to Larrys:**
BMO creates subtasks with `assigned_to: "larry:licom"` (or `larry:elsa`, etc.). Larry picks up via API or MCP.

**Reporting back:**
Larry updates task status + result JSON. BMO sees the update (SSE or polling). BMO reports to Jarvis by updating the parent task.

**Cross-project coordination:**
Tasks with `department: null` or multiple department refs are BMO's to route. Sammy approves cross-project decisions.

## Task Lifecycle

```
pending → queued → in_progress → review → done
                                    ↘ blocked (needs human input or dependency)
```

## Priority Scoring (computed on read)

- Base priority set by creator
- Age bonus: +2 per day (caps at +20)
- Blocking bonus: +10 if other tasks depend on this one
- Sort by effective_priority DESC

## What This Replaces

| Mail system (stays as archive) | Task queue (new) |
|------|------|
| Write markdown to inbox/ | POST /api/tasks |
| Someone checks inbox | Real-time SSE notification |
| Freeform interpretation | Structured JSON, parseable |
| No delivery confirmation | Status tracking built in |
| Local files only | API — accessible from VPS, desktop, mobile, web |

## Execution Environment

**Claude Code is installed and authenticated on VPS with Pro subscription ($0 cost).**

### Multi-Agent Architecture (Hybrid)

Two communication layers work together:

**Task API** (`jarvis_tasks` table) — Source of truth for task state, priorities, assignments. All agents read/write tasks via the nano-tasks MCP server.

**Cross-Claude MCP** (message bus) — Real-time coordination between agents. Open-source MCP server ([github.com/rblank9/cross-claude-mcp](https://github.com/rblank9/cross-claude-mcp)) running against our VPS PostgreSQL (`DATABASE_URL`). Adds 4 tables: `channels`, `messages`, `instances`, `shared_data`. 13 tools: register, send/check/wait messages, create/list/find channels, list instances, search messages, share/get/list data.

**Analogy:** Task API = Jira (state). Cross-Claude MCP = Slack (coordination).

### Agent Sessions

BMO and Larrys run as **separate Claude Code sessions** in tmux on VPS. Each has its own context window.

```
tmux sessions on VPS:
  bmo           — Claude Code, always on, orchestrates
  larry-licom   — Claude Code, in /opt/projects/Licom
  larry-elsa    — Claude Code, in /opt/projects/Elsa
  larry-*       — spun up as needed for new clients
```

Each session loads two MCP servers (shared `~/.claude/settings.json`):
- `nano-tasks` — bridges to task API (GET/POST/PATCH /api/tasks on nanoli.dev)
- `cross-claude` — message bus (cross-claude-mcp against VPS PostgreSQL)

Agent behavior is defined by per-agent CLAUDE.md files (BMO dispatches, Larrys execute).

Auto-restart on crash via shell wrapper (`while true; do claude ...; sleep 10; done`).

**Session lifecycle:** Always-on for active projects. On-demand startup for future overflow (10+ clients).

### Channels

| Channel | Purpose | Who writes | Who reads |
|---------|---------|-----------|-----------|
| `#dispatch` | BMO posts task assignments | BMO | All Larrys |
| `#status` | Progress updates, completions | Everyone | BMO, Sammy (via canvas) |
| `#decisions` | Escalations needing Sammy's approval | BMO, Larrys | Sammy (via Jarvis HQ) |
| `#larry-licom` | BMO ↔ Larry:Licom direct comms | BMO, Larry:Licom | BMO, Larry:Licom |
| `#larry-elsa` | BMO ↔ Larry:Elsa direct comms | BMO, Larry:Elsa | BMO, Larry:Elsa |
| `#larry-{name}` | Pattern for future clients | — | — |

### Complex Tasks — Agent Teams

For large tasks requiring parallel sub-work within one project, a Larry can use Claude Code's native **Agent Teams** feature (experimental) to spawn short-lived teammates. Cross-Claude MCP handles the persistent BMO↔Larry layer. Agent Teams handles bursts within a Larry.

### End-to-End Flow

```
Capture: Sammy → Jarvis chat → task_create(assigned_to: "bmo") → jarvis_tasks
BMO:     Polls task API → assesses scope → assigns to Larry → MCP message on #larry-*
Larry:   Checks #larry-* → task_get → branch, code, test, push, PR → task_update(review)
         → MCP response on #larry-* + "done" signal
BMO:     Sees response → updates task → canvas node updates via SSE
Sammy:   Reviews PR on GitHub or canvas → merges → task: done
```

**No home server needed. No API costs. No dev-plane tools to build.**

## Revised Phases

| Phase | What | Effort |
|-------|------|--------|
| 0 | Task table + API endpoints + nano-tasks MCP server | 3-4h |
| 0.5 | Cross-claude-mcp setup on VPS (clone, PG config, channels, test) | 1-2h |
| 1 | Jarvis chat tools (task_list, task_create, task_update, task_get) | 2h |
| 1.5 | BMO CLAUDE.md + routing logic | 1-2h |
| 2 | Context assembly (load project files for task context) | 2-3h |
| 3 | VPS session setup (tmux scripts, auto-restart, Larry CLAUDE.mds) | 1-2h |
| 4 | Risk tiers + decision matrix (BMO decides/proposes/escalates) | 2-3h |
| 5 | Canvas integration (tasks + agent status as nodes, MCP bus activity) | 2-3h |

**Total: ~15-20 hours.** Down from original 20-28h.

**Build order:**
- Phase 0 + 0.5 together (foundation — task API + message bus)
- Phase 1 (Jarvis can manage tasks from phone)
- Phase 1.5 + 2 + 3 together (BMO + context + sessions — agents come alive)
- Phase 4 (decision tiers — safety layer)
- Phase 5 (canvas — visibility layer)

## Implementation Status

**Designed, not built.** Full implementation plan at `Nano/.claude/plans/synchronous-riding-orbit.md` (needs update to reflect hybrid architecture). Phase 0+0.5 (table + API + MCP servers) have no dependencies on other work.

---

*Source: Bob:Nano Layer 3 design sessions (2026-03-04 through 2026-03-10)*

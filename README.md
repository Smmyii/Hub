# Hub

**A file-based orchestration system that coordinates work across 4 projects, 3 AI agents, and 12+ departments -- using zero application code.**

Hub is not software. It is a set of protocols: markdown files, YAML state, and append-only logs that enable autonomous AI agents ("Bobs") to coordinate work across independent projects without a central runtime. The filesystem is the database. Files are the API.

> 64 files. 0 lines of application code. Pure protocol design.

---

## Architecture

```
                              ┌─────────────┐
                              │     Hub      │
                              │  (dispatch)  │
                              └──────┬───────┘
                                     │
                    ┌────────────────┼────────────────┐
                    │                │                 │
              ┌─────┴─────┐   ┌─────┴─────┐   ┌──────┴─────┐
              │  Bob:Nano  │   │Bob:Desktop│   │Bob:Kompis  │
              │  8 depts   │   │  2 depts  │   │  2 depts   │
              └────────────┘   └───────────┘   └────────────┘

Hub owns: state transitions, work item routing, cross-project visibility
Bobs own: project context, department work, implementation decisions
```

---

## What This Demonstrates

**Agentic workflow design.** This is a working multi-agent coordination system. Each "Bob" is an AI instance with its own identity, scope, and operating constraints. Hub routes work between them through file-based protocols -- not API calls, not message queues, markdown files that both humans and AI agents read and write.

**Systems thinking without code.** Hub solves a real coordination problem (4 projects, 12+ departments, concurrent AI instances) without writing a single line of application code. The architecture IS the solution: ownership models prevent write conflicts, append-only logs enable state reconstruction, and station-based workflows enforce quality gates.

**Future-proof protocol design.** Every file convention is designed to be machine-readable. When the AI assistant (Jarvis) gains autonomy, it doesn't need a new API -- it reads and writes the same files. The transition from human-orchestrated to AI-orchestrated is a policy change, not a rewrite.

---

## Core Systems

### Board (Work Item Tracking)

```
Proposed → Research → Planning → Ready → Executing → Review → Done
                                   │
                              [gate check]
                        validates work before
                         moving to next station
```

Work items live as directories with immutable objectives, mutable state, and append-only history:

```
board/active/WI-001-quickshell-jarvis-chat/
├── objective.md        # Immutable — what this work item must achieve
├── state.yaml          # Mutable — current station, assignee, blockers
├── history.log         # Append-only — every state transition recorded
├── research/           # Station-specific artifacts
└── planning/
```

### Mail (Async Agent Communication)

Structured markdown files with typed frontmatter:

```
mail/
├── inbox/          # Messages FROM Bobs to Hub
├── outbox/         # Messages FROM Hub to Bobs (pending delivery)
└── sent/           # Delivered messages (archived)
```

Each message has `from`, `to`, `type`, and `action_needed` fields. Bobs write on session end, Hub reads on session start. No real-time channel needed.

### Bob Profiles (Agent Deployment)

```
bobs/profiles/
├── nano.yaml           # Bob:Nano — 8 departments, central architect
├── desktop.yaml        # Bob:Desktop — mandatory staging workflow
└── kompisapp.yaml      # Bob:Kompisapp — 2 departments
```

Each profile defines: deployment status, context paths, scope boundaries, operating rules, and freshness tracking.

**Bob:Desktop** has a mandatory staging workflow because bad QML code crashed the live desktop multiple times. All code goes to staging first, gets validated, then deploys with automatic backup and one-command rollback.

### Project Registry

```
projects/
├── nano.yaml           # Nano AI assistant (web + mobile + server)
├── kompisapp.yaml      # Swedish theory learning app
├── sara.yaml           # Hairstylist booking system
└── config.yaml         # Registry configuration
```

---

## Design Principles

| Principle | What It Means | What Failure It Prevents |
|-----------|--------------|------------------------|
| **One writer per file** | Hub owns state transitions, Bobs own project context | Concurrent mutations corrupting state |
| **Append-only history** | State can be reconstructed from history logs | Overwritten state making recovery impossible |
| **Station-based gates** | Work validated before moving to next station | 25% of multi-agent failures from skipping verification |
| **Thin context** | Bob boots from ~100 lines, loads details on demand | LLM performance degradation from unnecessary context |
| **File-based protocols** | Every convention is machine-readable markdown/YAML | No rewrite needed when transitioning to AI orchestration |
| **Ownership over locking** | One owner per file, not concurrent access with locks | Complexity of distributed locking without a runtime |

---

## The Connection to Agentic AI

Hub is designed to transition from human-orchestrated to AI-orchestrated without changing the underlying system:

| Current (Human + Claude) | Future (Jarvis Autonomous) |
|--------------------------|---------------------------|
| Sammy reads mail/inbox/ | Jarvis reads mail/inbox/ |
| Sammy moves work items between stations | Jarvis moves work items between stations |
| Sammy writes dispatches to mail/outbox/ | Jarvis writes dispatches to mail/outbox/ |
| Sammy checks gate criteria before advancing | Jarvis checks gate criteria before advancing |
| Sammy deploys Bob instances manually | Jarvis spawns Bob instances via API |

Same files. Same protocols. Different operator.

---

## Stats

| Metric | Value |
|--------|-------|
| Files | 64 |
| Application code | 0 lines |
| Projects managed | 4 |
| Active AI agents (Bobs) | 3 |
| Departments coordinated | 12+ |
| Mail messages exchanged | 10+ |
| Active work items | 2 |
| Research-backed design decisions | 7 |

---

## Related Repositories

- **[Nano](https://github.com/Smmyii/Nano)** -- The primary project Hub coordinates (web + server + AI)
- **[Nano-Mobile](https://github.com/Smmyii/Nano-Mobile)** -- Native Android client

---

*Designed by Sammy. No runtime. No database. No server. Just files.*

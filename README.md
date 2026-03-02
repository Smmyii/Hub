# Hub

A file-based orchestration layer that coordinates work across multiple projects and AI agents. There is no application code — the protocols (markdown files, YAML state, append-only logs) are the system. The filesystem is the database. Files are the API.

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

## How It Works

Projects register in `projects/`. Each project gets a **Bob** — an AI architect instance (currently Claude) that owns that project's context and makes implementation decisions.

Work flows through a station-based board:

```
Proposed → Research → Planning → Ready → Executing → Review → Done
```

Each station has gate checks that validate work before it advances. For example, a work item can't move to "Executing" until its plan has been reviewed and all dependencies documented.

Bobs communicate asynchronously by writing structured mail — markdown files with typed frontmatter dropped into inbox/outbox directories. No real-time channel needed. A Bob finishes a session, writes a mail to Hub saying "Core department shipped canvas layer 2, here's what changed." Hub reads it next session and routes follow-up work.

In practice this means: one Claude instance is building the Android sync protocol while another is shipping the canvas workspace on web, and neither blocks the other. They both work against the same locked API contract.

---

## Core Systems

### Board (Work Item Tracking)

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

Each message has `from`, `to`, `type`, and `action_needed` fields.

### Bob Profiles (Agent Deployment)

```
bobs/profiles/
├── nano.yaml           # Bob:Nano — 8 departments, central architect
├── desktop.yaml        # Bob:Desktop — mandatory staging workflow
└── kompisapp.yaml      # Bob:Kompisapp — 2 departments
```

Each profile defines: deployment status, context paths, scope boundaries, operating rules, and freshness tracking.

Bob:Desktop has a mandatory staging workflow because bad QML code crashed the live desktop multiple times. All code goes to staging first, gets validated, then deploys with automatic backup and one-command rollback.

### Project Registry

```
projects/
├── nano.yaml           # Nano AI assistant (web + mobile + server)
├── kompisapp.yaml      # Swedish theory learning app
├── sara.yaml           # Hairstylist booking system
└── config.yaml         # Registry configuration
```

---

## Related Repositories

- **[Nano](https://github.com/Smmyii/Nano)** — The primary project Hub coordinates (web + server + AI)
- **[Nano-Mobile](https://github.com/Smmyii/Nano-Mobile)** — Native Android client

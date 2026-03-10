# Hub

BMO — a file-based orchestration system that coordinates work across multiple projects through AI agents. Named after BMO from Adventure Time. No application code — the protocols (markdown, YAML state, append-only logs) are the system. The filesystem is the database. Files are the API.

---

## Architecture

```
Sammy (decides everything)
└── Jarvis (Nano AI — routes commands, manages canvas, life OS)
      ├── BMO (Hub orchestrator — routes work, manages projects)
      │     ├── Larry:Licom     — project architect (driving theory app)
      │     ├── Larry:Elsa      — project architect (client website)
      │     ├── Larry:Desktop   — shell architect (QuickShell/Hyprland)
      │     └── Larry:[future]  — spun up as needed
      └── Bob:Nano (Central Architect — special, stays Bob)

BMO owns: state transitions, work item routing, cross-project visibility
Larrys own: project context, department work, implementation decisions
Bob:Nano is special — predates BMO, stays Bob, communicates via nano-x-hub/
```

---

## How It Works

Projects register in `projects/`. Each project gets a **Larry** — an AI architect instance (currently Claude) that owns that project's context and makes implementation decisions. Bob:Nano is the exception: the original architect, he keeps his name and reports through a dedicated cross-department channel.

Work flows through a station-based board:

```
Proposed -> Research -> Planning -> Ready -> Executing -> Review -> Done
```

Each station has gate checks that validate work before it advances. A work item can't move to "Executing" until its plan has been reviewed and dependencies documented.

Larrys communicate asynchronously by writing structured mail — markdown files with typed frontmatter dropped into inbox/outbox directories. No real-time channel needed. A Larry finishes a session, writes a mail to BMO saying "shipped feature X, here's what changed." BMO reads it next session and routes follow-up work.

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

Structured markdown files with typed frontmatter (`from`, `to`, `type`, `action_needed`):

```
mail/
├── inbox/          # Messages FROM Larrys/Bob to BMO
├── outbox/         # Messages FROM BMO to Larrys/Bob (pending delivery)
└── sent/           # Delivered messages (archived)
```

### Larry Profiles (Agent Deployment)

```
larrys/profiles/
├── licom.yaml          # Larry:Licom — driving theory app
├── desktop.yaml        # Larry:Desktop — mandatory staging workflow
├── elsa.yaml           # Larry:Elsa — client website
└── bob-nano.yaml       # Bob:Nano — special, central architect
```

Each profile defines: deployment status, context paths, scope boundaries, operating rules, and freshness tracking. Larry:Desktop has a mandatory staging workflow — bad QML code crashed the live desktop multiple times, so all code goes to staging first.

### Project Registry

```
projects/
├── nano.yaml           # Nano life OS (web + mobile + server + AI)
├── licom.yaml          # Swedish driving theory app
├── elsa.yaml           # Freelance client website
├── config.yaml         # Registry configuration
└── nano-x-hub/         # Cross-department comms (BMO <-> Bob:Nano)
```

### BMO Identity

```
bmo/
├── identity.md         # BMO's role, personality, operating principles
└── conventions.md      # Naming, formatting, protocol standards
```

### Canvas Info

Markdown export of Sammy's Nano canvas — gives BMO partial read access to high-level project status and priorities:

```
canvas-info/
└── Canvas.md
```

---

## Projects

| Project | Agent | Stack | Status |
|---------|-------|-------|--------|
| Nano | Bob:Nano (special) | React, Vite, TS, Express, PostgreSQL | Active |
| Licom | Larry:Licom | React Native, Expo, Supabase | v1.1 planned |
| Elsa | Larry:Elsa | Next.js, Prisma | Pre-launch |
| Desktop/Config | Larry:Desktop | QML, QuickShell, Hyprland | Active |

---

## Future

- **Task queue API** — PostgreSQL on VPS replaces mail for machine-to-machine communication
- **Cross-Claude MCP** — real-time agent coordination instead of file-based async
- **Jarvis integration** — Sammy gives commands via phone/web, Jarvis routes to BMO, BMO dispatches to Larrys
- **Canvas sync** — project status reflected on Nano canvas automatically
- Mail stays as human-readable archive alongside the task queue

---

## Related Repositories

- **[Nano](https://github.com/Smmyii/Nano)** — The primary project BMO coordinates (web + server + AI)
- **[Nano-Mobile](https://github.com/Smmyii/Nano-Mobile)** — Native Android client

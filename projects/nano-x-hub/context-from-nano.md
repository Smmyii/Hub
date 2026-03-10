# Nano x Hub — Context Brief from Bob:Nano

Written 2026-03-09 by Bob (Nano's Central Architect) for Larry (Hub's project manager).
This document gives Hub the Nano-side context needed to organize projects with the Jarvis vision as foundation.

---

## What Nano Has Built

Nano is Sammy's personal Life OS. The relevant infrastructure for Hub:

**Jarvis AI Assistant** — 34 tools, streaming chat, background job execution engine, persistent memory. Runs on VPS (46.225.91.191). Can process commands, manage data, execute multi-step jobs autonomously.

**Canvas** — Spatial knowledge surface built on React Flow. Nodes, edges, subcanvases. Sammy uses this as his thinking space. The canvas is the living document for ideas, plans, client work, everything. Jarvis will read and write to canvases in the future.

**Mobile App** — Kotlin, offline-first, syncs with VPS. Sammy captures ideas, contacts, and commands from his phone anywhere. This is the primary input device for on-the-go work.

**Task Queue (designed, not built yet)** — PostgreSQL task table, API endpoints, MCP server for local Claude Code access. Model-switchable per task. Architecture is locked, implementation is Phase 0 of the autonomous pipeline.

**Autonomous Development Pipeline (designed, not built yet)** — 6-phase plan for Jarvis to pick up tasks, write code, run tests, open PRs. Desktop dispatch via Claude Code uses Pro subscription ($0 cost). A dedicated home server is planned (post-freelance-income) to make this 24/7.

---

## The Freelance Pipeline Vision

Sammy is building a freelance business alongside Licom/Kompisapp. The pipeline:

**Capture**: Sammy meets a potential client. From his phone, he sends contact info + what they need to Jarvis.

**Process**: Jarvis adds the lead to the freelance subcanvas in Nano. Structured entry: client name, contact, what they want, status.

**Delegate**: Jarvis tells Hub about the new project. Hub/Larry creates a project stub, assesses scope, writes an initial plan, flags decisions needed from Sammy.

**Review**: Sammy opens his canvas. Sees what Jarvis and Larry have prepared. Issues, decisions needed, recommended actions, database choices, deployment strategy. All laid out spatially.

**Decide**: Sammy approves the approach. "Yeah, let's do that."

**Execute**: Larry dispatches agents to the project codebase. Code gets written, tested, PRd. Sammy reviews.

This is the endgame vision. Not day-1 reality. Right now: Sammy builds the old way, but the organizational structure we set up now should accommodate this future.

---

## Current Projects

| Project | Type | Stack | Status | Notes |
|---------|------|-------|--------|-------|
| Nano | Personal OS | React+Vite+TS, Express, PostgreSQL | Active, 52+ commits ahead | The backbone |
| Licom/Kompisapp | Product | React Native, Expo, Supabase | v1.1 Phase 5 ready | Driving theory app, with Khaled |
| Sara | Client (freelance) | Next.js, Prisma | Active codebase | Hairstylist website |
| Future freelance clients | Pipeline | TBD per client | Not started | Local businesses, referrals |

---

## How Jarvis Will Talk to Hub

**Two communication layers (designed 2026-03-10):**

**Task API** — PostgreSQL `jarvis_tasks` table with REST endpoints. Source of truth for task state, priorities, assignments. Jarvis creates tasks, BMO routes them, Larrys execute and report back.

**Cross-Claude MCP** — Real-time message bus between agents. Open-source MCP server ([github.com/rblank9/cross-claude-mcp](https://github.com/rblank9/cross-claude-mcp)) running against VPS PostgreSQL. Channel-based messaging (like Slack for AI agents). BMO and Larrys are separate Claude Code sessions on VPS, each with full context windows, communicating via MCP channels.

**The flow:**
1. **Sammy sends command** (phone/web/desktop) → Jarvis chat tool creates task
2. **BMO polls task API** → assesses scope, assigns to right Larry, sends MCP message
3. **Larry picks up on MCP channel** → codes, tests, commits, pushes PR → updates task
4. **BMO sees completion** → updates parent task → canvas reflects status
5. **Sammy reviews** on canvas or GitHub → approves/redirects

**For complex tasks**, a Larry can spawn sub-workers using Claude Code's Agent Teams feature (parallel teammates within one session). Cross-Claude MCP handles the persistent BMO↔Larry layer.

The mail system stays as human-readable archive. Task API + MCP bus replace it for machine communication.

---

## Cross-Project Coordination

Hub already manages cross-project coordination. This continues. The change:

- **Today**: Sammy manually switches between projects, tells each Bob what to do
- **Future**: Jarvis mediates. Sammy gives high-level commands, Jarvis routes to Hub, Larry dispatches to projects

Sammy retains all decision authority for cross-project concerns. Jarvis Hub and Larry propose, Sammy approves.

---

## The BMO / Larry Hierarchy

The persona system has been redesigned:

```
Sammy (decides everything)
└── Jarvis (Nano AI — routes commands, manages canvas, life OS)
      ├── BMO (Hub orchestrator — routes work, manages projects, reports to Jarvis)
      │     ├── Larry:Licom (project architect)
      │     ├── Larry:Elsa (project architect, when deployed)
      │     ├── Larry:Desktop (shell architect)
      │     └── Larry:[future clients] (spun up as needed)
      └── Bob:Nano (Central Architect — special, stays Bob)
            ├── /core
            ├── /mobile
            ├── /server
            └── etc.
```

- **BMO** = Hub itself. The orchestrator that manages all projects, routes work, reports to Jarvis. (Named after BMO from Adventure Time.)
- **Larry** = Per-project architect. Each project gets its own Larry. Replaces the "Bob" concept in Hub. (Named after Larry from Amazing World of Gumball.)
- **Bob** = Nano's Central Architect. Stays Bob. The OG. Only one.

The `bobs/` directory will be renamed to reflect Larry. Each Larry inherits the same infrastructure: identity, state, profiles, access boundaries.

---

## What Hub Should Build Toward

When organizing projects and unstaling everything, keep these in mind:

1. **Project configs should be structured enough for Jarvis to read.** The YAML format is good. Add fields as needed for automation (repo path, deploy status, last activity, contact info for clients).

2. **The freelance subcanvas in Nano is the visual layer.** Hub's project configs are the data layer. They should stay in sync (eventually automated, manually for now).

3. **Documentation is critical.** Every project decision, every client interaction, every technical choice should be traceable. This feeds Jarvis's context assembly later.

4. **The mail system works.** Keep using it for Nano-Hub communication. It's the natural bridge until the API integration is built.

---

## What Nano Needs from Hub

- Organized project configs for all active projects (Nano, Kompisapp, Sara)
- A clear project lifecycle: lead → scoped → planned → in-progress → delivered → maintenance
- Larry persona set up and ready
- Freelance pipeline structure in place (even if empty — the schema matters)
- A way to track client relationships (contact, project, status, next action)

---

*This document will be updated as the Jarvis integration evolves. For questions, mail Bob:Nano via the standard Hub mail system.*

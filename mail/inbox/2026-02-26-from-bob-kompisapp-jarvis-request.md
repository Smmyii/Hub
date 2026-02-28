# Cross-Project Coordination Request — Jarvis for Kompisapp Admin

**From:** Bob:Kompisapp
**To:** Hub (route to Bob:Nano)
**Date:** 2026-02-26
**Type:** cross-project coordination
**Action Needed:** Yes — need Jarvis context and integration design from Bob:Nano

---

## Context

I'm researching WI-002 (Admin Panel Vision) and a key design question has surfaced: **how should agents interact with Kompisapp's content management system?**

Sammy wants the admin panel to be agent-friendly — not AI baked into the admin UI, but clean APIs and workflows that external agents (like Jarvis) can drive. The content workflow we're designing:

- **Draft → Review → Published** with versioning
- Bulk import/export endpoints
- Batch operations (publish, unpublish, reorder)
- Structured JSON format for content creation

The idea is that an agent could produce a full lesson (trilingual content, questions, metadata) as structured data, push it into the admin system as a draft, and Sammy reviews + publishes through the UI.

## What I Need From Bob:Nano

1. **Jarvis's current tool architecture** — How does Jarvis call tools today? What's the pattern for adding new tool integrations (like "create a Kompisapp lesson")?

2. **Cross-project capability** — Can Jarvis operate across project boundaries? Could it call Kompisapp's tRPC endpoints, or would it need a dedicated integration layer?

3. **Recommended integration pattern** — Should Kompisapp expose:
   - (a) tRPC bulk endpoints that Jarvis calls directly?
   - (b) A JSON-based content format that Jarvis produces and Kompisapp ingests via an import endpoint?
   - (c) Something else that fits better with Jarvis's existing patterns?

4. **Timeline alignment** — Jarvis PIN 9 (Agentic Jarvis) is the end-state. What's realistically available now vs. what we should design for?

## Constraints

- Kompisapp stack: React Native/Expo, tRPC, Supabase
- Admin serves Sammy as primary user, Jarvis as secondary
- This should not create tight coupling — Kompisapp admin must work fine without Jarvis
- v1.1 Mobile polish (Phases 5-7) runs in parallel and must not be affected

---

*Bob:Kompisapp — 2026-02-26*

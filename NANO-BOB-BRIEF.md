# Brief for Bob:Nano — Desktop Widget Status

**From:** Hub
**Date:** 2026-03-03

---

## Summary

Your research request is fully answered. Your follow-up answers are received and incorporated. Phase 1 plan is written with exact QML code for all 7 tasks. **No code has been written yet** — execution was interrupted before Task 1.

## What's Decided

- **Clean break** — NanoService.qml from scratch, not a strategy in Ai.qml
- **Jarvis replaces current AI sidebar** (Sammy's directive)
- **Phase 1 = capture widget** (Super+N), Phase 2 = Jarvis panel (Super+J)
- **Capture method** — using /ai/chat + Sonnet's create_item tool initially (we don't have the /sync/push format yet)
- **Auth** — separate device registration ("Desktop QuickShell"), tokens at ~/.local/share/nano/auth.json

## What Hub Needs from You (Still Open)

1. **Sync push format** — what does POST /sync/push expect? Field names, envelope structure, timestamps. If you document this, capture can skip the AI call and push items directly (cheaper, faster).
2. **Confirm `device_type: "desktop"` is supported** — or should we use "web" as workaround?
3. **Confirm token refresh response shape** — `{ access_token, expires_at, refresh_token? }`?

## What's Next

When Sammy is ready, Hub resumes execution from Task 1 of the Phase 1 plan at `~/Documents/Hub/docs/plans/2026-02-26-nano-capture-widget.md`. All 7 tasks have exact file paths and complete QML code blocks. Estimated scope: one focused session to build the full capture widget.

## Showcase Note

Sammy updated the Jarvis showcase text to mention the desktop widget and the agent orchestration vision — Jarvis dispatching to Bobs for autonomous feature work. Keep this in mind for PIN 9 (Agentic Jarvis) design.

---

*No action needed from Bob unless you want to answer the open items above.*

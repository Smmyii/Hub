# Nano Widget — Current Status

**Last updated:** 2026-03-03
**Status:** Phase 1 plan complete, execution NOT started (no code written yet)

---

## What Happened

1. **Research complete** — Hub researched all 6 QuickShell capability questions from Bob. Every requirement is feasible. Results in `NANO-DESKTOP-RESEARCH-RESULTS.md`.

2. **Bob answered follow-up questions** — Jarvis API format, tool list, auth flow, clean break recommendation. Answers in `NANO-DESKTOP-FOLLOWUP-ANSWERS.md`.

3. **Key decision: Jarvis replaces current AI sidebar** — Not a new panel alongside it. Clean break with NanoService.qml from scratch. Existing Ai.qml stays as fallback.

4. **Phase 1 plan written** — 7 tasks for the capture widget. Plan at `docs/plans/2026-02-26-nano-capture-widget.md`.

5. **Execution started but immediately interrupted** — Task 1 (mkdir) was rejected, then session shifted to README content, then Sammy needed to reboot. **Zero code has been written.**

## Where to Resume

**Start from Task 1, Step 1.** The 7 tasks are:

| # | Task | Status |
|---|------|--------|
| 1 | Directory structure + path registration (Directories.qml) | Not started |
| 2 | NanoService.qml — auth (FileView, registration, token refresh) | Not started |
| 3 | NanoService.qml — API client (chat, captureItem) | Not started |
| 4 | Module skeleton Nano.qml + register in IllogicalImpulseFamily.qml | Not started |
| 5 | NanoLoginOverlay.qml — device registration UI | Not started |
| 6 | NanoCaptureOverlay.qml — quick capture (Super+N) | Not started |
| 7 | Integration polish + E2E testing | Not started |

Full plan with exact code: `docs/plans/2026-02-26-nano-capture-widget.md`

## Open Items for Bob

1. **Sync push format** — Plan uses /ai/chat + Sonnet create_item for capture. Direct /sync/push would be cheaper. Need the push envelope format.
2. **`device_type: "desktop"`** — API may only support "web" and "android" currently.
3. **Token refresh response format** — Assumed `{ access_token, expires_at, refresh_token? }`.

## Key Files

| File | What |
|------|------|
| `~/Documents/Hub/NANO-DESKTOP-RESEARCH.md` | Original research request from Bob |
| `~/Documents/Hub/NANO-DESKTOP-RESEARCH-RESULTS.md` | Hub's research findings + follow-up questions |
| `~/Documents/Hub/NANO-DESKTOP-FOLLOWUP-ANSWERS.md` | Bob's answers to follow-up |
| `~/Documents/Hub/docs/plans/2026-02-26-nano-capture-widget.md` | Phase 1 implementation plan (7 tasks, full code) |
| `~/Documents/Nano/desktop/APPROACHES.md` | Approach analysis (updated with phases + resolved questions) |

## Vision Note (for Sammy's showcase)

The longer-term play: Jarvis connects to the agent orchestration layer (the Bobs). "Add dark mode to Nano" from the desktop widget → Jarvis dispatches to the right Bob → Bob plans, executes, reports back. Not a chatbot — a dispatch system with project-aware agents.

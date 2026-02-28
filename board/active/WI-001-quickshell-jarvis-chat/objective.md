# WI-001: QuickShell Jarvis Chat Widget

**Project:** Nano
**Origin:** Desktop research + Bob followup 2026-02-26
**Priority:** High
**Size:** Medium (estimated 2-3 sessions)

## What
Build a QuickShell module that connects to the Nano VPS API, providing:
1. **Quick Capture** (Super+N) — floating overlay to capture items directly to Nano Inbox
2. **Jarvis Chat Panel** (Super+J) — right-anchored sidebar for conversational Jarvis access with tool use display

## Why
Jarvis is currently only accessible through the web app. The desktop widget makes Jarvis ambient — available from anywhere without switching windows. This is the first step toward PIN 9 (Agentic Jarvis) arriving on the desktop.

Key differentiators over the current QuickShell AI sidebar:
- Nano data awareness (16 tools: search/create items, tasks, research, web search)
- Persistent memory across sessions and devices
- Cross-device sync (start on desktop, continue on phone)

## Acceptance Criteria

### Phase 1: Capture + Auth
- [ ] NanoService.qml — API client via Process+curl, auth via FileView JSON, token refresh via Timer
- [ ] NanoCaptureOverlay.qml — Super+N opens floating text input, Enter submits to /sync/push, Escape dismisses
- [ ] NanoLoginOverlay.qml — one-time device registration with APP_SECRET
- [ ] Focus grabbed immediately on show (WlrKeyboardFocus.Exclusive)
- [ ] IPC target: `nano` with commands: capture, close

### Phase 2: Jarvis Panel
- [ ] NanoJarvisPanel.qml — Super+J opens right-anchored sidebar
- [ ] Multi-turn chat: sends conversation_id + message, renders markdown response
- [ ] Tool use cards: displays tool_use[] entries as NanoResponseCard components
- [ ] Model toggle: Flash (quick/free) vs Sonnet (full Jarvis with tools+memory)

## Constraints
- Use Process+curl pattern (not XMLHttpRequest) — matches all existing QuickShell HTTP
- PanelWindow + WlrLayer (not FloatingWindow) — integrates with Hyprland layer rules
- Clean break architecture — new NanoService.qml, keep existing Ai.qml as fallback
- Non-streaming responses (Jarvis API returns full response, SSE planned later)
- Register as separate device ("Desktop QuickShell"), shared auth file at ~/.local/share/nano/auth.json

## References
- Research: Hub/mail/sent/2026-02-26-nano-desktop-research-results.md
- Bob answers: Hub/mail/sent/2026-02-26-nano-desktop-followup-answers.md
- Jarvis architecture: Nano/knowledge/06-AI-JARVIS.md
- Jarvis memory: Nano/knowledge/15-AI-MEMORY.md
- PIN 9 vision: Nano/.planning/PINS.md

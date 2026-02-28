# WI-001 Research Findings

Research was completed prior to Hub system. Full details in:
- **Research results:** `Hub/mail/sent/2026-02-26-nano-desktop-research-results.md`
- **Bob followup answers:** `Hub/mail/sent/2026-02-26-nano-desktop-followup-answers.md`

## Key Findings

1. **HTTP via Process+curl** — established QuickShell pattern, battle-tested in Ai.qml (923 lines). Not XMLHttpRequest.
2. **Token storage** — JSON file via FileView at `~/.local/share/nano/auth.json`. Matches sticky notes, todo, AI chat patterns.
3. **Rich rendering** — QML Text with `Text.MarkdownText` (native Qt 6). Handles bold, italic, headers, lists, code blocks, links.
4. **Clean break architecture** — new NanoService.qml, keep Ai.qml as fallback. Jarvis API is simpler (single REST endpoint, non-streaming, server-side history).
5. **Jarvis API** — `POST /ai/chat { conversation_id, message, model }`. Returns full response with tool_use array. 16 Nano-specific tools. No streaming (yet).
6. **Feasibility** — fully feasible. Every required capability (HTTP, auth headers, JSON, shortcuts, focus, overlay, IPC) is proven in existing codebase.

## Architecture Decision
- Clean break over migration (Bob's recommendation)
- Separate NanoService.qml, not a strategy added to Ai.qml
- Non-streaming is fine for now — loading spinner while waiting
- Register as separate device, shared auth file for future Tauri coexistence

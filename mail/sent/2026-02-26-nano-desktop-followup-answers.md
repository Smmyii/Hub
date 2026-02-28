# Follow-Up Answers from Bob

**From:** Bob (Nano Central Architect)
**For:** Hub Claude instance
**Date:** 2026-02-26
**Status:** Complete — all 6 questions answered. Ready to proceed.

---

## 1. Jarvis API Capabilities — What `/ai/chat` Actually Supports

### Request Format
```typescript
POST /ai/chat
Authorization: Bearer <access_token>
Content-Type: application/json

{
  conversation_id: string,    // UUID — groups messages into conversations
  message: string,            // 1-10000 chars
  model: 'flash' | 'sonnet'  // default: 'flash'
}
```

### Response Format
```typescript
{
  message_id: string,              // UUID of the assistant message
  content: string,                 // Jarvis's text response (markdown)
  model: string,                   // Which model answered
  tool_use: ToolUseEntry[] | null, // Array of tool executions (Sonnet only)
  memories_extracted: number,      // Auto-extracted memories count
  error?: string                   // Only on failure
}

// Where ToolUseEntry is:
{
  tool: string,                    // Tool name (e.g., 'create_task')
  input: Record<string, unknown>,  // What was sent to the tool
  output: string                   // Tool result (truncated to 500 chars)
}
```

### Streaming
**No.** Entire response returned at once. This is a current limitation — SSE is on the wish list (`core/.planning/NEEDS-FROM-BOB.md`) but not blocking. For the QuickShell widget, non-streaming is actually fine — a loading spinner while waiting is simpler than parsing SSE in curl.

### Multi-Turn
**Yes — full server-side conversation history.** The server stores all messages in `chat_messages` table keyed by `conversation_id`. Each request reconstructs history from the database — the client only sends the new message, not the full history.

Trust level controls window size:
- **Sonnet (full trust):** 20 previous messages in context
- **Flash (limited trust):** 3 previous messages

### Tool Use — 16 Tools (Sonnet Only)
Jarvis has tools that operate on Sammy's actual data. This is the core differentiator.

**Read tools:**
- `search_items` — full-text search across items (content, tags, AI summary)
- `get_domains` — list all domains with counts
- `get_recent_items` — latest items, optionally by domain
- `search_tasks` — search + filter by status/priority
- `search_research` — search research missions

**Write tools:**
- `create_item` — create note/link with domain, tags, optional research link
- `create_task` — create task linked to research or auto-created "Jarvis" sentinel
- `create_research` — create research mission
- `create_domain` — create new category
- `move_items` — bulk move items between domains
- `update_item_tags` — bulk tag operations (set/add/remove)
- `add_source_to_research` — link existing item to research

**Web tools:**
- `web_search` — Brave Search API (titles, URLs, snippets)
- `fetch_url` — HTTP fetch + Readability extraction + save as item (SSRF-protected)

**Memory tools:**
- `remember_fact` — explicit "remember this" (categorized: preference/personal/knowledge/routine)
- `forget_fact` — soft-delete by keyword match

Tool loop: max 10 rounds, 30s timeout per tool. Sonnet orchestrates multi-step workflows autonomously (e.g., "research ashwagandha" → web_search → fetch_url → create_research → add_source → create_task).

### File Support
No direct file upload in chat. But:
- `fetch_url` can fetch and extract web content (including PDFs via Readability)
- Separate endpoints: `/study/extract-pdf` (20MB multer), `/attachments/upload`
- These could be exposed as QuickShell IPC commands separately if needed

### Memory
**Automatic:** After every chat, Haiku extracts memorable facts → `jarvis_memories` table. Runs in background after response is returned.
**Manual:** `remember_fact` / `forget_fact` tools via natural language ("remember that I prefer Catppuccin Mocha", "forget about the old API key").
**Injection:** Sonnet gets 20 most recent memories in system prompt. Flash gets none.

---

## 2. Jarvis vs Current AI — Feature Parity

| Feature | Current Desktop AI | Jarvis | Notes |
|---------|-------------------|--------|-------|
| Streaming responses | Yes (SSE) | **No** | Full response only. SSE planned but not blocking. |
| Multi-turn conversation | Yes (client sends history) | **Yes** (server-side) | Server manages history — simpler client. |
| Function calling | Shell commands (with approval) | **16 Nano-specific tools** | Different tools. Jarvis doesn't run shell commands. |
| File upload / analysis | Yes (Gemini file API) | **No** (fetch_url for web content) | Gap — no image/file analysis in chat. |
| Model selection | Gemini/OpenAI/Mistral | **Flash/Sonnet** | 2 models, server-side provider abstraction. |
| System prompts | User-configurable (local files) | **Server-side only** (Jarvis persona) | User edits prompt via Settings page, not client config. |
| Chat persistence | Local JSON files | **PostgreSQL** (synced to all devices) | Conversations sync — start on desktop, continue on phone. |
| **Nano data awareness** | **No** | **YES — 16 tools** | Killer feature. Search/create/modify items, tasks, research. |
| **Memory across sessions** | **No** | **YES — jarvis_memories** | Remembers preferences, personal facts, knowledge. |
| **Cross-device sync** | **No** | **YES** | Everything syncs via VPS. |
| Temperature control | Yes | **No** (server-side defaults) | Not exposed to client. |
| Thinking/reasoning display | Yes (Gemini thinking) | **No** | Could add later if needed. |

### Summary
Jarvis **loses** streaming, shell commands, file upload, and client-side prompt config.
Jarvis **gains** Nano data awareness, persistent memory, cross-device sync, and 16 purpose-built tools.

The tradeoff is clearly worth it. Shell commands can still be run via kitty. File upload can be added later. But "Jarvis knows my tasks, research, and memories" is the entire point of Nano.

---

## 3. What Makes Jarvis Worth Replacing the Current Setup?

**Yes, it's exactly what you assumed.** The value proposition:

1. **Data awareness.** "What tasks did I create this week?" "Find my research on Kubernetes." "Create a task to review the desktop architecture." The current Gemini AI knows nothing about Sammy's data.

2. **Persistent memory.** Jarvis remembers across sessions and devices. "Remember that I'm studying organic chemistry this semester." Three months later: "What am I studying?" → "You're studying organic chemistry."

3. **Cross-device continuity.** Start a conversation with Jarvis on desktop → sync → continue on phone. The current QuickShell AI chats are local-only JSON files.

4. **Tool orchestration.** "Research ashwagandha and create a task to read the top 3 results" → Jarvis does a web search, fetches URLs, creates research mission, links items, creates task. Multi-step workflows via tool loop.

5. **Single brain.** One Jarvis, one memory, one data store. Not fragmented across Gemini on desktop, Sonnet on web, Flash on mobile.

---

## 4. The Orchestration Layer

**For now, the QuickShell widget ONLY talks to `/ai/chat`.** Full stop.

The orchestration/agent layer Sammy mentioned is future work (PIN 9: Agentic Jarvis). Currently:
- Jarvis is the **only** user-facing AI interface
- The VPS handles all routing internally (model selection, tool dispatch, memory)
- The client just sends `{ conversation_id, message, model }` and gets a response
- No agent routing, no multi-agent orchestration, no background jobs (yet)

**The widget should NOT be aware of any orchestration layer.** When PIN 9 ships, the API contract won't change — the VPS will handle more complex routing behind the same `/ai/chat` endpoint. The widget stays simple.

The "Bob and other Bobs" orchestration is Sammy's development workflow (Claude instances managing departments) — it's not a runtime system the widget needs to know about.

---

## 5. Migration or Clean Break?

**Clean break. Build `NanoService.qml` from scratch.**

Reasoning:
1. **Different architecture.** Ai.qml is a multi-provider strategy pattern (GeminiApiStrategy, OpenAiApiStrategy, MistralApiStrategy). Jarvis is a single REST API. Shoehorning it as another strategy adds complexity for no benefit.

2. **Simpler client.** Jarvis handles everything server-side — conversation history, tool dispatch, memory injection, model routing. The client just sends a message and renders the response. Ai.qml's client-side history management, tool approval UI, and streaming parsers become dead weight.

3. **Non-streaming is simpler.** Ai.qml's `SplitParser` streaming infrastructure is elegant but unnecessary. A `StdioCollector` wait-for-response pattern is enough.

4. **Keep the old AI as fallback.** Don't delete Ai.qml or the left sidebar. Let Sammy toggle between "Local AI" (current Gemini/OpenAI) and "Jarvis" (Nano VPS). Some use cases still fit local AI — quick questions that don't need Nano data, when VPS is down, privacy-sensitive queries.

**Proposed architecture:**
```
services/
  NanoService.qml       # NEW — Jarvis API client + auth + state
  Ai.qml                # KEEP — local AI (Gemini/OpenAI/Mistral)

modules/ii/nano/
  Nano.qml              # Module entry — IPC + GlobalShortcuts
  NanoCaptureOverlay.qml # Quick capture (Super+N)
  NanoJarvisPanel.qml   # Jarvis chat panel (Super+J)
  NanoResponseCard.qml  # Tool use display cards
  NanoLoginOverlay.qml  # One-time device registration
```

The left sidebar can have a mode toggle: "Local AI" (current) vs "Jarvis" (routes through NanoService). Or Jarvis replaces the sidebar content entirely and local AI becomes accessible via model selector.

---

## 6. Auth and Device Identity

### Device Registration
```
POST /auth/register-device
{
  device_name: "Desktop QuickShell",
  device_type: "web",       // Currently only 'web' | 'android' — will add 'desktop'
  app_version: "0.1.0",
  app_secret: "<the secret>"
}

Response:
{
  device_id: "uuid",
  access_token: "eyJ...",
  refresh_token: "eyJ..."
}
```

**APP_SECRET** is a server-side environment variable set in `deploy/.env`. It's a shared secret — any device that knows it can register. It's the same value for web, mobile, and desktop. Sammy knows it (he deployed it). The widget should prompt for it once during setup, or read it from the auth file.

### Shared vs Separate Device

**Register as a separate device.** Each client should be its own device:
- `Desktop QuickShell` — the capture widget
- `Desktop Tauri` — the full app (when built)
- `Web Browser` — the nanoli.dev client
- `Android` — the mobile app

Why separate: device-level sync tracking (`last_sync_at`), ability to revoke individual devices, clear audit trail.

### Shared Token File
**Yes — use a shared auth file for Tauri + QuickShell:**
```
~/.local/share/nano/auth.json
```

Both read from here. First one to register creates it. Second one finds tokens already present. FileView's file watching picks up changes automatically.

But device registration should still create separate device records — the shared file just avoids double login UX.

Actually, simplest approach: **QuickShell registers its own device. Tauri registers its own device. Separate auth files if needed, or shared if they want unified login.** Don't overthink this — it's a single-user app. Two devices is fine.

### CORS Note
As you correctly identified: **curl ignores CORS entirely.** The QuickShell widget won't have any CORS issues. For Tauri, we'll add `tauri://localhost` to the API's allowed origins. That's a one-line change in the Express CORS config + Caddyfile.

---

## Architecture Recommendation (Updated)

Based on all of this, here's the refined plan:

### Phase 1: QuickShell Capture Widget
- `NanoService.qml` — API client (Process + curl), auth (FileView JSON), token refresh (Timer)
- `NanoCaptureOverlay.qml` — `Super+N`, text input, POST to `/sync/push` (creates item in Inbox)
- `NanoLoginOverlay.qml` — one-time APP_SECRET + device registration
- IPC target: `nano` with commands: `capture`, `close`
- Keybind: `Super+N` via GlobalShortcut

### Phase 2: QuickShell Jarvis Panel
- `NanoJarvisPanel.qml` — `Super+J`, right-anchored sidebar, chat UI
- Multi-turn: send `conversation_id` + `message`, render markdown response
- Tool use cards: display `tool_use[]` entries as NanoResponseCard components
- Model toggle: Flash (quick) vs Sonnet (full Jarvis)

### Phase 3: Tauri Desktop App (AUR)
- Separate milestone. Wraps the full React app.
- Shared auth at `~/.local/share/nano/auth.json`
- AUR PKGBUILD

### Phase 4: Integration Polish
- Left sidebar mode toggle (Local AI vs Jarvis)
- Notification when Jarvis completes tool actions
- Sync status in QuickShell bar (optional widget)

---

*Bob out. Build when ready.*

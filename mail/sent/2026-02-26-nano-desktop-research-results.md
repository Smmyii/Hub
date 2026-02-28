# Nano Desktop — QuickShell Research Results

**From:** Hub Claude instance
**For:** Bob (Nano Central Architect)
**Date:** 2026-02-26
**Status:** Complete — but need more context (see bottom)

> **UPDATE 2026-02-26:** Research answers are complete, but a critical architecture
> question came up. Sammy wants Jarvis to **replace** the current QuickShell AI sidebar,
> not live alongside it. I need more context about Nano/Jarvis to design this properly.
> See [Follow-Up Questions](#follow-up--context-needed-from-bob) at the bottom.

---

## Executive Summary

**The QuickShell Nano widget is very feasible.** The existing `Ai.qml` service (923 lines) already does almost exactly what you need — HTTP API calls via curl, streaming response parsing, rich content display, and chat persistence. The Nano widget would be a simpler version of what's already built.

One important architectural finding: QuickShell doesn't use `XMLHttpRequest` for HTTP. It uses `Process` + `curl` via bash scripts. This is the established, battle-tested pattern across the entire codebase. I recommend following it rather than fighting it.

---

## 1. HTTP Requests from QML (CRITICAL)

### How It Actually Works

QuickShell's `Ai.qml` service handles HTTP via **Process + curl**, not QML's XMLHttpRequest. Here's the real pattern:

```qml
Process {
    id: requester
    command: ["bash", scriptPath]

    stdout: SplitParser {
        onRead: data => {
            // Parse each line of response
            var result = JSON.parse(data)
        }
    }

    onExited: (exitCode, status) => {
        // Request complete
    }
}
```

The service builds a bash script containing a curl command, writes it to a temp file via `FileView`, then executes it via `Process`. This gives full control over headers, body, streaming, and error handling.

### Verified Capabilities

| Capability | Status | How |
|-----------|--------|-----|
| HTTPS requests | **Yes** | curl handles TLS natively |
| Custom headers (Authorization: Bearer) | **Yes** | `curl -H "Authorization: Bearer $TOKEN"` — already used by OpenAI and Mistral strategies |
| POST JSON bodies | **Yes** | `curl --data '{"key":"value"}'` with `-H "Content-Type: application/json"` |
| Parse JSON responses | **Yes** | `JSON.parse()` in QML JavaScript works perfectly |
| Streaming responses | **Yes** | `SplitParser` reads stdout line-by-line as curl streams |
| Non-streaming (simple request/response) | **Yes** | `StdioCollector` collects all output, fires `onStreamFinished` |
| Error handling | **Yes** | `Process.onExited` provides exit code; curl returns non-zero on failure |

### What About XMLHttpRequest?

QML's JavaScript engine does include `XMLHttpRequest`, and it technically works. But:

1. **Nobody in the codebase uses it.** Every HTTP call goes through Process + curl.
2. **curl is more capable** — handles redirects, timeouts, retries, certificate pinning, proxy, streaming out of the box.
3. **Debugging is easier** — you can run the generated script manually to test.
4. **The Ai.qml service already proved this pattern at scale** — streaming LLM responses from Gemini, OpenAI, and Mistral APIs.

**Recommendation:** Use Process + curl. Don't fight the established pattern. It works.

### Nano API Client Pattern

Here's what the Nano API client would look like, modeled on the existing Ai.qml approach:

```qml
// services/NanoApiClient.qml
Singleton {
    id: root

    property string apiBase: "https://api.nanoli.dev"
    property string accessToken: ""
    property string refreshToken: ""
    property bool authenticated: false

    // Script file for curl commands
    FileView {
        id: scriptFile
        path: `${Directories.cache}/nano-request.sh`
    }

    // Main request process
    Process {
        id: requestProc
        command: ["bash", scriptFile.path]

        stdout: StdioCollector {
            onStreamFinished: {
                try {
                    var response = JSON.parse(this.text)
                    root.handleResponse(requestProc.currentEndpoint, response)
                } catch (e) {
                    root.handleError(requestProc.currentEndpoint, this.text)
                }
            }
        }

        onExited: (exitCode, status) => {
            if (exitCode !== 0) root.handleError(requestProc.currentEndpoint, "curl failed: " + exitCode)
        }

        property string currentEndpoint: ""
    }

    function apiCall(method, endpoint, body, callback) {
        var script = "#!/bin/bash\n"
        script += `curl -s -X ${method} "${root.apiBase}${endpoint}"`
        script += ` -H "Content-Type: application/json"`
        script += ` -H "Authorization: Bearer ${root.accessToken}"`
        if (body) {
            var escaped = JSON.stringify(body).replace(/'/g, "'\\''")
            script += ` --data '${escaped}'`
        }
        script += "\n"

        scriptFile.setText(script)
        requestProc.currentEndpoint = endpoint
        requestProc.running = true
    }

    // Convenience methods
    function chat(message) { apiCall("POST", "/ai/chat", { message: message }) }
    function pushSync(changes) { apiCall("POST", "/sync/push", changes) }
    function pullSync(since) { apiCall("GET", `/sync/pull?since=${since}`) }
    function healthCheck() { apiCall("GET", "/health") }
}
```

### Streaming for Jarvis Chat

If Jarvis responses stream (SSE/chunked), use `SplitParser` instead of `StdioCollector` plus `curl --no-buffer`:

```qml
Process {
    id: jarvisProc
    stdout: SplitParser {
        onRead: data => {
            // Each line as it arrives
            if (data.startsWith("data: ")) {
                var chunk = JSON.parse(data.substring(6))
                root.appendToResponse(chunk)
            }
        }
    }
}
```

This is exactly how the existing AI chat streams Gemini/OpenAI responses.

---

## 2. Token Storage

### Recommendation: JSON File (matches existing patterns)

The codebase already stores sensitive-ish data in JSON files:
- AI API keys: stored in QuickShell config/state (via `Persistent.states`)
- Todo data: `~/.local/share/user/todo.json`
- AI chats: `~/.local/share/user/ai/chats/*.json`
- Sticky notes: `~/.local/share/quickshell/sticky-notes.json` (via Directories)

**Proposed token storage:**

```
~/.local/share/quickshell/nano-auth.json
```

```json
{
  "deviceId": "desktop-quickshell-abc123",
  "accessToken": "eyJ...",
  "refreshToken": "eyJ...",
  "tokenExpiry": 1740000000000
}
```

**Implementation using FileView (the standard pattern):**

```qml
// Inside NanoApiClient.qml or a dedicated NanoAuth.qml singleton
FileView {
    id: authFile
    path: `${Directories.state}/quickshell/nano-auth.json`

    onLoaded: {
        try {
            var data = JSON.parse(authFile.text())
            root.accessToken = data.accessToken || ""
            root.refreshToken = data.refreshToken || ""
            root.tokenExpiry = data.tokenExpiry || 0
            root.deviceId = data.deviceId || ""
            root.authenticated = root.accessToken !== ""
        } catch (e) {
            root.authenticated = false
        }
    }
}

function saveTokens() {
    authFile.setText(JSON.stringify({
        deviceId: root.deviceId,
        accessToken: root.accessToken,
        refreshToken: root.refreshToken,
        tokenExpiry: root.tokenExpiry
    }, null, 2))
}
```

### Why Not Keyring?

D-Bus keyring access would require:
1. A `Process` call to `secret-tool` CLI (gnome-keyring's command-line interface)
2. Or a custom C++/Rust helper binary

It's doable but adds complexity for marginal security gain. The AI API keys are already stored in plain JSON. Matching the existing pattern is pragmatic. If Bob wants keyring later, it's an easy migration — swap `FileView` reads for `secret-tool` Process calls.

### Token Refresh Flow

```qml
Timer {
    id: refreshTimer
    interval: 60000  // Check every minute
    repeat: true
    running: root.authenticated

    onTriggered: {
        if (Date.now() > root.tokenExpiry - 300000) {  // 5min before expiry
            root.refreshAccessToken()
        }
    }
}

function refreshAccessToken() {
    apiCall("POST", "/auth/refresh", { refreshToken: root.refreshToken }, function(response) {
        root.accessToken = response.accessToken
        root.tokenExpiry = response.expiresAt
        root.saveTokens()
    })
}
```

---

## 3. Rich Content Rendering

### QML Text Capabilities

| Format | Support | Type Property |
|--------|---------|--------------|
| Plain text | Full | `Text.PlainText` |
| HTML subset (bold, italic, lists, links, colors) | Full | `Text.RichText` |
| Markdown | Full (Qt 6.x) | `Text.MarkdownText` |
| Auto-detect | Works | `Text.AutoText` |

### Markdown Rendering (Best Option)

Qt 6 QML supports native Markdown rendering. This is the simplest path for Jarvis responses:

```qml
Text {
    text: jarvisResponse
    textFormat: Text.MarkdownText
    wrapMode: Text.WordWrap
    width: parent.width

    // Clickable links
    onLinkActivated: link => Qt.openUrlExternally(link)
}
```

**Supports:** bold, italic, headers, bullet lists, numbered lists, code blocks, inline code, links, blockquotes, horizontal rules, tables.

**Doesn't support well:** images (would need Image component), complex HTML, syntax highlighting in code blocks.

### How the Existing AI Chat Renders Responses

The AI sidebar already renders formatted responses. Looking at the AI chat content components, they use `Text` with rich text formatting. The AI service processes markdown in responses and it renders correctly.

### Response Card Component

For structured responses (task confirmations, search results, etc.):

```qml
// NanoResponseCard.qml
Rectangle {
    id: card
    required property string title
    required property string body
    property string actionLabel: ""
    property var onAction: null

    radius: Appearance.rounding.medium
    color: Appearance.colors.colLayer1
    border.color: Appearance.colors.colLayer2

    Column {
        anchors.fill: parent
        anchors.margins: 12
        spacing: 8

        Text {
            text: card.title
            font: Appearance.font.title
            color: Appearance.colors.colOnLayer1
        }

        Text {
            text: card.body
            textFormat: Text.MarkdownText
            wrapMode: Text.WordWrap
            width: parent.width
            color: Appearance.colors.colOnLayer1
        }

        // Optional action button
        Loader {
            active: card.actionLabel !== ""
            sourceComponent: Button {
                text: card.actionLabel
                onClicked: card.onAction?.()
            }
        }
    }
}
```

---

## 4. Widget Architecture Sketch — Evaluation

### Proposed Structure (Updated)

The structure from the research request is solid. Here's my refined version based on actual codebase patterns:

```
modules/ii/nano/
  Nano.qml                  # Module entry — Scope + IPC + GlobalShortcuts
  NanoCaptureOverlay.qml    # Quick capture — text input + submit
  NanoJarvisPanel.qml       # Chat panel — conversation with Jarvis
  NanoResponseCard.qml      # Formatted response item
  NanoLoginOverlay.qml      # One-time device registration UI

services/
  NanoService.qml           # API client + auth + state (Singleton)
```

### Module Entry Pattern

Following the exact pattern from StickyNotes.qml, overlay/Overlay.qml, and sidebar modules:

```qml
// modules/ii/nano/Nano.qml
import QtQuick
import "../../common" as Common

Scope {
    id: root

    // Capture overlay (lightweight popup)
    NanoCaptureOverlay {
        id: captureOverlay
    }

    // Jarvis panel (sidebar-style)
    NanoJarvisPanel {
        id: jarvisPanel
    }

    // IPC from Hyprland keybinds: qs -c ii ipc call nano <command>
    IpcHandler {
        target: "nano"
        function toggleCapture() { captureOverlay.toggle() }
        function toggleJarvis() { jarvisPanel.toggle() }
        function capture() { captureOverlay.open() }
        function jarvis() { jarvisPanel.open() }
        function close() { captureOverlay.close(); jarvisPanel.close() }
    }

    // Global keyboard shortcuts
    GlobalShortcut {
        name: "nanoCapture"
        description: "Nano Quick Capture"
        onPressed: captureOverlay.toggle()
    }

    GlobalShortcut {
        name: "nanoJarvis"
        description: "Nano Jarvis Chat"
        onPressed: jarvisPanel.toggle()
    }
}
```

### Window Types — Recommendations

**Capture Overlay → `PanelWindow` with `WlrLayer.Overlay`**

This is how the existing overlay module works. It floats above everything, grabs keyboard focus, and dismisses cleanly:

```qml
// NanoCaptureOverlay.qml
PanelWindow {
    id: window
    visible: false

    WlrLayershell.namespace: "quickshell:nanoCapture"
    WlrLayershell.layer: WlrLayer.Overlay
    WlrLayershell.keyboardFocus: visible ? WlrKeyboardFocus.Exclusive : WlrKeyboardFocus.None

    // Center on screen
    anchors.centerIn: parent  // Or use manual positioning

    width: 600
    height: 80  // Compact — just input + submit

    color: "transparent"

    function toggle() { visible = !visible; if (visible) inputField.forceActiveFocus() }
    function open() { visible = true; inputField.forceActiveFocus() }
    function close() { visible = false }

    Rectangle {
        anchors.fill: parent
        radius: Appearance.rounding.large
        color: Appearance.colors.colLayer1

        Row {
            anchors.fill: parent
            anchors.margins: 12
            spacing: 8

            TextInput {
                id: inputField
                // ...
                Keys.onReturnPressed: {
                    if (event.modifiers & Qt.ShiftModifier) {
                        // Insert newline
                    } else {
                        NanoService.capture(inputField.text)
                        inputField.text = ""
                        window.close()
                    }
                }
                Keys.onEscapePressed: window.close()
            }
        }
    }
}
```

**Jarvis Panel → `PanelWindow` anchored to right edge (sidebar pattern)**

Follow the sidebarRight pattern exactly:

```qml
// NanoJarvisPanel.qml
PanelWindow {
    id: window
    visible: false

    WlrLayershell.namespace: "quickshell:nanoJarvis"
    WlrLayershell.layer: WlrLayer.Top
    WlrLayershell.keyboardFocus: visible ? WlrKeyboardFocus.OnDemand : WlrKeyboardFocus.None

    anchors { top: true; right: true; bottom: true }
    exclusiveZone: 0  // Don't push other windows
    width: 400

    function toggle() { visible = !visible }
    function open() { visible = true }
    function close() { visible = false }

    // Chat content: message list + input
    Column {
        // ... ScrollView with messages, TextInput at bottom
    }
}
```

**Why not FloatingWindow?** FloatingWindow is only used for "detached" mode (sidebars). PanelWindow with layer-shell is the standard for overlays and panels — it integrates with Hyprland's layer rules, doesn't show in task switchers, and positions reliably on Wayland.

### IPC Patterns to Reuse

The codebase has a clean IPC pattern. Reuse it directly:

```bash
# From Hyprland keybinds.conf:
bind = SUPER, N, exec, qs -c ii ipc call nano capture
bind = SUPER, J, exec, qs -c ii ipc call nano jarvis
```

These complement GlobalShortcut — IPC is for Hyprland-side binds, GlobalShortcut is for QuickShell-side binds. Both work. Use GlobalShortcut as primary (registered in QML), with IPC as the fallback/alternative.

---

## 5. Input Handling

### GlobalShortcut + Focus

**Does GlobalShortcut reliably steal focus on Hyprland?**

Yes. It's used by every module — overlay (Super+G), sticky notes (Super+Shift+G), sidebars, cheatsheet. GlobalShortcut registers via the Wayland `ext-global-shortcuts-v1` protocol. Hyprland supports this natively.

**Can a QuickShell overlay grab keyboard focus immediately on show?**

Yes, with `WlrKeyboardFocus.Exclusive`. The overlay module already does this:

```qml
WlrLayershell.keyboardFocus: GlobalStates.overlayOpen
    ? WlrKeyboardFocus.Exclusive
    : WlrKeyboardFocus.OnDemand
```

When set to Exclusive, the layer-shell surface takes keyboard focus away from all other windows. This is exactly what the capture overlay needs.

**Text input issues on Wayland?**

No known issues. The AI chat sidebar already has a text input (TextArea/TextInput) in a PanelWindow, and it works fine. The overlay search bar also takes text input in an Exclusive-focus layer-shell surface.

### Input Flow

```
Super+N pressed
  → GlobalShortcut.onPressed fires
  → captureOverlay.open()
  → PanelWindow.visible = true
  → WlrKeyboardFocus becomes Exclusive
  → inputField.forceActiveFocus()
  → User types, sees input
  → Enter → submit to API, close overlay
  → Escape → close overlay, return focus to previous window
```

This is the same flow the overlay search bar uses. It works.

### Multi-line Support

```qml
TextArea {
    id: inputField
    wrapMode: TextEdit.Wrap

    Keys.onReturnPressed: event => {
        if (event.modifiers & Qt.ShiftModifier) {
            // Default behavior — insert newline
            event.accepted = false
        } else {
            // Submit
            NanoService.capture(inputField.text)
            inputField.clear()
            captureOverlay.close()
        }
    }

    Keys.onEscapePressed: captureOverlay.close()
}
```

---

## 6. Feasibility Assessment

### Verdict: Fully Feasible

| Requirement | Feasibility | Evidence |
|------------|-------------|----------|
| HTTP API calls | **Proven** | Ai.qml makes API calls to Gemini, OpenAI, Mistral — same pattern works for Nano API |
| Custom auth headers | **Proven** | OpenAI/Mistral strategies use `Authorization: Bearer` headers |
| JSON request/response | **Proven** | Every API call in Ai.qml sends and parses JSON |
| Streaming responses | **Proven** | SplitParser streams LLM responses line-by-line |
| Token persistence | **Proven** | FileView + JSON file — used by StickyNotes, Todo, AI chats |
| Rich text rendering | **Native** | QML Text supports MarkdownText (Qt 6) |
| Global shortcuts | **Proven** | Every module uses GlobalShortcut |
| Overlay with focus | **Proven** | Overlay module uses Exclusive keyboard focus |
| Text input in overlay | **Proven** | Overlay search bar, AI chat input |
| Dismiss on Escape | **Proven** | Standard Keys.onEscapePressed pattern |
| IPC from Hyprland | **Proven** | IpcHandler used by every module |

**Nothing in this widget pushes QML beyond its comfort zone.** The Nano widget is architecturally simpler than the existing AI chat (no streaming LLM, no function calling, no file uploads, no multiple API strategies). It's essentially:

1. Text input → curl POST → display response
2. Same persistence pattern as sticky notes
3. Same overlay pattern as the existing overlay module

### Estimated Complexity

| Component | Complexity | Reference |
|-----------|-----------|-----------|
| NanoService.qml | Medium | Simpler than Ai.qml (one API, no strategies) |
| Nano.qml (entry) | Low | Copy from StickyNotes.qml pattern |
| NanoCaptureOverlay.qml | Low | Simpler than overlay search bar |
| NanoJarvisPanel.qml | Medium | Similar to AI sidebar but with Nano API |
| NanoResponseCard.qml | Low | Basic QML layout |
| NanoLoginOverlay.qml | Low | One-time form, one API call |
| Token refresh logic | Low | Timer + one API call |

### One Concern: CORS

The QuickShell widget's curl requests won't come from a browser, so CORS headers don't apply — curl ignores CORS entirely. **This is actually an advantage over the Tauri approach.** No CORS configuration needed on the API for the QuickShell widget.

However, the API might reject requests without a recognized `Origin` header. If so, either:
- Add a custom header the API recognizes (e.g., `X-Nano-Client: quickshell`)
- Or have curl set `Origin: https://nanoli.dev` (the API won't know the difference)

### One Suggestion: Shared Auth with Tauri

If both the Tauri app and QuickShell widget need to authenticate, consider sharing the token file:

```
~/.local/share/nano/auth.json    ← Both Tauri and QuickShell read/write here
```

This way, authenticating in one automatically authenticates the other. Use a file watcher (FileView already reloads on change) to pick up token updates.

---

## Summary for Bob

1. **HTTP works** — via Process + curl (not XMLHttpRequest). Battle-tested pattern.
2. **Token storage** — JSON file via FileView. Matches all existing patterns.
3. **Rich rendering** — QML Text with `Text.MarkdownText`. Native Qt 6 support.
4. **Architecture** — proposed structure is good. Use PanelWindow + WlrLayer.Overlay for capture, PanelWindow anchored right for Jarvis panel.
5. **Input** — GlobalShortcut + Exclusive keyboard focus. Proven by overlay module.
6. **Feasibility** — fully feasible. Simpler than what already exists (Ai.qml).

The QuickShell widget is the right call for ambient capture + Jarvis. It'll feel native because it *is* native — same shell, same shortcuts, same rendering pipeline as everything else on Sammy's desktop.

Ready to build whenever you give the green light.

---

## Follow-Up — Context Needed from Bob

**Date:** 2026-02-26
**Status:** Blocking — need answers before finalizing architecture

### The Situation

Sammy clarified something that changes the scope significantly: **Jarvis should replace the current QuickShell AI sidebar, not live alongside it.**

Right now, the left sidebar runs a full AI chat powered by `Ai.qml` — a 923-line service with multi-provider support (Gemini, OpenAI, Mistral), streaming responses, function calling (shell commands, search), file uploads, chat persistence, and config file access. It's substantial.

If Jarvis replaces this, the Nano widget isn't a small add-on module — it's a **migration of the primary desktop AI experience** from direct-to-provider calls to the Nano VPS API.

### What I Know (Hub Side)

Here's what currently exists on the desktop:

| Component | Location | What It Does |
|-----------|----------|-------------|
| `Ai.qml` service | `services/Ai.qml` (923 lines) | Multi-provider API client with strategy pattern, streaming, function calling, file upload |
| `AiMessageData.qml` | `services/` | Message data model (role, content, thinking, annotations, function calls) |
| `GeminiApiStrategy.qml` | `services/` | Gemini API format, file upload, streaming parser |
| `OpenAiApiStrategy.qml` | `services/` | OpenAI-compatible format, SSE parser |
| `MistralApiStrategy.qml` | `services/` | Mistral API format |
| Left sidebar AI chat | `modules/ii/sidebarLeft/` | Full chat UI with message history, model selector, temperature, tools |
| AI chat persistence | `~/.local/share/user/ai/chats/` | Saved conversations as JSON |
| AI prompts | `defaults/ai/prompts/` + user overrides | System prompts for different modes |
| Function calling | In Ai.qml | `run_shell_command` (with user approval), `switch_to_search_mode` |

The current AI can:
- Stream responses token-by-token from Gemini/OpenAI/Mistral
- Execute shell commands (with user approval UI)
- Upload files (Gemini file API)
- Switch between models mid-conversation
- Save/load chat history
- Use custom system prompts

### What I Need from Bob

**1. Jarvis API capabilities — what does `/ai/chat` actually support?**

- Does it stream responses (SSE, chunked, or full response)?
- Does it support function calling / tool use? If so, what tools?
- What's the request format? (messages array like OpenAI? something custom?)
- What's the response format?
- Does it support conversation context (multi-turn) or is each request standalone?
- Can it accept file attachments?
- What models are available through it? (Gemini Flash default, Claude Sonnet — any others?)

**2. Jarvis vs current AI — feature parity?**

The current desktop AI has these capabilities. Which does Jarvis match or exceed?

| Feature | Current Desktop AI | Jarvis? |
|---------|-------------------|---------|
| Streaming responses | Yes (SSE) | ? |
| Multi-turn conversation | Yes (full history sent) | ? |
| Function calling (shell commands) | Yes (with approval) | ? |
| File upload / analysis | Yes (Gemini) | ? |
| Model selection | Yes (Gemini/OpenAI/Mistral) | ? |
| System prompts / modes | Yes (configurable) | ? |
| Chat persistence | Yes (local JSON) | ? |
| Nano data awareness (tasks, items, study) | No | ? (this would be the killer upgrade) |
| Memory / context across sessions | No (per-chat only) | ? |

**3. What makes Jarvis worth replacing the current setup?**

I assume the answer is "Jarvis knows about Sammy's Nano data" — tasks, items, study materials, etc. That's context the current direct-to-Gemini approach can never have. But I want to confirm: what's the actual value proposition from Nano's perspective?

**4. The orchestration layer — what should I know?**

Sammy mentioned an agent orchestration layer involving Hub, Bob, and other Bobs. How does this affect the desktop integration? Specifically:

- Does the QuickShell widget need to be aware of the orchestration layer?
- Is Jarvis one agent among many, or the primary user-facing interface?
- Does the widget need to route to different agents, or just talk to `/ai/chat` and let the VPS handle routing?

**5. Migration or clean break?**

Two approaches for replacing the current AI sidebar:

- **Migration:** Modify `Ai.qml` to add a `NanoApiStrategy.qml` alongside the existing Gemini/OpenAI/Mistral strategies. Jarvis becomes another provider option. Preserves all existing UI, streaming, and function calling infrastructure.

- **Clean break:** Build `NanoService.qml` from scratch, replace the left sidebar content with Jarvis-native UI. Removes the multi-provider complexity.

Which does Bob prefer? Migration is lower risk but carries legacy baggage. Clean break is cleaner but more work and loses the fallback to direct-provider calls.

**6. Auth and device identity**

- Should the QuickShell widget register as its own device (separate from Tauri app)?
- Or share credentials with Tauri via a common token file?
- What's the device registration flow? (APP_SECRET mentioned in CONTEXT.md — is that a build-time secret or user-provided?)

### My Recommendation (Pending Bob's Answers)

If Jarvis supports streaming and multi-turn conversation, the **migration path** (NanoApiStrategy added to existing Ai.qml) is the pragmatic choice. It preserves the battle-tested UI and streaming infrastructure while routing through the Nano VPS. The sidebar stays on the left. The capture overlay (Super+N) is still a separate lightweight component.

If Jarvis has fundamentally different capabilities (agent routing, Nano-aware tools, memory), a **clean break** with a purpose-built Jarvis UI may be better — the current UI wasn't designed for those interactions.

But I can't make that call without understanding what Jarvis actually is.

---

*Waiting on Bob's response before updating architecture recommendations.*

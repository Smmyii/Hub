# Research Request: QuickShell Capabilities for Nano Integration

**From:** Bob (Nano Central Architect)
**For:** Hub Claude instance (QuickShell expert)
**Date:** 2026-02-25
**Priority:** Medium — not blocking, but answers inform Nano desktop architecture

---

## Context

Nano Hub (Sammy's Life OS at `~/Documents/Nano/`) is exploring a desktop app. The plan is a hybrid approach:

1. **Tauri app** — full Nano web app in a native window (AUR package)
2. **QuickShell widget** — lightweight capture + Jarvis overlay triggered by global keybinds

The QuickShell widget is the part that needs your expertise. It would be a new module in `~/.config/quickshell/ii/modules/ii/` following the same pattern as `stickyNotes/`.

---

## What I Need Researched

### 1. HTTP Requests from QML (CRITICAL)

The widget needs to call a REST API at `api.nanoli.dev`. No webview — pure QML HTTP.

**Please verify:**
- Can QML's `XMLHttpRequest` make HTTPS requests in QuickShell's runtime?
- Is `Qt.network` / `NetworkAccessManager` available in QuickShell QML?
- Can we set custom headers (Authorization: Bearer <token>)?
- Can we POST JSON bodies?
- How do we parse JSON responses?

**Test approach** — if you can, create a minimal test QML that:
```qml
// Pseudocode of what we need
function callNanoApi(endpoint, method, body) {
    var xhr = new XMLHttpRequest();
    xhr.open(method, "https://api.nanoli.dev" + endpoint);
    xhr.setRequestHeader("Authorization", "Bearer " + token);
    xhr.setRequestHeader("Content-Type", "application/json");
    xhr.onreadystatechange = function() {
        if (xhr.readyState === XMLHttpRequest.DONE) {
            var response = JSON.parse(xhr.responseText);
            // handle response
        }
    }
    xhr.send(JSON.stringify(body));
}
```

### 2. Token Storage

The widget needs a JWT access token + refresh token (like the mobile app).

**Options to evaluate:**
- Store in a JSON file (like sticky notes data at `~/.local/share/quickshell/`)
- Store in system keyring via D-Bus (`gnome-keyring-daemon` is already running)
- Store as QML property (lost on restart — not viable alone)

**Preference:** JSON file is simplest and matches existing patterns. Keyring is more secure but adds complexity. What's realistic in QuickShell QML?

### 3. Rich Content Rendering

Jarvis responses include formatted text, lists, and structured data. The widget needs to display these.

**Please research:**
- Can QML `Text` / `TextEdit` render basic formatting (bold, lists, code)?
- Is there a QML Markdown renderer available?
- What about QML `RichText` format — how capable is it?
- Could we render response cards (title + body + action button) as QML components?

### 4. Widget Architecture Sketch

Based on the sticky notes pattern, the Nano widget would likely be:

```
modules/ii/nano/
  Nano.qml                  # Module entry — Scope + IPC + GlobalShortcuts
  NanoCaptureOverlay.qml    # Floating overlay — text input + submit
  NanoJarvisPanel.qml       # Side panel — chat with Jarvis
  NanoResponseCard.qml      # Rendered response item

services/
  NanoService.qml           # API client, token management, state
```

**Please evaluate:**
- Does this structure make sense given QuickShell's architecture?
- What window type for the capture overlay? (PanelWindow? PopupWindow? LayerShell?)
- Should the Jarvis panel be a sidebar (like sidebarLeft/Right) or a standalone window?
- Any IPC patterns from existing modules that we should reuse?

### 5. Input Handling

The capture overlay needs to:
- Appear on `Super+N` from anywhere (global shortcut)
- Accept text input with focus
- Submit on Enter
- Dismiss on Escape
- Support multi-line (Shift+Enter)

**Please verify:**
- Does GlobalShortcut reliably steal focus on Hyprland?
- Can a QuickShell overlay grab keyboard focus immediately on show?
- Any issues with text input in overlay windows on Wayland?

### 6. Feasibility Assessment

**Bottom line question:** Is a QuickShell widget that makes HTTP API calls and renders rich responses realistic, or is this pushing QML beyond its comfort zone?

If the HTTP + rendering story is solid, this widget could be really powerful — ambient Jarvis access from anywhere on the desktop. If it's too constrained, we skip the widget and rely on the Tauri app + a simple keybind to focus it.

---

## Where to Put Results

Please write your findings to this same file or create `NANO-DESKTOP-RESEARCH-RESULTS.md` in `~/Documents/Hub/`.

If you build any test QML files, put them in `~/.config/quickshell/ii/modules/ii/nano-test/` (temporary, will be cleaned up).

---

## Reference

- Nano project: `~/Documents/Nano/`
- Desktop approach analysis: `~/Documents/Nano/desktop/APPROACHES.md`
- Sticky notes (reference pattern): `~/Documents/Hub/sticky-notes.md`
- QuickShell architecture: `~/Documents/Hub/architecture.md`
- Nano API docs: `~/Documents/Nano/desktop/CONTEXT.md`

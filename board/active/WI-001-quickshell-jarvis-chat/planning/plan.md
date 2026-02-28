# WI-001 Implementation Plan

Plan derived from Bob's followup answers (2026-02-26).

## File Structure

```
~/.config/quickshell/ii/
  services/
    NanoService.qml           # API client + auth + state (Singleton)
  modules/ii/nano/
    Nano.qml                  # Module entry — Scope + IPC + GlobalShortcuts
    NanoCaptureOverlay.qml    # Quick capture (Super+N)
    NanoJarvisPanel.qml       # Jarvis chat panel (Super+J)
    NanoResponseCard.qml      # Tool use display cards
    NanoLoginOverlay.qml      # One-time device registration
```

## Phase 1: Capture + Auth

### Step 1: NanoService.qml
- Singleton service with Process+curl pattern
- FileView for auth tokens at ~/.local/share/nano/auth.json
- apiCall(method, endpoint, body) function — builds bash script, writes via FileView, executes via Process
- Token refresh Timer (check every 60s, refresh 5min before expiry)
- Convenience: chat(message), capture(text), healthCheck()

### Step 2: NanoLoginOverlay.qml
- PanelWindow with WlrLayer.Overlay
- APP_SECRET input field
- Device name input (default: "Desktop QuickShell")
- POST /auth/register-device → save tokens → close overlay
- Only shown when NanoService.authenticated === false

### Step 3: NanoCaptureOverlay.qml
- PanelWindow with WlrLayer.Overlay, WlrKeyboardFocus.Exclusive when visible
- 600x80px centered floating input
- TextArea with Enter=submit, Shift+Enter=newline, Escape=dismiss
- Submits to NanoService.capture() → POST /sync/push (creates Inbox item)
- "Jarvis," prefix detection → routes to /ai/chat instead (optional, Phase 2)

### Step 4: Nano.qml (module entry)
- Scope with NanoCaptureOverlay + NanoJarvisPanel children
- IpcHandler target: "nano" with toggleCapture, toggleJarvis, capture, jarvis, close
- GlobalShortcut "nanoCapture" (Super+N) and "nanoJarvis" (Super+J)

## Phase 2: Jarvis Panel

### Step 5: NanoResponseCard.qml
- Rectangle with title + markdown body + optional action button
- Used to display tool_use[] entries from Jarvis responses
- Styled with Appearance.colors/rounding

### Step 6: NanoJarvisPanel.qml
- PanelWindow anchored right, WlrLayer.Top, exclusiveZone: 0, width: 400
- WlrKeyboardFocus.OnDemand (not Exclusive — user may want to interact with other windows)
- ScrollView with message list (ListView + model)
- TextArea input at bottom (Enter=send, Escape=close)
- conversation_id management (new conversation button, persist last ID)
- Model toggle: Flash/Sonnet (Ctrl+M or button)
- Loading state while waiting for response
- Renders response.content as MarkdownText
- Renders response.tool_use as NanoResponseCard list

## Keybinds (hyprland)
```conf
bind = SUPER, N, exec, qs -c ii ipc call nano capture
bind = SUPER, J, exec, qs -c ii ipc call nano jarvis
```

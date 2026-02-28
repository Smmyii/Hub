# Phase 1: Foundation - Research

**Researched:** 2026-02-02
**Domain:** Quickshell layer shell positioning, IPC, illogical-impulse integration
**Confidence:** HIGH

## Summary

This research investigated how to implement sticky notes as a Quickshell widget within the existing illogical-impulse shell configuration. The codebase already contains a notes widget within the overlay system (`Super+G`) that provides an excellent reference implementation. The foundation phase requires creating an independent sticky notes module that can work outside the overlay context, with dedicated IPC commands and keybinds.

Key findings:
- An existing notes widget exists at `/home/sammy/.config/quickshell/ii/modules/ii/overlay/notes/` using `StyledOverlayWidget` as base
- The overlay system provides dragging, resizing, pinning, and persistence patterns we can adapt
- IPC uses `IpcHandler` components with `target` and function definitions
- Layer shell positioning uses `PanelWindow` with `WlrLayershell` for Wayland layer control
- Data persistence uses `FileView` with JSON for state and `Directories` singleton for paths
- Fullscreen detection pattern exists in `Background.qml` using `workspace.toplevels.values.filter(window => window.wayland?.fullscreen)`

**Primary recommendation:** Create a new `stickyNotes` module under `modules/ii/` following the overlay widget patterns, but as an independent layer shell surface controllable via IPC.

## Standard Stack

The established libraries/tools for this domain:

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| Quickshell | current | Shell framework | Already in use, provides layer shell, IPC |
| QtQuick | 6.x | UI framework | Standard for Quickshell widgets |
| Quickshell.Wayland | current | Wayland integration | Provides WlrLayershell, layer control |
| Quickshell.Hyprland | current | Hyprland integration | Provides workspace/monitor awareness |

### Supporting
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| Quickshell.Io | current | File operations | Data persistence (FileView, Process) |
| Qt5Compat.GraphicalEffects | 6.x | Visual effects | Shadows, blur (DropShadow) |
| QtQuick.Effects | 6.x | Modern effects | RectangularShadow |

### Alternatives Considered
| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| Independent module | Extend overlay notes | Independence vs. reuse - overlay notes require Super+G open |
| Custom persistence | Existing Persistent.qml | Custom gives cleaner separation for user data |

**Installation:** No additional packages needed - all components already available in illogical-impulse.

## Architecture Patterns

### Recommended Project Structure
```
~/.config/quickshell/ii/modules/ii/stickyNotes/
    StickyNotes.qml           # Main module, IPC handlers, note management
    StickyNoteWindow.qml      # Individual note PanelWindow
    StickyNoteContent.qml     # Note content (text area, controls)

~/.config/quickshell/ii/services/
    StickyNotesService.qml    # Data persistence, note state management

~/.local/share/quickshell/
    sticky-notes.json         # Note data (survives config resets)
```

### Pattern 1: IPC Handler Pattern
**What:** Register commands via IpcHandler component
**When to use:** Any keybind-triggered functionality
**Example:**
```qml
// Source: /home/sammy/.config/quickshell/ii/modules/ii/overlay/Overlay.qml
IpcHandler {
    target: "overlay"

    function toggle(): void {
        GlobalStates.overlayOpen = !GlobalStates.overlayOpen;
    }
}
```

### Pattern 2: Layer Shell Window
**What:** PanelWindow with WlrLayershell for layer positioning
**When to use:** Any widget that floats over desktop
**Example:**
```qml
// Source: /home/sammy/.config/quickshell/ii/modules/ii/overlay/Overlay.qml
PanelWindow {
    id: overlayWindow
    exclusionMode: ExclusionMode.Ignore
    WlrLayershell.namespace: "quickshell:overlay"
    WlrLayershell.layer: WlrLayer.Overlay
    WlrLayershell.keyboardFocus: WlrKeyboardFocus.OnDemand
    visible: true
    color: "transparent"
}
```

### Pattern 3: Data Persistence with FileView
**What:** FileView for JSON file read/write with debouncing
**When to use:** Persisting note content/state
**Example:**
```qml
// Source: /home/sammy/.config/quickshell/ii/services/Todo.qml
FileView {
    id: todoFileView
    path: Qt.resolvedUrl(root.filePath)
    onLoaded: {
        root.list = JSON.parse(todoFileView.text())
    }
    onLoadFailed: (error) => {
        if(error == FileViewError.FileNotFound) {
            root.list = []
            todoFileView.setText(JSON.stringify(root.list))
        }
    }
}
```

### Pattern 4: Fullscreen Detection
**What:** Hide pinned notes when app goes fullscreen
**When to use:** Note visibility control
**Example:**
```qml
// Source: /home/sammy/.config/quickshell/ii/modules/ii/background/Background.qml
property list<HyprlandWorkspace> workspacesForMonitor:
    Hyprland.workspaces.values.filter(workspace =>
        workspace.monitor && workspace.monitor.name == monitor.name)
property var activeWorkspaceWithFullscreen:
    workspacesForMonitor.filter(workspace =>
        ((workspace.toplevels.values.filter(window =>
            window.wayland?.fullscreen)[0] != undefined) && workspace.active))[0]
visible: !(activeWorkspaceWithFullscreen != undefined) || !hideWhenFullscreen
```

### Pattern 5: Drag Handler for Positioning
**What:** DragHandler or MouseArea.drag for widget movement
**When to use:** Draggable notes
**Example:**
```qml
// Source: /home/sammy/.config/quickshell/ii/modules/common/widgets/widgetCanvas/AbstractWidget.qml
MouseArea {
    draggable: true
    drag.target: draggable ? root : undefined
    cursorShape: (draggable && containsPress) ? Qt.ClosedHandCursor :
                 draggable ? Qt.OpenHandCursor : Qt.ArrowCursor
}
```

### Pattern 6: Focus Handling for Text Input
**What:** forceActiveFocus() + WlrKeyboardFocus.OnDemand
**When to use:** Making text areas immediately typeable
**Example:**
```qml
// Source: /home/sammy/.config/quickshell/ii/modules/ii/overlay/notes/NotesContent.qml
function focusAtEnd() {
    textInput.forceActiveFocus();
    const endPos = root.content.length;
    applySelection(endPos, endPos);
}
```

### Anti-Patterns to Avoid
- **Putting sticky notes inside overlay:** Would require Super+G to be open for notes to be visible
- **Using GlobalStates for note visibility:** Better to use independent state per note
- **Absolute screen coordinates:** Use per-monitor relative positioning
- **Hand-rolling drag logic:** Use existing DragHandler or AbstractWidget patterns
- **Blocking file writes:** Use debounced Timer for saves

## Don't Hand-Roll

Problems that look simple but have existing solutions:

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Drag-to-move | Custom mouse tracking | DragHandler or AbstractWidget | Handles edge cases, cursor states |
| Persistence | Raw file I/O | FileView with JSON | Built-in reload/watch, error handling |
| Text editing | Custom TextInput | StyledTextArea or MaterialTextArea | Matches ii/ aesthetic, accessibility |
| Shadow/border | Manual Rectangle | StyledDropShadow/StyledRectangularShadow | Consistent with theme |
| Layer positioning | Raw coordinates | WlrLayershell anchors | Proper Wayland protocol |
| Focus grabbing | Complex focus logic | HyprlandFocusGrab | Handles edge cases |
| IPC | Custom socket | IpcHandler | Already integrated with qs CLI |

**Key insight:** The overlay notes widget already solves most UI problems - adapt its patterns rather than reinventing.

## Common Pitfalls

### Pitfall 1: Layer Shell Focus Stealing
**What goes wrong:** Note steals keyboard focus from active window
**Why it happens:** Using `WlrKeyboardFocus.Exclusive` instead of `OnDemand`
**How to avoid:** Use `WlrKeyboardFocus.OnDemand` - only grabs focus when clicked
**Warning signs:** Can't type in other apps when note is visible

### Pitfall 2: Note Persists Across Config Reset
**What goes wrong:** Note data lost when illogical-impulse updates
**Why it happens:** Storing data under `~/.config/illogical-impulse/`
**How to avoid:** Store in `~/.local/share/` per XDG spec
**Warning signs:** Data in config dirs that could be wiped

### Pitfall 3: Z-Order Confusion
**What goes wrong:** Notes appear behind other layer shell surfaces
**Why it happens:** Using wrong WlrLayer or not handling z-order
**How to avoid:** Use `WlrLayer.Overlay` for visibility, manage z-index for multiple notes
**Warning signs:** Notes disappearing behind bar/dock

### Pitfall 4: Multi-Monitor Coordinate Mismatch
**What goes wrong:** Note position wrong on different monitor
**Why it happens:** Using absolute screen coords instead of monitor-relative
**How to avoid:** Store position as percentage or relative to monitor bounds
**Warning signs:** Notes jumping to wrong monitor

### Pitfall 5: Overlay Conflict
**What goes wrong:** Super+G overlay interferes with sticky notes
**Why it happens:** Not properly separating concerns
**How to avoid:** Sticky notes are INDEPENDENT of overlay - different layer shell surfaces
**Warning signs:** Notes only visible when overlay is open

## Code Examples

Verified patterns from official sources:

### IPC Command Registration with GlobalShortcut
```qml
// Source: /home/sammy/.config/quickshell/ii/modules/ii/overlay/Overlay.qml
IpcHandler {
    target: "stickyNotes"  // Call with: qs -c ii ipc call stickyNotes toggle

    function toggle(): void {
        // Toggle last active note
    }

    function create(): void {
        // Create new note
    }

    function showAll(): void {
        // Show all notes
    }

    function hideAll(): void {
        // Hide all notes
    }
}

GlobalShortcut {
    name: "stickyNotesToggle"  // Bind with: global, quickshell:stickyNotesToggle
    description: "Toggle sticky notes"

    onPressed: {
        // Toggle action
    }
}
```

### Panel Window with Focus Control
```qml
// Source: /home/sammy/.config/quickshell/ii/modules/ii/overlay/Overlay.qml
PanelWindow {
    id: noteWindow
    exclusionMode: ExclusionMode.Ignore
    WlrLayershell.namespace: "quickshell:stickyNote"
    WlrLayershell.layer: WlrLayer.Overlay
    // OnDemand = only grabs focus when clicked, not always
    WlrLayershell.keyboardFocus: WlrKeyboardFocus.OnDemand
    visible: noteVisible
    color: "transparent"

    // Positioning - no anchors for floating window
    // Use x, y, width, height instead
}
```

### User Notification for Errors
```qml
// Pattern from /home/sammy/.config/quickshell/ii services
// Use notify-send or existing notification system
Quickshell.execDetached(["notify-send",
    "Sticky Notes",
    "Error: " + errorMessage,
    "-a", "Quickshell"
])
```

### Keybind Configuration
```conf
# Source: /home/sammy/.config/hypr/hyprland/keybinds.conf pattern
# Add to custom/keybinds.conf for update resilience

##! Sticky Notes
bind = Super+Shift, G, global, quickshell:stickyNotesToggle # Toggle sticky note
# Include fallback:
bind = Super+Shift, G, exec, qs -c ii ipc call stickyNotes toggle # [hidden]
```

### Directory Path for User Data
```qml
// Source: /home/sammy/.config/quickshell/ii/modules/common/Directories.qml
// Add to Directories.qml singleton:
property string stickyNotesPath: FileUtils.trimFileProtocol(
    `${StandardPaths.standardLocations(StandardPaths.GenericDataLocation)[0]}/quickshell/sticky-notes.json`
)
// Results in: ~/.local/share/quickshell/sticky-notes.json
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| ags/eww widgets | Quickshell | 2024 | Better Wayland integration |
| Custom drag | DragHandler | Qt6 | Cleaner code, better UX |
| Manual layering | WlrLayershell | Current | Proper Wayland protocol |

**Deprecated/outdated:**
- Qt5 GraphicalEffects in Qt6 requires `Qt5Compat.GraphicalEffects` import
- Old QML `MouseArea { drag.target }` works but `DragHandler` is preferred for complex drag

## Open Questions

Things that couldn't be fully resolved:

1. **DragHandler + Layer Shell Compatibility**
   - What we know: AbstractWidget uses MouseArea.drag, overlay widgets use DragHandler
   - What's unclear: Whether DragHandler works correctly on layer shell surfaces
   - Recommendation: Start with DragHandler (used in StyledOverlayWidget), fall back to MouseArea.drag if issues

2. **Multiple Notes Z-Order**
   - What we know: Overlay widgets handle z-order via `z: dragHandler.active ? 2 : 1`
   - What's unclear: How to persist z-order across sessions
   - Recommendation: Track lastActive timestamp, highest timestamp = front

3. **Cascade Placement for New Notes**
   - What we know: No existing cascade pattern in codebase
   - What's unclear: Optimal offset for visibility
   - Recommendation: Offset 30px right, 30px down from last note position

## Sources

### Primary (HIGH confidence)
- `/home/sammy/.config/quickshell/ii/modules/ii/overlay/` - Existing overlay widget patterns
- `/home/sammy/.config/quickshell/ii/modules/common/widgets/widgetCanvas/` - Drag/positioning
- `/home/sammy/.config/quickshell/ii/services/Todo.qml` - FileView persistence pattern
- `/home/sammy/.config/quickshell/ii/modules/common/Directories.qml` - XDG paths
- `/home/sammy/.config/hypr/hyprland/keybinds.conf` - IPC/keybind patterns

### Secondary (MEDIUM confidence)
- `/home/sammy/.config/quickshell/ii/modules/ii/background/Background.qml` - Fullscreen detection
- `/home/sammy/.config/quickshell/ii/modules/common/Persistent.qml` - State persistence pattern

### Tertiary (LOW confidence)
- DragHandler on layer shell surfaces - needs validation during implementation

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH - Verified from codebase
- Architecture patterns: HIGH - Verified from multiple codebase examples
- Pitfalls: HIGH - Derived from codebase patterns and user requirements
- DragHandler compatibility: LOW - Needs runtime testing

**Research date:** 2026-02-02
**Valid until:** 2026-03-02 (30 days - stable codebase)

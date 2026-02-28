# Technology Stack: Sticky Notes Widget

**Project:** Sticky Notes Enhancement
**Dimension:** QML Components for Drag/Resize/Persistence
**Researched:** 2026-02-02
**Overall Confidence:** MEDIUM (based on Qt 6 training knowledge; unable to verify with current documentation due to tool access limitations)

## Executive Summary

This stack research focuses on the specific QML components and patterns needed to add draggable, resizable, persistent sticky notes to the existing Quickshell/illogical-impulse setup. The existing Quickshell config already provides the shell framework; this research identifies the specific QML types and persistence patterns needed.

## Recommended Stack

### Drag Implementation

| Technology | Version | Purpose | Confidence |
|------------|---------|---------|------------|
| **DragHandler** | Qt 6.x | Modern declarative drag handling | HIGH |
| **MouseArea.drag** | Qt 6.x | Alternative/fallback drag approach | HIGH |

**Recommendation: DragHandler**

Use `DragHandler` from `QtQuick` for dragging because:
- Declarative, cleaner than MouseArea.drag for single-purpose drag
- Works well with touch and pointer input
- Better composability with other input handlers
- Part of Qt Quick Input Handlers (modern approach)

```qml
import QtQuick

Rectangle {
    id: stickyNote
    x: 100; y: 100
    width: 200; height: 150

    DragHandler {
        target: stickyNote
        // Constrain to parent if needed
        xAxis.minimum: 0
        xAxis.maximum: parent.width - stickyNote.width
        yAxis.minimum: 0
        yAxis.maximum: parent.height - stickyNote.height
    }
}
```

**Alternative: MouseArea.drag**

If DragHandler causes issues with Quickshell layer shell, fallback to MouseArea:

```qml
Rectangle {
    id: stickyNote
    x: 100; y: 100
    width: 200; height: 150

    MouseArea {
        anchors.fill: parent
        drag.target: stickyNote
        drag.axis: Drag.XAndYAxis
        drag.minimumX: 0
        drag.maximumX: parent.width - stickyNote.width
        drag.minimumY: 0
        drag.maximumY: parent.height - stickyNote.height
    }
}
```

### Resize Implementation

| Technology | Version | Purpose | Confidence |
|------------|---------|---------|------------|
| **Custom resize handles** | - | Manual resize via edge MouseAreas | HIGH |
| **PinchHandler** | Qt 6.x | Touch/gesture resize (optional) | MEDIUM |

**Recommendation: Custom Resize Handles**

QML does not have a built-in "resize handle" component. Build custom resize handles using MouseAreas positioned at corners/edges:

```qml
Rectangle {
    id: stickyNote
    x: 100; y: 100
    width: 200; height: 150

    property int minWidth: 100
    property int minHeight: 80
    property int maxWidth: 600
    property int maxHeight: 400

    // Bottom-right resize handle
    Rectangle {
        id: resizeHandle
        width: 16; height: 16
        color: "transparent"
        anchors.right: parent.right
        anchors.bottom: parent.bottom

        MouseArea {
            anchors.fill: parent
            cursorShape: Qt.SizeFDiagCursor

            property point clickPos
            property size startSize

            onPressed: (mouse) => {
                clickPos = Qt.point(mouse.x, mouse.y)
                startSize = Qt.size(stickyNote.width, stickyNote.height)
            }

            onPositionChanged: (mouse) => {
                if (pressed) {
                    let newWidth = Math.max(stickyNote.minWidth,
                        Math.min(stickyNote.maxWidth,
                            startSize.width + mouse.x - clickPos.x))
                    let newHeight = Math.max(stickyNote.minHeight,
                        Math.min(stickyNote.maxHeight,
                            startSize.height + mouse.y - clickPos.y))
                    stickyNote.width = newWidth
                    stickyNote.height = newHeight
                }
            }
        }
    }
}
```

**Why custom vs PinchHandler:**
- PinchHandler is designed for touch gestures, not mouse resize
- Custom handles provide visual affordance (cursor change)
- More control over resize behavior and constraints

### Persistence

| Technology | Version | Purpose | Confidence |
|------------|---------|---------|------------|
| **JSON file** | - | Note data storage | HIGH |
| **Qt.labs.settings** | Qt 6.x | Simple key-value persistence | MEDIUM |
| **FileIO (Quickshell)** | - | File read/write in Quickshell | MEDIUM |

**Recommendation: JSON File via Quickshell FileIO**

Based on the codebase integrations analysis, Quickshell already uses file-based storage patterns. Use JSON for structured note data:

**Storage location:** `~/.local/share/quickshell/sticky-notes.json`

**Data schema:**
```json
{
    "notes": [
        {
            "id": "uuid-1",
            "x": 100,
            "y": 200,
            "width": 250,
            "height": 180,
            "content": "Note text here",
            "color": "#ffeb3b",
            "pinned": true,
            "created": "2026-02-02T14:30:00Z",
            "modified": "2026-02-02T15:45:00Z"
        }
    ],
    "version": 1
}
```

**Why JSON over alternatives:**

| Option | Pros | Cons | Verdict |
|--------|------|------|---------|
| JSON file | Human-readable, easy debug, structured | Manual serialization | RECOMMENDED |
| Qt.labs.settings | Simple API | Flat key-value only, not ideal for arrays | Not for note lists |
| SQLite | Query capability | Overkill for <100 notes | Unnecessary complexity |
| QSettings | Qt standard | INI format, not ideal for nested data | Not suitable |

**Implementation approach:**
```qml
import Quickshell.Io

// In Quickshell, FileIO or Process for file operations
// Pseudo-code pattern:

Component.onCompleted: {
    loadNotes()
}

function loadNotes() {
    // Read JSON file, parse, populate ListModel
    let content = FileIO.read(notesFilePath)
    let data = JSON.parse(content)
    data.notes.forEach(note => notesModel.append(note))
}

function saveNotes() {
    // Serialize ListModel to JSON, write to file
    let notes = []
    for (let i = 0; i < notesModel.count; i++) {
        notes.push(notesModel.get(i))
    }
    let json = JSON.stringify({ notes: notes, version: 1 }, null, 2)
    FileIO.write(notesFilePath, json)
}
```

### Multi-Instance Management

| Technology | Version | Purpose | Confidence |
|------------|---------|---------|------------|
| **ListModel** | Qt 6.x | Note collection management | HIGH |
| **Repeater** | Qt 6.x | Instantiate note components | HIGH |
| **Component** | Qt 6.x | Note template definition | HIGH |

**Recommendation: ListModel + Repeater Pattern**

Standard QML pattern for multiple dynamic items:

```qml
import QtQuick

Item {
    id: notesContainer
    anchors.fill: parent

    ListModel {
        id: notesModel
        // Populated from JSON on load
    }

    Repeater {
        model: notesModel

        delegate: StickyNote {
            noteId: model.id
            x: model.x
            y: model.y
            width: model.width
            height: model.height
            noteContent: model.content
            noteColor: model.color

            onPositionChanged: {
                // Update model when dragged
                notesModel.setProperty(index, "x", x)
                notesModel.setProperty(index, "y", y)
                saveNotesDebounced()
            }

            onSizeChanged: {
                // Update model when resized
                notesModel.setProperty(index, "width", width)
                notesModel.setProperty(index, "height", height)
                saveNotesDebounced()
            }
        }
    }

    function createNote(x, y) {
        notesModel.append({
            id: generateUUID(),
            x: x,
            y: y,
            width: 200,
            height: 150,
            content: "",
            color: "#ffeb3b",
            pinned: false,
            created: new Date().toISOString(),
            modified: new Date().toISOString()
        })
        saveNotes()
    }

    function deleteNote(noteId) {
        for (let i = 0; i < notesModel.count; i++) {
            if (notesModel.get(i).id === noteId) {
                notesModel.remove(i)
                break
            }
        }
        saveNotes()
    }
}
```

### Context Menu

| Technology | Version | Purpose | Confidence |
|------------|---------|---------|------------|
| **Menu** | QtQuick.Controls 6.x | Right-click context menu | HIGH |
| **MenuItem** | QtQuick.Controls 6.x | Menu items | HIGH |

**Implementation:**
```qml
import QtQuick.Controls

Menu {
    id: noteContextMenu

    MenuItem {
        text: "New Note"
        onTriggered: createNote(contextMenu.x, contextMenu.y)
    }
    MenuItem {
        text: "Delete Note"
        onTriggered: deleteNote(currentNoteId)
    }
    MenuItem {
        text: "Pin/Unpin"
        onTriggered: togglePin(currentNoteId)
    }
    MenuSeparator {}
    Menu {
        title: "Color"
        MenuItem { text: "Yellow"; onTriggered: setColor("#ffeb3b") }
        MenuItem { text: "Green"; onTriggered: setColor("#a5d6a7") }
        MenuItem { text: "Blue"; onTriggered: setColor("#90caf9") }
        MenuItem { text: "Pink"; onTriggered: setColor("#f48fb1") }
    }
}

// In MouseArea:
MouseArea {
    acceptedButtons: Qt.LeftButton | Qt.RightButton
    onClicked: (mouse) => {
        if (mouse.button === Qt.RightButton) {
            noteContextMenu.popup()
        }
    }
}
```

### Auto-Scroll TextArea

| Technology | Version | Purpose | Confidence |
|------------|---------|---------|------------|
| **TextArea** | QtQuick.Controls 6.x | Multi-line text input | HIGH |
| **ScrollView** | QtQuick.Controls 6.x | Scrollable container | HIGH |
| **Flickable** | Qt 6.x | Alternative scroll container | HIGH |

**Implementation:**
```qml
import QtQuick.Controls

ScrollView {
    anchors.fill: parent
    anchors.margins: 8

    TextArea {
        id: noteText
        wrapMode: TextEdit.Wrap
        placeholderText: "Type your note..."

        // Auto-scroll follows cursor
        onCursorRectangleChanged: {
            // ScrollView automatically handles this when
            // TextArea is inside ScrollView
        }

        onTextChanged: {
            notesModel.setProperty(noteIndex, "content", text)
            notesModel.setProperty(noteIndex, "modified", new Date().toISOString())
            saveNotesDebounced()
        }
    }
}
```

## Supporting Libraries

| Library | Purpose | When to Use |
|---------|---------|-------------|
| `QtQuick` | Core QML types | Always - base framework |
| `QtQuick.Controls` | UI controls (Menu, TextArea, ScrollView) | For controls |
| `QtQuick.Layouts` | Layout management | For internal note layout |
| `Quickshell` | Shell integration | For window/layer management |
| `Quickshell.Io` | File I/O | For persistence |

## Alternatives Considered

| Category | Recommended | Alternative | Why Not Alternative |
|----------|-------------|-------------|---------------------|
| Drag | DragHandler | MouseArea.drag | Less clean, more code |
| Resize | Custom handles | PinchHandler | PinchHandler is for touch gestures |
| Persistence | JSON file | SQLite | Overkill for note storage |
| Persistence | JSON file | Qt.labs.settings | Not suited for array data |
| Multi-instance | ListModel+Repeater | Dynamic object creation | Repeater is standard pattern |

## Quickshell-Specific Considerations

Based on the codebase analysis showing Quickshell patterns:

### Layer Shell Integration

Notes need to work with Hyprland layer rules. The existing config shows:
- Layer rules in `/home/sammy/.config/hypr/hyprland/rules.conf`
- Quickshell panels use `layerrule` for blur, animation control
- Pin functionality likely uses layer shell "overlay" or "top" layer

**Recommendation:** Create notes as a Quickshell panel with configurable layer:
- Unpinned: Normal layer (below other windows)
- Pinned: Overlay layer (above other windows)

### IPC Integration

The codebase shows Quickshell IPC pattern:
```bash
qs -c $qsConfig ipc call {SERVICE} {METHOD}
```

**Recommendation:** Expose sticky note actions via IPC:
- `stickyNotes create` - Create new note
- `stickyNotes toggle` - Show/hide all notes
- `stickyNotes save` - Force save

### State File Location

Following Quickshell patterns, store state in:
- Primary: `~/.local/share/quickshell/sticky-notes.json`
- Alternative: `~/.config/quickshell/ii/data/sticky-notes.json`

## Implementation Order

1. **Phase 1: Basic Note Component**
   - ListModel for note collection
   - Repeater with basic Rectangle delegate
   - DragHandler for movement
   - TextArea for content

2. **Phase 2: Persistence**
   - JSON save/load functions
   - Auto-save on change (debounced)
   - Load on startup

3. **Phase 3: Resize**
   - Custom resize handle component
   - Min/max constraints
   - Save size to model

4. **Phase 4: Context Menu**
   - Menu component with actions
   - Create/delete/pin actions
   - Color picker submenu

## Confidence Assessment

| Component | Confidence | Rationale |
|-----------|------------|-----------|
| DragHandler | MEDIUM | Standard Qt 6 pattern; unable to verify Quickshell compatibility |
| MouseArea.drag | HIGH | Well-established fallback, widely documented |
| Custom resize handles | HIGH | Standard pattern, no framework dependency |
| ListModel + Repeater | HIGH | Core QML pattern, certain to work |
| JSON persistence | MEDIUM | Depends on Quickshell FileIO availability |
| Context Menu | HIGH | Qt Quick Controls standard component |
| TextArea + ScrollView | HIGH | Standard components |

## Gaps and Uncertainties

1. **Quickshell FileIO API** - Need to verify exact API for file operations in Quickshell
2. **Layer shell behavior** - Need to test how pinned notes interact with Hyprland layers
3. **DragHandler + layer shell** - Uncertain if DragHandler works correctly with Quickshell panels
4. **Multi-monitor** - Note position storage may need monitor-relative coordinates

## Verification Needed

Before implementation:
- [ ] Verify Quickshell FileIO or Process API for file operations
- [ ] Test DragHandler in Quickshell panel context
- [ ] Confirm layer shell "pin" behavior with overlay layer
- [ ] Check existing illogical-impulse note implementation patterns

---

*Stack research: 2026-02-02*
*Confidence: MEDIUM - Based on Qt 6 training knowledge; web verification tools unavailable*

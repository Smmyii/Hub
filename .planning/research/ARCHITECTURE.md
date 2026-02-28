# Architecture Patterns: QML Sticky Notes

**Domain:** Multi-instance draggable/resizable widgets in Quickshell
**Researched:** 2026-02-02
**Confidence:** MEDIUM (based on established QML patterns; verify against current Quickshell API)

## Recommended Architecture

```
+------------------------------------------------------------------+
|                        Shell Root (shell.qml)                     |
|  +------------------------------------------------------------+  |
|  |                    NoteManager (Singleton)                  |  |
|  |  - noteModel: ListModel                                     |  |
|  |  - createNote() / deleteNote(id)                           |  |
|  |  - loadNotes() / saveNotes()                               |  |
|  +------------------------------------------------------------+  |
|                              |                                    |
|              +---------------+---------------+                    |
|              |                               |                    |
|  +-----------v-----------+     +-------------v-----------+       |
|  |   Overlay Layer       |     |   Standalone Windows    |       |
|  |   (Super+G toggle)    |     |   (pinned notes)        |       |
|  |                       |     |                         |       |
|  |  +------------------+ |     |  +------------------+   |       |
|  |  | Repeater         | |     |  | Repeater         |   |       |
|  |  |  model: filtered | |     |  |  model: filtered |   |       |
|  |  |  delegate: Note  | |     |  |  delegate:       |   |       |
|  |  +------------------+ |     |  |   PanelWindow    |   |       |
|  +-----------------------+     |  |    + Note        |   |       |
|                                |  +------------------+   |       |
|                                +-------------------------+       |
+------------------------------------------------------------------+
                              |
                    +---------v---------+
                    |  PersistenceService |
                    |  (JSON file I/O)   |
                    +--------------------+
```

### Component Boundaries

| Component | Responsibility | Communicates With |
|-----------|---------------|-------------------|
| **NoteManager** | CRUD operations, model management, persistence orchestration | PersistenceService, Note instances |
| **Note** | Visual representation, drag/resize interaction, content editing | NoteManager (via signals) |
| **PersistenceService** | File I/O, JSON serialization/deserialization | NoteManager only |
| **OverlayLayer** | Container for notes in overlay mode, visibility toggle | NoteManager (reads model) |
| **StandaloneNoteWindow** | PanelWindow wrapper for pinned notes | NoteManager (reads model) |

### Data Flow

```
User Interaction                    State                         Persistence
      |                               |                               |
      v                               v                               v
+------------+    signal       +-------------+    call         +------------+
| Note UI    | --------------> | NoteManager | --------------> | Persistence|
| (drag/edit)|  noteChanged()  | (ListModel) |  saveNotes()   | Service    |
+------------+                 +-------------+                 +------------+
      ^                               |                               |
      |                               v                               |
      |                        model binding                          |
      +------------------------(automatic)----------------------------+
                                                   loadNotes() at startup
```

**Data flow is unidirectional:**
1. User interacts with Note component
2. Note emits signal with new state
3. NoteManager updates ListModel
4. ListModel binding automatically updates all Note views
5. NoteManager triggers async save to persistence

## Component Specifications

### NoteManager.qml (Singleton)

```qml
pragma Singleton
import QtQuick

QtObject {
    id: noteManager

    // Central data store - all notes live here
    property ListModel noteModel: ListModel {}

    // Track next available ID
    property int nextId: 0

    // Signals for external communication
    signal notesLoaded()
    signal notesSaved()

    function createNote(x, y) {
        var noteData = {
            noteId: nextId++,
            x: x,
            y: y,
            width: 200,
            height: 150,
            content: "",
            color: "#ffeb3b",
            pinned: false,       // false = overlay only, true = standalone window
            zOrder: noteModel.count
        }
        noteModel.append(noteData)
        scheduleSave()
        return noteData.noteId
    }

    function updateNote(noteId, properties) {
        for (var i = 0; i < noteModel.count; i++) {
            if (noteModel.get(i).noteId === noteId) {
                for (var prop in properties) {
                    noteModel.setProperty(i, prop, properties[prop])
                }
                break
            }
        }
        scheduleSave()
    }

    function deleteNote(noteId) {
        for (var i = 0; i < noteModel.count; i++) {
            if (noteModel.get(i).noteId === noteId) {
                noteModel.remove(i)
                break
            }
        }
        scheduleSave()
    }

    // Debounced save - don't write on every pixel of drag
    property Timer saveTimer: Timer {
        interval: 500
        onTriggered: PersistenceService.saveNotes(noteModel)
    }

    function scheduleSave() {
        saveTimer.restart()
    }

    Component.onCompleted: {
        PersistenceService.loadNotes(noteModel)
        nextId = calculateNextId()
    }
}
```

### Note.qml (Reusable Component)

```qml
import QtQuick
import QtQuick.Controls

Rectangle {
    id: note

    // Required properties bound from model
    required property int noteId
    required property string content
    required property string color
    required property int zOrder

    // Dimensions (can be bound or set)
    property alias noteWidth: note.width
    property alias noteHeight: note.height

    // Signals to parent/manager
    signal contentChanged(int id, string newContent)
    signal positionChanged(int id, real newX, real newY)
    signal sizeChanged(int id, real newWidth, real newHeight)
    signal deleteRequested(int id)
    signal pinToggled(int id, bool pinned)

    color: note.color
    radius: 4
    z: zOrder

    // Header with drag handle and controls
    Rectangle {
        id: header
        height: 24
        anchors { left: parent.left; right: parent.right; top: parent.top }
        color: Qt.darker(note.color, 1.1)

        MouseArea {
            id: dragArea
            anchors.fill: parent
            drag.target: note
            drag.minimumX: 0
            drag.minimumY: 0

            onReleased: {
                note.positionChanged(noteId, note.x, note.y)
            }
        }

        Row {
            anchors { right: parent.right; verticalCenter: parent.verticalCenter }
            spacing: 4

            // Pin button
            Button {
                width: 20; height: 20
                text: note.pinned ? "U" : "P"  // Unpin/Pin icons
                onClicked: note.pinToggled(noteId, !note.pinned)
            }

            // Delete button
            Button {
                width: 20; height: 20
                text: "X"
                onClicked: note.deleteRequested(noteId)
            }
        }
    }

    // Content area
    TextArea {
        id: contentArea
        anchors {
            left: parent.left; right: parent.right
            top: header.bottom; bottom: resizeHandle.top
            margins: 4
        }
        text: note.content
        wrapMode: TextArea.Wrap
        background: null

        onTextChanged: {
            if (text !== note.content) {
                note.contentChanged(noteId, text)
            }
        }
    }

    // Resize handle (bottom-right corner)
    Rectangle {
        id: resizeHandle
        width: 16; height: 16
        anchors { right: parent.right; bottom: parent.bottom }
        color: Qt.darker(note.color, 1.2)

        MouseArea {
            anchors.fill: parent
            cursorShape: Qt.SizeFDiagCursor

            property point pressPos

            onPressed: (mouse) => {
                pressPos = Qt.point(mouse.x, mouse.y)
            }

            onPositionChanged: (mouse) => {
                var deltaX = mouse.x - pressPos.x
                var deltaY = mouse.y - pressPos.y
                note.width = Math.max(100, note.width + deltaX)
                note.height = Math.max(80, note.height + deltaY)
            }

            onReleased: {
                note.sizeChanged(noteId, note.width, note.height)
            }
        }
    }
}
```

### PersistenceService.qml (Singleton)

```qml
pragma Singleton
import QtQuick
import Quickshell.Io  // For file operations

QtObject {
    id: persistence

    readonly property string notesPath: StandardPaths.writableLocation(
        StandardPaths.ConfigLocation) + "/quickshell/ii/notes.json"

    function saveNotes(model) {
        var notes = []
        for (var i = 0; i < model.count; i++) {
            var item = model.get(i)
            notes.push({
                noteId: item.noteId,
                x: item.x,
                y: item.y,
                width: item.width,
                height: item.height,
                content: item.content,
                color: item.color,
                pinned: item.pinned,
                zOrder: item.zOrder
            })
        }

        var json = JSON.stringify(notes, null, 2)
        // Use Quickshell's file writing mechanism
        FileIO.write(notesPath, json)
    }

    function loadNotes(model) {
        model.clear()

        if (!FileIO.exists(notesPath)) {
            return
        }

        var json = FileIO.read(notesPath)
        var notes = JSON.parse(json)

        for (var i = 0; i < notes.length; i++) {
            model.append(notes[i])
        }
    }
}
```

### Integration with Overlay System

```qml
// In your existing overlay component (triggered by Super+G)
import "components" as Components

Item {
    id: overlayRoot
    visible: overlayActive  // Bound to your Super+G toggle

    // Only show non-pinned notes in overlay
    Repeater {
        model: NoteManager.noteModel

        delegate: Loader {
            active: !model.pinned  // Only load non-pinned notes

            sourceComponent: Components.Note {
                noteId: model.noteId
                x: model.x
                y: model.y
                width: model.width
                height: model.height
                content: model.content
                color: model.color
                zOrder: model.zOrder

                // Wire signals to manager
                onContentChanged: (id, text) => NoteManager.updateNote(id, {content: text})
                onPositionChanged: (id, x, y) => NoteManager.updateNote(id, {x: x, y: y})
                onSizeChanged: (id, w, h) => NoteManager.updateNote(id, {width: w, height: h})
                onDeleteRequested: (id) => NoteManager.deleteNote(id)
                onPinToggled: (id, pinned) => NoteManager.updateNote(id, {pinned: pinned})
            }
        }
    }

    // Create new note on double-click
    MouseArea {
        anchors.fill: parent
        onDoubleClicked: (mouse) => {
            NoteManager.createNote(mouse.x - 100, mouse.y - 75)
        }
    }
}
```

### Standalone Pinned Windows

```qml
// StandaloneNotes.qml - instantiate pinned notes as separate windows
import QtQuick
import Quickshell

Item {
    // Create a window for each pinned note
    Repeater {
        model: NoteManager.noteModel

        delegate: Loader {
            active: model.pinned  // Only load pinned notes

            sourceComponent: PanelWindow {
                // Quickshell PanelWindow for standalone display
                visible: true

                // Position from note data
                // Note: Exact positioning API depends on Quickshell version

                width: model.width
                height: model.height

                Components.Note {
                    anchors.fill: parent
                    noteId: model.noteId
                    content: model.content
                    color: model.color
                    zOrder: model.zOrder

                    // Wire signals same as overlay
                    onContentChanged: (id, text) => NoteManager.updateNote(id, {content: text})
                    onSizeChanged: (id, w, h) => NoteManager.updateNote(id, {width: w, height: h})
                    onDeleteRequested: (id) => NoteManager.deleteNote(id)
                    onPinToggled: (id, pinned) => NoteManager.updateNote(id, {pinned: pinned})
                }
            }
        }
    }
}
```

## Patterns to Follow

### Pattern 1: Singleton Manager with ListModel
**What:** Central NoteManager singleton holds all state in a ListModel
**When:** Multi-instance widgets that need shared state across contexts
**Why:**
- ListModel provides automatic view updates via bindings
- Singleton ensures single source of truth
- Easy to serialize/deserialize

### Pattern 2: Signal-Based State Updates
**What:** Child components emit signals; parent/manager updates model
**When:** User interactions that modify state
**Why:**
- Maintains unidirectional data flow
- Components remain reusable (not coupled to specific manager)
- Easy to debug (can log all state changes in one place)

### Pattern 3: Filtered Views via Loader
**What:** Use Repeater + Loader with `active` property bound to filter condition
**When:** Same data shown differently in different contexts (overlay vs standalone)
**Why:**
- Single data source, multiple presentations
- Loader prevents instantiation of inactive delegates
- Filter changes automatically create/destroy views

### Pattern 4: Debounced Persistence
**What:** Timer-based save delay after state changes
**When:** Frequent updates (drag operations, typing)
**Why:**
- Prevents excessive file I/O
- User sees immediate feedback
- Final state persisted reliably

## Anti-Patterns to Avoid

### Anti-Pattern 1: Direct Property Binding for Position
**What:** Binding note.x directly to model.x bidirectionally
**Why bad:** Drag operations create binding loops and performance issues
**Instead:** Use one-way binding for initial position; emit signal on drag end to update model

### Anti-Pattern 2: Separate State per Context
**What:** Overlay notes and pinned notes each managing own state
**Why bad:** Drift between states, double persistence, sync bugs
**Instead:** Single NoteManager, filtered views

### Anti-Pattern 3: Synchronous Save on Every Change
**What:** Calling saveNotes() in every property change handler
**Why bad:** File I/O blocks UI thread, especially during drags
**Instead:** Debounced save with Timer

### Anti-Pattern 4: Imperative Note Creation in Views
**What:** Views directly instantiating Note objects
**Why bad:** State escapes model, notes lost on view destruction
**Instead:** All CRUD through NoteManager; views are pure projections

## Directory Structure

```
~/.config/quickshell/ii/
  shell.qml                    # Root - imports singletons
  qmldir                       # Module definition

  components/
    Note.qml                   # Reusable note component
    NoteHeader.qml             # Optional: extracted header
    ResizeHandle.qml           # Optional: extracted resize logic

  services/
    NoteManager.qml            # Singleton - CRUD and state
    PersistenceService.qml     # Singleton - file I/O
    qmldir                     # singleton declarations

  panels/
    Overlay.qml                # Super+G overlay with notes
    StandaloneNotes.qml        # Spawns PanelWindows for pinned

  data/
    notes.json                 # Persisted note data
```

### qmldir for services/

```
module services
singleton NoteManager 1.0 NoteManager.qml
singleton PersistenceService 1.0 PersistenceService.qml
```

## Scalability Considerations

| Concern | At 10 notes | At 100 notes | At 1000 notes |
|---------|-------------|--------------|---------------|
| Memory | Negligible | ~10MB | Consider lazy loading |
| Save time | Instant | ~100ms | Chunk saves, background thread |
| Search | Linear scan OK | Add index property | Separate index structure |
| Z-order | Simple increment | Track explicitly | Use stable sort |

For typical sticky note usage (10-50 notes), no special scaling needed.

## Build Order Implications

**Suggested implementation phases:**

1. **Phase 1: Core Note Component**
   - Build Note.qml with drag/resize/edit
   - Test in isolation (hardcoded data)
   - No persistence, no manager

2. **Phase 2: State Management**
   - Build NoteManager singleton
   - Build PersistenceService
   - Connect Note signals to manager
   - JSON save/load working

3. **Phase 3: Overlay Integration**
   - Integrate with existing Super+G overlay
   - Repeater with model binding
   - Create-on-double-click

4. **Phase 4: Standalone Windows**
   - Add pinned property
   - Build StandaloneNotes.qml
   - PanelWindow integration
   - Test overlay-to-pinned flow

**Dependency chain:** Note (isolated) -> NoteManager -> Overlay Integration -> Standalone Windows

## Integration Points with Existing System

| Integration Point | What Exists | How Notes Integrate |
|-------------------|-------------|---------------------|
| Super+G overlay | Existing toggle mechanism | Add notes Repeater to overlay Item |
| Shell root | Existing shell.qml | Import services/ singletons |
| Config directory | ~/.config/quickshell/ii/ | Add components/, services/, data/ |
| Theme/styling | Existing color scheme | Notes can use same color variables |

## Sources

- QML ListModel documentation (Qt official)
- QML singleton pattern (Qt official)
- Qt Quick drag and drop patterns (Qt official)
- Quickshell PanelWindow API (verify against current Quickshell docs)

**Confidence notes:**
- Core QML patterns (ListModel, signals, Repeater): HIGH - stable Qt patterns
- Quickshell-specific APIs (PanelWindow, FileIO): MEDIUM - verify against current Quickshell version
- File path APIs: MEDIUM - StandardPaths may differ; check Quickshell docs

# Sticky Notes Widget

Desktop sticky notes for Hyprland/Quickshell. Create, edit, pin, and persist notes across reboots.

---

## Status

**Progress:** Phase 1, Plan 1 of 2 complete (~7%)
**Last worked on:** 2026-02-02
**Next step:** Phase 1, Plan 2 — Window and IPC integration with keybinds

## What Exists Today

The foundation layer is built. The data service works, the content component renders, and the module skeleton is wired up with IPC and keyboard shortcuts. What's missing is the remaining Phase 1 work (Plan 02) and the 6 phases after that.

### Files Created

| File | Purpose | Lines |
|------|---------|-------|
| `~/.config/quickshell/ii/services/StickyNotesService.qml` | Data persistence singleton — CRUD, archiving, z-ordering, debounced save | 358 |
| `~/.config/quickshell/ii/modules/ii/stickyNotes/StickyNotes.qml` | Main module — Scope wrapper, IPC handler, GlobalShortcuts | 62 |
| `~/.config/quickshell/ii/modules/ii/stickyNotes/StickyNotesPanel.qml` | Panel window that hosts all notes | — |
| `~/.config/quickshell/ii/modules/ii/stickyNotes/StickyNotesPanelContent.qml` | Panel content layout (grid of notes) | — |
| `~/.config/quickshell/ii/modules/ii/stickyNotes/StickyNoteWindow.qml` | Individual note window component | — |
| `~/.config/quickshell/ii/modules/ii/stickyNotes/StickyNoteContent.qml` | Text area with auto-scroll, debounced edits | 117 |
| `~/.config/quickshell/ii/modules/common/Directories.qml` | Modified — added `stickyNotesPath` property | — |

### Data Storage

Notes persist at `~/.local/share/quickshell/sticky-notes.json` (outside config dir, survives config resets).

Schema (version 1):
```json
{
  "version": 1,
  "nextZIndex": 5,
  "notes": [{
    "id": "lz3k5m-a8b2c9d1",
    "content": "Note text",
    "x": 115, "y": 97,
    "width": 280, "height": 220,
    "visible": true, "archived": false,
    "monitor": "", "color": "#d4c896",
    "lastActive": 1738512000000,
    "zIndex": 3
  }]
}
```

### Service API

`StickyNotesService` (singleton) provides:

| Function | Description |
|----------|-------------|
| `createNote(options)` | Create with optional content, position, color. Smart positioning avoids overlap. |
| `deleteNote(id)` | Permanent delete |
| `archiveNote(id)` | Hide but keep data |
| `restoreNote(id)` | Unarchive |
| `updateNote(id, props)` | Update any properties, emits signal per property |
| `toggleNote(id)` | Toggle visibility |
| `toggleLastActive()` | Smart toggle — shows all if hidden, toggles last active if visible, creates if none exist |
| `showAll()` / `hideAll()` | Bulk visibility |
| `bringToFront(id)` | Highest z-index |
| `getNote(id)` | Lookup |
| `getActiveNotes()` / `getArchivedNotes()` | Filtered lists |
| `getVisibleCount()` | Count visible non-archived |

### IPC Commands

Target: `stickyNotes`
```bash
qs -c ii ipc call stickyNotes toggle
qs -c ii ipc call stickyNotes open
qs -c ii ipc call stickyNotes close
qs -c ii ipc call stickyNotes create
```

### Keyboard Shortcuts

| Shortcut | Action |
|----------|--------|
| `Super + Shift + G` | Toggle sticky notes panel |
| `Super + Shift + N` | Create new sticky note |

### Color Palette

8 muted colors, randomly assigned on creation:

| Name | Hex |
|------|-----|
| Sand | `#d4c896` |
| Rose | `#c9a0a0` |
| Slate | `#9bb0c4` |
| Sage | `#a3b89c` |
| Peach | `#cdb091` |
| Mauve | `#b5a0b8` |
| Mist | `#94b5ad` |
| Stone | `#a8adb0` |

## Roadmap

7 phases. Phases 5-7 can run in parallel after Phase 2.

| Phase | Goal | Status |
|-------|------|--------|
| **1. Foundation** | Layer shell positioning + IPC | Plan 1 done, Plan 2 pending |
| **2. Multi-Note** | Multiple notes with persistence | Not started |
| **3. Overlay** | Notes in Super+G overlay with drag | Not started |
| **4. Standalone** | Pinned always-on-top windows | Not started |
| **5. Resize** | User-controlled note sizing | Not started |
| **6. Context Menu** | Right-click actions, color picker | Not started |
| **7. Text Polish** | Auto-scroll, editing refinements | Not started |

## Requirements Coverage

26 v1 requirements across 7 categories: Foundation (3), Multi-Note (5), Interaction (5), Text Editing (3), Context Menu (4), Customization (3), Display Modes (3).

Full requirements: `~/.config/hypr/.planning/REQUIREMENTS.md`
Full roadmap: `~/.config/hypr/.planning/ROADMAP.md`
Project state: `~/.config/hypr/.planning/STATE.md`

## Architecture Decisions

| Decision | Rationale |
|----------|-----------|
| Data in `~/.local/share/` (not config dir) | Survives config resets |
| FileView + debounced Timer for persistence | Matches existing Todo.qml pattern |
| Signal-based updates (not array recreation) | Performance — windows listen to `notePropertyChanged` |
| Version field in JSON | Future schema migrations |
| zIndex tracking in data | Window stacking order survives reload |

## Known Concerns

- DragHandler + layer shell compatibility unverified (may need fallback to MouseArea.drag)
- PanelWindow API specifics need verification for Phase 4
- Quickshell config is not a git repo — changes tracked in planning docs only

---

*-> See [architecture.md](architecture.md) for how this fits into the desktop*
*-> See [keybindings.md](keybindings.md) for all shortcuts including sticky notes*

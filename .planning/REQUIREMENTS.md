# Requirements: Sticky Notes Enhancement

**Defined:** 2026-02-02
**Core Value:** Sticky notes should behave like real sticky notes — always accessible, movable, and resizable without needing to open an overlay.

## v1 Requirements

Requirements for initial release. Each maps to roadmap phases.

### Foundation

- [ ] **FOUND-01**: Note widget works within Quickshell layer shell (proper positioning)
- [ ] **FOUND-02**: Note integrates with existing Super+G overlay without breaking it
- [ ] **FOUND-03**: IPC interface allows keybind control (toggle, create, etc.)

### Multi-Note Management

- [ ] **NOTE-01**: User can create multiple independent notes
- [ ] **NOTE-02**: User can delete individual notes
- [ ] **NOTE-03**: Each note has its own position, size, and content
- [ ] **NOTE-04**: Notes persist across session restarts (reboot survival)
- [ ] **NOTE-05**: Persistence uses atomic writes (no data loss on crash)

### Interaction

- [ ] **DRAG-01**: User can drag note to any position when pinned (no overlay needed)
- [ ] **DRAG-02**: Note position persists after drag
- [ ] **RESIZE-01**: User can resize note by dragging edges/corners
- [ ] **RESIZE-02**: Note has minimum size constraint (not too small to use)
- [ ] **RESIZE-03**: Note size persists after resize

### Text Editing

- [ ] **TEXT-01**: User can type text in note
- [ ] **TEXT-02**: Text area auto-scrolls to follow cursor while typing
- [ ] **TEXT-03**: Text content persists across sessions

### Context Menu

- [ ] **MENU-01**: Right-click on note opens context menu
- [ ] **MENU-02**: Context menu has "New Note" option
- [ ] **MENU-03**: Context menu has "Delete Note" option
- [ ] **MENU-04**: Context menu has color selection options

### Customization

- [ ] **CUST-01**: User can set note color (from predefined palette)
- [ ] **CUST-02**: User can adjust note opacity
- [ ] **CUST-03**: Color and opacity persist per note

### Display Modes

- [ ] **MODE-01**: Notes appear in Super+G overlay (unpinned mode)
- [ ] **MODE-02**: Notes can be pinned as standalone always-on-top windows
- [ ] **MODE-03**: User can toggle between overlay and pinned mode

## v2 Requirements

Deferred to future release. Tracked but not in current roadmap.

### Enhanced Features

- **V2-01**: Note titles/headers
- **V2-02**: Snap-to-edges when dragging
- **V2-03**: Minimize notes to tray/hidden state
- **V2-04**: Keyboard navigation between notes
- **V2-05**: Search across all notes
- **V2-06**: Timestamps (created/modified)

## Out of Scope

Explicitly excluded. Documented to prevent scope creep.

| Feature | Reason |
|---------|--------|
| Cloud sync | Adds complexity; local-only is sufficient |
| Rich text formatting | Plain text covers use case; markdown adds complexity |
| Categories/folders | Keep it simple; multiple notes is enough organization |
| Reminders/alarms | Scope creep; use dedicated reminder app |
| Collaboration | Single user tool |
| Drawing/attachments | Text-only keeps implementation simple |
| Import/export | v2 consideration if needed |

## Traceability

Which phases cover which requirements. Updated during roadmap creation.

| Requirement | Phase | Status |
|-------------|-------|--------|
| FOUND-01 | Phase 1 | Pending |
| FOUND-02 | Phase 1 | Pending |
| FOUND-03 | Phase 1 | Pending |
| NOTE-01 | Phase 2 | Pending |
| NOTE-02 | Phase 2 | Pending |
| NOTE-03 | Phase 2 | Pending |
| NOTE-04 | Phase 2 | Pending |
| NOTE-05 | Phase 2 | Pending |
| MODE-01 | Phase 3 | Pending |
| DRAG-01 | Phase 3 | Pending |
| DRAG-02 | Phase 3 | Pending |
| MODE-02 | Phase 4 | Pending |
| MODE-03 | Phase 4 | Pending |
| RESIZE-01 | Phase 5 | Pending |
| RESIZE-02 | Phase 5 | Pending |
| RESIZE-03 | Phase 5 | Pending |
| MENU-01 | Phase 6 | Pending |
| MENU-02 | Phase 6 | Pending |
| MENU-03 | Phase 6 | Pending |
| MENU-04 | Phase 6 | Pending |
| CUST-01 | Phase 6 | Pending |
| CUST-02 | Phase 6 | Pending |
| CUST-03 | Phase 6 | Pending |
| TEXT-01 | Phase 7 | Pending |
| TEXT-02 | Phase 7 | Pending |
| TEXT-03 | Phase 7 | Pending |

**Coverage:**
- v1 requirements: 26 total
- Mapped to phases: 26
- Unmapped: 0

---
*Requirements defined: 2026-02-02*
*Last updated: 2026-02-02 after roadmap creation*

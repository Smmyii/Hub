# Roadmap: Sticky Notes Enhancement

## Overview

This roadmap transforms the existing single sticky note component into a full-featured multi-note system. Starting with layer shell positioning (the critical foundation for Wayland), we build multi-note management with persistence, integrate with the existing Super+G overlay, add standalone pinned window mode, then polish with resize, context menus, and text editing enhancements. Each phase delivers a coherent, testable capability.

## Phases

**Phase Numbering:**
- Integer phases (1, 2, 3): Planned milestone work
- Decimal phases (2.1, 2.2): Urgent insertions (marked with INSERTED)

Decimal phases appear between their surrounding integers in numeric order.

- [ ] **Phase 1: Foundation** - Layer shell positioning and IPC integration
- [ ] **Phase 2: Multi-Note Management** - Multiple notes with persistence
- [ ] **Phase 3: Overlay Integration** - Notes in Super+G overlay with dragging
- [ ] **Phase 4: Standalone Mode** - Pinned always-on-top windows
- [ ] **Phase 5: Resize** - User-controlled note sizing
- [ ] **Phase 6: Context Menu** - Right-click actions and customization
- [ ] **Phase 7: Text Polish** - Auto-scroll and editing refinements

## Phase Details

### Phase 1: Foundation
**Goal**: Establish working layer shell positioning and IPC interface for keyboard control
**Depends on**: Nothing (first phase)
**Requirements**: FOUND-01, FOUND-02, FOUND-03
**Success Criteria** (what must be TRUE):
  1. User can see a note widget positioned on the layer shell (not floating randomly)
  2. User can toggle note visibility via keybind (IPC works)
  3. Super+G overlay still functions correctly (no regression)
**Plans**: 2 plans

Plans:
- [ ] 01-01-PLAN.md - Core module infrastructure with data persistence
- [ ] 01-02-PLAN.md - Window and IPC integration with keybinds

### Phase 2: Multi-Note Management
**Goal**: Users can create, manage, and persist multiple independent notes
**Depends on**: Phase 1
**Requirements**: NOTE-01, NOTE-02, NOTE-03, NOTE-04, NOTE-05
**Success Criteria** (what must be TRUE):
  1. User can create a new note and see it appear
  2. User can delete a note and see it disappear
  3. Each note maintains its own position, size, and content independently
  4. Notes survive system reboot (reopen Quickshell, notes are there)
  5. Killing Quickshell mid-edit does not corrupt saved notes (atomic writes)
**Plans**: TBD

Plans:
- [ ] 02-01: TBD

### Phase 3: Overlay Integration
**Goal**: Notes work seamlessly within the Super+G overlay with dragging capability
**Depends on**: Phase 2
**Requirements**: MODE-01, DRAG-01, DRAG-02
**Success Criteria** (what must be TRUE):
  1. User sees notes when opening Super+G overlay
  2. User can drag a note to any position within the overlay
  3. Note position persists after drag (close/reopen overlay, note is where user left it)
  4. Super+G toggle continues to work as before (regression-free)
**Plans**: TBD

Plans:
- [ ] 03-01: TBD

### Phase 4: Standalone Mode
**Goal**: Notes can be pinned as always-on-top windows outside the overlay
**Depends on**: Phase 3
**Requirements**: MODE-02, MODE-03
**Success Criteria** (what must be TRUE):
  1. User can pin a note and it becomes a standalone window above other applications
  2. User can unpin a note and it returns to overlay-only mode
  3. User can toggle between pinned and unpinned mode freely
  4. Pinned notes remain visible when overlay is closed
**Plans**: TBD

Plans:
- [ ] 04-01: TBD

### Phase 5: Resize
**Goal**: Users can resize notes by dragging edges or corners
**Depends on**: Phase 2
**Requirements**: RESIZE-01, RESIZE-02, RESIZE-03
**Success Criteria** (what must be TRUE):
  1. User can resize a note by dragging its edges or corners
  2. Note cannot be resized smaller than a usable minimum (content remains visible)
  3. Note size persists after resize (close/reopen, note is same size)
**Plans**: TBD

Plans:
- [ ] 05-01: TBD

### Phase 6: Context Menu
**Goal**: Right-click menu provides discoverable actions and note customization
**Depends on**: Phase 2
**Requirements**: MENU-01, MENU-02, MENU-03, MENU-04, CUST-01, CUST-02, CUST-03
**Success Criteria** (what must be TRUE):
  1. User can right-click a note to open a context menu
  2. User can create a new note from the context menu
  3. User can delete a note from the context menu
  4. User can change note color from the context menu
  5. User can adjust note opacity
  6. Color and opacity settings persist per note
**Plans**: TBD

Plans:
- [ ] 06-01: TBD

### Phase 7: Text Polish
**Goal**: Text editing is polished with auto-scroll following the cursor
**Depends on**: Phase 2
**Requirements**: TEXT-01, TEXT-02, TEXT-03
**Success Criteria** (what must be TRUE):
  1. User can type text into a note
  2. Text area auto-scrolls to follow cursor as user types (cursor never hidden below viewport)
  3. Text content persists across sessions
**Plans**: TBD

Plans:
- [ ] 07-01: TBD

## Progress

**Execution Order:**
Phases execute in numeric order. Phases 5, 6, 7 can proceed in parallel after Phase 2.

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. Foundation | 0/2 | Planned | - |
| 2. Multi-Note Management | 0/? | Not started | - |
| 3. Overlay Integration | 0/? | Not started | - |
| 4. Standalone Mode | 0/? | Not started | - |
| 5. Resize | 0/? | Not started | - |
| 6. Context Menu | 0/? | Not started | - |
| 7. Text Polish | 0/? | Not started | - |

---
*Roadmap created: 2026-02-02*
*Last updated: 2026-02-02*

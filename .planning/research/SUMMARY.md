# Project Research Summary

**Project:** Sticky Notes Widget Enhancement
**Domain:** Desktop widget development (QML/Quickshell on Wayland/Hyprland)
**Researched:** 2026-02-02
**Confidence:** MEDIUM (based on Qt 6/QML knowledge and codebase analysis; Quickshell-specific APIs need runtime verification)

## Executive Summary

This project transforms an existing single sticky note component in the illogical-impulse Quickshell setup into a full-featured multi-note system with draggable positioning, resizable windows, and persistent state. The recommended approach uses standard QML patterns (ListModel + Repeater for note management, DragHandler for movement, custom MouseArea handles for resizing) with JSON file persistence. The architecture centers on a singleton NoteManager that maintains all state in a ListModel, with filtered views for overlay mode (Super+G toggle) versus standalone pinned windows.

The core technical challenge is bridging QML widget patterns with Wayland layer shell semantics. Unlike regular windows, layer shell surfaces have anchor-based positioning that conflicts with standard drag-and-drop patterns. This requires manual position tracking and explicit Quickshell PanelWindow configuration. The second major risk is breaking the existing illogical-impulse dashboard integration—the current single-note is embedded in the overlay system, and changes must preserve the Super+G toggle functionality.

Success hinges on three things: (1) understanding layer shell positioning from day one, (2) creating new components rather than modifying existing dashboard files, and (3) implementing robust atomic persistence to prevent data loss. The phased approach should start with foundation work proving layer shell positioning works, then add multi-note management, then enhance with resize/context menu features.

## Key Findings

### Recommended Stack

The stack focuses on standard Qt Quick components with Quickshell-specific extensions for shell integration. No exotic dependencies—everything needed ships with Qt 6 and Quickshell.

**Core technologies:**
- **DragHandler** (Qt Quick): Modern declarative drag handling for note positioning—cleaner than MouseArea.drag, better composability with other input handlers
- **ListModel + Repeater**: Standard QML pattern for multi-instance management—automatic view updates via property bindings, efficient for 10-50 notes
- **JSON file storage**: Human-readable persistence in `~/.local/share/quickshell/sticky-notes.json`—simpler than SQLite, more structured than Qt.labs.settings
- **Custom resize handles**: MouseArea-based corner/edge handles with min/max constraints—QML has no built-in resize component
- **Quickshell PanelWindow**: Layer shell surface wrapper for standalone pinned notes—enables per-note window management
- **Menu + MenuItem** (Qt Quick Controls): Standard right-click context menu for create/delete/color actions

**Critical stack decisions:**
- DragHandler over MouseArea.drag (fallback if layer shell interaction fails)
- JSON over Qt.labs.settings (settings are flat key-value, notes need array structures)
- Custom resize handles over PinchHandler (PinchHandler is touch-gesture oriented, not mouse)

**Quickshell-specific patterns:**
- Layer shell integration via `layerrule` in Hyprland config (already established pattern in illogical-impulse)
- IPC exposure via `qs ipc call` pattern for keyboard shortcuts (matches existing cliphistService, wallpaperSelectorToggle)
- State files in `~/.local/share/quickshell/` or `~/.config/quickshell/ii/data/` (follows existing conventions)

### Expected Features

**Must have (table stakes):**
- **Multiple notes** — every sticky note app supports this; single note is a toy
- **Persistence across sessions** — notes that vanish on reboot are useless
- **Draggable positioning** — physical sticky notes can be moved; digital ones must too
- **Text input with auto-scroll** — long notes must scroll to follow cursor
- **Note creation/deletion** — basic note lifecycle management
- **Always-on-top option** — core use case is keeping notes visible while working (already exists via pin functionality)

**Should have (competitive):**
- **Resizable notes** — users control screen real estate per note (medium complexity: resize handles, constraints)
- **Right-click context menu** — discoverable actions without memorizing shortcuts (user specifically requested this)
- **Color customization** — visual organization via color-coding notes
- **Auto-save on edit** — never lose content; no explicit save action needed (debounced write on change)

**Defer (v2+):**
- Search within notes (low priority until note count grows)
- Minimize to icon/tray (nice but not essential)
- Snap to edges/grid (polish feature)
- Import/export (backup feature for later)
- Timestamp display (metadata)
- Note titles/headers (optional identification)

**Anti-features (deliberately exclude):**
- Cloud sync (auth complexity, privacy concerns)
- Rich text formatting (parsing complexity, cursor management issues)
- Note categories/folders (over-engineering for quick notes)
- Reminders/alarms (scope creep into todo territory)
- Collaboration/sharing (network complexity)
- Handwriting/drawing (requires canvas, touch input)

### Architecture Approach

The architecture uses a singleton manager pattern with centralized state and filtered views. A single NoteManager singleton holds all note data in a ListModel. Two separate view layers (overlay mode and standalone windows) filter this model based on the `pinned` property. Overlay mode shows unpinned notes in the existing Super+G toggle context; standalone mode creates PanelWindow instances for pinned notes.

**Major components:**

1. **NoteManager (singleton)** — Central orchestrator handling CRUD operations, model management, and persistence coordination. Exposes createNote(), updateNote(), deleteNote() methods. Implements debounced save timer to prevent excessive file I/O during drag operations.

2. **Note (reusable component)** — Visual representation with drag/resize interaction and content editing. Emits signals for state changes (contentChanged, positionChanged, sizeChanged, deleteRequested, pinToggled). Stateless—all state lives in NoteManager's ListModel.

3. **PersistenceService (singleton)** — File I/O abstraction handling JSON serialization/deserialization. Uses atomic writes (temp file + rename) to prevent corruption on crash. Validates JSON before writing, handles UTF-8 encoding explicitly.

4. **OverlayLayer** — Container for notes in overlay mode, integrated with existing Super+G toggle. Uses Repeater with filtered model (only unpinned notes). Handles note creation on double-click.

5. **StandaloneNoteWindow** — PanelWindow wrapper for pinned notes, creates one window per pinned note. Uses Repeater with filtered model (only pinned notes).

**Data flow is unidirectional:**
User interaction → Note emits signal → NoteManager updates ListModel → Model binding updates all views → Async save to persistence

**Key patterns:**
- Singleton manager with ListModel (multi-instance shared state)
- Signal-based state updates (maintains unidirectional flow, component reusability)
- Filtered views via Loader (single data source, multiple presentations)
- Debounced persistence (frequent updates like drag don't overwhelm I/O)

### Critical Pitfalls

Research identified 5 critical pitfalls that could cause rewrites or major issues:

1. **Layer shell vs regular window confusion** — Layer shells use anchor-based positioning, not free x/y coordinates. Standard QML drag patterns fail silently. Prevention: Use Quickshell PanelWindow with `anchors: Edges.None`, manual position tracking. This is the #1 reason developers get stuck early. Test positioning on day one.

2. **Breaking existing illogical-impulse bindings** — Modifying shared QML files breaks Super+G overlay or dashboard widgets. Prevention: Create NEW files for standalone functionality, keep existing dashboard note intact initially, test Super+G after every change. Regression testing must be continuous.

3. **State persistence without proper serialization** — State seems to save but corrupts/loses data on reload. Issues include async Settings, non-atomic writes (crash = corrupted file), JSON fails on special characters. Prevention: Atomic file writes (temp + rename), validate JSON before writing, explicit UTF-8 handling, test by killing Quickshell mid-edit (SIGKILL).

4. **Z-order and layer shell namespace collisions** — Multiple widgets overlap incorrectly or appear behind other shell elements. Using same layer shell namespace as existing components causes conflicts. Prevention: Unique namespace per widget `quickshell:stickynote-{id}`, understand Hyprland layer precedence (background < bottom < top < overlay), review existing layerrule entries.

5. **Quickshell IPC pattern violations** — Widget doesn't integrate with existing IPC patterns, can't be toggled via keybinds. Prevention: Study existing IPC services (cliphistService pattern), expose new widget as IPC-callable service (stickyNoteService with toggle/new/show/hide methods), follow naming conventions.

**Moderate pitfalls:**
- Mouse event propagation (text editing triggers drag—use separate drag handle region)
- Resize constraints not enforced (widget shrinks to 0x0 or fills screen—define min/max from start)
- Text input focus management (layer shell focus semantics differ from regular windows)
- Scroll behavior not following cursor (TextArea needs explicit scroll management)
- Right-click context menu platform issues (Wayland positioning rules stricter than X11)

## Implications for Roadmap

Based on research, the phase structure should follow a bottom-up dependency chain: prove layer shell positioning works first, then build multi-note infrastructure, then add polish features. Each phase addresses specific table stakes features while avoiding identified pitfalls.

### Phase 1: Foundation & Layer Shell Integration
**Rationale:** Must understand layer shell positioning and prove basic architecture before building features. Pitfalls #1 (layer shell confusion) and #5 (IPC patterns) are blockers for all subsequent work.

**Delivers:**
- NoteManager singleton with empty ListModel
- Single Note component with drag capability
- Quickshell PanelWindow integration working
- IPC service skeleton for keyboard shortcuts
- Manual position tracking proven

**Addresses features:**
- Draggable positioning (table stakes)
- Foundation for always-on-top (pin functionality architecture)

**Avoids pitfalls:**
- Layer shell vs window confusion (critical)
- IPC pattern violations (critical)
- Mouse event propagation (moderate—separate drag handle from start)

**Research needs:** Low—uses standard patterns, but must verify DragHandler compatibility with layer shell. May need `/gsd:research-phase` if DragHandler fails and MouseArea.drag fallback needed.

### Phase 2: Multi-Note Management & Persistence
**Rationale:** Transforms single-note into usable multi-note system. Persistence is critical—users will lose trust if notes vanish. Pitfall #3 (state persistence) must be addressed comprehensively.

**Delivers:**
- ListModel populated with multiple notes
- Note creation/deletion methods
- JSON persistence with atomic writes
- Auto-save with debounced timer
- Load notes on startup

**Addresses features:**
- Multiple notes (table stakes)
- Persistence across sessions (table stakes)
- Note creation/deletion (table stakes)
- Auto-save on edit (should-have)

**Avoids pitfalls:**
- State persistence without serialization (critical—dedicated focus)
- Configuration file overwriting (moderate—correct XDG paths)

**Research needs:** Low—JSON patterns well-documented. Standard phase, skip `/gsd:research-phase`.

### Phase 3: Overlay Integration
**Rationale:** Integrate with existing Super+G toggle system. Must not break existing dashboard. Pitfall #2 (breaking bindings) is the main risk.

**Delivers:**
- Repeater in overlay layer showing unpinned notes
- Filter logic (pinned vs unpinned)
- Create note on double-click in overlay
- Super+G toggle continues working
- Regression tests for existing overlay features

**Addresses features:**
- Integration with existing workflow (table stakes)
- Visual note distinction (table stakes via positioning)

**Avoids pitfalls:**
- Breaking existing illogical-impulse bindings (critical—test continuously)

**Research needs:** Low—integration patterns clear from codebase analysis. Standard phase.

### Phase 4: Standalone Pinned Windows
**Rationale:** Enable always-on-top mode for notes that should persist outside overlay. Uses filtered views architecture established in Phase 3.

**Delivers:**
- StandaloneNoteWindow component
- PanelWindow per pinned note
- Pin/unpin toggle button
- Layer shell namespace management

**Addresses features:**
- Always-on-top option (table stakes)

**Avoids pitfalls:**
- Z-order and namespace collisions (critical—unique namespaces per note)

**Research needs:** Medium—PanelWindow API details need verification. Consider `/gsd:research-phase` if Quickshell PanelWindow API unclear.

### Phase 5: Resize & Constraints
**Rationale:** Users need to control note size. Separate phase because resize is orthogonal to drag (different interaction model).

**Delivers:**
- Custom resize handles (bottom-right corner initially)
- Min/max size constraints
- Size persisted to model
- Cursor shape changes on hover

**Addresses features:**
- Resizable notes (should-have)

**Avoids pitfalls:**
- Resize constraints not enforced (moderate—constraints from day one)
- Mouse event propagation (moderate—resize handle separate from content)

**Research needs:** Low—custom resize pattern well-documented. Standard phase.

### Phase 6: Context Menu & Actions
**Rationale:** Improves discoverability. Right-click menu is user-requested feature. Wayland positioning needs attention (pitfall warning).

**Delivers:**
- Right-click context menu component
- Menu items: New Note, Delete Note, Pin/Unpin
- Color picker submenu
- Wayland-compatible positioning

**Addresses features:**
- Right-click context menu (should-have, explicitly requested)
- Color customization (should-have)

**Avoids pitfalls:**
- Right-click menu platform issues (moderate—test on Wayland)

**Research needs:** Low—Qt Quick Controls Menu is standard. Standard phase.

### Phase 7: Text Editing Polish
**Rationale:** Table stakes feature (text input with auto-scroll) needs proper implementation. Separate phase allows focus on UX details.

**Delivers:**
- TextArea with ScrollView/Flickable
- Cursor position tracking
- ensureVisible() on cursorPositionChanged
- Text content saved to model

**Addresses features:**
- Text input with auto-scroll (table stakes)

**Avoids pitfalls:**
- Scroll behavior not following cursor (moderate)
- Text input focus management (moderate—explicit focus handling)

**Research needs:** Low—TextArea patterns standard. Standard phase.

### Phase Ordering Rationale

**Dependency chain:**
```
Phase 1 (Foundation) → Phase 2 (Multi-note) → Phase 3 (Overlay)
                                              → Phase 4 (Standalone)
Phase 1 → Phase 5 (Resize) [parallel to 2-4]
Phase 2 → Phase 6 (Context Menu)
Phase 2 → Phase 7 (Text Polish)
```

**Why this order:**
- Foundation first: Layer shell positioning is blocker for everything (pitfall #1)
- Multi-note before integration: Need working model before integrating with overlay
- Overlay before standalone: Test integration with existing system first, then add complexity
- Resize parallel: Can develop alongside multi-note work—orthogonal concerns
- Context menu after multi-note: Menu triggers CRUD operations that need working model
- Text polish last: Core editing works from Phase 1, polish is refinement

**Grouping rationale:**
- Phases 1-2: Core infrastructure (no UI integration yet)
- Phases 3-4: Integration with existing system (two deployment modes)
- Phases 5-7: Feature enhancements (polish existing functionality)

**Pitfall avoidance:**
- Phase 1 directly addresses critical pitfall #1 (layer shell)
- Phase 2 directly addresses critical pitfall #3 (persistence)
- Phase 3 directly addresses critical pitfall #2 (breaking bindings)
- Phase 4 directly addresses critical pitfall #4 (z-order/namespaces)
- Phase 1 IPC work addresses critical pitfall #5 (IPC patterns)

### Research Flags

**Phases likely needing deeper research:**
- **Phase 1:** Possible `/gsd:research-phase` if DragHandler fails with layer shell—need to research MouseArea.drag fallback patterns
- **Phase 4:** Possible `/gsd:research-phase` if PanelWindow API unclear—need to verify Quickshell's exact API for standalone window creation

**Phases with standard patterns (skip research-phase):**
- **Phase 2:** JSON persistence—well-documented pattern, multiple sources
- **Phase 3:** Repeater + filtered views—standard QML pattern
- **Phase 5:** Custom resize handles—established pattern, no framework dependency
- **Phase 6:** Qt Quick Controls Menu—standard component
- **Phase 7:** TextArea + ScrollView—standard components

**Research confidence by phase:**
| Phase | Confidence | Notes |
|-------|------------|-------|
| 1 | MEDIUM | DragHandler + layer shell interaction unverified |
| 2 | HIGH | Standard patterns, JSON well-documented |
| 3 | HIGH | Codebase analysis shows clear integration points |
| 4 | MEDIUM | PanelWindow API specifics need verification |
| 5 | HIGH | Custom handles are standard pattern |
| 6 | HIGH | Qt Quick Controls well-documented |
| 7 | HIGH | TextArea patterns standard |

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack | MEDIUM | Qt 6/QML patterns HIGH confidence; Quickshell-specific APIs (PanelWindow, FileIO) MEDIUM—need runtime verification |
| Features | MEDIUM | Feature list derived from desktop sticky note app analysis (Windows Sticky Notes, macOS Stickies, KNotes, Xpad); complexity estimates based on QML patterns; no external web verification available |
| Architecture | MEDIUM | Core QML patterns (ListModel, signals, Repeater) HIGH confidence—stable Qt patterns; Quickshell-specific integration (layer shell, IPC) MEDIUM—inferred from codebase analysis but not directly tested |
| Pitfalls | MEDIUM-HIGH | Layer shell fundamentals HIGH (well-documented Wayland/wlroots behavior); illogical-impulse integration HIGH (based on detailed codebase analysis); QML mouse/focus handling MEDIUM (general patterns, would benefit from Quickshell-specific verification) |

**Overall confidence:** MEDIUM

**Confidence drivers:**
- Qt 6/QML fundamentals are solid (training knowledge)
- illogical-impulse architecture is well-understood (detailed codebase analysis)
- Quickshell-specific APIs not directly verified (no access to Quickshell docs during research)
- Layer shell behavior well-documented (Wayland standard)

**Confidence limiters:**
- Could not examine actual Quickshell API documentation
- Could not read existing dashboard QML files in `~/.config/quickshell/ii/`
- Exact current sticky note implementation unknown
- DragHandler + layer shell compatibility unverified

### Gaps to Address

**During planning:**
1. **Quickshell FileIO API** — Research shows `Quickshell.Io` import and `FileIO.read()/write()` pattern, but exact API needs verification before Phase 2. Consider `/gsd:research-phase` if persistence implementation blocked.

2. **PanelWindow positioning API** — Research assumes PanelWindow supports explicit x/y positioning, but Quickshell API details unclear. May need Phase 4 `/gsd:research-phase` to determine correct approach for standalone windows.

3. **Current sticky note location** — Exact location of existing single-note implementation unknown. Before Phase 3, need to locate and understand current integration to avoid breaking it.

4. **DragHandler + layer shell** — Uncertain if DragHandler works correctly with Quickshell PanelWindow. Phase 1 should test this early; fallback to MouseArea.drag if issues found.

**During implementation:**
1. **Multi-monitor positioning** — Note positions may need monitor-relative coordinates rather than absolute screen coordinates. Test multi-monitor setup during Phase 2.

2. **Layer shell layer precedence** — Need to verify Hyprland layer shell behavior: background < bottom < top < overlay. Phase 4 must test that pinned notes appear correctly above regular windows.

3. **IPC service naming** — Verify illogical-impulse IPC naming conventions before implementing. Phase 1 IPC work should follow existing patterns (cliphistService, wallpaperSelectorToggle).

4. **Theme integration** — Colors hardcoded initially; later phase should integrate with matugen-generated theme system. Defer to post-MVP.

**Validation strategies:**
- Phase 1: Build minimal prototype with single draggable note on layer shell—proves positioning works
- Phase 2: Test persistence by killing Quickshell mid-edit (SIGKILL)—proves atomic writes work
- Phase 3: Test Super+G toggle after every change—proves integration doesn't break existing system
- Phase 4: Test with 5+ pinned notes—proves namespace/z-order handling works

## Sources

### Primary Sources (HIGH confidence)

**Codebase analysis:**
- `.planning/codebase/CONCERNS.md` — Configuration management patterns, identified test code in production, missing error handling
- `.planning/codebase/ARCHITECTURE.md` — illogical-impulse structure, Quickshell integration patterns, IPC mechanisms
- `.planning/codebase/STRUCTURE.md` — File organization, module boundaries, config directory layout
- `.planning/codebase/INTEGRATIONS.md` — Hyprland integration, matugen theming, existing widget patterns
- `.planning/codebase/STACK.md` — Technology inventory (Hyprland, Quickshell, matugen, etc.)

**Project context:**
- `PROJECT.md` — Original request for sticky notes enhancement, pin/drag/resize requirements

### Secondary Sources (MEDIUM confidence)

**Qt/QML documentation (from training knowledge):**
- Qt Quick ListModel patterns — multi-instance management
- Qt Quick Input Handlers (DragHandler) — declarative drag handling
- Qt Quick Controls (Menu, TextArea, ScrollView) — UI components
- QML singleton pattern — shared state management
- QML Repeater + filtered views — dynamic instance creation

**Domain knowledge:**
- Desktop sticky note applications analysis (Windows Sticky Notes, macOS Stickies, KDE KNotes, Xpad)
- Wayland layer shell protocol — anchor-based positioning, exclusive zones
- wlroots layer shell semantics — z-order, namespace management

### Tertiary Sources (LOW confidence, needs validation)

**Quickshell-specific APIs (inferred from codebase, not verified):**
- `Quickshell.Io` module and `FileIO` API — file operations
- `PanelWindow` component — layer shell surface creation
- IPC pattern via `qs ipc call` — service/method invocation
- StandardPaths for XDG directories — state file locations

**Assumptions requiring validation:**
- DragHandler compatibility with Quickshell layer shell surfaces
- PanelWindow supports explicit x/y positioning for standalone windows
- FileIO.read()/write() API signature and error handling
- Layer shell namespace requirements for multiple instances

---

**Research completed:** 2026-02-02
**Ready for roadmap:** Yes

**Next steps for orchestrator:**
1. Review research summary and phase suggestions
2. Proceed to requirements definition phase
3. Use phase structure as starting point for ROADMAP.md
4. Flag Phase 1 and Phase 4 for potential `/gsd:research-phase` during planning
5. Ensure regression testing (Super+G toggle) in every phase

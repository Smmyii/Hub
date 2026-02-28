# Feature Landscape: Desktop Sticky Notes

**Domain:** Desktop sticky notes widget (Linux/Wayland/Quickshell)
**Researched:** 2026-02-02
**Confidence:** MEDIUM (based on analysis of established desktop sticky note applications: Windows Sticky Notes, macOS Stickies, KNotes, Xpad, and similar)

## Table Stakes

Features users expect from any sticky notes implementation. Missing these makes the product feel incomplete or broken.

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| **Multiple notes** | Every sticky note app supports this; single note is a toy | Medium | Requires note management, z-ordering |
| **Persistence across sessions** | Notes that vanish on reboot are useless | Medium | JSON/file storage, state restoration |
| **Draggable positioning** | Physical sticky notes can be moved; digital ones must too | Low | Already partially exists when pinned |
| **Text input with auto-scroll** | Long notes must scroll to follow cursor | Low | Standard TextArea behavior |
| **Note creation/deletion** | Must be able to add new notes and remove old ones | Low | UI for create/delete actions |
| **Visual note distinction** | Multiple notes must be visually distinguishable | Low | Different colors or positions |
| **Always-on-top option** | Core use case is keeping notes visible while working | Low | Already exists via pin functionality |

## Differentiators

Features that set the implementation apart. Not strictly required but provide competitive advantage and improved UX.

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| **Resizable notes** | Users control how much screen real estate each note takes | Medium | Requires resize handles, min/max constraints |
| **Right-click context menu** | Discoverable actions without memorizing shortcuts | Low | Standard pattern; user specifically requested |
| **Color customization** | Organize notes visually; match desktop aesthetic | Low | Color picker or preset palette |
| **Note titles/headers** | Quick identification of note purpose | Low | Optional text field at top |
| **Auto-save on edit** | Never lose content; no explicit save action needed | Low | Debounced write on text change |
| **Minimize to icon/tray** | Reduce clutter while keeping notes accessible | Medium | Requires tray integration or compact mode |
| **Keyboard navigation** | Power users want keyboard-only workflow | Low | Tab between notes, hotkeys for actions |
| **Note opacity control** | See through notes to content below | Low | Slider for window opacity |
| **Snap to edges/grid** | Organized layout without manual pixel-pushing | Medium | Snap zones, magnetic edges |
| **Search within notes** | Find content across all notes quickly | Medium | Text search with highlighting |
| **Timestamp display** | Know when note was created/modified | Low | Metadata display in corner or tooltip |
| **Import/export** | Backup, share, or migrate notes | Low | JSON export, plain text export |

## Anti-Features

Features to deliberately NOT build. These add complexity without proportional value for a lightweight desktop widget.

| Anti-Feature | Why Avoid | What to Do Instead |
|--------------|-----------|-------------------|
| **Cloud sync** | Adds auth complexity, privacy concerns, network dependencies | Local-only storage; user can manually backup |
| **Rich text formatting** | Markdown/HTML adds parsing complexity, cursor management issues | Plain text; use separate editor for formatted docs |
| **Note categories/folders** | Over-engineering for quick notes; adds navigation friction | Use colors for visual organization |
| **Reminders/alarms** | Scope creep into calendar/todo territory | Keep notes as passive reference |
| **Collaboration/sharing** | Network complexity, conflicts, privacy | Single-user local notes |
| **Handwriting/drawing** | Requires canvas, touch input, stylus support | Text-only; use dedicated drawing app |
| **Embedded images/attachments** | Storage complexity, rendering complexity | Text-only; link to external files if needed |
| **Note linking/wiki-style** | Database complexity, link management | Keep notes independent |
| **Version history** | Storage overhead, UI complexity | Auto-save current state only |
| **Encryption** | Key management, recovery complexity | Use OS-level disk encryption |
| **OCR/voice input** | External service dependencies, accuracy issues | Manual text entry |
| **Templates** | Over-engineering for quick notes | Start with blank note |

## Feature Dependencies

```
Core dependencies (must build in order):
  Multiple notes
    |
    +---> Note creation/deletion (requires note collection)
    |
    +---> Persistence (requires note collection to serialize)
    |
    +---> Visual distinction (requires multiple notes to distinguish)

Drag/resize (can be parallel):
  Draggable positioning (independent)
  Resizable notes (independent, but same interaction model)

UI layer (after core):
  Right-click context menu
    |
    +---> Requires: Note creation/deletion (menu triggers these)
    +---> Requires: Color customization (menu item for color)
```

## MVP Recommendation

For MVP, prioritize table stakes that transform single-note into usable multi-note system:

### Phase 1: Core Infrastructure
1. **Multiple notes support** - Foundation for everything else
2. **Note creation/deletion** - Basic note management
3. **Persistence** - Notes survive session restart

### Phase 2: Interaction Polish
4. **Draggable positioning** - Already partially exists; extend to work outside overlay
5. **Auto-scroll on text input** - Basic usability fix
6. **Right-click context menu** - Discoverable UI for note actions

### Phase 3: Enhanced Usability
7. **Resizable notes** - User-controlled sizing
8. **Color customization** - Visual organization
9. **Note opacity control** - Non-intrusive notes

### Defer to Post-MVP
- Search within notes: Low priority until note count grows
- Minimize to icon: Nice but not essential
- Snap to edges: Polish feature
- Import/export: Backup feature for later
- Timestamps: Metadata display

## Complexity Estimates

| Feature | Effort | Risk | Notes |
|---------|--------|------|-------|
| Multiple notes | 2-3 days | Low | ListModel + Repeater in QML |
| Persistence | 1-2 days | Low | JSON file read/write |
| Note creation/deletion | 1 day | Low | Model manipulation |
| Draggable positioning | 1-2 days | Medium | MouseArea drag handling; layer interaction |
| Auto-scroll | 0.5 days | Low | TextArea property |
| Right-click menu | 1 day | Low | QML Menu component |
| Resizable | 1-2 days | Medium | Resize handles, min/max constraints |
| Color customization | 0.5-1 day | Low | Color property + palette |
| Note opacity | 0.5 days | Low | opacity property |

## Platform Considerations

**Wayland/Hyprland constraints:**
- Layer shell rules affect how notes appear above/below other windows
- Quickshell panels have specific positioning behaviors
- Window decorations differ from X11; resize handles must be custom
- Multi-monitor support requires explicit handling

**Quickshell/QML advantages:**
- ListModel provides efficient note collection management
- Property bindings simplify auto-save
- Component reuse for consistent note appearance
- IPC integration already established for overlay interaction

## Sources

**Confidence notes:**
- Feature list derived from analysis of established desktop sticky note applications
- Windows Sticky Notes, macOS Stickies, KDE KNotes, Xpad (Linux), Stickies (Windows third-party)
- Complexity estimates based on QML/Quickshell patterns from codebase analysis
- No external web search verification available; estimates based on framework knowledge

---

*Feature landscape analysis: 2026-02-02*

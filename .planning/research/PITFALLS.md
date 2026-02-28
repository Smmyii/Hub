# Domain Pitfalls

**Domain:** QML Widget Development for Quickshell/illogical-impulse
**Researched:** 2026-02-02
**Focus:** Draggable, resizable, persistent widgets with Hyprland layer shell integration

## Critical Pitfalls

Mistakes that cause rewrites or major issues.

### Pitfall 1: Layer Shell vs Regular Window Confusion

**What goes wrong:** Developers assume layer shell surfaces work like regular windows. They don't. Layer shells are positioned absolutely relative to screen edges/anchors, not freely draggable by default. Attempting to use standard `x`, `y` positioning or `MouseArea` drag handlers fails silently or produces erratic behavior.

**Why it happens:** QML tutorials show `MouseArea { drag.target: parent }` patterns that work for regular QML windows but not layer shell surfaces. The developer applies the pattern, sees no movement or jumpy behavior, and doesn't understand why.

**Consequences:**
- Widget appears stuck to anchor position
- Drag attempts cause visual glitching
- Hours spent debugging "working" code that simply doesn't apply to layer shell context

**Prevention:**
- Use Quickshell's `PanelWindow` with `anchors: Edges.None` to create freely-positioned layer shell surfaces
- Position via explicit `x` and `y` properties bound to stored state
- Implement manual position tracking: capture mouse delta and update position state

**Detection:**
- Widget won't move despite correct-looking MouseArea code
- Widget snaps back to original position after release
- Position changes work in Qt preview but fail in Quickshell runtime

**Phase mapping:** Phase 1 (Foundation) - Must understand layer shell model before any implementation

---

### Pitfall 2: Breaking Existing illogical-impulse Bindings

**What goes wrong:** Modifying shared QML files (like dashboard components) inadvertently breaks the Super+G overlay or other integrated features. The developer tests their new widget but doesn't verify the overlay still works.

**Why it happens:** illogical-impulse has tightly integrated components. The sticky note widget is currently embedded in the dashboard system. Changes to make it standalone can break parent-child relationships, signal connections, or property bindings.

**Consequences:**
- Super+G overlay stops showing
- Other dashboard widgets break
- Quickshell crashes on startup
- User has to restore from backup (which fortunately exists)

**Prevention:**
- Create NEW QML files for standalone widget functionality
- Keep existing dashboard sticky note intact initially
- Test overlay functionality (Super+G) after every change
- Use Quickshell's component isolation patterns

**Detection:**
- Quickshell startup errors in journal
- Super+G keybind does nothing
- Dashboard shows partially or with missing elements

**Phase mapping:** Every phase - Regression testing must be continuous

---

### Pitfall 3: State Persistence Without Proper Serialization

**What goes wrong:** Widget state (position, size, content) seems to save but corrupts or loses data on reload. Developer stores state in a way that doesn't survive Quickshell restart or crashes mid-write.

**Why it happens:**
- Using QML `Settings` without understanding its async nature
- Writing to files without atomic save (crash = corrupted file)
- Storing state in QML properties that don't persist
- JSON serialization fails on special characters in note content

**Consequences:**
- Notes lost after reboot
- Partial saves leave corrupted state files
- Multi-note state becomes inconsistent (some notes load, others don't)

**Prevention:**
- Use atomic file writes: write to temp file, then rename
- Validate JSON before writing
- Handle text encoding explicitly (UTF-8)
- Test persistence by killing Quickshell (SIGKILL) mid-edit

**Detection:**
- Notes empty after reboot
- Error messages about JSON parsing
- Some notes load, others are missing
- File permissions issues in state directory

**Phase mapping:** Phase 2 or 3 (Persistence) - Dedicated phase for robust state management

---

### Pitfall 4: Z-Order and Layer Shell Namespace Collisions

**What goes wrong:** Multiple widget instances overlap incorrectly, or widgets appear behind other shell elements (bar, notifications). New widgets use same layer shell namespace as existing components.

**Why it happens:** Quickshell layer shell surfaces use namespaces for identification and z-ordering. illogical-impulse already uses `quickshell:*` namespace patterns. Adding widgets without unique namespaces causes conflicts.

**Consequences:**
- Sticky notes appear behind the bar
- Notes stack incorrectly (can't bring one to front)
- Layer rules in Hyprland affect wrong widgets

**Prevention:**
- Use unique namespace per widget type: `quickshell:stickynote-{id}`
- Understand Hyprland layer shell precedence: background < bottom < top < overlay
- Review existing `layerrule` entries in `hyprland/rules.conf`
- Test with multiple instances open simultaneously

**Detection:**
- Widget appears but is unclickable (behind invisible surface)
- Focus doesn't work as expected
- Blur/transparency rules apply incorrectly

**Phase mapping:** Phase 1 (Foundation) and Phase 3 (Multi-note)

---

### Pitfall 5: Quickshell IPC Pattern Violations

**What goes wrong:** Developer creates widget that doesn't integrate with existing Quickshell IPC patterns. Widget can't be toggled via keybinds, doesn't respond to `qs ipc call`, or conflicts with existing services.

**Why it happens:** illogical-impulse uses `qs -c $qsConfig ipc call {SERVICE} {METHOD}` pattern extensively (see keybinds.conf). New widgets that don't expose IPC interfaces require separate keybind mechanisms, breaking consistency.

**Consequences:**
- Inconsistent UX: some widgets toggle via Quickshell IPC, new widget needs different approach
- Can't create keybinds to show/hide notes
- Can't integrate with existing overlay toggle behavior

**Prevention:**
- Study existing IPC services in illogical-impulse (cliphistService, brightness, wallpaperSelectorToggle)
- Expose new widget as IPC-callable service
- Follow naming conventions: `stickyNoteService` with methods like `toggle`, `new`, `show`, `hide`

**Detection:**
- Keybind using `qs ipc call` fails with "unknown service"
- Widget has no keyboard shortcut support
- Can't programmatically control widget visibility

**Phase mapping:** Phase 1 (Foundation) - IPC design should be early

---

## Moderate Pitfalls

Mistakes that cause delays or technical debt.

### Pitfall 1: Mouse Event Propagation Issues

**What goes wrong:** Clicking inside the sticky note text area accidentally triggers drag. Or dragging the note accidentally types in the text area. Mouse events don't route correctly between interactive areas.

**Why it happens:** QML mouse event propagation is subtle. MouseArea `propagateComposedEvents` and `preventStealing` properties are often misunderstood. Default behavior varies by component.

**Prevention:**
- Use explicit drag handle region (title bar) separate from content area
- Set `propagateComposedEvents: true` only where needed
- Test with: click in text, drag title bar, resize corner
- Consider using `DragHandler` (Qt 5.12+) instead of MouseArea for cleaner handling

**Detection:**
- Text editing triggers window movement
- Dragging doesn't work inside text areas
- Resize handles conflict with content scrolling

**Phase mapping:** Phase 1 (Dragging) and Phase 2 (Resizing)

---

### Pitfall 2: Resize Constraints Not Enforced

**What goes wrong:** Widget can be resized to unusably small (0x0), too large (off-screen), or aspect ratio gets distorted. No minimum/maximum size constraints.

**Why it happens:** Implementing resize without boundaries. Focus on "it resizes" without "it resizes sensibly."

**Prevention:**
- Define minimum size (e.g., 150x100) that keeps content usable
- Define maximum size (e.g., 80% of screen) to prevent occlusion
- Store size in state and restore on reload
- Clamp values in resize handler before applying

**Detection:**
- Widget disappears when dragged to tiny size
- Widget covers entire screen and can't be controlled
- Widget size changes randomly on restart

**Phase mapping:** Phase 2 (Resizing)

---

### Pitfall 3: Text Input Focus Management

**What goes wrong:** Clicking the sticky note doesn't focus the text area for typing. Or clicking outside doesn't unfocus. Keyboard input goes to wrong widget.

**Why it happens:** Layer shell surfaces have different focus semantics than regular windows. Focus must be explicitly requested and managed. illogical-impulse dashboard may have existing focus management that conflicts.

**Prevention:**
- Set `exclusiveZone: 0` for floating widgets (don't reserve screen space)
- Implement `onClicked` to explicitly request focus
- Handle focus loss gracefully (save state, visual indication)
- Test with other Quickshell panels open simultaneously

**Detection:**
- Typing doesn't work after clicking note
- Note captures all keyboard input even when minimized
- Can't type in terminal while note is open

**Phase mapping:** Phase 1 (Foundation)

---

### Pitfall 4: Scroll Behavior Not Following Cursor

**What goes wrong:** User types beyond visible area, but view doesn't scroll to follow. Cursor position is lost. User can't see what they're typing.

**Why it happens:** `TextArea` in QML requires explicit scroll management. The component doesn't auto-scroll by default in all configurations.

**Prevention:**
- Wrap TextArea in `Flickable` with proper contentHeight binding
- Connect to `cursorPositionChanged` to ensure visible
- Use `ensureVisible()` or manual scroll position calculation
- Test with long notes (100+ lines)

**Detection:**
- Cursor disappears below visible area while typing
- Page down/up doesn't work
- Copy/paste of long text leaves cursor off-screen

**Phase mapping:** Phase 1 (Foundation - basic editing) or Phase 2 (enhanced UX)

---

### Pitfall 5: Right-Click Context Menu Platform Issues

**What goes wrong:** Right-click menu doesn't appear, appears in wrong location, or uses X11-style menus that look wrong on Wayland.

**Why it happens:** QML context menus have platform-specific behavior. Wayland has stricter positioning rules. Some menu implementations assume X11 popup positioning.

**Prevention:**
- Use Quickshell's menu components if available
- Position menu relative to mouse cursor at time of right-click
- Test on Wayland specifically (not Qt preview)
- Handle edge cases: menu near screen edge should stay visible

**Detection:**
- Right-click does nothing
- Menu appears at 0,0 instead of cursor position
- Menu extends off-screen

**Phase mapping:** Phase 3 (Context menu feature)

---

### Pitfall 6: Configuration File Overwriting

**What goes wrong:** Auto-save or state persistence overwrites user configuration. Or widget writes to files that other tools (nwg-displays) auto-generate.

**Why it happens:** Putting state files in wrong location. Not understanding which files are "owned" by which tools. (See CONCERNS.md about monitors.conf/workspaces.conf being auto-generated.)

**Prevention:**
- Store widget state in dedicated directory: `~/.local/state/quickshell/stickynotes/`
- Never write to `~/.config/hypr/` from widget code
- Document state file locations clearly
- Use XDG base directories correctly

**Detection:**
- Configuration resets unexpectedly
- Other tools complain about file format
- State files appear in unexpected locations

**Phase mapping:** Phase 2 (Persistence)

---

## Minor Pitfalls

Mistakes that cause annoyance but are fixable.

### Pitfall 1: Hardcoded Colors Don't Match Theme

**What goes wrong:** Sticky note uses hardcoded colors that clash with matugen-generated theme. User changes wallpaper, theme updates, but sticky notes look out of place.

**Prevention:**
- Use color references from illogical-impulse's color system
- Import colors.conf values or equivalent QML bindings
- Support theme change signal if available
- At minimum, make colors configurable

**Phase mapping:** Later phase (polish)

---

### Pitfall 2: No Keyboard Shortcuts for Common Actions

**What goes wrong:** User must always use mouse for operations. No Ctrl+N for new note, no Escape to close, no Ctrl+S visual feedback.

**Prevention:**
- Implement standard shortcuts: Ctrl+N (new), Ctrl+W (close), Escape (hide)
- Use `Shortcut` QML component with appropriate context
- Don't conflict with existing illogical-impulse keybinds (check keybinds.conf)

**Phase mapping:** Later phase (usability)

---

### Pitfall 3: No Visual Feedback During Drag/Resize

**What goes wrong:** User drags but can't tell if widget is responding. No cursor change on hover over resize handles. No visual indication of drop target.

**Prevention:**
- Change cursor on hover over drag regions
- Add subtle highlight when dragging active
- Show resize cursor on corner hover
- Consider drop shadow change during drag

**Phase mapping:** Phase 1 and 2 (polish during implementation)

---

### Pitfall 4: Memory Leaks with Multiple Notes

**What goes wrong:** Opening and closing many notes causes increasing memory usage. QML components not properly destroyed.

**Prevention:**
- Use `Component.onDestruction` to clean up
- Verify objects are garbage collected (no lingering references)
- Test with: create 10 notes, close all, check memory
- Use Loader pattern for dynamic note creation

**Phase mapping:** Phase 3 (Multiple notes)

---

## Phase-Specific Warnings

| Phase Topic | Likely Pitfall | Mitigation |
|-------------|---------------|------------|
| Foundation/Layer Shell | Layer shell vs window confusion | Study Quickshell PanelWindow docs; test positioning early |
| Dragging | Mouse event propagation | Separate drag handle from content area |
| Resizing | No size constraints | Define min/max from start |
| Persistence | Non-atomic writes, data loss | Use temp file + rename pattern |
| Multi-note | Z-order conflicts, namespaces | Unique namespace per instance |
| Context Menu | Wayland positioning | Test on actual Wayland, not preview |
| Integration | Breaking existing overlay | Test Super+G after every change |

## illogical-impulse Specific Warnings

Based on CONCERNS.md analysis:

| Existing Concern | Impact on Widget Project | Mitigation |
|------------------|-------------------------|------------|
| Config file proliferation | Don't add more .old/.new files | Use git or single backup strategy |
| Hardcoded fallbacks | Widget should have fallback if Quickshell down | Consider graceful degradation |
| Test code in production | Don't leave debug widgets active | Use conditional compilation or feature flags |
| Missing error handling | Widget should handle errors gracefully | Add try-catch equivalent patterns in QML |
| No documentation | Document new widget architecture | Add inline comments explaining design |

## Confidence Assessment

| Pitfall Category | Confidence | Rationale |
|-----------------|------------|-----------|
| Layer shell fundamentals | HIGH | Well-documented Wayland/wlroots behavior |
| illogical-impulse integration | HIGH | Based on detailed codebase analysis |
| QML mouse/focus handling | MEDIUM | Based on general Qt/QML patterns; would benefit from Quickshell-specific verification |
| Persistence patterns | MEDIUM | Standard patterns but Quickshell-specific APIs unverified |
| Multi-note z-ordering | MEDIUM | Layer shell namespace behavior is standard but implementation details vary |

## Research Gaps

- **Quickshell-specific APIs:** Could not directly examine Quickshell documentation for widget patterns. Pitfalls based on general QML + layer shell knowledge.
- **illogical-impulse dashboard internals:** Could not read actual QML files in `~/.config/quickshell/ii/`. Integration pitfalls based on architecture analysis.
- **Current sticky note implementation:** Exact location and implementation of existing sticky note unknown. May need phase-specific research.

## Recommendations for Roadmap

1. **Start with Foundation phase** that proves layer shell positioning and IPC integration before adding features
2. **Include explicit regression tests** for Super+G overlay in every phase
3. **Create backup/restore scripts** that can quickly revert Quickshell config (backup already exists at `/home/sammy/.config/hypr-backup-20260202-133654`)
4. **Plan a dedicated Persistence phase** given the criticality of not losing user notes
5. **Flag Multi-note phase** for potential deeper research into z-ordering and namespace patterns

---

*Pitfalls research: 2026-02-02*
*Sources: Codebase analysis (CONCERNS.md, ARCHITECTURE.md, STRUCTURE.md, INTEGRATIONS.md, STACK.md), PROJECT.md, QML/Wayland layer shell domain knowledge*

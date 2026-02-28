# Sticky Notes Enhancement

## What This Is

An improved sticky notes widget for the Hyprland/illogical-impulse desktop environment. The existing sticky note feature in the Super+G overlay works but has usability limitations. This project enhances it with proper window management, multi-note support, and persistence.

## Core Value

Sticky notes should behave like real sticky notes — always accessible, movable, and resizable without needing to open an overlay.

## Requirements

### Validated

- Sticky note widget exists in Super+G overlay — existing
- Note can be pinned above other windows — existing
- Note accepts text input — existing

### Active

- [ ] Draggable anytime (without needing overlay open)
- [ ] Manually resizable by user
- [ ] Auto-scroll follows cursor/typing position
- [ ] Multiple notes support
- [ ] Right-click context menu for note management (new, close, etc.)
- [ ] Persistent across reboots (notes survive session restart)
- [ ] Keep integration with Super+G overlay

### Out of Scope

- Cloud sync — local-only for simplicity
- Rich text formatting — plain text sufficient
- Note categories/folders — keep it simple
- Keyboard shortcuts for note management — right-click menu covers this

## Context

**Technical environment:**
- Hyprland window manager on Wayland
- Quickshell (QML-based) provides the UI shell via illogical-impulse config
- Quickshell config location: `~/.config/quickshell/ii/` (referenced as `$qsConfig`)
- illogical-impulse dotfiles: `~/.config/illogical-impulse/`
- Current sticky note is part of the dashboard/overlay system

**Config backup:**
- Full backup created: `/home/sammy/.config/hypr-backup-20260202-133654`

**Known constraints:**
- Must work within Quickshell/QML framework
- Must maintain compatibility with existing overlay system
- Changes should follow illogical-impulse patterns

## Constraints

- **Framework**: Must use Quickshell/QML — existing UI stack
- **Compatibility**: Must not break existing Super+G overlay functionality
- **Safety**: Original config backed up; changes should be reversible

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Keep overlay integration | User wants both: overlay access AND standalone pinned notes | — Pending |
| Right-click for note management | Simpler than keyboard shortcuts, discoverable | — Pending |
| Local persistence only | Avoids complexity of sync; notes stored locally | — Pending |

---
*Last updated: 2026-02-02 after initialization*

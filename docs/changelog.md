# Changelog

Work done with Claude, ordered newest first.

---

## 2026-02-16 — Storage diagnostics + btrfs fix

**What happened:**
- Wallpaper selector stopped changing wallpapers (image stayed same, but theme colors updated)
- Investigated and found root cause: btrfs allocation exhaustion
- Device fully allocated (1.05 MiB unallocated) despite 24 GiB free space inside chunks
- File writes failing with "No space left on device" (ENOSPC)

**Diagnosis process:**
1. Tested `switchwall.sh` manually — revealed `mv` failing with ENOSPC
2. Checked `df -h` — showed 24 GiB free (misleading with btrfs)
3. Ran `btrfs filesystem usage /` — found device 100% allocated with only 1.05 MiB unallocated
4. Identified gap: plenty of space *inside* data chunks, but no raw device space for new metadata chunks

**Fix applied:**
```bash
sudo btrfs balance start -dusage=50 /
```
Rebalanced underutilized data chunks to free up unallocated device space.

**Prevention:**
- Run periodic `btrfs balance start -dusage=20 /` when disk >85% full
- Consider `btrfsmaintenance` package for automated balancing

**Files created:**
- `~/Documents/Hub/storage.md` — comprehensive filesystem diagnostics guide

**Files modified:**
- `~/Documents/Hub/README.md` — added storage.md to navigation
- `~/Documents/Hub/changelog.md` — this entry

**Impact:**
- Wallpaper switching works immediately after balance completes
- All file write operations restored (sticky notes saves, config updates, package installs)

**Knowledge gained:**
- Btrfs has two-layer space management: device allocation vs file usage
- `df -h` shows file-level free space, not device-level allocation
- "No space left" with apparent free space = allocation fragmentation
- Regular balancing essential for long-term btrfs health

---

## 2026-02-12 — Hub expansion + architecture doc

**What changed:**
- Built full architecture document covering Hyprland, Quickshell, Kitty, Fish, and how they connect
- Created sticky notes project doc (status, API reference, roadmap, architecture decisions)
- Updated README to Desktop Hub format with stack diagram, active projects, and navigation table
- Added sticky notes shortcuts to keybindings doc
- Updated changelog with sticky notes project history

**Files created:**
- `~/Documents/Hub/architecture.md` — full desktop file map and component details
- `~/Documents/Hub/sticky-notes.md` — sticky notes widget project reference

**Files modified:**
- `~/Documents/Hub/README.md` — expanded to Desktop Hub format
- `~/Documents/Hub/keybindings.md` — added Sticky Notes section
- `~/Documents/Hub/changelog.md` — backfilled sticky notes history

---

## 2026-02-12 — Kitty vim cleanup + Hub creation

**What changed:**
- Removed vim-style split navigation (`Ctrl+Shift+H/J/K/L`) from Kitty
- Replaced with arrow-key navigation: `Ctrl+Shift+Arrow` for split nav, `Ctrl+Alt+Arrow` for resize
- Created initial documentation hub at `~/Documents/Hub/`

**Files modified:**
- `~/.config/kitty/kitty.conf` — replaced vim keybindings with arrow keys

**Files created:**
- `~/Documents/Hub/README.md`
- `~/Documents/Hub/keybindings.md`
- `~/Documents/Hub/changelog.md`

---

## 2026-02-11 — Kitty terminal upgrade

**What changed:**
- Added tab bar with powerline style (rounded), color-coded to match Hyprland theme
- Added split/layout support (tall, fat, splits, stack layouts)
- Added tab keybindings (new tab, close, cycle, jump, move, rename)
- Added split keybindings (create, navigate, resize, zoom)
- Added copy, search, scroll, and zoom keybindings
- Matched terminal colors to Hyprland theme (cyan accent, dark background)

**Files modified:**
- `~/.config/kitty/kitty.conf` — expanded from 37 to 104 lines

**Backup:**
- `~/.config/kitty/kitty.conf.backup` — original config

---

## 2026-02-02 — Sticky Notes widget (Phase 1, Plan 1)

**What changed:**
- Initialized sticky notes project with full planning (PROJECT.md, REQUIREMENTS.md, ROADMAP.md)
- Built StickyNotesService — data persistence singleton with CRUD, archiving, z-ordering
- Built StickyNoteContent — text area with auto-scroll and debounced save
- Built StickyNotes module — Scope wrapper with IPC handler and GlobalShortcuts
- Built StickyNotesPanel, StickyNotesPanelContent, StickyNoteWindow components
- Added `stickyNotesPath` to Directories.qml
- Data stored at `~/.local/share/quickshell/sticky-notes.json` (survives config resets)
- 8-color muted palette: Sand, Rose, Slate, Sage, Peach, Mauve, Mist, Stone

**Files created:**
- `~/.config/quickshell/ii/services/StickyNotesService.qml`
- `~/.config/quickshell/ii/modules/ii/stickyNotes/StickyNotes.qml`
- `~/.config/quickshell/ii/modules/ii/stickyNotes/StickyNotesPanel.qml`
- `~/.config/quickshell/ii/modules/ii/stickyNotes/StickyNotesPanelContent.qml`
- `~/.config/quickshell/ii/modules/ii/stickyNotes/StickyNoteWindow.qml`
- `~/.config/quickshell/ii/modules/ii/stickyNotes/StickyNoteContent.qml`
- `~/.config/hypr/.planning/PROJECT.md`
- `~/.config/hypr/.planning/REQUIREMENTS.md`
- `~/.config/hypr/.planning/ROADMAP.md`

**Files modified:**
- `~/.config/quickshell/ii/modules/common/Directories.qml` — added stickyNotesPath

**Backup:**
- Full config backup at `/home/sammy/.config/hypr-backup-20260202-133654`

---

## 2026-02-02 — Hyprland codebase mapping

**What changed:**
- Ran full codebase analysis on Hyprland configuration
- Generated architecture, structure, stack, integrations, conventions, concerns, and testing docs

**Files created:**
- `~/Documents/Config/hypr/.planning/codebase/ARCHITECTURE.md`
- `~/Documents/Config/hypr/.planning/codebase/STRUCTURE.md`
- `~/Documents/Config/hypr/.planning/codebase/STACK.md`
- `~/Documents/Config/hypr/.planning/codebase/INTEGRATIONS.md`
- `~/Documents/Config/hypr/.planning/codebase/CONVENTIONS.md`
- `~/Documents/Config/hypr/.planning/codebase/CONCERNS.md`
- `~/Documents/Config/hypr/.planning/codebase/TESTING.md`

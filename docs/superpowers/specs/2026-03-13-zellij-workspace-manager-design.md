# Zellij Workspace Manager — Design Spec

**Date:** 2026-03-13
**Status:** Approved

## Overview

A native Zellij plugin (Rust/WASM) that brings cmux-style workspace management to Linux. A persistent sidebar shows all open projects with live status, Claude Code notification badges, and git branch info. All workspaces stay alive in the background — switching is instant.

---

## Architecture

**Single Zellij session** with one plugin pane pinned to the left as a sidebar. All workspaces run within this session. The plugin replaces Zellij's native tab bar as the primary navigation surface.

**The sidebar pane is rendered entirely by the plugin — no shell runs inside it.** This is a pure plugin-rendered UI, which means Fish keybinds never fire there. Ctrl+Z in the sidebar is handled exclusively by the plugin's keybind layer.

**Two components:**

1. **`workspace-manager` plugin** — Rust/WASM. Renders the sidebar, manages workspace state, handles keybinds, maintains undo stack, listens for `wm-notify` pipe messages.
2. **`wm-notify` CLI** — small shell script. Wired into Claude Code hooks. Sends pipe messages to the plugin via `zellij pipe --plugin workspace-manager`.

---

## Workspace Model

Workspaces are defined in `~/.config/zellij/workspace-manager/workspaces.kdl`.

```kdl
workspace "Nano" {
    root "~/Nano"
    pane command="claude" name="claude-1"
    pane command="claude" name="claude-2"
    pane command="ssh sammy@vps" name="ssh:vps"
    pane name="git"
}
```

**State persistence:** Runtime state (open workspaces, notification badges, undo stack) is written to `~/.config/zellij/workspace-manager/state.json` on every change. JSON is used (not KDL) because serde_json makes programmatic read/write ergonomic for machine-owned state. The plugin's KV store is used as a cache only; the file is the source of truth and survives Zellij restarts.

**Creating a workspace:**
- `n` in the sidebar prompts for name + root dir, then launches the workspace
- `s` snapshots the current active workspace layout into `workspaces.kdl`, overwriting the existing entry for that workspace after a confirmation prompt. Panes with a `command` are snapshotted with their command; bare panes (no command) are snapshotted as `pane name="<name>"`. The confirmation prompt shows what will be overwritten.

**Deleting a workspace (`d`):**
- If any pane in the workspace has a running process, the plugin shows a warning listing those panes and asks for confirmation before proceeding.
- On confirm: all panes in the workspace are closed and the workspace entry is removed from `workspaces.kdl`.
- Delete is **not undoable**.

**All workspaces stay alive** (keep-alive model). No suspend/restore. RAM scales with number of open workspaces — no Electron overhead.

---

## Sidebar UI

Fixed-width pane of **28 columns** (configurable in `workspaces.kdl` via `sidebar-width 28`). Always visible on the left.

Each workspace entry shows:
- Name
- Git branch
- Active process badges (`claude ×2`, `ssh vps`, etc.) or `idle`
- Amber notification dot + `⏳ waiting` badge when Claude Code needs input

**Active workspace:** highlighted with a purple left border.
**Waiting workspace:** amber left border + dot indicator.

**Keybinds (sidebar focused):**

| Key | Action |
|-----|--------|
| `↑ ↓` | Navigate |
| `Enter` | Switch to workspace |
| `Tab` | Jump to next waiting workspace |
| `n` | New workspace |
| `d` | Delete workspace (with confirmation if processes running) |
| `s` | Snapshot current layout (with confirmation) |
| `Ctrl+Z` | Undo last layout operation |

**Notifications are sidebar-only.** Zellij's plugin model does not support drawing overlays over other panes. When a background workspace has a waiting Claude Code, its sidebar entry shows the amber indicator. Switching to that workspace clears the badge.

---

## Claude Code Integration

### Message Protocol

`wm-notify` reads `$ZELLIJ_PANE_ID` (injected by Zellij into every pane's environment) and sends a structured pipe message:

```
waiting pane_id=<ZELLIJ_PANE_ID>
active  pane_id=<ZELLIJ_PANE_ID>
```

The plugin maintains a `pane_id → workspace` map built from its own tab/pane state. Unknown pane IDs (panes not belonging to any managed workspace) are silently ignored.

### Hooks

```json
"hooks": {
  "Stop":        [{ "type": "command", "command": "wm-notify waiting" }],
  "PreToolUse":  [{ "type": "command", "command": "wm-notify active" }]
}
```

### Notification State Machine

```
idle → active  (SessionStart: Claude Code session opens)
active → waiting  (Stop: Claude finishes a turn, waits for user input)
waiting → active  (PreToolUse: user has responded, Claude is processing)
any → idle  (user switches to the workspace — badge is cleared on focus)
```

Hook patching is automated during install — the user does not manually edit settings.json.

---

## Undo Stack

The plugin subscribes to Zellij's `PaneUpdate` and `TabUpdate` events and maintains an in-memory operation log.

**Undoable operations:**

| Operation | Undo action |
|-----------|-------------|
| Close pane | Reopen in same position, rerun startup command |
| Close tab | Reopen tab with all panes restored |
| Rename workspace | Restore previous name |
| Reorder workspaces | Restore previous order |

**Not undoable:** Delete workspace.

**Scope:** Ctrl+Z fires when the sidebar pane is focused. Because the sidebar is a pure plugin-rendered pane (no shell), there is no Fish keybind conflict. In terminal panes, Ctrl+Z passes through as SIGTSTP — no conflict.

---

## Ctrl+Z While Typing

One line added to `~/.config/fish/config.fish` at install time:

```fish
bind \cZ undo
```

Fish keybinds fire only when the shell is idle at the prompt. When a process is running, the kernel sends SIGTSTP directly — this binding does not interfere.

**Rationale:** Nordic keyboard layout makes `Ctrl+_` (fish's default undo) inaccessible. `Ctrl+Z` is the natural choice and is safe to remap at the prompt level.

---

## Project Location

New standalone project directory: `~/projects/workspace-manager/`

Not under Hub (this is a personal tool, not a Hub-managed project). Larry:Desktop is the natural agent to build this given the desktop/config domain.

---

## Out of Scope

- Restoring live process state (OS limitation — not possible for arbitrary processes)
- Cross-machine sync
- Mouse support in sidebar (keyboard-only for now)
- Notifications for tools other than Claude Code (can be added later)

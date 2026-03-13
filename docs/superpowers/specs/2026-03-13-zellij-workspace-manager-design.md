# Zellij Workspace Manager — Design Spec

**Date:** 2026-03-13
**Status:** Approved

## Overview

A native Zellij plugin (Rust/WASM) that brings cmux-style workspace management to Linux. A persistent sidebar shows all open projects with live status, Claude Code notification badges, and git branch info. All workspaces stay alive in the background — switching is instant.

---

## Architecture

**Single Zellij session** with one plugin pane pinned to the left as a sidebar. All workspaces run within this session. The plugin replaces Zellij's native tab bar as the primary navigation surface.

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

**Runtime state** is persisted in the plugin's KV store (survives Zellij restarts).

**Creating a workspace:**
- `n` in the sidebar prompts for name + root dir, then launches the workspace
- `s` snapshots the current active workspace layout into the config file

**All workspaces stay alive** (keep-alive model). No suspend/restore. RAM scales with number of open workspaces — no Electron overhead.

---

## Sidebar UI

Fixed-width pane (~200px) on the left. Always visible.

Each workspace entry shows:
- Name
- Git branch
- Active process badges (`claude ×2`, `ssh vps`, etc.) or `idle`
- Amber notification dot + `⏳ waiting` badge when Claude Code needs input

**Active workspace:** highlighted with a purple left border.
**Waiting workspace:** amber left border + glow dot.

**Keybinds (sidebar focused):**

| Key | Action |
|-----|--------|
| `↑ ↓` | Navigate |
| `Enter` | Switch to workspace |
| `Tab` | Jump to next waiting workspace |
| `n` | New workspace |
| `d` | Delete workspace |
| `s` | Snapshot current layout |
| `Ctrl+Z` | Undo last layout operation |

A toast notification appears in the bottom-right of the active workspace when a background workspace needs attention.

---

## Claude Code Integration

Wired via Claude Code hooks in `~/.claude/settings.json`:

```json
"hooks": {
  "Stop":         [{ "type": "command", "command": "wm-notify waiting" }],
  "SessionStart": [{ "type": "command", "command": "wm-notify active" }]
}
```

`wm-notify` calls `zellij pipe --plugin workspace-manager -- <event>`. The plugin receives the message, identifies which pane/workspace it came from, and updates the sidebar badge.

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

**Scope:** Ctrl+Z only fires when the sidebar pane is focused. In terminal panes, Ctrl+Z passes through as SIGTSTP (process suspension) — no conflict.

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

# Sammy's Desktop Hub

**Your desktop. Your reference. Find anything in seconds.**

---

## How to Use This

Each file covers one domain. When you need to find something, check the relevant file. Files cross-reference each other with `-> See [filename]` links.

---

## Quick Navigation

| File | What's Inside | When to Read |
|------|--------------|--------------|
| [architecture.md](architecture.md) | Full file tree, how Hyprland/Quickshell/Kitty connect, data flow | "Where does X live?" |
| [keybindings.md](keybindings.md) | All keyboard shortcuts — Hyprland, Kitty, Sticky Notes | "What's the shortcut for X?" |
| [sticky-notes.md](sticky-notes.md) | Sticky notes widget — status, API, roadmap, architecture decisions | "What's the state of sticky notes?" |
| [storage.md](storage.md) | Filesystem diagnostics, btrfs troubleshooting, disk space issues | "Why is something failing to write?" |
| [changelog.md](changelog.md) | History of all configuration work done with Claude | "What changed and when?" |

---

## Desktop Stack

```
Hyprland (compositor) → Quickshell (UI shell) → Kitty (terminal) → Fish (shell)
     ↑                       ↑
  keybinds              IPC commands
  window rules          services + modules
  scripts               sticky notes widget
```

## Config Locations

| Component | Path | What It Does |
|-----------|------|-------------|
| Hyprland | `~/.config/hypr/` | Window management, keybindings, display layout |
| Quickshell | `~/.config/quickshell/ii/` | Desktop shell UI — bar, panels, overlays, widgets |
| Kitty | `~/.config/kitty/kitty.conf` | Terminal — tabs, splits, theme |
| Fish | `~/.config/fish/` | Shell prompt and environment |
| Sticky Notes data | `~/.local/share/quickshell/sticky-notes.json` | Note persistence (outside config dir) |
| Planning docs | `~/Documents/Hub/.planning/` | Project roadmaps, requirements, state |
| This hub | `~/Documents/Hub/` | Central documentation |

## Theme Colors

All components share this palette:

| Role | Hex | Used In |
|------|-----|---------|
| Background | `#121318` | Hyprland, Kitty tab bar |
| Text | `#e2e2e9` | Kitty inactive tabs, general text |
| Accent (active) | `#0DB7D4` | Active borders, active tabs, highlights |
| Inactive border | `#313136` | Inactive borders, inactive splits |
| Secondary accent | `#b0c6ff` | Bell border, secondary highlights |

## Active Projects

| Project | Status | Details |
|---------|--------|---------|
| Sticky Notes widget | ~7% — Phase 1 in progress | [sticky-notes.md](sticky-notes.md) |
| Kitty terminal config | Complete | [architecture.md](architecture.md#kitty--terminal-emulator) |
| Hyprland config | Maintained | [architecture.md](architecture.md#hyprland--window-manager) |

---

*Created: 2026-02-12*

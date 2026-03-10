# Hyprland 0.54 Update — What Happened

**Date:** 2026-03-03
**Outcome:** Downgraded back to 0.53.3. Pinned in pacman.

---

## What I did

Full system update (492 packages). Hyprland went from 0.53.3 to 0.54.0.

## What broke

### 1. Config errors — finger count
Lines 175-176 in `general.conf` (`gesture_distance` and `gesture_positive` under `hyprexpo` plugin) were removed in 0.54. Easy fix — just delete those lines.

### 2. Desktop lag
The entire compositor felt sluggish — opening/closing windows left visual outlines, workspace switching was choppy. Dual monitor setup (eDP-2 165Hz + HDMI-A-1 144Hz) with mixed VRR probably didn't help. This is likely a renderer regression in 0.54's rewrite, not a config issue.

### 3. Three-finger swipe to move windows between monitors
`gesture = 3, swipe, move` still works but now only rearranges within the current monitor's layout. It no longer pushes windows across monitors. The old cross-monitor behavior seems gone.

### 4. Widget colors not updating with wallpaper
QuickShell clock and workspace widgets stuck on wrong colors. The Material You pipeline (`switchwall.sh` → `generate_colors_material.py`) was generating colors, but something in the rendering or color application was off. Didn't fully debug this.

## What I fixed / changed

- Removed `gesture_distance` and `gesture_positive` from hyprexpo config (those are gone in 0.54)
- Added `workspace_swipe = true` to gestures block → turns out that was also removed in 0.54, caused more errors
- Added `Super+Shift+Arrow` keybinds in `custom/keybinds.conf` to move windows between monitors (these are keepers)
- Downgraded hyprland 0.54.0 → 0.53.3 and hyprwire 0.3.0 → 0.2.1
- Restored original `general.conf` from backup
- Pinned in `/etc/pacman.conf`: `IgnorePkg = hyprland hyprwire linux-cachyos linux-cachyos-headers linux-cachyos-nvidia-open`
- Booting LTS kernel manually from GRUB (not set as default yet)

## Backup location

`~/Documents/backup-pre-update-20260303-115601/`
- `hypr-config/` — full Hyprland config snapshot
- `keybinds/` — separate keybind files
- `quickshell-config/` — QuickShell runtime
- `hub/` — Hub planning state

## The actual lag culprit — kernel 6.19.5

After downgrading Hyprland, the lag persisted. GPU was idle (1W, 0% util), so it wasn't NVIDIA.
Booted into LTS kernel (6.18.15) — lag completely gone. The main kernel (linux-cachyos 6.19.5) + NVIDIA 590.48 open modules don't play well together on this setup (RTX 4060 Laptop).

**Current state:** LTS kernel alone wasn't enough. Lag only appeared with second monitor connected — NVIDIA 590.48 dual-monitor regression confirmed.

## Final fix — btrfs snapshot rollback

LTS kernel fixed single-monitor lag, but dual-monitor was still broken. NVIDIA 580 → 590 driver jump is the real culprit. Too many NVIDIA packages to downgrade cleanly, so rolling back via btrfs snapshot.

`@home` is a separate subvolume — rollback only affects system packages, not ~/Documents or configs.

**After rollback, pin these in pacman.conf:**
`IgnorePkg = hyprland hyprwire linux-cachyos linux-cachyos-headers linux-cachyos-nvidia-open nvidia-utils lib32-nvidia-utils opencl-nvidia libva-nvidia-driver`

## When to try again

**Hyprland 0.54:** Wait for 0.54.1+. Check for renderer fixes for mixed-refresh dual monitors.
**Kernel 6.19.x:** Wait for 6.19.6+ or 6.20.
**NVIDIA 590.x:** The big one. Wait for 590.48.02+ or 591. Check CachyOS/NVIDIA forums for RTX 4060 Laptop dual-monitor fixes.

Remove pins with: `sudo sed -i 's/^IgnorePkg.*/##IgnorePkg =/' /etc/pacman.conf`

## Interesting stuff in 0.54 worth revisiting

- **Per-workspace layouts** — run different tiling engines per workspace
- **iGPU improvements** — 50-500% perf gains, could make iGPU-only desktop viable
- **Scroll and Monocle layouts** — now built into core

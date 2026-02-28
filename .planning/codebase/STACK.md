# Technology Stack

**Analysis Date:** 2026-02-02

## Languages

**Primary:**
- Bash - Shell scripts for system integration and utilities (version 4.x assumed from shebang usage)
- Configuration Language - Hyprland/Quickshell declarative config syntax (`.conf` files)

**Secondary:**
- GLSL - OpenGL Shading Language for visual effects and filters
- SQL - Data queries for system information via Ollama API

## Runtime

**Environment:**
- Wayland display server (not X11)
- Hyprland window manager v1.x (compositing Wayland window manager)

**System Requirements:**
- systemd for dbus and user activation environment
- GeoClue 2.0 for location services
- D-Bus for inter-process communication

## Frameworks

**Core:**
- **Hyprland** - Tiling Wayland window manager
  - Config: `hyprland.conf`, `hyprland/*.conf`, `custom/*.conf`
  - Entry point for all display/window management

- **Quickshell (qs)** - QML-based desktop shell framework
  - Config: Referenced as `$qsConfig` (default: `ii`)
  - Location: `~/.config/quickshell/ii/`
  - Used for: Bar, dashboard, panels, search, clipboard UI, emoji picker

- **Hypridle** - Hyprland idle management daemon
  - Config: `hypridle.conf`
  - Purpose: Lock screen triggers, sleep states

**Audio/Media:**
- **EasyEffects** - PipeWire audio effects processor
  - Runs as background service in audio processing mode

**Supporting Services:**
- **Gnome Keyring** - Credential/secret management via `gnome-keyring-daemon`
- **Cliphist** - Wayland clipboard history manager
- **Hyprpicker** - Color picker utility
- **Grim/Slurp** - Screenshot and region selection tools
- **Tesseract** - OCR (Optical Character Recognition) engine
- **Wlogout** - Session menu (logout/reboot/shutdown)

## Key Dependencies

**Critical System Packages:**
- `curl` - HTTP client for API calls (Ollama, image upload to uguu.se)
- `jq` - JSON processor for parsing API responses
- `hyprctl` - Hyprland IPC client for runtime queries and dispatch
- `wl-copy` / `wl-paste` - Wayland clipboard utilities
- `brightnessctl` - Display brightness control
- `wpctl` - PipeWire volume/audio control
- `dbus-update-activation-environment` - D-Bus environment sync
- `hyprpm` - Hyprland plugin manager (for loading plugins/extensions)

**Development/Utility:**
- `fuzzel` - Launcher/menu (fallback when Quickshell unavailable)
- `xdg-open` - Standard app launcher for URLs/files
- `imagemagick` (implied by `grim`) - Image manipulation
- `pgrep` / `ps` - Process introspection utilities

## Configuration

**Environment Variables:**
- `ELECTRON_OZONE_PLATFORM_HINT=auto` - Electron Wayland support
- `QT_QPA_PLATFORM=wayland` - Qt Wayland backend
- `QT_QPA_PLATFORMTHEME=kde` - Qt KDE theme integration
- `XDG_MENU_PREFIX=plasma-` - KDE Plasma menu compatibility
- `TERMINAL=kitty -1` - Terminal emulator specification
- `ILLOGICAL_IMPULSE_VIRTUAL_ENV=~/.local/state/quickshell/.venv` - Quickshell Python venv

**Build/Load:**
- `hyprpm reload` - Plugin reloading at startup
- Modular sourcing: Files source each other in layered fashion
  - Base configs in `hyprland/`
  - User overrides in `custom/`
  - Workspace/monitor config in separate files

## File Organization

**Configuration Files (`.conf`):**
- Main: `hyprland.conf` (master entry point that sources all others)
- Environment: `hyprland/env.conf`, `custom/env.conf`
- Execution/startup: `hyprland/execs.conf`, `custom/execs.conf`
- General settings: `hyprland/general.conf`, `custom/general.conf`
- Window rules: `hyprland/rules.conf`, `custom/rules.conf`
- Colors: `hyprland/colors.conf`
- Keybindings: `hyprland/keybinds.conf`, `custom/keybinds.conf`
- Monitor/workspace: `monitors.conf`, `workspaces.conf`
- Lock screen: `hyprlock.conf`, `hypridle.conf`

**Shell Scripts (`.sh`):**
- AI integration: `hyprland/scripts/ai/`
  - `show-loaded-ollama-models.sh` - Queries Ollama API for active LLM models
  - `primary-buffer-query.sh` - AI text processing/summarization
- System utilities: `hyprland/scripts/`
  - `launch_first_available.sh` - Multi-command launcher (failover pattern)
  - `start_geoclue_agent.sh` - Geolocation service initialization
  - `snip_to_search.sh` - Screenshot-to-Google-Lens integration
  - `workspace_action.sh` - Workspace manipulation dispatch
  - `zoom.sh` - Cursor zoom control
  - `fuzzel-emoji.sh` - Emoji picker fallback
- Wallpaper: `custom/scripts/__restore_video_wallpaper.sh`

**Shaders (`.frag`):**
- Visual effects in `shaders/`
  - Chromatic aberration, CRT effect, color inversion, solarized theme

## Platform Requirements

**Development:**
- Linux with Wayland support
- Hyprland 1.x installed and functional
- D-Bus daemon running
- systemd for activation environments

**Runtime:**
- Wayland compositor (Hyprland)
- PipeWire sound server (with easyeffects)
- X11/Wayland protocol support in applications
- Optional but recommended: Ollama server (localhost:11434) for AI features
- Optional: GeoClue 2.0 agent for location services

---

*Stack analysis: 2026-02-02*

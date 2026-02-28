# External Integrations

**Analysis Date:** 2026-02-02

## APIs & External Services

**AI/LLM Services:**
- **Ollama** - Local LLM inference server
  - SDK/Client: `curl` HTTP client (bash scripts)
  - URL: `http://localhost:11434` (configurable via `-u/--ollama_url` flag)
  - Endpoints:
    - `GET /api/tags` - List available models
    - `POST /api/show` - Get model details and modelfile
  - Used by: `hyprland/scripts/ai/show-loaded-ollama-models.sh` for model discovery
  - Auth: None (assumed localhost-only)
  - Reference: `hyprland/scripts/ai/license_show-loaded-ollama-models.txt`

**Image Upload/Sharing:**
- **uguu.se** - Temporary image hosting and CDN
  - Service URL: `https://uguu.se/upload`
  - Method: Multipart form upload (`files[]=@filename`)
  - Response: JSON with URL in `.files[0].url`
  - Used by: `hyprland/scripts/snip_to_search.sh` for screenshot sharing
  - Auth: None (public API)

**Search Services:**
- **Google Lens** - Image-based search
  - Service: Web-based, accessed via `https://lens.google.com/uploadbyurl?url={imageLink}`
  - Integration: Screenshot → upload to uguu.se → open in Google Lens
  - Trigger: `Super+Shift+A` keybinding (regional screenshot search)
  - Reference: `hyprland/scripts/snip_to_search.sh`

## Data Storage

**Clipboard Management:**
- **Wayland Clipboard (wl-paste/wl-copy)**
  - Storage: In-memory clipboard ring
  - Tool: `cliphist` for clipboard history persistence
  - Integration: `wl-paste --watch` monitors text/image clipboard changes
  - Triggers Quickshell update via IPC: `qs -c $qsConfig ipc call cliphistService update`
  - Reference: `hyprland/execs.conf` lines 18-19

**Credential Storage:**
- **Gnome Keyring**
  - Implementation: `gnome-keyring-daemon --start --components=secrets`
  - Purpose: Stores system secrets, SSH keys, passwords
  - Used by: Wayland authentication, window manager credentials
  - Reference: `hyprland/execs.conf` line 7

**File Storage:**
- **Local filesystem only**
  - Screenshot storage: `$(xdg-user-dir PICTURES)/Screenshots/`
  - Config storage: `~/.config/hypr/`, `~/.config/quickshell/`
  - Temporary files: `/tmp/` for images, OCR results
  - Wallpaper storage: Referenced by wallpaper selector (location not explicitly shown)

**Caching:**
- **None** (direct API calls, no caching layer)

## Authentication & Identity

**Local Services:**
- **D-Bus** - Inter-process communication authority
  - Components: `dbus-update-activation-environment`
  - Purpose: System service activation, IPC messages to Quickshell
  - Reference: `hyprland/execs.conf` lines 9-10

**System Integration:**
- **Custom authentication** - None detected
- Uses system-level Wayland/Hyprland authentication
- Gnome Keyring handles credential management

## Monitoring & Observability

**Error Tracking:**
- None detected

**Logging:**
- Console-based logging via bash `echo` statements
- Startup messages printed to terminal/journal
- Example: `hyprland/scripts/ai/show-loaded-ollama-models.sh` outputs status to stdout
- No centralized log aggregation detected

**Debug Output:**
- Process monitoring: `ps aux`, `pgrep` for detecting running services
- Status queries: `hyprctl` IPC for window manager state
- Model introspection: Ollama API calls for detecting active models

## CI/CD & Deployment

**Hosting:**
- Local desktop environment (personal workstation)
- No cloud deployment

**Reload/Update Mechanism:**
- `hyprpm reload` - Plugin manager reload
- `Ctrl+Super+R` keybinding - Restart shell widgets (Quickshell/AGS)
  - Command: `killall ags agsv1 gjs ydotool qs quickshell; qs -c $qsConfig &`
- Config reload: Manual via editing `hyprland.conf` or `custom/*.conf` files

**Version Control:**
- Not detected (no .git, package.json, or version manifest)

## Environment Configuration

**Required Environment Variables:**
- `TERMINAL` - Terminal emulator command (default: `kitty -1`)
- `qsConfig` - Quickshell config variant (default: `ii`)
- `XDG_DATA_DIRS` - Data directory paths for Flatpak/system apps

**Optional Environment Variables:**
- `ELECTRON_OZONE_PLATFORM_HINT=auto` - Electron Wayland compatibility
- `QT_QPA_PLATFORM=wayland` - Qt Wayland backend
- `QT_QPA_PLATFORMTHEME=kde` - Qt theme
- `SLURP_ARGS` - Additional slurp region selection arguments (referenced in OCR keybind)

**Secrets Location:**
- Gnome Keyring daemon manages secrets in encrypted storage
- No explicit `.env` files detected
- System-wide via `dbus-update-activation-environment`
- Reference: `hyprland/execs.conf` lines 9-10, 7

## Webhooks & Callbacks

**Incoming:**
- None detected

**Outgoing:**
- **Quickshell IPC calls** - Asynchronous commands to running shell instance
  - Format: `qs -c $qsConfig ipc call {SERVICE} {METHOD}`
  - Examples:
    - `qs -c $qsConfig ipc call TEST_ALIVE` - Health check
    - `qs -c $qsConfig ipc call cliphistService update` - Update clipboard UI
    - `qs -c $qsConfig ipc call brightness increment/decrement` - Brightness control
    - `qs -c $qsConfig ipc call wallpaperSelectorToggle` - Wallpaper UI toggle
  - Reference: Throughout `hyprland/keybinds.conf`

- **Hyprland IPC dispatch** - Window manager event dispatching
  - Format: `hyprctl dispatch {ACTION} {ARGUMENTS}`
  - Examples:
    - `hyprctl dispatch submap global` - Keymap switching
    - `hyprctl dispatch movewindow {direction}` - Window movement
    - `hyprctl activeworkspace -j` - Query active workspace (JSON output)
  - Reference: `hyprland/scripts/workspace_action.sh`

## System Service Integration

**Startup Services (exec-once):**
- `hypridle` - Idle timeout and lock screen daemon
- `easyeffects --hide-window --service-mode` - Audio effects processor
- `gnome-keyring-daemon` - Credential manager
- `hyprctl dispatch submap global` - Initialize keymap
- `dbus-update-activation-environment` - System activation
- `hyprpm reload` - Load plugins
- `wl-paste --type text --watch` - Clipboard monitoring
- `wl-paste --type image --watch` - Clipboard monitoring
- GeoClue agent script - Location services (geolocation)
- Quickshell instance - UI shell framework
- Wallpaper restoration script - Visual state persistence

**Optional/Feature-Gated:**
- Ollama server (localhost:11434) - Must be running for AI features
- GeoClue 2.0 daemon - For location-based features

## External API Patterns

**HTTP Requests:**
```bash
# Ollama API (show-loaded-ollama-models.sh)
curl -s "$ollama_url:$port/api/tags"                                    # List models
curl -s "$ollama_url:$port/api/show" -d '{"name":"model","modelfile":true}'  # Model details

# Image Upload (snip_to_search.sh)
curl -sF files[]=@/tmp/image.png 'https://uguu.se/upload'               # Upload to uguu.se

# Health Check (via Quickshell)
qs -c $qsConfig ipc call TEST_ALIVE                                     # Check if Quickshell running
```

**Fallback Patterns:**
All keybindings use dual execution strategy:
1. Try IPC call to Quickshell: `qs -c $qsConfig ipc call {SERVICE}`
2. On failure/missing service: Execute fallback command
Example: `bind = Super, Super_L, exec, qs -c $qsConfig ipc call TEST_ALIVE || pkill fuzzel || fuzzel`

---

*Integration audit: 2026-02-02*

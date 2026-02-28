# Codebase Concerns

**Analysis Date:** 2026-02-02

## Configuration File Proliferation & Version Management

**Backup and Version Files Everywhere:**
- Issue: Multiple `.old`, `.new`, and `.backup` files exist without clear versioning strategy
- Files:
  - `hypridle.conf.new`, `hypridle.conf.old`
  - `hyprland.conf.backup`, `hyprland.conf.new`, `hyprland.conf.old`
  - `hyprlock.conf.new`, `hyprlock.conf.old`
  - `monitors.conf.new`, `monitors.conf.old`
  - `workspaces.conf.new`, `workspaces.conf.old`
  - `hyprland.confy` (typo or abandoned file)
- Impact: Unclear which is the active configuration; hard to track what changed; clutters directory; potential for accidental loading of old configs
- Fix approach: Keep only the active config files. Use git versioning or a single backup strategy. Delete `.new` and `.old` variants once verified.

## Unsafe Shell Script Patterns

**Unquoted Variables in `launch_first_available.sh`:**
- Issue: Uses `eval "$cmd"` without proper quoting/escaping of user input
- Files: `hyprland/scripts/launch_first_available.sh` (lines 4-5)
- Impact: Command injection vulnerability if config variables contain malicious strings
- Fix approach: Replace `eval` with safer alternatives like `bash -c` or use proper argument arrays

**Unquoted Variables in `snip_to_search.sh`:**
- Issue: Direct use of `$RANDOM_IMAGE` in command without quotes
- Files: `hyprland/scripts/snip_to_search.sh` (line 4)
- Impact: File paths with spaces will break; potential for command injection
- Fix approach: Quote all variables: `"$imageLink"`

**Weak IFS Handling in `show-loaded-ollama-models.sh`:**
- Issue: Line 56 uses `IFS=',' read` incorrectly; creates potential parsing issues
- Files: `hyprland/scripts/ai/show-loaded-ollama-models.sh` (line 56)
- Impact: Array parsing may fail on edge cases; IFS not properly restored in all paths
- Fix approach: Use safer parsing with `readarray` or `mapfile` builtins

## Hardcoded Network Endpoints

**Ollama Service Dependency:**
- Issue: Hardcoded localhost:11434 for Ollama API with no fallback
- Files:
  - `hyprland/scripts/ai/primary-buffer-query.sh` (line 34)
  - `hyprland/scripts/ai/show-loaded-ollama-models.sh` (line 7)
- Impact: Keybind execution will fail silently if Ollama is not running; no user feedback
- Fix approach: Check service availability before executing; provide meaningful error messages

**External Image Upload Service:**
- Issue: Hardcoded uguu.se for temporary image hosting in Google Lens workflow
- Files: `hyprland/scripts/snip_to_search.sh` (line 3)
- Impact: Depends on external service availability; privacy concern if sensitive images uploaded; no error handling if service is down
- Fix approach: Add error handling; consider local caching; use IPFS or privacy-respecting alternative; document privacy implications

## Excessive and Confusing Keybinding Duplication

**Same Keybind Mapped Multiple Times:**
- Issue: Multiple keybinds for same key with different fallbacks (primary + fallback patterns)
- Files: `hyprland/keybinds.conf` (e.g., lines 7-10, 42-43, 62-64, 234-236)
- Impact: Confusing to understand which binding actually fires; makes maintenance harder; hidden binds marked `# [hidden]` but still present; 271 lines of keybinds is difficult to maintain
- Fix approach: Consolidate fallback logic into separate launcher script; remove duplicate entries; clarify why multiple variants exist

**Example:** Lines 42-43 in keybinds.conf:
```
bindle=, XF86MonBrightnessUp, exec, qs -c $qsConfig ipc call brightness increment || brightnessctl s 5%+ # [hidden]
bindle=, XF86MonBrightnessDown, exec, qs -c $qsConfig ipc call brightness decrement || brightnessctl s 5%- # [hidden]
```

## Inappropriate Content Filtering

**Questionable Image Filtering in Test Keybinds:**
- Issue: Keybinds 222-223 use vulgar content filters for test notification images
- Files: `hyprland/keybinds.conf` (lines 222-223)
- Impact: Shows test code was never cleaned up; inappropriate for configuration; makes repository unsuitable for professional contexts
- Fix approach: Remove test keybinds entirely or move to separate test config file not sourced by default; clean up before committing

## Missing Error Handling Throughout

**No Error Handling in Startup Commands:**
- Issue: Multiple `exec-once` commands in execs.conf with no error handling
- Files: `hyprland/execs.conf` (lines 2-4, 7-11, 14-18)
- Impact: If one fails, no user notification; silent failures make debugging harder; some commented-out workarounds (lines 24-26) suggest known issues were not properly resolved
- Fix approach: Wrap critical commands with error checking; add logging; notify user on startup failure

**No Error Handling in AI Scripts:**
- Issue: `primary-buffer-query.sh` makes API calls without proper error handling
- Files: `hyprland/scripts/ai/primary-buffer-query.sh` (line 34)
- Impact: If curl fails or jq parsing fails, notification will contain jq error message to user
- Fix approach: Check command exit codes; provide fallback messages

## Conflicting Configuration Sources

**Duplicate Configuration in Multiple Files:**
- Issue: Both `/hyprland/execs.conf` and `/hyprland.confy` appear to contain startup logic
- Files:
  - `hyprland.confy` (typo filename, unclear purpose)
  - `execs.conf`
- Impact: Unclear which is authoritative; potential for conflicting settings; maintenance nightmare
- Fix approach: Consolidate into single execs.conf; remove or document hyprland.confy

**Duplicate Keybind Variable:**
- Issue: `$qsConfig` variable defined in both `hyprland.conf` (line 4) and `hyprland.confy` (line 22)
- Files: Multiple
- Impact: Creates confusion about configuration source
- Fix approach: Define once in central location (env.conf)

## Layout Configuration Inconsistency

**Keyboard Layout Set Multiple Times:**
- Issue: Swedish keyboard layout (`se`) set in both exec-once and input config
- Files: `hyprland.confy` (lines 4-5 and 7)
- Impact: Redundant; confusing which takes precedence
- Fix approach: Set once in input section only

## Complex Conditional Bash in Keybinds

**Overly Complex Inline Bash:**
- Issue: Long bash commands with multiple pipes and conditionals embedded in keybinds
- Files: `hyprland/keybinds.conf` (lines 69-70, 222-223)
- Impact: Hard to read; hard to test; hard to debug; maintainability nightmare
- Fix approach: Extract to separate shell scripts; source and call from keybind

**Example (lines 69-70):** Complex OCR pipeline should be extracted to dedicated script

## Workspace Script with Questionable Logic

**Potential Exit Code Bug:**
- Issue: `workspace_action.sh` line 17 has `exit 1` in the else branch but it should succeed
- Files: `hyprland/scripts/workspace_action.sh` (line 17)
- Impact: When dispatcher receives string target (special workspaces), script exits with error code even though command succeeded
- Fix approach: Remove `exit 1` from line 17; let hyprctl command succeed

## Monitor Configuration Fragility

**Hardcoded Monitor Parameters:**
- Issue: Specific resolution and refresh rates hardcoded (2560x1600@60, 2560x1440@144)
- Files: `hyprland.confy` (lines 53-55)
- Impact: Configuration breaks on monitor change; manual F9 toggle required for dual/single monitor gaming setups
- Fix approach: Use dynamic detection or environment variables for multi-monitor setups

## Missing Platform Dependencies

**Undocumented External Dependencies:**
- Issue: Scripts require curl, jq, slurp, grim, hyprctl, notify-send, etc. with no dependency checker
- Files: Multiple scripts
- Impact: Silent failures if tools not installed; no helpful error messages
- Fix approach: Create dependency checker script; document all requirements; add guards checking `command -v` before use

## Cleanup Tasks Not Documented

**Temporary Files Created Without Cleanup Documentation:**
- Issue: Scripts create `/tmp/image.png` and `/tmp/ocr_image.png` with explicit cleanup
- Files: `snip_to_search.sh`, `keybinds.conf`
- Impact: Good practice in some scripts but inconsistent; no cleanup for failed operations
- Fix approach: Add consistent cleanup with trap handlers for error conditions

## Security: Data Privacy Concerns

**Privacy Implications Not Documented:**
- Issue: Image data sent to external services (uguu.se, Google Lens) without privacy notice
- Files: `snip_to_search.sh`
- Impact: Users may not realize their screenshots are being uploaded
- Fix approach: Add privacy notice to keybind description; document data flow; provide local-only alternatives

**GeoClue Agent Running Without User Control:**
- Issue: GeoClue starts automatically in execs.conf without explicit user control
- Files: `hyprland/execs.conf` (line 2)
- Impact: Location data collected and potentially shared; privacy concern
- Fix approach: Add explicit opt-in; document privacy implications; provide disable mechanism

## Fragile AI Integration

**Hardcoded AI Model Selection:**
- Issue: Falls back to hardcoded `llama3.2` if no model loaded
- Files: `hyprland/scripts/ai/primary-buffer-query.sh` (line 14)
- Impact: Assumes specific model installed; fails silently if model not available
- Fix approach: Check available models first; provide user-friendly error messages; allow model selection

**Clipboard Size Limit Not Explained:**
- Issue: 2000 character limit hardcoded without explanation
- Files: `hyprland/scripts/ai/primary-buffer-query.sh` (line 26)
- Impact: Large text selections silently truncated; user won't know why AI response seems incomplete
- Fix approach: Notify user if text was truncated; make limit configurable

## Test Code in Production Config

**Test/Debug Keybinds Never Removed:**
- Issue: Keybinds starting at line 221 labeled "Testing" still active
- Files: `hyprland/keybinds.conf` (lines 221-224)
- Impact: Test notifications with inappropriate content will trigger; clutters active config
- Fix approach: Move to separate test config; remove from default startup

## Documentation Gaps

**Comments Explain "What" Not "Why":**
- Issue: Most comments describe what a keybind does but not why design decisions were made
- Files: Throughout keybinds.conf
- Impact: Future maintenance harder; unclear which keybinds are critical vs. experimental
- Fix approach: Add section headers; explain complex sequences; document design rationale

**No Configuration Audit Log:**
- Issue: Multiple backup files suggest changes but no clear changelog
- Files: Directory structure
- Impact: Impossible to understand evolution of configuration or when/why things changed
- Fix approach: Initialize git repository; create CHANGELOG.md documenting major changes

---

*Concerns audit: 2026-02-02*

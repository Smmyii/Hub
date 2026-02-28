# Phase 1: Foundation - Context

**Gathered:** 2026-02-02
**Status:** Ready for planning

<domain>
## Phase Boundary

Establish working layer shell positioning and IPC interface for keyboard control. This phase proves the technical foundation works before building features on top. Notes must position correctly on Wayland layer shell, respond to keybind commands via IPC, and not break the existing Super+G overlay.

</domain>

<decisions>
## Implementation Decisions

### Initial Positioning
- Claude's discretion on initial note placement (center, corner, or near cursor)
- Claude's discretion on cascade/offset behavior for multiple notes
- Notes must remember position per-monitor (not absolute screen coords)
- Claude's discretion on screen bounds constraints

### IPC Commands
- Super+Shift+G toggles last active note (dedicated shortcut)
- Super+G remains for overlay (unchanged)
- Include show-all and hide-all commands
- IPC naming follows existing illogical-impulse patterns (Claude to inspect and match)
- Errors should notify user (not silent)

### Layer Behavior
- Clicking a note focuses it immediately (can type right away)
- Pinned notes hide when apps go fullscreen (not always-on-top over fullscreen)
- Multiple notes have z-order stacking (click to bring forward)
- Claude's discretion on visual styling (shadow/border to match ii/ aesthetic)

### Integration Approach
- Claude's discretion on structure (prioritize update-resilience — separate module, override layer, etc.)
- Note data stored in `~/.local/share/` (survives config resets)
- Include backup/restore script for sticky notes config
- Code location should be practical for future improvements (user plans ongoing customization)

### Claude's Discretion
- Initial note placement strategy
- New note cascade/offset behavior
- Screen bounds constraints
- IPC naming conventions (match existing patterns)
- Visual styling (shadow/border)
- Code structure for update resilience
- Git tracking approach

</decisions>

<specifics>
## Specific Ideas

- User is concerned about illogical-impulse and Hyprland updates resetting customizations — structure changes to be resilient
- User plans ongoing improvements to their setup — code should be organized for future modifications
- Already created full config backup at `/home/sammy/.config/hypr-backup-20260202-133654`

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope

</deferred>

---

*Phase: 01-foundation*
*Context gathered: 2026-02-02*

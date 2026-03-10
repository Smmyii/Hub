# Ideas Backlog

Things to explore, revisit, or build when the time is right.

---

## GPU / iGPU Hybrid Setup
**Added:** 2026-03-03
**Context:** Hyprland 0.54 brought 50-500% iGPU performance improvements. Previously could hot-switch dGPU on/off but stopped because it required BIOS tinkering.

**Goal:** Find a clean solution to dynamically switch between iGPU and dGPU (or offload to dGPU only when needed) without BIOS changes.

**Leads:**
- Hyprland 0.54 iGPU rendering improvements may make iGPU-primary viable for daily use
- `supergfxctl` (from asus-linux, already have `asusctl`) — switchable graphics daemon
- PRIME offload rendering (`__NV_PRIME_RENDER_OFFLOAD=1`) for per-app dGPU
- `envycontrol` — switch between integrated/hybrid/dedicated modes
- Check if new Hyprland perf gains reduce the need for dGPU entirely for desktop tasks

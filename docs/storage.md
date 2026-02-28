# Storage & Filesystem

System-level storage knowledge, disk usage patterns, and troubleshooting guides.

---

## System Overview

| Component | Details |
|-----------|---------|
| **Filesystem** | Btrfs |
| **Root partition** | `/dev/nvme0n1p4` (294 GiB) |
| **Boot partition** | `/dev/nvme0n1p1` (256 MiB, FAT32) |
| **Subvolumes** | `/`, `/home`, `/srv`, `/var/cache`, `/var/log`, `/var/tmp`, `/root` |

---

## Known Issues & Fixes

### "No space left on device" with 24 GiB free (2026-02-16)

**Symptom:** Wallpaper selector unable to change wallpaper. Theme colors update correctly, but the wallpaper image stays the same. File writes fail with `ENOSPC` (no space left) despite `df -h` showing 24 GiB available.

**Root Cause:** Btrfs space allocation exhaustion. The entire device was allocated to data/metadata chunk groups, leaving only 1.05 MiB of unallocated space. Btrfs needs unallocated device space to create new metadata chunks when writing files.

#### How Btrfs Space Works

Btrfs has two layers of space management:

1. **Device-level allocation** — Raw disk divided into chunk groups (Data, Metadata, System)
2. **File-level usage** — Actual file data within those chunks

You can have plenty of free space *inside* allocated chunks, but if the device has no unallocated raw space, writes fail.

**Example from this issue:**
```
Device size:         293.62 GiB
Device allocated:    293.62 GiB   ← entire disk allocated to chunks
Device unallocated:    1.05 MiB   ← no room for new chunks!

Data chunks:         287.61 GiB total, 263.81 GiB used
  → 23.79 GiB free INSIDE data chunks

Metadata (DUP):        3.00 GiB total, 2.62 GiB used
  → 0.38 GiB free, but can't create new metadata chunks
```

#### Diagnosis Process

1. **Observed symptoms:**
   - Wallpaper selector works (UI opens, thumbnails load)
   - Color theme changes when selecting new wallpaper
   - Wallpaper image doesn't change on screen

2. **Tested wallpaper switch script manually:**
   ```bash
   ~/.config/quickshell/ii/scripts/colors/switchwall.sh \
     --image "/path/to/wallpaper.jpg" --mode dark
   ```
   Output revealed:
   ```
   mv: cannot overwrite '/home/sammy/.config/illogical-impulse/config.json':
       No space left on device
   ```

3. **Checked disk usage:**
   ```bash
   df -h /          # Showed 24 GiB free (92% used)
   df -i /          # Inodes: no limit on btrfs
   ```

4. **Checked btrfs-specific stats:**
   ```bash
   btrfs filesystem df /
   btrfs filesystem usage /
   ```
   Revealed: Device unallocated = 1.05 MiB

#### The Fix

Run a btrfs balance to reclaim underutilized data chunks:

```bash
sudo btrfs balance start -dusage=50 /
```

**What this does:**
- `-dusage=50` targets data chunks less than 50% full
- Relocates files from sparse chunks into fuller ones
- Frees up the emptied chunks back to unallocated space
- Gives btrfs room to allocate new metadata chunks

**Safer conservative option:**
```bash
sudo btrfs balance start -dusage=10 /
```
Only rebalances chunks <10% full. Faster, less aggressive.

**Progress monitoring:**
```bash
sudo btrfs balance status /
```

**Time estimate:** 5-15 minutes depending on disk speed and data movement.

#### Why It Happened

Over time, as files are created/deleted, btrfs allocates new chunk groups but doesn't automatically free underutilized ones. With a 92% full disk, the allocation fragmentation accumulated until no unallocated space remained.

This is a **known btrfs behavior**, not a bug. Regular balancing prevents it.

#### Prevention

Run periodic maintenance balance:
```bash
# Monthly or when disk >85% full
sudo btrfs balance start -dusage=20 /
```

Or install `btrfsmaintenance` package (if available on CachyOS):
```bash
yay -S btrfsmaintenance
sudo systemctl enable btrfs-balance.timer
```

#### Post-Fix Verification

After balance completes:
```bash
# Check unallocated space increased
btrfs filesystem usage /

# Test wallpaper switching
~/.config/quickshell/ii/scripts/colors/switchwall.sh \
  --image "/home/sammy/Pictures/Wallpapers/test.jpg" --mode dark
```

Wallpaper changes should work immediately.

---

## Diagnostic Commands Reference

### Quick Health Check
```bash
# Disk usage summary
df -h /

# Btrfs allocation
btrfs filesystem df /
btrfs filesystem usage /

# Find large directories
du -h --max-depth=1 / 2>/dev/null | sort -h | tail -20
```

### Detailed Analysis
```bash
# Check for snapshots (Timeshift/Snapper)
sudo btrfs subvolume list /

# Check quota groups (if enabled)
sudo btrfs qgroup show /

# Verify filesystem health
sudo btrfs scrub start /
sudo btrfs scrub status /
```

### Space Reclamation
```bash
# Clear package cache (CachyOS/Arch)
sudo pacman -Sc          # Remove uninstalled package cache
sudo pacman -Scc         # Remove ALL package cache (aggressive)

# Clear journal logs
sudo journalctl --vacuum-time=7d   # Keep 7 days
sudo journalctl --vacuum-size=500M # Or limit to 500M

# Find largest files
sudo find / -type f -size +1G 2>/dev/null

# Check trash
du -sh ~/.local/share/Trash/
```

---

## Impact on System Components

### Wallpaper System

The wallpaper switching pipeline failed at the config write step:

1. ✅ User selects wallpaper in Quickshell selector
2. ✅ `Wallpapers.select()` service called
3. ✅ `switchwall.sh` script executes
4. ❌ **Config write fails** — `jq ... > tmp && mv tmp config.json` (ENOSPC)
5. ✅ `matugen` color generation runs (uses temp space or memory)
6. ✅ Colors apply to system
7. ❌ Quickshell `Config.qml` never sees new wallpaper path
8. ❌ `Background.qml` keeps displaying old wallpaper

**Why colors still changed:** The `matugen` tool generates theme colors from the selected image and writes to different paths (likely in `/tmp` or memory-backed locations), which succeeded. But the critical config.json write that tells Quickshell *which* wallpaper to display failed silently.

### Other Affected Operations

Any operation requiring file writes would fail:
- Creating new sticky notes
- Saving Quickshell config changes
- Installing packages (`pacman` needs space for downloads)
- Git commits (creating new objects)
- Application data saves

---

## Related Files

- Wallpaper system: `~/Documents/Hub/architecture.md` → Quickshell Background module
- Config file: `~/.config/illogical-impulse/config.json`
- Switch script: `~/.config/quickshell/ii/scripts/colors/switchwall.sh`

---

*Last updated: 2026-02-16*

# Ubuntu Disk Usage Cheat Sheet

A quick reference for checking disk usage at every level — from whole-disk overviews down to individual files.

---

## Level 1 — Filesystem & Volume Overview

### `df` — Disk free by mounted filesystem

Shows every mounted volume with total, used, and available space.

```bash
df -h                        # All filesystems, human-readable
df -hT                       # + filesystem type (ext4, tmpfs, etc.)
df -h /                      # Root partition only
df -h /home                  # Specific mount point
df -h --exclude-type=tmpfs   # Exclude tmpfs entries
```

### `lsblk` — Block device tree

Shows physical drives, partitions, and where they're mounted.

```bash
lsblk
lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINT,LABEL
```

> Good for seeing the disk → partition → mount relationship at a glance.

### `fdisk` / `parted` — Partition details

Low-level partition table view (sizes are raw, not usage).

```bash
sudo fdisk -l
sudo parted -l
```

---

## Level 2 — Directory & Folder Usage

### `du` — Disk usage by directory

Walks a directory tree and reports how much space each directory consumes.

```bash
du -sh *                          # Size of each item in the current directory
du -sh /var/*                     # Size of each item under /var
du -sh ~                          # Total size of your home directory
du -h --max-depth=1 /var          # One level deep under /var
du -ah /etc | sort -rh | head -20 # Top 20 largest items under /etc
```

**Key flags:**
| Flag | Meaning |
|------|---------|
| `-s` | Summarise (don't recurse into subdirs) |
| `-h` | Human-readable sizes (MB, GB) |
| `-a` | Include files, not just directories |
| `--max-depth=N` | Limit recursion depth |

### Find the biggest space hogs

Sort all directories under a path by size, largest first.

```bash
du -h --max-depth=2 / 2>/dev/null | sort -rh | head -30
```

> `2>/dev/null` suppresses permission errors. Pipe to `less` if output is long.

---

## Level 3 — File-Level Analysis

### `find` — Locate large files

```bash
find / -type f -size +500M 2>/dev/null
find /var -type f -size +100M -exec ls -lh {} \;
find . -type f -size +10M | sort       # In the current directory
```

### `df -i` — Inode exhaustion check

A disk can show free space but fail to create files if inodes run out.

```bash
df -i       # Inode usage per filesystem
df -ih      # Same, human-readable
```

> **IUse% near 100% = inode exhaustion.** Common culprit: millions of tiny files
> (mail spools, PHP sessions, container layers).

---

## Level 4 — Interactive & GUI Tools

### `ncdu` — Interactive terminal disk analyser

Navigate directories with arrow keys, drill in/out, sorted by size. The fastest
tool for interactive exploration.

```bash
sudo apt install ncdu
ncdu /     # Scan from root
ncdu ~     # Scan home directory
```

### Disk Usage Analyzer (Baobab) — GUI

Graphical treemap viewer. Pre-installed on Ubuntu GNOME; install manually on Xubuntu.

```bash
sudo apt install baobab
baobab
```

---

## Quick Reference

| Goal | Command |
|------|---------|
| How full is my disk? | `df -h` |
| What's eating space in this folder? | `du -sh *` |
| Find biggest subdirs under /var | `du -h --max-depth=1 /var \| sort -rh` |
| Drill down interactively | `ncdu /` |
| Find files > 500 MB anywhere | `find / -size +500M 2>/dev/null` |
| Check drive/partition layout | `lsblk` |
| Disk can't create files (inodes full)? | `df -i` |

---

## Common Ubuntu Disk Hogs

These locations are the usual suspects when space runs low:

| Path | Cause | Fix |
|------|-------|-----|
| `/var/log` | Log accumulation | `sudo journalctl --vacuum-size=500M` |
| `/var/cache/apt` | Package cache | `sudo apt clean` |
| `/var/lib/docker` | Images & volumes | `docker system prune` |
| `~/.local/share/Trash` | Trash bin | Empty trash |
| `/tmp`, `/var/tmp` | Temp files | Usually auto-cleaned on reboot |
| `/var/lib/snapd/snaps` | Snap packages | `snap list --all` + remove old revisions |

---

## Typical Troubleshooting Workflow

```bash
# 1. Check overall disk fill level
df -h

# 2. Find which top-level directory is largest
du -h --max-depth=1 / 2>/dev/null | sort -rh

# 3. Drill into the biggest offender (e.g. /var)
du -h --max-depth=1 /var | sort -rh

# 4. Keep drilling until you find the culprit
# OR launch ncdu for interactive navigation:
sudo ncdu /var
```

---

*Applies to: Ubuntu Desktop, Ubuntu Server, Xubuntu, and most Debian-based distributions.*

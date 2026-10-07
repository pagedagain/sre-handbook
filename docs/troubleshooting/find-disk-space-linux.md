---
title: How to Find What Is Using Disk Space on Linux
description: Use df, du, find, and lsof to locate full filesystems, inode exhaustion, deleted-open files, logs, and Docker data safely.
tags:
  - linux
  - disk
  - df
  - du
  - inodes
  - lsof
  - logs
  - docker
  - troubleshooting
---

# How to Find What Is Using Disk Space on Linux

Start with the full filesystem, then narrow the search without crossing into other mounts. Do not delete the first large file you see until you know which process owns it and whether the data can be recreated.

## Quick Reference

| Command | What it does |
|---------|--------------|
| `df -hT` | Shows block usage and filesystem type for mounted filesystems |
| `df -ih` | Shows inode usage instead of bytes |
| `du -xhd1 /` | Totals first-level directories without crossing filesystems |
| `findmnt -T /var/log` | Shows which filesystem contains a path |
| `lsof +L1` | Finds open files whose directory entry has been deleted |
| `journalctl --disk-usage` | Shows space used by the systemd journal |
| `docker system df -v` | Breaks down Docker image, container, volume, and cache usage |

## Core Concepts

### `df` measures filesystems

`df` reports allocated blocks from the filesystem's view. Find the mount that is full before running a recursive search, because `/`, `/var`, and application data may be separate filesystems. A filesystem can also reject new files when its inodes are exhausted even if bytes remain.

Show both block and inode pressure:

```bash
df -hT && printf '\nInodes:\n' && df -ih
```

### `du` measures reachable files

`du` walks directory entries and totals reachable file blocks. Use `-x` to stay on one filesystem and `-d1` to narrow the search one level at a time. When `df` shows much more used space than `du`, the usual causes are a deleted file that is still open, or files written into a directory before another filesystem was mounted over it. Run `du` with `sudo`; without root it skips directories it cannot read.

Rank top-level directories on the root filesystem:

```bash
sudo du -xhd1 / 2>/dev/null | sort -h
```

### Deletion does not always release space

Linux removes a file's directory entry when it is deleted, but the blocks remain allocated while a process holds the file open. `du` cannot see that file because it has no pathname; `df` still counts its blocks. The blocks are freed when the process closes the file, either because it reopens its log or because it restarts. Confirm the service can be restarted safely first.

Find deleted files that still consume blocks:

```bash
sudo lsof +L1
```

## Common Scenarios

### One filesystem is full

Identify the full mount in `df`, set `mnt` to it, then rank its largest first-level directories without crossing mount boundaries:

```bash
df -hT
mnt=/var
sudo du -xhd1 "$mnt" 2>/dev/null | sort -h
```

### Logs grew unexpectedly

List the largest regular files under `/var/log` and check journal usage:

```bash
sudo find /var/log -xdev -type f -printf '%s %p\n' 2>/dev/null | sort -n | tail -20; sudo journalctl --disk-usage
```

### Writes fail but `df -h` shows free space

Confirm inode exhaustion, then count entries per directory on that filesystem to find where small files pile up. Every file, directory, and symlink uses an inode, so the count does not filter by type:

```bash
df -ih /var; sudo find /var -xdev -printf '%h\n' 2>/dev/null | sort | uniq -c | sort -n | tail -20
```

### `df` is full but `du` is small

Look for deleted-open files, including the process and descriptor that still own the blocks:

```bash
sudo lsof +L1
```

If `lsof` finds nothing, check for files hidden under a mount point. Bind-mount the root filesystem to a temporary path, which shows the directories underneath other mounts, and measure it there:

```bash
tmp=$(mktemp -d)
sudo mount --bind / "$tmp"
sudo du -xhd1 "$tmp" 2>/dev/null | sort -h
sudo umount "$tmp"; rmdir "$tmp"
```

### Docker owns most of the disk

Check how much is reclaimable, then list per-image, per-container, and per-volume usage before pruning anything:

```bash
docker system df; docker system df -v
```

## Gotchas

- **`du` without `-x` crosses mounts**: Network filesystems and bind mounts can make the scan slow and the totals misleading.
- **Sparse files have two sizes**: `ls -lh` shows logical size; `du -h` shows allocated blocks.
- **Deleted files can stay allocated**: Removing a hot log file does not free its blocks until the writer closes or reopens it.
- **Inode exhaustion looks like disk exhaustion**: Check `df -i` when creating files fails with `No space left on device` even though `df -h` shows free space.
- **Blind pruning causes outages**: `docker system prune`, journal vacuuming, and log deletion can remove data needed by running services or an investigation.
- **Reserved blocks affect non-root users first**: ext4 reserves 5% of blocks by default so root can recover a full filesystem. `df` counts reserved blocks as neither used nor available, so `Used` plus `Avail` is less than `Size`.

## Related Challenges

<div class="practice-cta" markdown>

**Practice these failures in a live terminal.**

Find directory-level disk consumption, diagnose a deleted file that still occupies space, and trace inode exhaustion on a volume that looks half empty.

[Launch Disk Full Emergency](https://pagedagain.com/incidents/disk-full?utm_source=runbooks&utm_medium=troubleshooting&utm_campaign=find-disk-space-linux){ .md-button .md-button--primary }
[Launch No Space Left After Cleanup](https://pagedagain.com/incidents/deleted-file-held-open?utm_source=runbooks&utm_medium=troubleshooting&utm_campaign=find-disk-space-linux){ .md-button .md-button--primary }
[Launch No Space Left With Disk Free](https://pagedagain.com/incidents/inode-exhaustion?utm_source=runbooks&utm_medium=troubleshooting&utm_campaign=find-disk-space-linux){ .md-button .md-button--primary }

</div>

<a class="star-cta" href="https://github.com/pagedagain/sre-handbook">Found this useful? <span class="star-cta-link">Star the handbook repo</span> to help other SREs find it.</a>

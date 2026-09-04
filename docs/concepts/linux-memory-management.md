---
title: Linux Memory Management Explained
description: Read free, available, RSS, page cache, swap, pressure, cgroup limits, and OOM evidence without treating cached memory as a leak.
tags:
  - linux
  - memory
  - cgroups
  - oom
---

# Linux Memory Management Explained

Linux uses unused RAM for cache and reclaims it when applications need memory. During an incident, focus on available memory, reclaim pressure, swap activity, cgroup limits, and OOM evidence rather than the `free` column alone.

## Quick Reference

| Command | What it does |
|---------|--------------|
| `free -h` | Summarizes RAM, page cache, available memory, and swap |
| `vmstat 1 5` | Samples runnable tasks, memory, paging, I/O, and CPU |
| `ps -eo pid,user,rss,vsz,%mem,comm --sort=-rss` | Sorts processes by resident memory |
| `grep -E 'MemAvailable|Cached|SwapFree|Slab' /proc/meminfo` | Shows key kernel memory counters |
| `journalctl -k -g 'Out of memory|Killed process'` | Searches kernel logs for OOM kills |
| `systemd-cgtop -m -b -n 1` | Shows one snapshot of cgroup memory usage |

## Core Concepts

### Virtual memory is not resident memory

Each process sees a virtual address space that can be larger than physical RAM. VSZ includes mapped but untouched regions, shared libraries, and reserved address space; RSS counts pages currently resident in RAM. RSS is more useful for pressure, but summing it can double-count shared pages.

List the largest resident process sets in MiB:

```bash
ps -eo pid,ppid,user,rss,vsz,%mem,comm --sort=-rss | awk 'NR==1 {print; next} NR<=16 {$4=sprintf("%.1fM",$4/1024); $5=sprintf("%.1fM",$5/1024); print}'
```

### Cache is reclaimable memory

The kernel caches file data, directory metadata, and filesystem structures because RAM is faster than storage. `MemAvailable` estimates how much memory can be given to applications without heavy swapping, including reclaimable cache. A low `free` value with healthy `available` memory is normal.

Compare free, available, cache, and reclaimable slab:

```bash
free -h; grep -E '^(MemFree|MemAvailable|Cached|SReclaimable|SwapFree):' /proc/meminfo
```

### Limits and pressure decide OOM behavior

The host can run out of memory, but a process can also hit a cgroup limit while the host still has plenty available. Under sustained pressure, the kernel reclaims cache and may swap anonymous pages; if it cannot satisfy an allocation, the OOM killer selects a victim. Cgroup v2 records limit hits and kills in `memory.events`.

Show the current shell's cgroup usage, limit, and event counters:

```bash
CGROUP=$(awk -F: '$1=="0" {print $3}' /proc/self/cgroup); printf 'current='; cat "/sys/fs/cgroup${CGROUP}/memory.current"; printf 'max='; cat "/sys/fs/cgroup${CGROUP}/memory.max"; cat "/sys/fs/cgroup${CGROUP}/memory.events"
```

## Common Scenarios

### The host reports high memory usage

Check available memory, swap, and a short pressure sample before blaming page cache:

```bash
free -h; vmstat 1 5
```

### One process appears to be growing

Rank resident memory, then inspect the largest process's rollup counters:

```bash
PID=$(ps -eo pid=,rss= --sort=-rss | awk 'NR==1 {print $1}'); ps -p "$PID" -o pid,ppid,user,lstart,rss,vsz,%mem,cmd; grep -E '^(Rss|Pss|Private|Shared|Swap):' "/proc/$PID/smaps_rollup"
```

### A process disappeared without an application error

Search recent kernel messages for host or cgroup OOM evidence:

```bash
sudo journalctl -k --since '-1 hour' | grep -Ei 'out of memory|oom-kill|killed process|memory cgroup'
```

### Swap activity is causing latency

Sample swap-in, swap-out, run queue, I/O wait, and CPU steal once per second:

```bash
vmstat -w 1 10
```

## Gotchas

- **`free` is not available**: Use the `available` estimate to judge immediate headroom.
- **Dropping caches is not a normal fix**: It discards useful data and can create an I/O spike while hiding the real consumer.
- **RSS totals overcount shared pages**: Use proportional set size from `smaps_rollup` when attribution must be more precise.
- **Swap use is not automatically bad**: Active swap-in and swap-out under latency is stronger evidence than a nonzero used value.
- **A cgroup OOM can happen on a healthy host**: Inspect `memory.max` and `memory.events` for the affected service or container.
- **The OOM victim may not be the root cause**: The killed process can be the easiest victim rather than the process that created the pressure.

## Related Challenges

<div class="practice-cta" markdown>

**No matching hands-on challenge is published yet.**

Practice other Linux and production failure modes in an isolated terminal.

[Browse Paged Again challenges](https://pagedagain.com/incidents?utm_source=runbooks&utm_medium=concept&utm_campaign=linux-memory-management){ .md-button .md-button--primary }

</div>

<a class="star-cta" href="https://github.com/pagedagain/sre-handbook">Found this useful? <span class="star-cta-link">Star the handbook repo</span> to help other SREs find it.</a>

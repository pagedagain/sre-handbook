---
title: Linux File Permissions Explained
description: Read ownership, rwx and octal modes, directory permissions, umask, special bits, and ACLs without reaching for chmod 777.
tags:
  - linux
  - permissions
  - chmod
  - chown
---

# Linux File Permissions Explained

Linux checks the process identity against file ownership, mode bits, and optional ACLs. During an incident, inspect the process user and every directory in the path before changing permissions.

## Quick Reference

| Command | What it does |
|---------|--------------|
| `id` | Shows the current user, primary group, and supplementary groups |
| `stat -c '%A %a %U:%G %n' /etc/passwd` | Shows symbolic mode, octal mode, owner, and group |
| `namei -l /etc/ssh/sshd_config` | Shows permissions on every path component |
| `getfacl -p /etc/passwd` | Shows the access ACL and mask |
| `chmod 640 file` | Sets owner read/write, group read, and no access for others |
| `chown root:adm file` | Changes file owner and group |
| `umask` | Shows permission bits removed when new files are created |

## Core Concepts

### Ownership and mode bits

Each file has an owner, a group, and permission sets for owner, group, and other. Read is 4, write is 2, and execute is 1, so `640` means owner read/write, group read, and no permissions for others. The kernel uses one matching class, not the most permissive combination of all three.

Inspect a file in symbolic and octal form:

```bash
stat -c 'symbolic=%A octal=%a owner=%U group=%G path=%n' /etc/passwd
```

### Directory permissions are different

Read on a directory lists names, write adds or removes entries, and execute traverses the directory and accesses known names. A readable file still fails to open if the process lacks execute permission on any parent directory. Deleting a file depends mainly on the parent directory, not the file's own write bit.

Inspect every component leading to the shadow password database:

```bash
namei -l /etc/shadow
```

### Creation masks, special bits, and ACLs

`umask` removes permissions from an application's requested mode; common defaults are `022` and `027`. Setgid on a directory makes new entries inherit the directory's group, while the sticky bit limits deletion in shared writable directories. ACLs grant named users or groups access, but the ACL mask can silently reduce effective permissions.

Show the current mask, `/tmp` sticky bit, and ACLs on `/etc/passwd`:

```bash
printf 'umask='
umask
stat -c 'tmp=%A (%a)' /tmp
getfacl -p /etc/passwd
```

## Common Scenarios

### A service cannot read a configuration file

Compare the service identity with the file and every parent directory:

```bash
id nobody
namei -l /etc/shadow
stat -c '%A %a %U:%G %n' /etc/shadow
sudo -u nobody test -r /etc/shadow
printf 'read_status=%s\n' "$?"
```

### A shared directory needs group inheritance

Create a group-writable test directory with setgid, then verify the resulting mode:

```bash
sudo install -d -m 2775 -o root -g adm /tmp/pagedagain-shared && \
  stat -c '%A %a %U:%G %n' /tmp/pagedagain-shared
```

### A named user needs access without opening the file to everyone

Create a private test file, grant `nobody` read access with an ACL, and display the effective permissions:

```bash
printf 'service_config=true\n' | sudo tee /tmp/pagedagain-config >/dev/null
sudo chmod 600 /tmp/pagedagain-config
sudo setfacl -m u:nobody:r /tmp/pagedagain-config
getfacl -p /tmp/pagedagain-config
```

### New files are more restrictive than expected

Show how `umask 027` turns a requested file mode of `666` into `640`:

```bash
tmp=$(mktemp -d)
(umask 027; touch "$tmp/file"; mkdir "$tmp/dir")
stat -c '%a %n' "$tmp/file" "$tmp/dir"
rm -rf "$tmp"
```

## Gotchas

- **`chmod 777` hides the diagnosis**: It grants write access to every local user and usually creates a security problem.
- **Root tests can lie**: Root commonly bypasses discretionary read and write checks; reproduce the failure as the service user.
- **Parent directories need execute**: File permissions alone do not make a path reachable.
- **The ACL mask limits named entries**: `getfacl` marks reduced permissions as `effective`, even when the named entry looks broader.
- **Numeric `chown` may cross systems badly**: User and group names can map to different IDs across hosts and containers.
- **Setuid on scripts is ignored**: Linux does not honor setuid bits on interpreted scripts.

## Related Challenges

<div class="practice-cta" markdown>

**Practice this failure in a live terminal.**

Diagnose a non-root service and restore ownership and modes without making files world-writable.

[Launch Log Shipper Permission Denied](https://pagedagain.com/incidents/permission-denied?utm_source=runbooks&utm_medium=concept&utm_campaign=linux-file-permissions){ .md-button .md-button--primary }

</div>

<a class="star-cta" href="https://github.com/pagedagain/sre-handbook">Found this useful? <span class="star-cta-link">Star the handbook repo</span> to help other SREs find it.</a>

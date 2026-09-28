# Module 7: Special Permission Bits

## Introduction

Beyond the standard read, write, and execute permissions, Linux has three special permission bits: **SUID**, **SGID**, and the **Sticky Bit**. These bits modify how executables run and how directories behave, and they play critical roles in both system functionality and security.

---

## SUID (Set User ID) — Numeric: 4000

### What It Does

When the **SUID bit** is set on an executable file, the process runs with the **Effective UID (EUID) of the file's owner**, not the user who executed it.

### How It Works

```
Normal execution (no SUID):
  alice runs /usr/bin/myapp
  Process: RUID=1001(alice), EUID=1001(alice)

SUID execution:
  alice runs /usr/bin/passwd (SUID, owned by root)
  Process: RUID=1001(alice), EUID=0(root)
```

The process gains the privileges of the file owner while still knowing who actually started it (via RUID).

### Classic Example: /usr/bin/passwd

```bash
$ ls -l /usr/bin/passwd
-rwsr-xr-x. 1 root root 32648 Jul 21 2023 /usr/bin/passwd
```

Notice the `s` in the owner execute position (`rws`). This means:
- The file is owned by root
- SUID is set
- When any user runs `passwd`, it runs with EUID=0 (root)
- This allows it to write to `/etc/shadow`, which is only writable by root
- The RUID stays as the user, so `passwd` knows whose password to change

### How to Set SUID

```bash
# Using numeric
chmod 4755 myprogram     # SUID + rwxr-xr-x

# Using symbolic
chmod u+s myprogram
```

### How SUID Appears in ls -l

| Display | Meaning |
|---------|---------|
| `rws` (lowercase s) | SUID is set AND execute permission is set |
| `rwS` (uppercase S) | SUID is set BUT execute permission is NOT set |

The uppercase `S` indicates a potentially misconfigured SUID — SUID without execute is useless since the file cannot be executed.

```bash
# SUID with execute (correct)
$ chmod 4755 myapp
$ ls -l myapp
-rwsr-xr-x. 1 root root 1234 Sep 22 10:00 myapp

# SUID without execute (broken)
$ chmod 4644 myapp
$ ls -l myapp
-rwSr--r--. 1 root root 1234 Sep 22 10:00 myapp
```

### Security Risks of SUID

SUID executables owned by root are one of the biggest security concerns on a Linux system:

1. **Privilege escalation**: A vulnerability in a SUID root program can give an attacker root access.
2. **Buffer overflows**: Exploiting a bug in a SUID program may yield a root shell.
3. **Unintended capabilities**: Programs that allow arbitrary file access or command execution become root-level tools.

### SUID on Scripts

**SUID is ignored on scripts on most modern Linux systems** (including RHEL, Debian, and their derivatives). The kernel recognizes script interpreters (#!/bin/bash) and does not apply SUID to them. This is a security feature — scripts are too easy to manipulate (race conditions, environment variables, etc.).

### SUID on Directories

SUID has **no effect** on directories. Only SGID and the sticky bit are meaningful on directories.

### Common SUID Executables on RHEL 9

```bash
$ find / -type f -perm -4000 -ls 2>/dev/null
   ... /usr/bin/passwd
   ... /usr/bin/chage
   ... /usr/bin/gpasswd
   ... /usr/bin/newgrp
   ... /usr/bin/su
   ... /usr/bin/sudo
   ... /usr/bin/mount
   ... /usr/bin/umount
   ... /usr/bin/crontab
   ... /usr/sbin/unix_chkpwd
```

Each of these needs elevated privileges for a specific operation.

---

## SGID (Set Group ID) — Numeric: 2000

### On Executable Files

When SGID is set on an executable, the process runs with the **Effective GID of the file's group**, not the user's primary group.

```bash
$ ls -l /usr/bin/write
-rwxr-sr-x. 1 root tty 19544 Jul 21 2023 /usr/bin/write
```

The `s` in the group execute position means SGID is set. When any user runs `write`, the process gets the `tty` group's GID, allowing it to write to terminal devices owned by that group.

### On Directories (Most Important Use)

When SGID is set on a **directory**, new files and subdirectories created inside it **inherit the directory's group** instead of the creator's primary group.

**Without SGID:**

```bash
$ ls -ld /shared/
drwxrwxr-x. 2 root developers 4096 Sep 22 10:00 /shared/

# alice creates a file (her primary group is "alice", not "developers")
$ touch /shared/alice-file.txt
$ ls -l /shared/alice-file.txt
-rw-r--r--. 1 alice alice 0 Sep 22 10:01 alice-file.txt
                     ^^^^^ ← alice's primary group, NOT developers!
```

**With SGID:**

```bash
$ chmod g+s /shared/
$ ls -ld /shared/
drwxrwsr-x. 2 root developers 4096 Sep 22 10:00 /shared/
                  ^ ← SGID set

# alice creates a file
$ touch /shared/alice-file.txt
$ ls -l /shared/alice-file.txt
-rw-r--r--. 1 alice developers 0 Sep 22 10:01 alice-file.txt
                     ^^^^^^^^^^ ← inherits directory's group!
```

This is **essential for shared directories** where a team needs all files to belong to the same group.

### How to Set SGID

```bash
# Using numeric
chmod 2775 /shared/      # SGID + rwxrwxr-x

# Using symbolic
chmod g+s /shared/
```

### How SGID Appears in ls -l

| Display | Meaning |
|---------|---------|
| `rws` (lowercase s in group) | SGID set AND group execute set |
| `rwS` (uppercase S in group) | SGID set BUT group execute NOT set |

### Production Example: Shared Project Directory

```bash
# Create the directory
sudo mkdir /opt/project

# Create the group
sudo groupadd projteam

# Set ownership
sudo chown root:projteam /opt/project

# Set SGID + group write
sudo chmod 2775 /opt/project

# Now any file created inside inherits the projteam group
# Team members can all read and write each other's files
```

---

## Sticky Bit — Numeric: 1000

### What It Does

When the sticky bit is set on a **directory**, only the following can delete or rename files within it:
- The **file's owner**
- The **directory's owner**
- **root**

This prevents users from deleting other users' files in shared directories.

### Classic Example: /tmp

```bash
$ ls -ld /tmp
drwxrwxrwt. 15 root root 4096 Sep 22 10:00 /tmp
         ^ ← sticky bit
```

The permissions `1777` mean:
- Everyone can read, write, and enter the directory
- BUT users can only delete their own files
- Without the sticky bit (plain `777`), any user could delete any other user's files

### How to Set the Sticky Bit

```bash
# Using numeric
chmod 1777 /tmp          # Sticky + rwxrwxrwx

# Using symbolic
chmod +t /shared/
```

### How the Sticky Bit Appears in ls -l

| Display | Meaning |
|---------|---------|
| `rwt` (lowercase t) | Sticky bit set AND other execute set |
| `rwT` (uppercase T) | Sticky bit set BUT other execute NOT set |

### Demonstration

```bash
# Create a directory with sticky bit
$ sudo mkdir /shared-tmp
$ sudo chmod 1777 /shared-tmp

# alice creates a file
$ su - alice -c "touch /shared-tmp/alice-file.txt"

# bob tries to delete alice's file
$ su - bob -c "rm /shared-tmp/alice-file.txt"
rm: cannot remove '/shared-tmp/alice-file.txt': Operation not permitted

# alice can delete her own file
$ su - alice -c "rm /shared-tmp/alice-file.txt"
# Success
```

---

## Special Bits Summary Table

| Bit | Numeric | On Files | On Directories |
|-----|---------|----------|----------------|
| **SUID** | 4000 | Process runs with file owner's EUID | No effect |
| **SGID** | 2000 | Process runs with file group's EGID | New files inherit directory's group |
| **Sticky** | 1000 | No practical effect on modern Linux | Only owner/dir-owner/root can delete files |

## Permission Indicator Table

| Position | Symbol | Meaning |
|----------|--------|---------|
| Owner execute | `s` | SUID set + execute permission |
| Owner execute | `S` | SUID set, NO execute permission |
| Group execute | `s` | SGID set + execute permission |
| Group execute | `S` | SGID set, NO execute permission |
| Other execute | `t` | Sticky bit set + execute permission |
| Other execute | `T` | Sticky bit set, NO execute permission |

---

## Combined Numeric Examples

```bash
chmod 4755 file    # SUID + rwxr-xr-x     (-rwsr-xr-x)
chmod 2755 dir     # SGID + rwxr-xr-x     (drwxr-sr-x)
chmod 1777 dir     # Sticky + rwxrwxrwx   (drwxrwxrwt)
chmod 6755 file    # SUID+SGID + rwxr-xr-x (-rwsr-sr-x)
chmod 3775 dir     # SGID+Sticky + rwxrwxr-x (drwxrwsr-t)
```

---

## Finding SUID and SGID Files

### Security Auditing

Regularly auditing SUID and SGID files is a critical security practice:

```bash
# Find all SUID files
sudo find / -type f -perm -4000 -ls 2>/dev/null

# Find all SGID files
sudo find / -type f -perm -2000 -ls 2>/dev/null

# Find files with both SUID and SGID
sudo find / -type f -perm -6000 -ls 2>/dev/null

# Find SUID files NOT owned by root (suspicious)
sudo find / -type f -perm -4000 ! -user root -ls 2>/dev/null

# Find SGID directories
sudo find / -type d -perm -2000 -ls 2>/dev/null
```

### What to Look For

- SUID executables not on the known list for your distribution
- SUID executables in user-writable directories
- SUID executables owned by users other than root
- Recently modified SUID executables
- SUID set on scripts (should not be possible, but check)

### Removing Special Bits

```bash
# Remove SUID
chmod u-s /path/to/file
# or
chmod 0755 /path/to/file

# Remove SGID
chmod g-s /path/to/file

# Remove sticky bit
chmod -t /path/to/directory

# Remove all special bits
chmod 0755 /path/to/file
```

---

## Interview Questions

### Q1: What is the SUID bit and how does it work?

**Answer:** SUID (Set User ID) is a special permission bit set on executable files. When set, the process runs with the Effective UID of the file's owner rather than the user who executed it. The most common example is `/usr/bin/passwd`, which is owned by root with SUID set. When a normal user runs passwd, the process gets EUID=0, allowing it to write to `/etc/shadow`. The Real UID remains as the user, so the program knows whose password to change.

### Q2: What is the difference between SUID and SGID on a directory?

**Answer:** SUID has no effect on directories. SGID on a directory causes new files and subdirectories created inside to inherit the directory's group ownership instead of the creator's primary group. This is essential for shared directories where team members need all files to belong to the same group.

### Q3: What is the sticky bit and why does /tmp have it?

**Answer:** The sticky bit on a directory restricts file deletion — only the file's owner, the directory's owner, or root can delete or rename files. /tmp has permissions 1777 (sticky bit + rwxrwxrwx), which allows all users to create files but prevents them from deleting other users' files. Without the sticky bit, any user could delete any file in the directory.

### Q4: What does a capital `S` or `T` mean in permission output?

**Answer:** A capital `S` means SUID (or SGID) is set but the underlying execute permission is NOT set. A capital `T` means the sticky bit is set but the underlying execute permission for "other" is NOT set. Lowercase `s` and `t` mean the special bit AND the execute permission are both set. A capital letter usually indicates a misconfiguration — SUID without execute is useless, and sticky without other-execute is unusual.

### Q5: How would you audit a system for potentially dangerous SUID files?

**Answer:** Run `find / -type f -perm -4000 -ls 2>/dev/null` to list all SUID files. Compare against the known list for your distribution. Look for SUID files in unusual locations (user home directories, /tmp), files not owned by root, recently modified SUID executables, or SUID files not in the distribution's package database (`rpm -qf /path/to/file` on RHEL). Any unexpected SUID file could be a sign of compromise or misconfiguration.

### Q6: Why does SUID not work on scripts?

**Answer:** Most modern Linux kernels ignore the SUID bit on interpreted scripts (those starting with `#!`). This is a security measure because scripts are vulnerable to race conditions (the script could be modified between the kernel checking it and the interpreter running it) and environment manipulation (environment variables like `IFS` or `PATH` could change script behavior). For privileged script operations, use sudo with specific command rules instead.

---

**Next Module:** [Module 8: Linux ACLs](08-linux-acls.md) — Learn how Access Control Lists extend traditional permissions.

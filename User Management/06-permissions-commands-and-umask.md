# Module 6: Permission Commands and umask

## Introduction

This module covers the commands used to view and change file permissions, ownership, and how umask controls default permissions for newly created files and directories.

---

## chmod — Change File Permissions

### Purpose

Changes the permission bits of files and directories.

### Syntax

```bash
chmod [OPTIONS] MODE FILE...
```

### Numeric (Octal) Mode

```bash
chmod 755 script.sh    # rwxr-xr-x
chmod 644 config.txt   # rw-r--r--
chmod 700 private/     # rwx------
chmod 600 secret.key   # rw-------
chmod 750 project/     # rwxr-x---
chmod 640 data.conf    # rw-r-----
chmod 444 readonly.txt # r--r--r--
chmod 000 locked.txt   # ----------
```

### Symbolic Mode

Symbolic mode uses the format: `[who][operator][permissions]`

**Who:**

| Symbol | Meaning |
|--------|---------|
| `u` | User (owner) |
| `g` | Group |
| `o` | Other |
| `a` | All (user + group + other) |

**Operators:**

| Operator | Meaning |
|----------|---------|
| `+` | Add permission |
| `-` | Remove permission |
| `=` | Set exact permission (replaces) |

**Examples:**

```bash
# Add execute for owner
chmod u+x script.sh

# Remove write from group and other
chmod go-w file.txt

# Add read for everyone
chmod a+r file.txt

# Set owner to rwx, group to rx, other to nothing
chmod u=rwx,g=rx,o= file.txt

# Remove all permissions for other
chmod o= file.txt

# Add write for group only
chmod g+w shared.txt

# Make a file executable for owner only
chmod u+x,go-x script.sh

# Add execute for owner and group
chmod ug+x script.sh
```

### Recursive Permission Changes

```bash
# Change permissions recursively
chmod -R 755 /opt/project/

# WARNING: This sets the same permissions on files AND directories
# Files usually should NOT have execute permission
# Better approach - set directories and files separately:
find /opt/project/ -type d -exec chmod 755 {} \;
find /opt/project/ -type f -exec chmod 644 {} \;
```

### Common Mistakes with chmod

| Mistake | Problem | Fix |
|---------|---------|-----|
| `chmod -R 777 /` | Makes entire filesystem world-writable | Restore from backup or rebuild |
| `chmod -R 755 /opt/app/` | Makes all files executable | Use `find` to set files and dirs separately |
| `chmod 777 file.txt` | Insecure — anyone can modify | Use minimum required permissions |
| Forgetting `-R` for directories | Only top-level directory changed | Use `-R` for recursive, or use `find` |

---

## chown — Change File Ownership

### Purpose

Changes the owner and/or group of files and directories.

### Syntax

```bash
chown [OPTIONS] OWNER[:GROUP] FILE...
```

### Examples

```bash
# Change owner only
chown alice file.txt

# Change owner and group
chown alice:developers file.txt

# Change group only (note the colon)
chown :developers file.txt

# Recursive ownership change
chown -R alice:developers /opt/project/
```

### Important Options

| Option | Description |
|--------|-------------|
| `-R` | Recursive (change files and subdirectories) |
| `-h` | Change ownership of symbolic link itself (not the target) |
| `--from=OWNER:GROUP` | Only change if current owner/group matches |
| `-v` | Verbose (show each change) |

### Symbolic Links

By default, `chown` follows symbolic links (changes the target):

```bash
$ ls -l link.txt -> /opt/data/real.txt
$ chown alice link.txt
# Changes ownership of /opt/data/real.txt, NOT the link

# To change the link itself:
$ chown -h alice link.txt
```

### Dangerous Recursive Changes

```bash
# DANGEROUS: Follows symbolic links into other directories
$ sudo chown -R alice:alice /home/alice/

# If /home/alice/ contains a symlink to /etc:
# /home/alice/configs -> /etc
# chown -R would change ownership of files in /etc!

# SAFER: Don't follow symlinks (-h, or avoid -R on dirs with symlinks)
$ sudo chown -hR alice:alice /home/alice/
```

---

## chgrp — Change Group Ownership

### Purpose

Changes only the group ownership of files. Equivalent to `chown :group file`.

```bash
# Change group
chgrp developers file.txt

# Recursive
chgrp -R developers /opt/project/
```

A non-root user can only change a file's group to a group they belong to.

---

## stat — Detailed File Information

### Purpose

Displays detailed file information including permissions in multiple formats, timestamps, and inode information.

```bash
$ stat config.txt
  File: config.txt
  Size: 4096            Blocks: 8          IO Block: 4096   regular file
Device: 253,0   Inode: 8390657     Links: 1
Access: (0644/-rw-r--r--)  Uid: ( 1001/   alice)   Gid: ( 1002/developers)
Access: 2024-09-22 10:00:00.000000000 -0400
Modify: 2024-09-22 09:45:00.000000000 -0400
Change: 2024-09-22 09:45:00.000000000 -0400
 Birth: 2024-09-20 08:00:00.000000000 -0400
```

Key information:
- **Access (permissions):** Shows both octal (`0644`) and symbolic (`-rw-r--r--`)
- **Uid/Gid:** Both numeric and name
- **Access time:** Last time file was read
- **Modify time:** Last time file contents changed
- **Change time:** Last time file metadata changed (permissions, ownership, etc.)

---

## ls -l — List with Permissions

```bash
$ ls -l
total 16
drwxr-xr-x. 2 alice developers 4096 Sep 22 10:00 docs
-rw-r--r--. 1 alice developers 1234 Sep 22 09:30 readme.txt
-rwxr-xr-x. 1 alice developers  512 Sep 22 09:00 script.sh
lrwxrwxrwx. 1 alice developers   10 Sep 22 08:00 link -> readme.txt
```

| Column | Meaning |
|--------|---------|
| `drwxr-xr-x.` | File type + permissions (`.` = SELinux context present) |
| `2` | Number of hard links |
| `alice` | Owner |
| `developers` | Group |
| `4096` | Size in bytes |
| `Sep 22 10:00` | Last modification time |
| `docs` | Filename |

Useful variations:

```bash
ls -la     # Include hidden files (starting with .)
ls -ld     # Show directory itself, not its contents
ls -ln     # Show numeric UID/GID instead of names
ls -lR     # Recursive listing
```

---

## namei — Trace Path Permissions

### Purpose

Resolves a path and shows permissions at each level. Essential for diagnosing "Permission denied" errors.

```bash
$ namei -l /home/alice/projects/webapp/config.yml
f: /home/alice/projects/webapp/config.yml
dr-xr-xr-x root  root   /
drwxr-xr-x root  root   home
drwx------ alice  alice  alice         # ← Only alice can enter
drwxr-xr-x alice  alice  projects
drwxr-xr-x alice  alice  webapp
-rw-r--r-- alice  alice  config.yml
```

From this output, you can immediately see that `/home/alice` has `700` permissions, so only alice and root can access anything below it.

---

## find — Search by Permissions

### Purpose

Finds files matching specific permission criteria.

### Permission Matching Modes

| Syntax | Meaning |
|--------|---------|
| `-perm 755` | Permissions are **exactly** 755 |
| `-perm -755` | Permissions include **at least** all bits of 755 |
| `-perm /755` | Permissions include **any** bit of 755 |

### Examples

```bash
# Find files with exactly 777 permissions (security audit)
find /opt -type f -perm 777

# Find files where owner has at least read+write
find /opt -type f -perm -600

# Find files that are world-writable (any write bit for other)
find / -type f -perm -o+w -ls 2>/dev/null

# Find files owned by a specific user
find / -user alice -type f -ls 2>/dev/null

# Find files owned by a specific group
find /opt -group developers -type f

# Find files with no owner (orphaned)
find / -nouser -ls 2>/dev/null

# Find files with no group
find / -nogroup -ls 2>/dev/null

# Find SUID executables
find / -type f -perm -4000 -ls 2>/dev/null

# Find SGID executables
find / -type f -perm -2000 -ls 2>/dev/null
```

---

## install — Create Files with Specific Permissions

### Purpose

Copies files while setting permissions, ownership, and creating directories in one command.

```bash
# Create a directory with specific permissions and owner
install -d -m 750 -o appuser -g appgroup /opt/myapp

# Copy a file with specific permissions
install -m 640 -o root -g appgroup config.yml /etc/myapp/config.yml

# Copy an executable with correct permissions
install -m 755 -o root -g root myapp /usr/local/bin/myapp
```

Commonly used in Makefiles and deployment scripts.

---

## umask — Default Permission Mask

### What Is umask?

The **umask** (user file-creation mask) determines the default permissions for newly created files and directories. It is a permission **mask** — bits set in the umask are **removed** from the default permissions.

### Default Creation Modes

The operating system starts with these base permissions:

| Type | Base Permissions | Reason |
|------|-----------------|--------|
| Regular files | `0666` (rw-rw-rw-) | Files are not executable by default |
| Directories | `0777` (rwxrwxrwx) | Directories need execute for traversal |

### How umask Is Applied

The umask is applied using a **bitwise AND with the complement**:

```
New file permissions = Base mode AND (NOT umask)
New directory permissions = Base mode AND (NOT umask)
```

### Calculation Examples

#### umask 022 (most common default)

```
Files:       0666 AND NOT(0022) = 0666 AND 0755 = 0644 (rw-r--r--)
Directories: 0777 AND NOT(0022) = 0777 AND 0755 = 0755 (rwxr-xr-x)
```

#### umask 027 (more secure)

```
Files:       0666 AND NOT(0027) = 0666 AND 0750 = 0640 (rw-r-----)
Directories: 0777 AND NOT(0027) = 0777 AND 0750 = 0750 (rwxr-x---)
```

#### umask 077 (most restrictive)

```
Files:       0666 AND NOT(0077) = 0666 AND 0700 = 0600 (rw-------)
Directories: 0777 AND NOT(0077) = 0777 AND 0700 = 0700 (rwx------)
```

### Why Simple Subtraction Is Wrong

Many people explain umask as "subtract umask from base." This works for common values but is technically incorrect:

```
umask 033:
  Subtraction method: 0666 - 0033 = 0633 (rw--wx-wx)  ← WRONG
  Correct method:     0666 AND NOT(0033) = 0666 AND 0744 = 0644 (rw-r--r--)  ← CORRECT
```

The subtraction method fails because you cannot subtract a permission bit that is not present in the base. The base for files is `0666` — there is no execute bit (1) to subtract. The correct bitwise operation handles this correctly.

### umask Quick Reference

| umask | File Result | Dir Result | Use Case |
|-------|-------------|------------|----------|
| `022` | `644` (rw-r--r--) | `755` (rwxr-xr-x) | Default — public read |
| `027` | `640` (rw-r-----) | `750` (rwxr-x---) | Group-restricted |
| `077` | `600` (rw-------) | `700` (rwx------) | Private — owner only |
| `002` | `664` (rw-rw-r--) | `775` (rwxrwxr-x) | Group-writable |
| `007` | `660` (rw-rw----) | `770` (rwxrwx---) | Group-writable, no other |
| `066` | `600` (rw-------) | `711` (rwx--x--x) | No group/other read on files |
| `033` | `644` (rw-r--r--) | `744` (rwxr--r--) | Removes group/other write+execute |
| `000` | `666` (rw-rw-rw-) | `777` (rwxrwxrwx) | No mask — INSECURE |

### Viewing the Current umask

```bash
# Numeric
$ umask
0022

# Symbolic
$ umask -S
u=rwx,g=rx,o=rx
```

### Setting umask

```bash
# Set for current session
$ umask 027

# Verify
$ umask
0027

# Test
$ touch testfile
$ mkdir testdir
$ ls -l testfile
-rw-r-----. 1 alice alice 0 Sep 22 11:00 testfile
$ ls -ld testdir
drwxr-x---. 2 alice alice 4096 Sep 22 11:00 testdir
```

### Where umask Is Configured

The umask can be set in multiple places. They are applied in order, and later settings override earlier ones:

| Location | Scope | When Applied |
|----------|-------|-------------|
| `/etc/login.defs` (UMASK) | System default | At user creation |
| `/etc/profile` | All users, login shells | Login |
| `/etc/profile.d/*.sh` | All users, login shells | Login |
| `/etc/bashrc` | All users, all bash shells | Every bash start |
| `~/.bash_profile` | One user, login shells | Login |
| `~/.bashrc` | One user, all bash shells | Every bash start |
| PAM (`pam_umask`) | Per-service | Authentication |
| systemd unit file (`UMask=`) | Per-service | Service start |

### umask for Services

Services started by systemd can have their own umask:

```ini
# In a systemd unit file
[Service]
UMask=0027
```

This ensures files created by the service have restricted permissions.

---

## umask Calculation Exercises

### Exercise 1

**umask = 022. What permissions does a new file get?**

```
Base:  0666
Mask:  0022
NOT mask: 0755
Result: 0666 AND 0755 = 0644 (rw-r--r--)
```

### Exercise 2

**umask = 077. What permissions does a new directory get?**

```
Base:  0777
Mask:  0077
NOT mask: 0700
Result: 0777 AND 0700 = 0700 (rwx------)
```

### Exercise 3

**umask = 027. What permissions does a new file get?**

```
Base:  0666
Mask:  0027
NOT mask: 0750
Result: 0666 AND 0750 = 0640 (rw-r-----)
```

### Exercise 4

**umask = 002. What permissions does a new directory get?**

```
Base:  0777
Mask:  0002
NOT mask: 0775
Result: 0777 AND 0775 = 0775 (rwxrwxr-x)
```

### Exercise 5

**umask = 033. What permissions does a new file get?**

```
Base:    0666 = 110 110 110
Mask:    0033 = 000 011 011
NOT mask:0744 = 111 100 100
Result:  0666 AND 0744 = 0644 (rw-r--r--)
Note: NOT 0666 - 0033 = 0633 (the subtraction method gives the WRONG answer here)
```

### Exercise 6

**A new file is created with permissions `640`. What was the umask?**

```
Base:   0666
Result: 0640
Mask bits: 0666 XOR 0640... but actually:
We need: 0666 AND NOT(umask) = 0640
NOT(umask) must have at least: 0640
umask = NOT(something that produces 0640 from 0666)
umask = 0027
```

Verify: `0666 AND NOT(0027) = 0666 AND 0750 = 0640` ✓

---

## Combining chmod, chown, and umask in Production

### Setting Up an Application Directory

```bash
# Create directory structure
sudo mkdir -p /opt/myapp/{bin,config,data,logs}

# Set ownership
sudo chown -R appuser:appgroup /opt/myapp

# Set directory permissions
sudo find /opt/myapp -type d -exec chmod 750 {} \;

# Set file permissions
sudo find /opt/myapp -type f -exec chmod 640 {} \;

# Make binaries executable
sudo chmod 750 /opt/myapp/bin/*

# Make config readable by group
sudo chmod 640 /opt/myapp/config/*

# Make logs writable by the app
sudo chmod 770 /opt/myapp/logs
```

---

## Interview Questions

### Q1: What is the difference between numeric and symbolic chmod syntax?

**Answer:** Numeric (octal) syntax like `chmod 755` sets the entire permission at once using three digits. Symbolic syntax like `chmod u+x` modifies specific permissions relative to current values using who (u/g/o/a), operator (+/-/=), and permission (r/w/x). Numeric is more common in scripts and when you know the exact permissions you want. Symbolic is useful when you want to add or remove a specific permission without knowing the current state.

### Q2: How does umask work? Walk through a calculation.

**Answer:** umask is a permission mask that removes bits from default creation permissions. Files start with 0666 and directories with 0777. The umask is applied using bitwise AND with the complement: `result = base AND NOT(umask)`. For example, with umask 027: new files get `0666 AND NOT(0027) = 0666 AND 0750 = 0640 (rw-r-----)`, and new directories get `0777 AND NOT(0027) = 0777 AND 0750 = 0750 (rwxr-x---)`. The common explanation of "subtract umask from base" is technically wrong and gives incorrect results for values like umask 033.

### Q3: What happens when you run `chmod -R 755` on a directory tree?

**Answer:** It sets ALL files and directories to 755 (rwxr-xr-x). This is usually wrong for files because regular files should not have execute permission. The correct approach is to set directories and files separately: `find /path -type d -exec chmod 755 {} \;` for directories and `find /path -type f -exec chmod 644 {} \;` for files.

### Q4: How does chown behave with symbolic links?

**Answer:** By default, `chown` follows symbolic links and changes the ownership of the target file, not the link itself. To change the ownership of the symbolic link itself, use `chown -h`. When using `-R` (recursive), be careful because `chown -R` follows symbolic links, which could change ownership of files outside the intended directory tree. Using `chown -hR` avoids following symlinks.

### Q5: What is the `namei` command used for?

**Answer:** `namei -l` resolves a file path and shows the permissions, owner, and group at each directory level. It is invaluable for diagnosing "Permission denied" errors because it reveals exactly which directory in the path is blocking access. For example, a user may have permission on a file but be blocked because a parent directory lacks execute permission.

---

**Next Module:** [Module 7: Special Permission Bits](07-special-permissions-bits.md) — Learn about SUID, SGID, and the sticky bit.

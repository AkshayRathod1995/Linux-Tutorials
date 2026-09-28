# Module 2: User Management Commands

## Introduction

This module covers every important command for managing user accounts on Linux. For each command, you will learn the syntax, important options, practical examples with expected output, common mistakes, and troubleshooting tips. The primary distribution is AlmaLinux 9 / RHEL 9, with Debian/Ubuntu differences noted.

---

## useradd — Create a New User

### Purpose

Creates a new user account by adding entries to `/etc/passwd`, `/etc/shadow`, `/etc/group`, and optionally creating a home directory.

### Syntax

```bash
useradd [OPTIONS] USERNAME
```

### Important Options

| Option | Description | Example |
|--------|-------------|---------|
| `-u UID` | Specify the numeric UID | `useradd -u 1500 alice` |
| `-g GROUP` | Specify the primary group (name or GID) | `useradd -g developers alice` |
| `-G GROUPS` | Specify supplementary groups (comma-separated) | `useradd -G wheel,docker alice` |
| `-d DIR` | Specify home directory path | `useradd -d /opt/alice alice` |
| `-m` | Create the home directory if it doesn't exist | `useradd -m alice` |
| `-M` | Do NOT create a home directory | `useradd -M alice` |
| `-s SHELL` | Specify login shell | `useradd -s /bin/zsh alice` |
| `-c COMMENT` | Set the GECOS/comment field | `useradd -c "Alice Smith" alice` |
| `-e DATE` | Set account expiration date (YYYY-MM-DD) | `useradd -e 2025-12-31 alice` |
| `-r` | Create a system account (UID from system range) | `useradd -r myapp` |
| `-k SKEL_DIR` | Use a custom skeleton directory | `useradd -k /etc/skel.custom alice` |
| `-D` | Show or change default values | `useradd -D` |

### Examples

**Example 1: Create a basic user**

```bash
$ sudo useradd alice
```

What happens:
- Entry added to `/etc/passwd` with next available UID (1000+)
- Entry added to `/etc/shadow` with locked password (`!!`)
- A private group `alice` created in `/etc/group`
- On RHEL 9 with default config: home directory `/home/alice` is created (because `CREATE_HOME yes` in `/etc/login.defs`)
- Skeleton files from `/etc/skel/` copied to home directory

**Note on RHEL vs Debian:** On RHEL/AlmaLinux, `CREATE_HOME yes` is the default in `/etc/login.defs`, so `useradd` creates the home directory without `-m`. On Debian/Ubuntu, you typically need `-m` explicitly or should use `adduser` instead.

Verify:

```bash
$ getent passwd alice
alice:x:1001:1001::/home/alice:/bin/bash

$ ls -la /home/alice/
total 12
drwx------. 2 alice alice  83 Sep 22 10:00 .
drwxr-xr-x. 4 root  root   36 Sep 22 10:00 ..
-rw-r--r--. 1 alice alice  18 Jul 21  2023 .bash_logout
-rw-r--r--. 1 alice alice 141 Jul 21  2023 .bash_profile
-rw-r--r--. 1 alice alice 492 Jul 21  2023 .bashrc
```

**Example 2: Create a user with full options**

```bash
$ sudo useradd -u 1500 -g developers -G wheel,docker \
    -d /home/alice -m -s /bin/bash -c "Alice Smith" \
    -e 2025-12-31 alice
```

Verify:

```bash
$ id alice
uid=1500(alice) gid=1002(developers) groups=1002(developers),10(wheel),1003(docker)

$ getent passwd alice
alice:x:1500:1002:Alice Smith:/home/alice:/bin/bash

$ chage -l alice | grep "Account expires"
Account expires                     : Dec 31, 2025
```

**Example 3: Create a service account**

```bash
$ sudo useradd -r -s /sbin/nologin -d /opt/myapp -c "MyApp Service" myapp
```

Verify:

```bash
$ getent passwd myapp
myapp:x:988:984:MyApp Service:/opt/myapp:/sbin/nologin

$ id myapp
uid=988(myapp) gid=984(myapp) groups=984(myapp)
```

Notice: UID is below 1000 (system range) because `-r` was used. Shell is `/sbin/nologin` to prevent interactive login.

### Common Mistakes

| Mistake | Problem | Fix |
|---------|---------|-----|
| Forgetting to set a password | User exists but cannot log in | Run `passwd alice` after creating |
| Using `-G` without `-g` | Primary group may not be what you expect | Always check with `id alice` |
| Forgetting `-m` on Debian | Home directory not created | Use `useradd -m` or `adduser` |
| Specifying a nonexistent group | Command fails | Create the group first with `groupadd` |
| Using a UID that already exists | Command fails with error | Check first: `id 1500` or `getent passwd 1500` |

### Viewing Defaults

```bash
$ useradd -D
GROUP=100
HOME=/home
INACTIVE=-1
EXPIRE=
SHELL=/bin/bash
SKEL=/etc/skel
CREATE_MAIL_SPOOL=yes
```

These defaults are stored in `/etc/default/useradd`.

---

## adduser — Distribution-Specific User Creation

### RHEL / AlmaLinux / CentOS

On RHEL-family distributions, `adduser` is simply a symbolic link to `useradd`:

```bash
$ ls -l /usr/sbin/adduser
lrwxrwxrwx. 1 root root 7 Jul 21  2023 /usr/sbin/adduser -> useradd
```

It behaves identically to `useradd`.

### Debian / Ubuntu

On Debian-family distributions, `adduser` is a separate Perl script that provides an **interactive, user-friendly** interface:

```bash
$ sudo adduser alice
Adding user 'alice' ...
Adding new group 'alice' (1001) ...
Adding new user 'alice' (1001) with group 'alice' ...
Creating home directory '/home/alice' ...
Copying files from '/etc/skel' ...
New password: 
Retype new password:
passwd: password updated successfully
Changing the user information for alice
Enter the new value, or press ENTER for the default
    Full Name []: Alice Smith
    Room Number []: 
    Work Phone []: 
    Home Phone []: 
    Other []: 
Is the information correct? [Y/n] Y
```

**Key differences:**

| Feature | `useradd` (RHEL) | `adduser` (Debian) |
|---------|------------------|-------------------|
| Type | Binary command | Perl wrapper script |
| Interactive | No | Yes (prompts for password, info) |
| Home directory | Depends on config | Always created |
| Password | Must set separately | Prompts during creation |
| Skeleton files | Copied if home created | Always copied |

**Best practice:** Use `useradd` in scripts and automation (predictable behavior). Use `adduser` interactively on Debian (more user-friendly).

---

## usermod — Modify an Existing User

### Purpose

Changes the properties of an existing user account.

### Syntax

```bash
usermod [OPTIONS] USERNAME
```

### Important Options

| Option | Description | Example |
|--------|-------------|---------|
| `-l NEW_NAME` | Change the username | `usermod -l alice_new alice` |
| `-u NEW_UID` | Change the UID | `usermod -u 2000 alice` |
| `-g GROUP` | Change primary group | `usermod -g newgroup alice` |
| `-G GROUPS` | **SET** supplementary groups (REPLACES all existing!) | `usermod -G docker alice` |
| `-aG GROUPS` | **APPEND** to supplementary groups (SAFE) | `usermod -aG docker alice` |
| `-d DIR` | Change home directory path | `usermod -d /newhome/alice alice` |
| `-d DIR -m` | Change home directory AND move contents | `usermod -d /newhome/alice -m alice` |
| `-s SHELL` | Change login shell | `usermod -s /bin/zsh alice` |
| `-L` | Lock the account (adds `!` before password hash) | `usermod -L alice` |
| `-U` | Unlock the account (removes `!` prefix) | `usermod -U alice` |
| `-e DATE` | Set account expiration (YYYY-MM-DD) | `usermod -e 2025-12-31 alice` |
| `-c COMMENT` | Change the comment/GECOS field | `usermod -c "Alice Jones" alice` |

### Critical Warning: `-G` vs `-aG`

This is one of the most common and dangerous mistakes in Linux administration:

```bash
# alice is currently in: wheel, developers, docker

# DANGEROUS: -G REPLACES all supplementary groups
$ sudo usermod -G newgroup alice
$ id alice
uid=1001(alice) gid=1001(alice) groups=1001(alice),1005(newgroup)
# wheel, developers, and docker are GONE!

# SAFE: -aG APPENDS to supplementary groups
$ sudo usermod -aG newgroup alice
$ id alice
uid=1001(alice) gid=1001(alice) groups=1001(alice),10(wheel),1002(developers),1003(docker),1005(newgroup)
# All existing groups preserved, newgroup added
```

**Rule: Always use `-aG` when adding a user to a group, unless you intentionally want to replace all groups.**

### Examples

**Example 1: Add a user to a supplementary group**

```bash
$ sudo usermod -aG wheel alice
$ id alice
uid=1001(alice) gid=1001(alice) groups=1001(alice),10(wheel)
```

**Example 2: Lock and unlock an account**

```bash
# Lock the account
$ sudo usermod -L alice
$ sudo passwd -S alice
alice LK 2024-09-22 0 99999 7 -1 (Password locked.)

# Unlock the account
$ sudo usermod -U alice
$ sudo passwd -S alice
alice PS 2024-09-22 0 99999 7 -1 (Password set, SHA512 crypt.)
```

**Example 3: Move a user's home directory**

```bash
$ sudo usermod -d /newhome/alice -m alice

# Verify
$ getent passwd alice
alice:x:1001:1001:Alice Smith:/newhome/alice:/bin/bash

$ ls -la /newhome/alice/
# All files moved from old home directory
```

### Important Notes

- Changing a user's UID with `usermod -u` does **not** change the ownership of their existing files. You must manually update file ownership: `find / -user OLD_UID -exec chown NEW_UID {} \;`
- The user must be logged out before you can change their username, home directory, or UID.
- Group changes take effect at the next login, not immediately for running sessions.

---

## userdel — Delete a User

### Purpose

Removes a user account from the system.

### Syntax

```bash
userdel [OPTIONS] USERNAME
```

### Important Options

| Option | Description |
|--------|-------------|
| `-r` | Remove the user's home directory and mail spool |
| `-f` | Force removal even if user is logged in (dangerous) |

### Examples

**Example 1: Delete user but keep home directory**

```bash
$ sudo userdel alice

# User is removed from /etc/passwd, /etc/shadow, /etc/group
# Home directory /home/alice still exists
# Files in /home/alice now show numeric UID instead of username
$ ls -ln /home/alice/
total 0
drwx------. 2 1001 1001 83 Sep 22 10:00 .
```

**Example 2: Delete user and remove home directory**

```bash
$ sudo userdel -r alice

# User removed AND /home/alice deleted AND /var/spool/mail/alice deleted
```

### Safe Deletion Process

Before deleting a user, follow this process:

```bash
# 1. Check for running processes
$ ps -u alice
# If processes are running, decide whether to kill them

# 2. Check for cron jobs
$ sudo crontab -l -u alice

# 3. Check for at jobs
$ sudo atq | grep alice

# 4. Find all files owned by the user
$ sudo find / -user alice -ls 2>/dev/null | head -20

# 5. Backup home directory if needed
$ sudo tar czf /backup/alice-home.tar.gz /home/alice

# 6. Transfer important file ownership if needed
$ sudo find / -user alice -exec chown newowner {} \;

# 7. Delete the user
$ sudo userdel -r alice
```

### Finding Orphaned Files

After deleting a user, files they owned now show a numeric UID instead of a username:

```bash
# Find files with no matching user
$ sudo find / -nouser -ls 2>/dev/null

# Find files with no matching group
$ sudo find / -nogroup -ls 2>/dev/null
```

---

## passwd — Manage User Passwords

### Purpose

Sets or changes user passwords and manages password properties.

### Syntax

```bash
passwd [OPTIONS] [USERNAME]
```

### Important Options

| Option | Description | Example |
|--------|-------------|---------|
| (none) | Change your own password | `passwd` |
| `USERNAME` | Change another user's password (root only) | `passwd alice` |
| `-l` | Lock the password (add `!` before hash) | `passwd -l alice` |
| `-u` | Unlock the password (remove `!` prefix) | `passwd -u alice` |
| `-d` | Delete the password (allow login without password) | `passwd -d alice` |
| `-e` | Expire the password (force change at next login) | `passwd -e alice` |
| `-S` | Show password status | `passwd -S alice` |
| `-n DAYS` | Set minimum days between password changes | `passwd -n 1 alice` |
| `-x DAYS` | Set maximum days before password must change | `passwd -x 90 alice` |
| `-w DAYS` | Set warning days before expiry | `passwd -w 7 alice` |
| `-i DAYS` | Set inactive days after expiry | `passwd -i 30 alice` |

### Password Status Output

```bash
$ sudo passwd -S alice
alice PS 2024-09-22 0 99999 7 -1 (Password set, yescrypt crypt.)
```

| Field | Meaning | Values |
|-------|---------|--------|
| `alice` | Username | |
| `PS` | Password status | `PS` = set, `LK` = locked, `NP` = no password |
| `2024-09-22` | Last password change date | |
| `0` | Minimum days between changes | |
| `99999` | Maximum days before expiry | |
| `7` | Warning days before expiry | |
| `-1` | Inactive days after expiry | `-1` = disabled |

### Examples

**Example 1: Set a user's password**

```bash
$ sudo passwd alice
Changing password for user alice.
New password: 
Retype new password: 
passwd: all authentication tokens updated successfully.
```

**Example 2: Force password change at next login**

```bash
$ sudo passwd -e alice
Expiring password for user alice.
passwd: Success

# Next time alice logs in:
# WARNING: Your password has expired.
# You must change your password now and login again!
```

**Example 3: Lock and unlock a password**

```bash
# Lock
$ sudo passwd -l alice
Locking password for user alice.
passwd: Success

# Check
$ sudo passwd -S alice
alice LK 2024-09-22 0 99999 7 -1 (Password locked.)

# IMPORTANT: Locking the password does NOT prevent SSH key authentication!
# To fully prevent login, also expire the account:
# sudo usermod -e 1 alice
```

---

## chage — Change Password Aging Information

### Purpose

Manages password aging policies — when passwords expire, when accounts expire, and when users are warned.

### Syntax

```bash
chage [OPTIONS] USERNAME
```

### Important Options

| Option | Description | Example |
|--------|-------------|---------|
| `-l` | List all aging information | `chage -l alice` |
| `-d DAYS` | Set last password change date (0 = force change) | `chage -d 0 alice` |
| `-E DATE` | Set account expiration date (YYYY-MM-DD) | `chage -E 2025-12-31 alice` |
| `-E -1` | Remove account expiration | `chage -E -1 alice` |
| `-I DAYS` | Set inactive days after password expiry | `chage -I 30 alice` |
| `-m DAYS` | Set minimum days between password changes | `chage -m 1 alice` |
| `-M DAYS` | Set maximum password age in days | `chage -M 90 alice` |
| `-W DAYS` | Set warning days before password expiry | `chage -W 14 alice` |

### Examples

**Example 1: View all aging information**

```bash
$ sudo chage -l alice
Last password change                    : Sep 22, 2024
Password expires                        : Dec 21, 2024
Password inactive                       : Jan 20, 2025
Account expires                         : Dec 31, 2025
Minimum number of days between password change  : 1
Maximum number of days between password change  : 90
Number of days of warning before password expires : 14
```

**Example 2: Force password change at next login**

```bash
$ sudo chage -d 0 alice
# This sets "last password change" to epoch (Jan 1, 1970)
# The password is immediately considered expired
```

**Example 3: Set a standard password policy**

```bash
# Password must be changed every 90 days
# Cannot be changed more than once per day
# Warning 14 days before expiry
# Account disabled 30 days after password expires
$ sudo chage -M 90 -m 1 -W 14 -I 30 alice
```

---

## id — Display User Identity

### Purpose

Shows the UID, GID, and all group memberships for a user.

### Syntax

```bash
id [OPTIONS] [USERNAME]
```

### Options

| Option | Description | Output Example |
|--------|-------------|---------------|
| (none) | Full identity info | `uid=1001(alice) gid=1001(alice) groups=...` |
| `-u` | UID only | `1001` |
| `-g` | Primary GID only | `1001` |
| `-G` | All GIDs | `1001 10 1002` |
| `-n` | Show names instead of numbers | Used with `-u`, `-g`, `-G` |
| `-un` | Username | `alice` |
| `-gn` | Primary group name | `alice` |
| `-Gn` | All group names | `alice wheel developers` |

### Examples

```bash
$ id alice
uid=1001(alice) gid=1001(alice) groups=1001(alice),10(wheel),1002(developers)

$ id -u alice
1001

$ id -Gn alice
alice wheel developers
```

---

## whoami, who, w, and users — Session Information

### whoami

Prints the effective username of the current user:

```bash
$ whoami
alice

# Equivalent to:
$ id -un
alice
```

### who

Shows who is currently logged in:

```bash
$ who
alice    pts/0        2024-09-22 10:15 (192.168.1.100)
bob      pts/1        2024-09-22 11:30 (192.168.1.101)

# With headers
$ who -H
NAME     LINE         TIME             COMMENT
alice    pts/0        2024-09-22 10:15 (192.168.1.100)
bob      pts/1        2024-09-22 11:30 (192.168.1.101)

# Last system boot
$ who -b
         system boot  2024-09-20 08:00
```

### w

More detailed than `who` — shows what each user is doing:

```bash
$ w
 14:30:15 up 2 days,  6:30,  2 users,  load average: 0.15, 0.10, 0.05
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
alice    pts/0    192.168.1.100    10:15    0.00s  0.05s  0.00s w
bob      pts/1    192.168.1.101    11:30    3:00   0.02s  0.02s vim config.yml
```

### users

Simple list of logged-in usernames:

```bash
$ users
alice bob
```

---

## groups and getent — Group and Identity Lookups

### groups

Shows group memberships:

```bash
$ groups
alice wheel developers

$ groups alice
alice : alice wheel developers
```

### getent

Queries the Name Service Switch (NSS) for user, group, or shadow information. Unlike reading `/etc/passwd` directly, `getent` also returns users from LDAP, SSSD, and other configured sources.

```bash
# Look up a user
$ getent passwd alice
alice:x:1001:1001:Alice Smith:/home/alice:/bin/bash

# Look up a group
$ getent group developers
developers:x:1002:alice,charlie

# Look up shadow entry (requires root)
$ sudo getent shadow alice
alice:$y$j9T$...:19988:0:90:14:30:20088:

# List all users (including LDAP/SSSD)
$ getent passwd

# List all groups
$ getent group
```

**Production use:** `getent` is the correct way to check if a user or group exists, especially in environments with centralized identity management. Never rely solely on `grep /etc/passwd`.

---

## su — Switch User

### Purpose

Switches to another user account. Requires the **target user's password** (unlike sudo, which uses your own password).

### Syntax

```bash
su [OPTIONS] [USERNAME]
```

### Important Options

| Option | Description |
|--------|-------------|
| `-` or `-l` | Start a login shell (clean environment) |
| `-c COMMAND` | Run a single command as the user |
| `-s SHELL` | Use a specific shell |

### Examples

```bash
# Switch to root (requires root's password)
$ su -
Password: 
# You are now root with root's full environment

# Switch to alice (requires alice's password)
$ su - alice
Password:
$ whoami
alice

# Run a single command as another user
$ su - alice -c "id"
Password:
uid=1001(alice) gid=1001(alice) groups=1001(alice),10(wheel)
```

### su vs su - (with dash)

| Command | Environment | Working Directory | Profile Files |
|---------|-------------|-------------------|---------------|
| `su alice` | Keeps current user's environment | Stays in current directory | Reads `~alice/.bashrc` only |
| `su - alice` | Loads target user's full environment | Changes to `~alice` | Reads login profile files |

**Best practice:** Always use `su -` (with the dash) for a clean environment. Using `su` without the dash can cause confusing behavior because environment variables from the original user leak through.

---

## newgrp — Change Current Group

### Purpose

Changes the current primary group for the session without logging out.

### When to Use

After being added to a new group with `usermod -aG`, the new group does not appear in your current session. Instead of logging out and back in, you can use `newgrp`:

```bash
# Administrator added alice to the "developers" group
$ sudo usermod -aG developers alice

# In alice's CURRENT session, the group doesn't show up yet
$ id
uid=1001(alice) gid=1001(alice) groups=1001(alice),10(wheel)
# "developers" is missing!

# Use newgrp to activate it
$ newgrp developers
$ id
uid=1001(alice) gid=1002(developers) groups=1001(alice),10(wheel),1002(developers)
# Now "developers" appears, and it's the current primary group
```

**Note:** `newgrp` starts a new shell. Type `exit` to return to the previous shell.

---

## chsh — Change Login Shell

### Purpose

Changes a user's login shell. The new shell must be listed in `/etc/shells`.

### Examples

```bash
# Change your own shell
$ chsh -s /bin/zsh
Changing shell for alice.
Password: 
Shell changed.

# Root can change any user's shell
$ sudo chsh -s /bin/zsh alice

# List valid shells
$ cat /etc/shells
/bin/sh
/bin/bash
/usr/bin/bash
/bin/zsh
/usr/bin/zsh
```

---

## chfn — Change Finger Information

### Purpose

Changes the GECOS field in `/etc/passwd` (full name, room number, phone numbers).

```bash
$ chfn alice
Changing finger information for alice.
Name [Alice Smith]: Alice Jones
Office []: Room 301
Office Phone []: 555-1234
Home Phone []: 
Finger information changed.

$ getent passwd alice
alice:x:1001:1001:Alice Jones,Room 301,555-1234,:/home/alice:/bin/bash
```

---

## last — Login History

### Purpose

Shows a list of recent logins from `/var/log/wtmp`.

### Examples

```bash
$ last -n 10
alice    pts/0        192.168.1.100    Mon Sep 22 10:15   still logged in
bob      pts/1        192.168.1.101    Mon Sep 22 09:00 - 12:30  (03:30)
alice    pts/0        192.168.1.100    Sun Sep 21 14:00 - 18:45  (04:45)
reboot   system boot  6.1.0-1.el9      Fri Sep 20 08:00   still running

# Show failed login attempts
$ sudo lastb -n 5
baduser  ssh:notty    10.0.0.5         Mon Sep 22 03:15 - 03:15  (00:00)
```

---

## lastlog — Last Login Times

### Purpose

Shows the most recent login time for every user account.

```bash
$ lastlog
Username         Port     From             Latest
root             pts/0                     Mon Sep 22 08:00:00 -0400 2024
alice            pts/0    192.168.1.100    Mon Sep 22 10:15:00 -0400 2024
nginx                                     **Never logged in**
```

**Note:** On some newer systems using `systemd-logind`, `lastlog` may not be available or may show incomplete data. Use `last` or `journalctl` instead.

---

## faillock — Failed Login Tracking (RHEL 9)

### Purpose

Manages the `pam_faillock` module, which tracks failed login attempts and can lock accounts after too many failures.

### Examples

```bash
# Check failed login attempts for a user
$ sudo faillock --user alice
alice:
When                Type  Source                             Valid
2024-09-22 10:01:05 RHOST 192.168.1.200                     V
2024-09-22 10:01:08 RHOST 192.168.1.200                     V
2024-09-22 10:01:11 RHOST 192.168.1.200                     V

# Reset (unlock) a locked-out user
$ sudo faillock --user alice --reset

# Check the status again
$ sudo faillock --user alice
alice:
When                Type  Source                             Valid
```

**Note:** `faillock` replaced `pam_tally2` in RHEL 8+. On older systems, use `pam_tally2` instead. On Debian/Ubuntu, `faillock` may not be installed by default.

---

## Detailed Procedures

### Creating Users with Specific UIDs

When you need consistent UIDs across multiple servers (important for NFS, shared storage, and container environments):

```bash
# Check if UID 1500 is already in use
$ getent passwd 1500
# (no output means it's available)

# Create user with specific UID
$ sudo useradd -u 1500 -m alice
$ id alice
uid=1500(alice) gid=1500(alice) groups=1500(alice)
```

### Creating Non-Login Service Accounts

```bash
# Create a service account for myapp
$ sudo useradd -r -s /sbin/nologin -d /opt/myapp -c "MyApp Service Account" myapp

# Set ownership of the application directory
$ sudo mkdir -p /opt/myapp
$ sudo chown myapp:myapp /opt/myapp

# Verify the account cannot be used for interactive login
$ sudo su - myapp
This account is currently not available.

# The service can still run as this user via systemd:
# [Service]
# User=myapp
# Group=myapp
```

### Locking and Unlocking Accounts

There are two ways to lock a user account, and they are subtly different:

| Method | Command | What It Does | SSH Keys? |
|--------|---------|-------------|-----------|
| Password lock | `passwd -l alice` | Adds `!` before password hash | **Still work** |
| Password lock | `usermod -L alice` | Adds `!` before password hash | **Still work** |
| Account expire | `usermod -e 1 alice` | Sets account expiry to Jan 2, 1970 | **Blocked** |
| Full lockout | Both commands above | Locks password AND expires account | **Blocked** |

**To fully prevent all login methods:**

```bash
# Lock the password AND expire the account
$ sudo usermod -L -e 1 alice

# Verify
$ sudo passwd -S alice
alice LK 2024-09-22 0 99999 7 -1 (Password locked.)

$ sudo chage -l alice | grep "Account expires"
Account expires                     : Jan 02, 1970
```

**To restore access:**

```bash
# Unlock the password AND remove expiry
$ sudo usermod -U -e '' alice
```

### Handling UID Changes

If you need to change a user's UID:

```bash
# 1. Log the user out (required)
# 2. Change the UID
$ sudo usermod -u 2000 alice

# 3. Fix file ownership - the files still have the OLD UID
$ sudo find / -user 1001 -exec chown 2000 {} \; 2>/dev/null

# 4. Verify
$ id alice
uid=2000(alice) gid=1001(alice) groups=1001(alice)

$ ls -ln /home/alice/
# Should now show UID 2000, not 1001
```

### Why Group Changes Need a New Login

```bash
# Administrator adds alice to docker group
$ sudo usermod -aG docker alice

# In alice's current SSH session:
$ groups
alice wheel
# "docker" is NOT shown!

# This is because group memberships are loaded into process credentials at login time
# The current shell's credentials were set when alice logged in, BEFORE the change

# Solutions:
# Option 1: Log out and back in
$ exit
$ ssh alice@server

# Option 2: Use newgrp (for the current session)
$ newgrp docker

# Option 3: Start a new login shell (keeps existing session)
$ su - alice
```

---

## Quick Reference Table

| Task | Command |
|------|---------|
| Create a user | `useradd -m alice` |
| Create a system user | `useradd -r -s /sbin/nologin myapp` |
| Set a password | `passwd alice` |
| Add to a group | `usermod -aG groupname alice` |
| Change shell | `usermod -s /bin/zsh alice` |
| Lock an account | `usermod -L -e 1 alice` |
| Unlock an account | `usermod -U -e '' alice` |
| Delete a user and home | `userdel -r alice` |
| Check identity | `id alice` |
| Check password status | `passwd -S alice` |
| Check aging info | `chage -l alice` |
| Force password change | `chage -d 0 alice` |
| View login history | `last alice` |
| Find orphaned files | `find / -nouser -ls` |
| List all users | `getent passwd` |
| Check failed logins | `faillock --user alice` |

---

## Interview Questions

### Q1: What is the difference between `useradd` and `adduser`?

**Answer:** On RHEL/AlmaLinux/CentOS, `adduser` is a symbolic link to `useradd` — they are identical. On Debian/Ubuntu, `adduser` is a separate interactive Perl script that creates the home directory, prompts for a password, and asks for user information, making it more user-friendly. `useradd` is the low-level command available on all distributions and is preferred for scripting because its behavior is predictable and non-interactive.

### Q2: Why is `usermod -aG` safer than `usermod -G`?

**Answer:** `usermod -G` replaces ALL supplementary groups with the specified list. If a user is in groups `wheel`, `developers`, and `docker`, running `usermod -G newgroup alice` removes them from all three groups and adds them only to `newgroup`. `usermod -aG` (with the `-a` flag for "append") adds the new group while preserving all existing group memberships. Always use `-aG` unless you intentionally want to reset all supplementary groups.

### Q3: What happens when you lock a user's password with `passwd -l`?

**Answer:** It adds a `!` character before the password hash in `/etc/shadow`, which prevents password authentication. However, this does NOT prevent SSH key-based login. To fully prevent all login, you should also expire the account with `usermod -e 1`. The combination of locked password and expired account prevents all login methods.

### Q4: How do you find all files owned by a deleted user?

**Answer:** Use `find / -nouser -ls` to find files whose owner UID does not match any existing user. After deleting a user, their files remain on disk with the original numeric UID but no username mapping. You should reassign these files to another user with `chown` or remove them.

### Q5: Why must a user log out and back in after being added to a group?

**Answer:** Group memberships are loaded into the process credentials when the user logs in. The kernel caches these credentials and passes them to every child process via `fork()`. Changes to `/etc/group` are not retroactively applied to running sessions. The user must start a new login session for the new groups to take effect. Alternatively, `newgrp groupname` can be used to activate a new group in the current session.

---

**Next Module:** [Module 3: Linux Group Management](03-linux-group-management.md) — Learn how to create and manage groups for collaborative access.

# Linux User Management — Command Cheatsheet
**Platform:** AlmaLinux 9 / RHEL 9  
**Audience:** Linux System Administration learners

---

## Table of Contents

1. [User Management Commands](#1-user-management-commands)
2. [Group Management Commands](#2-group-management-commands)
3. [Account Inspection](#3-account-inspection)
4. [Password and Account Aging](#4-password-and-account-aging)
5. [Permissions and Ownership](#5-permissions-and-ownership)
6. [ACLs (Access Control Lists)](#6-acls-access-control-lists)
7. [sudo and sudoers](#7-sudo-and-sudoers)
8. [Authentication Troubleshooting](#8-authentication-troubleshooting)
9. [Security Auditing](#9-security-auditing)
10. [Log Inspection](#10-log-inspection)
11. [Quick Reference Tables](#11-quick-reference-tables)

---

## 1. User Management Commands

### Core User Commands

| Command | Syntax | Description |
|---------|--------|-------------|
| `useradd` | `useradd [OPTIONS] USERNAME` | Create a new user account |
| `usermod` | `usermod [OPTIONS] USERNAME` | Modify an existing user account |
| `userdel` | `userdel [OPTIONS] USERNAME` | Delete a user account |
| `passwd` | `passwd [OPTIONS] [USERNAME]` | Set or change a user password |
| `chage` | `chage [OPTIONS] USERNAME` | Change user password expiry information |
| `id` | `id [OPTIONS] [USERNAME]` | Display user and group identity |
| `whoami` | `whoami` | Print effective username of current user |
| `who` | `who [OPTIONS]` | Show who is logged in |
| `w` | `w [OPTIONS] [USER]` | Show who is logged in and what they are doing |
| `users` | `users` | Print usernames of logged-in users (space-separated) |
| `getent` | `getent DATABASE [KEY]` | Query NSS databases (passwd, group, shadow, etc.) |
| `su` | `su [OPTIONS] [USERNAME]` | Switch user identity |
| `newgrp` | `newgrp [GROUP]` | Switch effective group for current session |
| `chsh` | `chsh [OPTIONS] [USERNAME]` | Change the login shell |
| `chfn` | `chfn [OPTIONS] [USERNAME]` | Change user finger/GECOS information |
| `last` | `last [OPTIONS] [USERNAME]` | Show history of logins and reboots |
| `lastlog` | `lastlog [OPTIONS]` | Show last login for each user |
| `faillock` | `faillock [OPTIONS]` | Display or reset failed authentication attempts |

---

### `useradd` — Create User

| Option | Long Form | Description |
|--------|-----------|-------------|
| `-c COMMENT` | `--comment` | Set GECOS/comment field (e.g., full name) |
| `-d DIR` | `--home-dir` | Set home directory path |
| `-e DATE` | `--expiredate` | Set account expiration date (YYYY-MM-DD) |
| `-f DAYS` | `--inactive` | Days after password expiry before account is disabled |
| `-g GROUP` | `--gid` | Set primary group (name or GID) |
| `-G GROUP,...` | `--groups` | Set supplementary groups (comma-separated) |
| `-m` | `--create-home` | Create home directory if it does not exist |
| `-M` | `--no-create-home` | Do not create home directory |
| `-N` | `--no-user-group` | Do not create a group with the same name as user |
| `-r` | `--system` | Create a system account (UID < 1000, no home by default) |
| `-s SHELL` | `--shell` | Set the login shell |
| `-u UID` | `--uid` | Specify a specific UID |
| `-U` | `--user-group` | Create a group with the same name as the user (default) |
| `-k SKELDIR` | `--skel` | Specify skeleton directory (default: `/etc/skel`) |
| `-K KEY=VAL` | `--key` | Override `/etc/login.defs` defaults |
| `-p PASSWORD` | `--password` | Set encrypted password directly |
| `-D` | `--defaults` | Display or set `useradd` defaults |

**Examples:**
```bash
# Create a standard user with home directory
useradd -m -c "Alice Smith" -s /bin/bash alice

# Create user with specific UID, GID, and supplementary groups
useradd -u 1500 -g developers -G sudo,docker -m bob

# Create a system account (no home, no login shell)
useradd -r -s /sbin/nologin -c "Apache Service" apache

# Create user with account expiration
useradd -m -e 2026-12-31 contractor

# View useradd defaults
useradd -D

# Change default shell for new users
useradd -D -s /bin/bash
```

---

### `usermod` — Modify User

| Option | Long Form | Description |
|--------|-----------|-------------|
| `-a` | `--append` | Append supplementary groups (use with `-G`) |
| `-c COMMENT` | `--comment` | Change GECOS/comment field |
| `-d DIR` | `--home` | Change home directory path |
| `-e DATE` | `--expiredate` | Change account expiration date (YYYY-MM-DD, or -1 to clear) |
| `-f DAYS` | `--inactive` | Change inactivity period after password expiry |
| `-g GROUP` | `--gid` | Change primary group |
| `-G GROUP,...` | `--groups` | Set supplementary groups (replaces existing unless `-a` used) |
| `-l NEWNAME` | `--login` | Change login name |
| `-L` | `--lock` | Lock the user account (prepends `!` to shadow password) |
| `-m` | `--move-home` | Move home directory contents to new location (use with `-d`) |
| `-p PASSWORD` | `--password` | Set new encrypted password |
| `-s SHELL` | `--shell` | Change login shell |
| `-u UID` | `--uid` | Change the UID |
| `-U` | `--unlock` | Unlock the user account (removes `!` from shadow password) |

**Examples:**
```bash
# Add user to supplementary group without removing existing groups
usermod -aG wheel alice

# Add user to multiple groups
usermod -aG docker,developers,testers bob

# Change home directory and move contents
usermod -d /home/newhome -m alice

# Lock and unlock an account
usermod -L alice
usermod -U alice

# Rename a user (also rename home dir separately)
usermod -l alicia alice
usermod -d /home/alicia -m alicia

# Set account expiration
usermod -e 2027-01-01 contractor
usermod -e "" contractor   # Remove expiration
```

---

### `userdel` — Delete User

| Option | Long Form | Description |
|--------|-----------|-------------|
| `-f` | `--force` | Force deletion even if user is logged in; also removes home and mail spool |
| `-r` | `--remove` | Remove home directory and mail spool |
| `-Z` | `--selinux-user` | Remove SELinux user mapping for the user |

**Examples:**
```bash
# Delete user but keep home directory (for audit/archive)
userdel alice

# Delete user and remove home directory and mail spool
userdel -r bob

# Force deletion of logged-in user (use with caution)
userdel -rf contractor
```

---

### `passwd` — Manage Passwords

| Option | Long Form | Description |
|--------|-----------|-------------|
| `-d` | `--delete` | Delete (empty) the user password; account becomes passwordless |
| `-e` | `--expire` | Immediately expire the password (force change on next login) |
| `-i DAYS` | `--inactive` | Set inactivity period after password expiry |
| `-l` | `--lock` | Lock the account password |
| `-n DAYS` | `--mindays` | Set minimum days between password changes |
| `-q` | `--quiet` | Quiet mode |
| `-S` | `--status` | Display password status information |
| `-u` | `--unlock` | Unlock the account password |
| `-w DAYS` | `--warndays` | Set warning days before password expiry |
| `-x DAYS` | `--maxdays` | Set maximum days a password is valid |

**Examples:**
```bash
# Set password for current user (interactive)
passwd

# Set password for another user as root
passwd alice

# Force password change on next login
passwd -e bob

# Lock and unlock an account
passwd -l alice
passwd -u alice

# Show password status
passwd -S alice

# Set max password age
passwd -x 90 alice
```

---

### `chage` — Change Password Aging

| Option | Long Form | Description |
|--------|-----------|-------------|
| `-d DATE` | `--lastday` | Set date of last password change (YYYY-MM-DD or days since epoch; 0 forces change) |
| `-E DATE` | `--expiredate` | Set account expiration date (YYYY-MM-DD or -1 to disable) |
| `-I DAYS` | `--inactive` | Set inactivity period (days after expiry before account locks) |
| `-l` | `--list` | List account aging information |
| `-m DAYS` | `--mindays` | Set minimum days between password changes |
| `-M DAYS` | `--maxdays` | Set maximum days a password is valid |
| `-W DAYS` | `--warndays` | Set warning days before password expires |

**Examples:**
```bash
# View aging information for a user
chage -l alice

# Force password change on next login
chage -d 0 alice

# Set comprehensive password policy
chage -m 7 -M 90 -W 14 -I 30 alice

# Set account expiration
chage -E 2026-12-31 contractor

# Remove account expiration
chage -E -1 alice

# Interactive aging configuration
chage alice
```

---

### `id` — Display Identity

| Option | Description |
|--------|-------------|
| `-u` | Print effective UID only |
| `-g` | Print effective GID only |
| `-G` | Print all GIDs (supplementary + primary) |
| `-n` | Print name instead of number (use with `-u`, `-g`, or `-G`) |
| `-r` | Print real ID instead of effective ID |
| `-Z` | Print SELinux security context |

**Examples:**
```bash
id                    # All info for current user
id alice              # All info for alice
id -u alice           # UID of alice
id -gn alice          # Primary group name of alice
id -Gn alice          # All group names of alice
```

---

### `su` — Switch User

| Option | Description |
|--------|-------------|
| `-` or `-l` | Start a login shell (load full environment of target user) |
| `-c COMMAND` | Run command as the target user then exit |
| `-s SHELL` | Use specified shell instead of user's default |
| `-m` or `-p` | Preserve current environment (do not reset variables) |

**Examples:**
```bash
su alice              # Switch to alice, keep current environment
su - alice            # Switch to alice with full login environment
su -                  # Switch to root with full login environment
su -c "whoami" alice  # Run one command as alice
su -c "cat /etc/shadow" root   # Run command as root
```

---

### `last`, `lastlog`, `who`, `w`, `users`

| Command | Example | Output Description |
|---------|---------|-------------------|
| `last` | `last -n 20` | Last 20 logins and reboots from `/var/log/wtmp` |
| `last alice` | `last alice` | Login history for user alice |
| `last reboot` | `last reboot` | System reboot history |
| `lastlog` | `lastlog` | Last login for all users |
| `lastlog -u alice` | `lastlog -u alice` | Last login for alice |
| `lastlog -b 30` | `lastlog -b 30` | Users who have not logged in for 30+ days |
| `lastlog -t 7` | `lastlog -t 7` | Users who logged in within the last 7 days |
| `who` | `who` | Who is logged in with terminal and login time |
| `who -a` | `who -a` | All info: dead processes, login time, etc. |
| `who am i` | `who am i` | Your own login identity (with terminal) |
| `w` | `w` | Who is logged in and their current process/activity |
| `w alice` | `w alice` | Activity of specific user |
| `users` | `users` | Simple space-separated list of logged-in usernames |

---

## 2. Group Management Commands

| Command | Syntax | Description |
|---------|--------|-------------|
| `groupadd` | `groupadd [OPTIONS] GROUPNAME` | Create a new group |
| `groupmod` | `groupmod [OPTIONS] GROUPNAME` | Modify an existing group |
| `groupdel` | `groupdel GROUPNAME` | Delete a group |
| `gpasswd` | `gpasswd [OPTIONS] GROUP` | Administer group passwords and membership |
| `groups` | `groups [USERNAME]` | Show groups a user belongs to |
| `usermod -aG` | `usermod -aG GROUP USER` | Add user to a supplementary group |
| `newgrp` | `newgrp GROUP` | Switch active group in current session |
| `getent group` | `getent group GROUPNAME` | Query group database |

---

### `groupadd` — Create Group

| Option | Long Form | Description |
|--------|-----------|-------------|
| `-f` | `--force` | Exit successfully if group already exists |
| `-g GID` | `--gid` | Specify a specific GID |
| `-K KEY=VAL` | `--key` | Override `/etc/login.defs` defaults |
| `-o` | `--non-unique` | Allow creation of group with non-unique GID |
| `-p PASSWORD` | `--password` | Set encrypted group password |
| `-r` | `--system` | Create a system group (GID < 1000) |

**Examples:**
```bash
groupadd developers          # Create group with next available GID
groupadd -g 2000 devops      # Create group with specific GID
groupadd -r -g 500 svcgroup  # Create system group with specific GID
```

---

### `groupmod` — Modify Group

| Option | Long Form | Description |
|--------|-----------|-------------|
| `-g GID` | `--gid` | Change GID |
| `-n NEWNAME` | `--new-name` | Rename the group |
| `-o` | `--non-unique` | Allow non-unique GID |
| `-p PASSWORD` | `--password` | Change group password |

**Examples:**
```bash
groupmod -n engineering developers   # Rename group
groupmod -g 2500 engineering         # Change GID
```

---

### `groupdel` — Delete Group

```bash
groupdel developers      # Delete group (must not be a primary group of any user)
groupdel -f oldgroup     # Force deletion even if it is a primary group
```

> **Note:** A group cannot be deleted if it is the primary group of any existing user. Remove or reassign those users first.

---

### `gpasswd` — Manage Group Passwords and Members

| Option | Description |
|--------|-------------|
| `-a USER` | Add user to the group |
| `-d USER` | Remove user from the group |
| `-M USER,...` | Set the group member list (replaces existing) |
| `-A USER,...` | Set group administrators |
| `-r` | Remove group password |
| `-R` | Restrict access (only members can use `newgrp`) |

**Examples:**
```bash
gpasswd -a alice developers          # Add alice to developers
gpasswd -d bob developers            # Remove bob from developers
gpasswd -M alice,bob,carol devops    # Set exact membership list
gpasswd -A alice developers          # Make alice a group admin
gpasswd -r developers                # Remove group password
newgrp developers                    # Switch to developers group in session
```

---

### Group Membership — Quick Reference

```bash
# Show groups for current user
groups

# Show groups for specific user
groups alice

# Add user to supplementary group (persistent)
usermod -aG groupname username

# Add user to multiple supplementary groups
usermod -aG group1,group2,group3 username

# View group members
getent group developers
grep "^developers:" /etc/group

# Show all members of a group using getent
getent group wheel | cut -d: -f4 | tr ',' '\n'
```

---

## 3. Account Inspection

### `getent` — Query System Databases

| Command | Description |
|---------|-------------|
| `getent passwd` | List all user accounts from all NSS sources |
| `getent passwd alice` | Show `/etc/passwd`-style entry for alice |
| `getent passwd 1000` | Look up user by UID |
| `getent group` | List all groups |
| `getent group developers` | Show group entry for developers |
| `getent group 1001` | Look up group by GID |
| `getent shadow alice` | Show shadow entry (requires root) |
| `getent gshadow developers` | Show gshadow entry |
| `getent hosts` | Show hosts entries |

**Examples:**
```bash
# List all local users with UID >= 1000
getent passwd | awk -F: '$3 >= 1000 && $3 < 65534 {print $1, $3, $6, $7}'

# Check if user exists
getent passwd alice && echo "exists" || echo "not found"

# List all groups a user belongs to (cross-checks NSS)
getent group | grep -w alice
```

---

### `id` — Identity Details

```bash
id                        # uid, gid, groups for current user
id alice                  # uid, gid, groups for alice
id -u                     # Print UID only
id -un                    # Print username only
id -g                     # Print primary GID only
id -gn                    # Print primary group name only
id -G                     # Print all GIDs
id -Gn                    # Print all group names
id -Z                     # Print SELinux context
```

---

### `passwd -S` — Password Status

```bash
passwd -S alice
# Output fields:
# USERNAME STATUS DATE MIN MAX WARN INACTIVE
# alice PS 2025-01-15 7 90 14 30
```

| Status Code | Meaning |
|-------------|---------|
| `PS` | Password Set (usable) |
| `LK` | Locked (via `passwd -l` or `usermod -L`) |
| `NP` | No Password |

---

### `chage -l` — Aging Details

```bash
chage -l alice
# Shows:
# Last password change
# Password expires
# Password inactive
# Account expires
# Minimum number of days between password change
# Maximum number of days between password change
# Number of days of warning before password expires
```

---

### `last` and `lastlog`

```bash
# Show last 10 logins
last -n 10

# Show logins since a specific date
last -s "2026-01-01"

# Show logins for specific user
last alice

# Show only login/logout (no still-logged-in)
last -p now

# Users who have never logged in
lastlog | grep "Never logged in"

# Users who have not logged in for 60 days
lastlog -b 60

# Show only specific user
lastlog -u alice
```

---

### `faillock` — Failed Authentication

```bash
# Show all failed login records
faillock

# Show failed attempts for specific user
faillock --user alice

# Reset failed attempt counter for a user
faillock --user alice --reset

# Show faillock configuration
cat /etc/security/faillock.conf
```

---

## 4. Password and Account Aging

### `passwd` Complete Options Reference

| Option | Effect |
|--------|--------|
| `passwd USERNAME` | Set/change password interactively |
| `passwd -S USERNAME` | Show password status |
| `passwd -l USERNAME` | Lock account (prepend `!` to hash) |
| `passwd -u USERNAME` | Unlock account |
| `passwd -d USERNAME` | Delete password (passwordless login if PAM allows) |
| `passwd -e USERNAME` | Expire password immediately (force change at next login) |
| `passwd -n DAYS USERNAME` | Set minimum days between changes |
| `passwd -x DAYS USERNAME` | Set maximum days a password is valid |
| `passwd -w DAYS USERNAME` | Set warning days before expiry |
| `passwd -i DAYS USERNAME` | Set inactivity days after expiry |
| `passwd -n 0 -x 99999 USERNAME` | Effectively disable password aging |

---

### `chage` Complete Options Reference

| Option | Effect | Example |
|--------|--------|---------|
| `-l` | List current aging settings | `chage -l alice` |
| `-d DATE` | Set last password change date | `chage -d 2026-01-01 alice` |
| `-d 0` | Force password change on next login | `chage -d 0 alice` |
| `-m DAYS` | Minimum days between changes | `chage -m 7 alice` |
| `-M DAYS` | Maximum password age in days | `chage -M 90 alice` |
| `-W DAYS` | Warning days before expiry | `chage -W 14 alice` |
| `-I DAYS` | Inactivity period after expiry | `chage -I 30 alice` |
| `-E DATE` | Account expiration date (YYYY-MM-DD) | `chage -E 2026-12-31 alice` |
| `-E -1` | Remove account expiration | `chage -E -1 alice` |
| `-M 99999` | Disable maximum password age | `chage -M 99999 alice` |
| Interactive | Configure all settings interactively | `chage alice` |

---

### `usermod` Lock/Unlock Reference

| Command | Effect | Shadow Field |
|---------|--------|--------------|
| `usermod -L alice` | Lock account | Prepends `!` to password hash |
| `usermod -U alice` | Unlock account | Removes `!` from password hash |
| `usermod -e 2026-12-31 alice` | Set expiry | Sets field 8 of shadow |
| `usermod -e "" alice` | Clear expiry | Clears field 8 of shadow |
| `usermod -f 30 alice` | Set inactivity | Sets field 7 of shadow |
| `usermod -f -1 alice` | Disable inactivity | Sets field 7 to -1 |

---

### `/etc/login.defs` — System-Wide Password Policy

| Setting | Default (RHEL 9) | Description |
|---------|-----------------|-------------|
| `PASS_MAX_DAYS` | 99999 | Maximum days a password is valid |
| `PASS_MIN_DAYS` | 0 | Minimum days between password changes |
| `PASS_MIN_LEN` | 5 | Minimum password length (PAM overrides this) |
| `PASS_WARN_AGE` | 7 | Warning days before password expires |
| `UID_MIN` | 1000 | Minimum UID for regular users |
| `UID_MAX` | 60000 | Maximum UID for regular users |
| `SYS_UID_MIN` | 201 | Minimum UID for system accounts |
| `SYS_UID_MAX` | 999 | Maximum UID for system accounts |
| `GID_MIN` | 1000 | Minimum GID for regular groups |
| `GID_MAX` | 60000 | Maximum GID for regular groups |
| `CREATE_HOME` | yes | Create home directory by default |
| `UMASK` | 077 | Default umask for home directory creation |
| `USERGROUPS_ENAB` | yes | Create matching group for each new user |
| `ENCRYPT_METHOD` | yescrypt | Password hashing algorithm |

---

## 5. Permissions and Ownership

### `chmod` — Numeric (Octal) Mode

| Command | Syntax | Description |
|---------|--------|-------------|
| `chmod` | `chmod MODE FILE...` | Change file/directory permissions |
| `chmod -R` | `chmod -R MODE DIR` | Recursively change permissions |
| `chmod --reference` | `chmod --reference=REF FILE` | Copy permissions from reference file |

**Octal Permission Values:**

| Digit | Value | Permission |
|-------|-------|------------|
| `4` | Read | `r` |
| `2` | Write | `w` |
| `1` | Execute | `x` |
| `0` | None | `-` |

```bash
chmod 755 script.sh      # rwxr-xr-x
chmod 644 file.txt       # rw-r--r--
chmod 600 private.key    # rw-------
chmod 700 ~/scripts      # rwx------
chmod 1777 /tmp          # rwxrwxrwt (sticky bit)
chmod 4755 /usr/bin/prog # rwsr-xr-x (SUID)
chmod 2755 /usr/bin/prog # rwxr-sr-x (SGID)
chmod -R 750 /opt/app    # Recursively set
```

---

### `chmod` — Symbolic Mode

| Syntax | Description |
|--------|-------------|
| `chmod u+x file` | Add execute for owner (user) |
| `chmod g-w file` | Remove write from group |
| `chmod o=r file` | Set others to read-only |
| `chmod a+x file` | Add execute for all (user, group, others) |
| `chmod u=rwx,g=rx,o= file` | Set explicit permissions for all classes |
| `chmod u+s file` | Set SUID bit |
| `chmod g+s dir` | Set SGID bit on directory |
| `chmod +t dir` | Set sticky bit on directory |
| `chmod u-s file` | Remove SUID bit |
| `chmod a-x file` | Remove execute for all |

**Who specifiers:**

| Letter | Applies To |
|--------|-----------|
| `u` | User (owner) |
| `g` | Group |
| `o` | Others |
| `a` | All (u + g + o) |

**Operator specifiers:**

| Operator | Effect |
|----------|--------|
| `+` | Add permission |
| `-` | Remove permission |
| `=` | Set exactly (replaces existing) |

---

### `chown` — Change Ownership

| Command | Syntax | Description |
|---------|--------|-------------|
| `chown USER FILE` | `chown alice file.txt` | Change file owner |
| `chown USER:GROUP FILE` | `chown alice:devs file.txt` | Change owner and group |
| `chown :GROUP FILE` | `chown :devs file.txt` | Change group only (same as `chgrp`) |
| `chown -R USER:GROUP DIR` | `chown -R alice:devs /opt/app` | Recursive ownership change |
| `chown --reference=REF FILE` | `chown --reference=ref.txt target.txt` | Copy ownership from reference file |

**Examples:**
```bash
chown alice report.txt                  # Change owner to alice
chown alice:developers report.txt       # Change owner and group
chown :developers /opt/project          # Change group only
chown -R www-data:www-data /var/www     # Recursive for web directory
chown -R alice: /home/alice             # Set owner; group = alice's primary group
```

---

### `chgrp` — Change Group

```bash
chgrp developers file.txt          # Change group
chgrp -R developers /opt/project   # Recursive
chgrp --reference=ref.txt file.txt # Copy group from reference
```

---

### `stat` — Detailed File Information

```bash
stat file.txt
# Shows: File, Size, Blocks, IO Block, type,
#        Device, Inode, Links, Access (octal), Uid, Gid,
#        Access time, Modify time, Change time, Birth time

stat -c "%a %n" file.txt          # Print octal permissions and name
stat -c "%U:%G %a %n" /etc/passwd # User:Group, octal, name
stat -c "%F" file.txt             # File type only
```

---

### `ls -l` — Permission Listing

```bash
ls -l file.txt         # Long listing: permissions, links, owner, group, size, date, name
ls -la /home           # Include hidden files
ls -lZ file.txt        # Include SELinux context
ls -ld /directory      # Show directory itself, not contents
ls -l --sort=size      # Sort by size
```

**Permission string breakdown:**
```
-rwxr-xr-x  1  alice  developers  4096  Jan 15 12:00  script.sh
│└───┴───┴─ │  └───┘  └────────┘
│ u   g   o links owner   group
│
└── File type: - regular, d directory, l symlink, c char device, b block device, p pipe, s socket
```

---

### `namei` — Trace Path Permissions

```bash
# Trace permissions along each component of a path
namei -l /var/www/html/index.html

# Show all components with ownership
namei -om /etc/sudoers
```

---

### `find` with Permission Filters

```bash
# Find files with specific exact permission
find /home -perm 777

# Find files with at least these permissions
find /opt -perm -644

# Find files readable by any of these permissions
find /tmp -perm /022

# Find SUID files
find / -perm -4000 -type f 2>/dev/null

# Find SGID files
find / -perm -2000 -type f 2>/dev/null

# Find world-writable files
find / -perm -o+w -type f 2>/dev/null

# Find world-writable directories
find / -perm -o+w -type d 2>/dev/null

# Find files with sticky bit
find / -perm -1000 -type d 2>/dev/null
```

---

## 6. ACLs (Access Control Lists)

### ACL Commands Overview

| Command | Syntax | Description |
|---------|--------|-------------|
| `getfacl` | `getfacl [OPTIONS] FILE...` | Display ACL entries for files |
| `setfacl -m` | `setfacl -m ENTRY FILE` | Modify (add/update) an ACL entry |
| `setfacl -x` | `setfacl -x ENTRY FILE` | Remove a specific ACL entry |
| `setfacl -b` | `setfacl -b FILE` | Remove all ACL entries (keep base permissions) |
| `setfacl -d` | `setfacl -d -m ENTRY DIR` | Set default ACL on a directory |
| `setfacl -R` | `setfacl -R -m ENTRY DIR` | Apply ACL recursively |
| `setfacl -k` | `setfacl -k DIR` | Remove all default ACL entries |
| `setfacl --restore` | `setfacl --restore=FILE` | Restore ACLs from a saved file |

---

### `getfacl` — Display ACLs

```bash
getfacl file.txt                 # Show ACL for a file
getfacl -R /opt/project          # Show ACLs recursively
getfacl -p file.txt              # Show without file path prefix
getfacl -n file.txt              # Show UIDs/GIDs numerically

# Sample output:
# file: file.txt
# owner: alice
# group: developers
# user::rw-
# user:bob:r--
# group::r--
# mask::r--
# other::---
```

---

### `setfacl` — Modify ACLs

**ACL Entry Format:** `TYPE:QUALIFIER:PERMISSIONS`

| Type | Example | Meaning |
|------|---------|---------|
| `user::rwx` | `u::rwx` | Permissions for file owner |
| `user:alice:rw-` | `u:alice:rw` | Named user alice |
| `group::r-x` | `g::rx` | Permissions for file's group |
| `group:devs:rwx` | `g:devs:rwx` | Named group devs |
| `mask::r-x` | `m::rx` | Effective permission mask |
| `other::---` | `o::---` | All others |

**Examples:**
```bash
# Grant read-write to a specific user
setfacl -m u:bob:rw file.txt

# Grant read-execute to a specific group
setfacl -m g:developers:rx /opt/app

# Grant permissions to multiple entries at once
setfacl -m u:alice:rw,g:devs:r /opt/project/data

# Remove ACL entry for specific user
setfacl -x u:bob file.txt

# Remove ACL entry for specific group
setfacl -x g:oldgroup file.txt

# Remove all ACL entries (reset to standard permissions)
setfacl -b file.txt

# Set default ACL on directory (inherited by new files)
setfacl -d -m u:alice:rw /shared/project
setfacl -d -m g:developers:rwx /shared/project

# Apply ACL recursively to directory
setfacl -R -m g:developers:rx /opt/project

# Remove all default ACLs from directory
setfacl -k /shared/project

# Copy ACL from one file to another
getfacl source.txt | setfacl --set-file=- target.txt

# Backup ACLs
getfacl -R /opt/project > /backup/project_acls.bak

# Restore ACLs from backup
setfacl --restore=/backup/project_acls.bak
```

---

### ACL Mask and Effective Permissions

```bash
# The mask limits effective permissions for named users, named groups, and the file's group
# effective permission = stated permission AND mask

# Set the ACL mask explicitly
setfacl -m m::rx file.txt

# Recalculate mask automatically
setfacl --mask file.txt

# Check effective permissions in getfacl output:
# user:bob:rwx             #effective:r-x   <-- bob's effective is limited by mask
```

---

## 7. sudo and sudoers

### sudo Command Reference

| Command | Description |
|---------|-------------|
| `sudo COMMAND` | Run command as root |
| `sudo -u USER COMMAND` | Run command as specific user |
| `sudo -g GROUP COMMAND` | Run command as specific group |
| `sudo -i` | Start interactive login shell as root |
| `sudo -s` | Start interactive non-login shell as root (preserves some env) |
| `sudo -l` | List allowed (and forbidden) commands for current user |
| `sudo -l -U alice` | List allowed commands for alice (as root) |
| `sudo -k` | Invalidate cached credentials immediately |
| `sudo -K` | Remove credential cache file entirely |
| `sudo -v` | Extend credential cache timeout (re-authenticate) |
| `sudo -e FILE` | Edit file with $SUDOEDITOR (sudoedit) |
| `sudo -n COMMAND` | Non-interactive; fail if password required |
| `sudo -H` | Set HOME to target user's home directory |
| `sudo -b COMMAND` | Run command in the background |
| `sudo !!` | Re-run previous command with sudo (shell expansion) |

**Examples:**
```bash
sudo systemctl restart nginx          # Run as root
sudo -u postgres psql                 # Run as postgres user
sudo -i                               # Full root login shell
sudo -s                               # Root shell, keep environment
sudo -k                               # Forget cached password
sudo -l                               # What can I sudo?
sudo -l -U bob                        # What can bob sudo? (as root)
sudo visudo                           # Edit sudoers safely
sudo -e /etc/nginx/nginx.conf         # Edit file as root safely
```

---

### `visudo` — Edit sudoers Safely

| Command | Description |
|---------|-------------|
| `visudo` | Edit `/etc/sudoers` with syntax checking |
| `visudo -f /etc/sudoers.d/devops` | Edit a specific sudoers drop-in file |
| `visudo -c` | Check syntax of `/etc/sudoers` without editing |
| `visudo -c -f /etc/sudoers.d/custom` | Check syntax of a specific file |
| `visudo -s` | Strict mode; treat warnings as errors |

---

### sudoers File Syntax

**Basic Rule Format:**

```
WHO  WHERE=(AS_WHOM)  WHAT
│    │      │          │
│    │      │          └── Commands allowed (comma-separated, or ALL)
│    │      └── User/group to run as (defaults to root)
│    └── Hostname or ALL
└── Username, %groupname, or alias
```

**Examples:**
```bash
# Allow alice to run all commands as root on any host
alice   ALL=(ALL)   ALL

# Allow bob to run all commands without password
bob     ALL=(ALL)   NOPASSWD: ALL

# Allow members of wheel group (standard RHEL approach)
%wheel  ALL=(ALL)   ALL

# Allow members of wheel without password
%wheel  ALL=(ALL)   NOPASSWD: ALL

# Allow specific commands only
alice   ALL=(root)  /bin/systemctl restart nginx, /bin/systemctl status nginx

# Allow a command as a specific user
alice   ALL=(postgres) /usr/bin/psql

# Allow running as any user in the devops group
bob     ALL=(%devops) /bin/bash

# Deny a specific command (must appear before allow rules)
alice   ALL=(ALL)   !/bin/bash, !/bin/sh, !/usr/bin/su

# Include drop-in directory (already in default /etc/sudoers)
#includedir /etc/sudoers.d
```

---

### sudoers Aliases

```bash
# User Alias — group users under a single name
User_Alias ADMINS = alice, bob, carol
User_Alias DBAS   = dave, eve

# Command Alias — group commands
Cmnd_Alias SERVICES    = /bin/systemctl start *, /bin/systemctl stop *, /bin/systemctl restart *
Cmnd_Alias NETWORKING  = /sbin/ip, /sbin/ifconfig, /sbin/route
Cmnd_Alias DANGEROUS   = /bin/rm -rf *, /bin/bash, /bin/sh

# Host Alias — group hosts
Host_Alias WEBSERVERS  = web01, web02, web03
Host_Alias DBSERVERS   = db01, db02

# Runas Alias — group target users
Runas_Alias DB_USERS   = postgres, mysql, oracle

# Using aliases in rules
ADMINS      WEBSERVERS=(root)      SERVICES
DBAS        DBSERVERS=(DB_USERS)   NETWORKING
```

---

### sudoers Tags

| Tag | Effect |
|-----|--------|
| `NOPASSWD:` | Run following commands without password prompt |
| `PASSWD:` | Require password (overrides `NOPASSWD:` for specific commands) |
| `NOEXEC:` | Prevent command from executing further commands (uses LD_PRELOAD) |
| `EXEC:` | Allow command to execute other commands |
| `SETENV:` | Allow user to set environment variables |
| `NOSETENV:` | Prevent setting environment variables |
| `LOG_INPUT:` | Enable input logging for following commands |
| `NOLOG_INPUT:` | Disable input logging |
| `LOG_OUTPUT:` | Enable output logging |
| `NOLOG_OUTPUT:` | Disable output logging |

```bash
# Mix NOPASSWD and PASSWD in same rule
alice ALL=(ALL) NOPASSWD: /bin/systemctl status *, PASSWD: /bin/systemctl restart *

# Prevent shell escape from editors
alice ALL=(ALL) NOEXEC: /usr/bin/vim, /usr/bin/less

# Allow setting specific environment variables
Defaults>alice env_keep += "PATH EDITOR"
alice ALL=(ALL) SETENV: ALL
```

---

### sudoers Defaults

```bash
# In /etc/sudoers

# Require TTY (security measure, disabled by default in RHEL 9)
Defaults  requiretty

# Log sudo usage to syslog
Defaults  syslog=auth

# Set secure PATH for sudo commands
Defaults  secure_path="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"

# Set timeout for cached credentials (minutes)
Defaults  timestamp_timeout=15

# Warn about bad passwords
Defaults  badpass_message="Incorrect password. Try again."

# Set number of password retries
Defaults  passwd_tries=3

# Apply Defaults to specific user
Defaults:alice  !requiretty, timestamp_timeout=30

# Apply Defaults to specific command
Defaults!/usr/bin/vim  noexec

# Apply Defaults on specific host
Defaults@webservers  log_output
```

---

## 8. Authentication Troubleshooting

### Diagnostic Commands Reference

| Problem | Command | What to Check |
|---------|---------|---------------|
| User cannot log in | `passwd -S username` | Account locked (`LK`) or no password (`NP`) |
| Password expired | `chage -l username` | "Password expires" date in the past |
| Account expired | `chage -l username` | "Account expires" date in the past |
| Account locked after failures | `faillock --user username` | Count of failures and unlock time |
| User not found | `getent passwd username` | NSS resolution working |
| Group membership missing | `id username` | Supplementary groups listed |
| PAM configuration issue | `journalctl -u sshd` | PAM error messages |
| sudo not working | `sudo -l` | Check allowed commands |

---

### Step-by-Step Authentication Diagnosis

```bash
# Step 1: Does the account exist?
getent passwd alice
id alice

# Step 2: Is the password set and usable?
passwd -S alice
# PS = password set, LK = locked, NP = no password

# Step 3: Has the password expired?
chage -l alice
# Check "Password expires" and "Account expires" lines

# Step 4: Has the account been locked due to failures?
faillock --user alice
# Shows: failures, latest failure timestamp, unlock time

# Step 5: Reset failure counter if needed
faillock --user alice --reset

# Step 6: Check PAM configuration
cat /etc/pam.d/system-auth
cat /etc/pam.d/password-auth
cat /etc/security/faillock.conf

# Step 7: Check SSH-specific authentication
journalctl -u sshd --since "1 hour ago"
tail -f /var/log/secure

# Step 8: Test login manually (as root, switch to user)
su - alice

# Step 9: Check shell is valid
grep alice /etc/passwd | cut -d: -f7
# If shell is /sbin/nologin or /bin/false, login is prevented

# Step 10: Check home directory
ls -la /home/alice
```

---

### `faillock` — Failed Login Management

```bash
# View failure records for all users
faillock

# View failures for specific user
faillock --user alice

# Sample output:
# alice:
# When                Type  Source                                           Valid
# 2026-09-20 14:32:11 TTY   pts/1                                            V
# 2026-09-20 14:32:25 TTY   pts/1                                            V

# Reset (unlock) a user after too many failures
faillock --user alice --reset

# faillock configuration (RHEL 9)
cat /etc/security/faillock.conf
# Key settings:
# deny = 5              (lock after N failures)
# fail_interval = 900   (failure counting window in seconds)
# unlock_time = 600     (seconds before auto-unlock; 0 = never)
```

---

### Common PAM Files (RHEL 9)

| File | Purpose |
|------|---------|
| `/etc/pam.d/system-auth` | Authentication for most local services |
| `/etc/pam.d/password-auth` | Authentication for services using password auth |
| `/etc/pam.d/sshd` | SSH-specific PAM stack |
| `/etc/pam.d/sudo` | PAM for sudo authentication |
| `/etc/pam.d/login` | Console login PAM stack |
| `/etc/pam.d/su` | PAM for su command |
| `/etc/security/faillock.conf` | faillock module configuration |
| `/etc/security/pwquality.conf` | Password quality requirements |
| `/etc/security/limits.conf` | User resource limits |
| `/etc/security/access.conf` | Login access control |

---

## 9. Security Auditing

### Find SUID Files

```bash
# Find all SUID files system-wide (readable)
find / -perm -4000 -type f 2>/dev/null

# Find SUID files, show ownership and permissions
find / -perm -4000 -type f -exec ls -l {} \; 2>/dev/null

# Find SUID files not owned by root
find / -perm -4000 -not -user root -type f 2>/dev/null

# Find SUID in specific directories
find /usr /bin /sbin -perm -4000 -type f
```

---

### Find SGID Files

```bash
# Find all SGID files
find / -perm -2000 -type f 2>/dev/null

# Find SGID directories
find / -perm -2000 -type d 2>/dev/null

# Find both SUID and SGID
find / -perm /6000 -type f 2>/dev/null
```

---

### Find World-Writable Files and Directories

```bash
# World-writable files (excluding /proc and /sys)
find / -perm -o+w -type f -not -path "/proc/*" -not -path "/sys/*" 2>/dev/null

# World-writable directories
find / -perm -o+w -type d -not -path "/proc/*" -not -path "/sys/*" 2>/dev/null

# World-writable files not owned by root
find / -perm -o+w -type f -not -user root 2>/dev/null

# World-writable and NOT sticky bit (more dangerous)
find / -perm -o+w -not -perm -1000 -type d 2>/dev/null
```

---

### Find Files Without Valid Owner or Group

```bash
# Files with no valid user owner
find / -nouser -not -path "/proc/*" 2>/dev/null

# Files with no valid group owner
find / -nogroup -not -path "/proc/*" 2>/dev/null

# Both conditions
find / \( -nouser -o -nogroup \) -not -path "/proc/*" 2>/dev/null
```

---

### Password File Auditing with `awk`

```bash
# List users with UID 0 (should only be root)
awk -F: '$3 == 0 {print $1, $3}' /etc/passwd

# List users with empty password field in shadow
awk -F: '$2 == "" {print $1}' /etc/shadow

# List users with no login shell restriction
awk -F: '$7 !~ /nologin|false/ && $3 >= 1000 {print $1, $7}' /etc/passwd

# Find duplicate UIDs
awk -F: '{print $3}' /etc/passwd | sort | uniq -d

# Find duplicate usernames
awk -F: '{print $1}' /etc/passwd | sort | uniq -d

# List system accounts that have a valid shell
awk -F: '$3 < 1000 && $7 !~ /nologin|false|sync|shutdown|halt/ {print $1, $3, $7}' /etc/passwd

# List accounts with non-standard home directories
awk -F: '$6 !~ /^\/home\// && $3 >= 1000 {print $1, $6}' /etc/passwd

# Show users with password hash (not locked, not empty)
awk -F: '$2 !~ /^[!*]/ && $2 != "" {print $1}' /etc/shadow

# Find users who have not changed password in >90 days
awk -F: -v cutoff=$(( $(date +%s) / 86400 - 90 )) '$3 > 0 && $3 < cutoff {print $1}' /etc/shadow
```

---

### Group File Auditing

```bash
# Find duplicate GIDs
awk -F: '{print $3}' /etc/group | sort | uniq -d

# Find groups with no members
awk -F: '$4 == "" {print $1}' /etc/group

# Find users listed in group file but not in passwd
awk -F: '{if ($4 != "") split($4, a, ","); for (u in a) print a[u]}' /etc/group | \
  sort -u | \
  while read u; do getent passwd "$u" > /dev/null 2>&1 || echo "Missing: $u"; done

# List all groups a user belongs to (from group file)
awk -F: -v user="alice" '$4 ~ user {print $1}' /etc/group
```

---

### Additional Security Checks

```bash
# Check for .rhosts or .netrc files (dangerous legacy)
find /home -name ".rhosts" -o -name ".netrc" 2>/dev/null

# Check for world-readable shadow file (should not be readable)
ls -l /etc/shadow /etc/gshadow

# Check /etc/passwd and /etc/shadow permissions
stat /etc/passwd /etc/shadow /etc/group /etc/gshadow

# Identify users with sudo access
grep -E "^[^#].*ALL.*ALL" /etc/sudoers 2>/dev/null
grep -rE "^[^#].*ALL" /etc/sudoers.d/ 2>/dev/null

# Find SSH authorized_keys files
find /home -name "authorized_keys" -type f 2>/dev/null
find /root -name "authorized_keys" -type f 2>/dev/null

# Check for executable files in home directories
find /home -perm -o+x -type f 2>/dev/null
```

---

## 10. Log Inspection

### `journalctl` — systemd Journal Commands

| Command | Description |
|---------|-------------|
| `journalctl -u sshd` | All logs for SSH daemon |
| `journalctl -u sshd -n 50` | Last 50 lines for SSH daemon |
| `journalctl -u sshd -f` | Follow (tail) SSH daemon log |
| `journalctl _UID=1000` | All logs from UID 1000 |
| `journalctl _COMM=sudo` | All sudo-related log entries |
| `journalctl _COMM=su` | All su-related log entries |
| `journalctl -p err` | Only error-level and above entries |
| `journalctl -p warning..err` | Warning through error entries |
| `journalctl --since "1 hour ago"` | Entries from last hour |
| `journalctl --since "2026-09-01" --until "2026-09-22"` | Date range |
| `journalctl -b` | Entries since last boot |
| `journalctl -b -1` | Entries from previous boot |
| `journalctl -k` | Kernel messages only |
| `journalctl -g "Failed password"` | Grep within journal |
| `journalctl -o json-pretty` | JSON output format |

**Authentication-Specific journalctl Commands:**
```bash
# Failed SSH login attempts
journalctl -u sshd | grep "Failed password"

# Successful SSH logins
journalctl -u sshd | grep "Accepted"

# All authentication events (PAM)
journalctl -u sshd -u sudo -u login --since "today"

# sudo usage
journalctl _COMM=sudo

# su usage
journalctl _COMM=su

# Account lockout events (faillock/PAM)
journalctl | grep -i "faillock\|pam_faillock\|account locked"

# Password changes
journalctl | grep "password changed\|passwd"

# User creation/deletion
journalctl | grep "useradd\|userdel\|usermod"
```

---

### Log File Locations

| File | OS | Purpose |
|------|----|---------|
| `/var/log/secure` | RHEL / AlmaLinux / CentOS | Authentication, sudo, ssh, su events |
| `/var/log/auth.log` | Debian / Ubuntu | Authentication events |
| `/var/log/messages` | RHEL / AlmaLinux | General system messages |
| `/var/log/syslog` | Debian / Ubuntu | General system messages |
| `/var/log/wtmp` | All | Binary: login/logout history (read with `last`) |
| `/var/log/btmp` | All | Binary: failed login history (read with `lastb`) |
| `/var/log/lastlog` | All | Binary: last login per user (read with `lastlog`) |
| `/var/log/faillog` | Debian | Failed login attempts (legacy pam_tally2) |
| `/var/run/faillock/` | RHEL 9 | faillock per-user failure records |
| `/var/log/audit/audit.log` | All (auditd) | Detailed audit events |
| `/var/log/sudo.log` | All (if configured) | sudo command logs |

---

### Reading Binary Log Files

```bash
# Read /var/log/wtmp (login history)
last -f /var/log/wtmp

# Read /var/log/btmp (failed login attempts)
lastb                          # Read /var/log/btmp
lastb -n 20                    # Last 20 failed attempts
lastb alice                    # Failed attempts for alice

# Read /var/log/lastlog
lastlog
lastlog -u alice

# Search /var/log/secure (RHEL)
grep "Failed password" /var/log/secure
grep "Accepted publickey" /var/log/secure
grep "sudo:" /var/log/secure
grep "session opened" /var/log/secure | grep "for user root"

# Count failed logins by IP
grep "Failed password" /var/log/secure | awk '{print $(NF-3)}' | sort | uniq -c | sort -rn | head -20

# Count failed logins by username
grep "Failed password" /var/log/secure | awk '{print $(NF-5)}' | sort | uniq -c | sort -rn
```

---

### auditd Commands (Advanced)

```bash
# Search audit log for user modifications
ausearch -m ADD_USER,DEL_USER,MODIFY_USER

# Search for failed sudo attempts
ausearch -m USER_AUTH --success no

# Search for specific user activity
ausearch -ua alice

# Generate audit report
aureport

# Report on failed events
aureport --failed

# Report on authentication
aureport --auth

# Watch file for changes (add audit rule)
auditctl -w /etc/passwd -p wa -k passwd_changes
auditctl -w /etc/sudoers -p wa -k sudoers_changes

# List active audit rules
auditctl -l
```

---

## 11. Quick Reference Tables

### Common Numeric Permissions

| Permission | Octal | Symbolic | Meaning | Typical Use |
|------------|-------|----------|---------|-------------|
| `777` | `0777` | `rwxrwxrwx` | Full access for everyone | Temporary shared dirs (avoid in production) |
| `775` | `0775` | `rwxrwxr-x` | Full for owner/group, read-execute for others | Group collaboration dirs |
| `755` | `0755` | `rwxr-xr-x` | Full for owner, read-execute for others | Executables, public web dirs |
| `750` | `0750` | `rwxr-x---` | Full for owner, read-execute for group, none for others | Application directories |
| `700` | `0700` | `rwx------` | Full for owner only | Private scripts, user home dirs |
| `664` | `0664` | `rw-rw-r--` | Read-write for owner/group, read for others | Group-editable files |
| `644` | `0644` | `rw-r--r--` | Read-write for owner, read for others | Standard config files, web content |
| `640` | `0640` | `rw-r-----` | Read-write for owner, read for group, none for others | Sensitive config files |
| `600` | `0600` | `rw-------` | Read-write for owner only | SSH private keys, sensitive files |
| `444` | `0444` | `r--r--r--` | Read-only for everyone | Public read-only reference files |
| `440` | `0440` | `r--r-----` | Read for owner and group only | Sensitive read-only files |
| `400` | `0400` | `r--------` | Read for owner only | Maximally restricted files |
| `000` | `0000` | `----------` | No access for anyone | Temporarily revoked access |

---

### Special Permission Bits

| Bit | Numeric | Symbolic (Set) | On File | On Directory |
|-----|---------|---------------|---------|-------------|
| **SUID** | `4000` | `u+s` (shows as `s` in user execute) | Executes with **file owner's** UID instead of caller's UID | No standard effect |
| **SGID** | `2000` | `g+s` (shows as `s` in group execute) | Executes with **file group's** GID instead of caller's GID | New files inherit **directory's group** instead of creator's primary group |
| **Sticky Bit** | `1000` | `+t` (shows as `t` in other execute) | Legacy: keep executable in memory (ignored on modern Linux) | Only the file **owner** (or root) can delete files, even if directory is world-writable |

**Combined with base permissions:**
```bash
chmod 4755 /usr/bin/prog   # SUID + rwxr-xr-x  → shows: rwsr-xr-x
chmod 2755 /usr/bin/prog   # SGID + rwxr-xr-x  → shows: rwxr-sr-x
chmod 1777 /tmp            # Sticky + rwxrwxrwx → shows: rwxrwxrwt
chmod 6755 /usr/bin/prog   # SUID+SGID + 755   → shows: rwsr-sr-x

# Capital S or T = bit set but execute bit NOT set (potential misconfiguration)
chmod 4644 file            # SUID without execute → shows: rwSr--r-- (warning sign)
chmod 1666 /tmp/shared     # Sticky without execute → shows: rw-rw-rwT
```

---

### Common umask Values

| umask | File Result | Dir Result | Calculation | Typical Use Case |
|-------|-------------|------------|-------------|-----------------|
| `022` | `644` (rw-r--r--) | `755` (rwxr-xr-x) | 666-022=644, 777-022=755 | Default RHEL/system umask; files readable by all |
| `027` | `640` (rw-r-----) | `750` (rwxr-x---) | 666-027=640, 777-027=750 | Secure default; others have no access |
| `077` | `600` (rw-------) | `700` (rwx------) | 666-077=600, 777-077=700 | Maximum privacy; only owner has access |
| `002` | `664` (rw-rw-r--) | `775` (rwxrwxr-x) | 666-002=664, 777-002=775 | Group collaboration; group can write |
| `007` | `660` (rw-rw----) | `770` (rwxrwx---) | 666-007=660, 777-007=770 | Group-private files; others have no access |
| `033` | `644` (rw-r--r--) | `744` (rwxr--r--) | 666-033=633→644, 777-033=744 | Unusual; execute removed for group/others on dirs |
| `066` | `600` (rw-------) | `711` (rwx--x--x) | 666-066=600, 777-066=711 | Rare; dirs traversable but no read |
| `113` | `664` (rw-rw-r--) | `664` (rw-rw-r--) | Edge case; avoid umask affecting execute |

> **How umask works:**  
> Files start with max `0666` (no execute by default).  
> Directories start with max `0777`.  
> umask bits are subtracted (bitwise AND with complement).  
> `chmod = max_permissions & ~umask`

```bash
# View current umask
umask                  # Octal (e.g., 0022)
umask -S               # Symbolic (e.g., u=rwx,g=rx,o=rx)

# Set umask for current session
umask 027

# Set umask permanently (add to ~/.bashrc or /etc/profile)
echo "umask 027" >> ~/.bashrc

# Set system-wide umask
echo "UMASK 027" >> /etc/login.defs
# Or edit /etc/profile or /etc/bashrc
```

---

### Important Configuration Files

| File | Purpose | Key Contents |
|------|---------|-------------|
| `/etc/passwd` | User account database | username, UID, GID, GECOS, home, shell |
| `/etc/shadow` | Encrypted passwords and aging | password hash, min/max days, expiry |
| `/etc/group` | Group database | group name, GID, member list |
| `/etc/gshadow` | Group passwords and admins | encrypted group password, admins, members |
| `/etc/login.defs` | Default values for shadow suite | UID/GID ranges, password policy, home creation |
| `/etc/default/useradd` | Default values for `useradd` | SHELL, GROUP, HOME base, SKEL, EXPIRE, INACTIVE |
| `/etc/skel/` | Skeleton directory | Template files copied to new user home dirs |
| `/etc/sudoers` | Main sudo configuration | Access rules, aliases, defaults |
| `/etc/sudoers.d/` | Drop-in sudo rule files | Modular sudo rules (included by `/etc/sudoers`) |
| `/etc/security/faillock.conf` | faillock module config | deny, fail_interval, unlock_time, dir |
| `/etc/security/pwquality.conf` | Password quality rules | minlen, dcredit, ucredit, retry |
| `/etc/security/limits.conf` | User resource limits | max files, processes, memory per user/group |
| `/etc/security/access.conf` | Login access control | Allow/deny login by user, group, origin |
| `/etc/pam.d/system-auth` | PAM auth stack (RHEL) | Module stack for local authentication |
| `/etc/pam.d/password-auth` | PAM password auth (RHEL) | Module stack for password-based services |
| `/etc/pam.d/sshd` | PAM SSH config | SSH-specific authentication stack |
| `/etc/pam.d/sudo` | PAM sudo config | sudo authentication stack |
| `/etc/nsswitch.conf` | Name Service Switch | Order of resolution for passwd, group, shadow |

---

### Sudoers Syntax Quick Reference

```
# Rule format:
# USER  HOST=(RUNAS)  [TAGS:]  COMMANDS

# Basic examples:
alice        ALL=(ALL)        ALL                          # alice: any host, any user, any command
%wheel       ALL=(ALL)        NOPASSWD: ALL                # wheel group: no password required
bob          webserver=(root) /bin/systemctl restart nginx # bob: only on webserver, only restart nginx

# Alias definitions (must appear before use):
User_Alias   WEBADMINS  = alice, bob
Cmnd_Alias   WEB_CMDS   = /bin/systemctl restart nginx, /bin/systemctl reload nginx
Host_Alias   WEBSERVERS = web01, web02, web03
Runas_Alias  SERVICES   = www-data, nginx

# Using aliases:
WEBADMINS    WEBSERVERS=(SERVICES)  WEB_CMDS

# Defaults directives:
Defaults                  requiretty                     # Require real TTY
Defaults:alice            !requiretty                    # Override for alice
Defaults@webservers       log_output                     # Override for host
Defaults!/usr/bin/vim     noexec                         # Override for command

# Negation (deny before allow — order matters):
alice        ALL=(ALL)  !/bin/bash, !/bin/sh             # Deny shells
alice        ALL=(ALL)  NOPASSWD: ALL                    # Allow everything else

# Include drop-in files:
@include /etc/sudoers.d                                  # RHEL 9 modern syntax
#includedir /etc/sudoers.d                               # Legacy syntax (also works)
```

---

### `/etc/shadow` Field Reference

```
alice:$6$rounds=5000$salt$hash:19371:7:90:14:30:19723:
│     │                        │     │ │  │  │  │     │
│     │                        │     │ │  │  │  │     └── Reserved (unused)
│     │                        │     │ │  │  │  └── Account expiration (days since epoch)
│     │                        │     │ │  │  └── Inactivity period (days after expiry)
│     │                        │     │ │  └── Warning days before expiry
│     │                        │     │ └── Maximum days (password validity)
│     │                        │     └── Minimum days between changes
│     │                        └── Last change (days since Jan 1, 1970)
│     └── Encrypted password (! or !! = locked; * = disabled/system)
└── Username
```

---

### `/etc/passwd` Field Reference

```
alice:x:1001:1001:Alice Smith:/home/alice:/bin/bash
│     │ │    │    │           │           │
│     │ │    │    │           │           └── Login shell
│     │ │    │    │           └── Home directory
│     │ │    │    └── GECOS/Comment field
│     │ │    └── Primary GID
│     │ └── UID
│     └── Password placeholder (x = in shadow, empty = no password)
└── Username
```

---

### Common Shell One-Liners

```bash
# List all human users (UID 1000-60000)
awk -F: '$3 >= 1000 && $3 < 60000 {print $1}' /etc/passwd

# List locked accounts
awk -F: '$2 ~ /^!/ {print $1}' /etc/shadow

# List passwordless accounts
awk -F: '$2 == "" {print $1}' /etc/shadow

# Show all sudo-capable users (direct + wheel group)
grep -E "^%wheel|^sudo" /etc/sudoers | grep -v "^#"
getent group wheel | cut -d: -f4 | tr ',' '\n'

# List users by their shell
awk -F: '{print $7}' /etc/passwd | sort | uniq -c | sort -rn

# Show users with expiring passwords (within 14 days)
for u in $(awk -F: '$3 >= 1000 {print $1}' /etc/passwd); do
  chage -l "$u" 2>/dev/null | grep -q "never" || \
  chage -l "$u" | awk -v user="$u" '/Password expires/{print user": "$0}'
done

# Force all users to change password at next login
for u in $(awk -F: '$3 >= 1000 && $3 < 60000 {print $1}' /etc/passwd); do
  chage -d 0 "$u"
done

# List home directories and their owners
ls -la /home | awk 'NR>1 {print $3, $4, $9}'

# Check if any user's UID/GID don't match between passwd and group
awk -F: '{print $1, $4}' /etc/passwd | while read user gid; do
  getent group "$gid" | grep -q "$user" || echo "$user primary group $gid not found"
done
```

---

*Cheatsheet version: 2026-09 | Platform: AlmaLinux 9 / RHEL 9 | Part of Linux 101 — User Management series*

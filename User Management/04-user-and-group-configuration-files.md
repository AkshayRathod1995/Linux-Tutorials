# Module 4: User and Group Configuration Files

## Introduction

Linux stores user and group information in several plain-text configuration files. Understanding the format, purpose, and security implications of these files is essential for administration and troubleshooting. This module covers every important file in detail.

---

## /etc/passwd — User Account Database

### Purpose

The primary user account database. Every user on the system has an entry in this file. Programs like `ls`, `ps`, and `id` read this file to convert UIDs to usernames.

### File Properties

| Property | Value |
|----------|-------|
| Permissions | `-rw-r--r--` (644) — readable by everyone |
| Owner | `root:root` |
| Modified by | `useradd`, `usermod`, `userdel`, `vipw` |
| Read by | Almost every program that needs username information |

### Format

Each line represents one user account with 7 colon-separated fields:

```
username:password:UID:GID:GECOS:home_directory:login_shell
```

### Field-by-Field Explanation

| # | Field | Description | Example |
|---|-------|-------------|---------|
| 1 | Username | Login name (1-32 characters, lowercase convention) | `alice` |
| 2 | Password | `x` means password is in `/etc/shadow` | `x` |
| 3 | UID | Numeric user ID | `1001` |
| 4 | GID | Numeric primary group ID | `1001` |
| 5 | GECOS | Comment field (full name, contact info) | `Alice Smith` |
| 6 | Home directory | Absolute path to home directory | `/home/alice` |
| 7 | Login shell | Program started at login | `/bin/bash` |

### Example Entries

```
root:x:0:0:root:/root:/bin/bash
bin:x:1:1:bin:/bin:/sbin/nologin
daemon:x:2:2:daemon:/sbin:/sbin/nologin
alice:x:1001:1001:Alice Smith:/home/alice:/bin/bash
bob:x:1002:1002:Bob Jones,Room 301,555-1234,:/home/bob:/bin/bash
nginx:x:993:991:Nginx web server:/var/lib/nginx:/sbin/nologin
nfsnobody:x:65534:65534:Anonymous NFS User:/var/lib/nfs:/sbin/nologin
```

**Explanation of the `alice` entry:**
- `alice` — username
- `x` — password stored in `/etc/shadow`
- `1001` — UID 1001 (regular user range)
- `1001` — primary group GID 1001 (private group "alice")
- `Alice Smith` — full name in GECOS field
- `/home/alice` — home directory
- `/bin/bash` — login shell

**Explanation of the GECOS field for `bob`:**

The GECOS field can contain comma-separated subfields: `Full Name,Room,Work Phone,Home Phone,Other`. The `chfn` command sets these.

### Why Is `/etc/passwd` World-Readable?

Many programs need to convert UIDs to usernames:
- `ls -l` converts file owner UID to username
- `ps aux` converts process owner UID to username
- `id` reads both `/etc/passwd` and `/etc/group`
- Email systems, logging daemons, etc.

If `/etc/passwd` were not readable by all users, basic commands like `ls -l` would only show numeric UIDs. Since the actual password is stored in `/etc/shadow` (not in this file), the security risk is minimal.

### Commands That Use This File

| Command | How It Uses `/etc/passwd` |
|---------|--------------------------|
| `getent passwd` | Reads all entries (via NSS) |
| `useradd` | Adds a new line |
| `usermod` | Modifies an existing line |
| `userdel` | Removes a line |
| `vipw` | Safe manual editing with file locking |
| `id` | Looks up UID-to-name mapping |
| `ls -l` | Converts file owner UID to username |

---

## /etc/shadow — Password Hash Database

### Purpose

Stores password hashes and password aging information. Separated from `/etc/passwd` to protect sensitive password data.

### File Properties

| Property | Value |
|----------|-------|
| Permissions | `----------` (0000) on RHEL 9, or `-rw-r-----` (0640) root:shadow on some distros |
| Owner | `root:root` |
| Modified by | `passwd`, `chage`, `useradd`, `usermod` |
| Read by | PAM modules during authentication (running as root) |

### Format

```
username:password_hash:lastchanged:min:max:warn:inactive:expire:reserved
```

### Field-by-Field Explanation

| # | Field | Description | Example |
|---|-------|-------------|---------|
| 1 | Username | Must match `/etc/passwd` | `alice` |
| 2 | Password hash | Hashed password or lock marker | `$y$j9T$abc...xyz` |
| 3 | Last changed | Days since Jan 1, 1970 when password was last changed | `19988` |
| 4 | Minimum | Minimum days between password changes | `0` |
| 5 | Maximum | Maximum days before password must be changed | `99999` |
| 6 | Warn | Days before expiry to warn user | `7` |
| 7 | Inactive | Days after expiry before account is disabled | `30` |
| 8 | Expire | Account expiration date (days since epoch) | `20088` |
| 9 | Reserved | Reserved for future use | (empty) |

### Password Hash Format

The password hash field uses the format: `$id$salt$hash`

| ID | Algorithm | Security | Notes |
|----|-----------|----------|-------|
| `$1$` | MD5-crypt | Weak — do not use | Legacy, breakable |
| `$5$` | SHA-256 | Adequate | Common default on older systems |
| `$6$` | SHA-512 | Strong | Default on RHEL 7/8 |
| `$y$` | yescrypt | Strongest | Default on RHEL 9 / AlmaLinux 9 |

Example hash (yescrypt):
```
$y$j9T$F5Jx8fS3d2sYk7G4Lz9Mn/$R7Hp2Q5vK3jN8mW1xB6tL9dC4fA0yE5i
```

- `$y$` — algorithm identifier (yescrypt)
- `j9T` — yescrypt parameters
- `F5Jx8fS3d2sYk7G4Lz9Mn/` — salt (random data)
- `R7Hp2Q5vK3jN8mW1xB6tL9dC4fA0yE5i` — the actual hash

### Special Password Field Values

| Value | Meaning | Login Allowed? |
|-------|---------|---------------|
| `$y$j9T$...` | Valid password hash | Yes (with correct password) |
| `!!` | No password ever set | No (password auth denied) |
| `!` | Password locked (prefix before hash) | No (password auth denied) |
| `!$y$j9T$...` | Locked password (hash preserved) | No (password auth denied, but hash can be restored) |
| `*` | Account disabled for password login | No (never had a password) |
| (empty) | No password required | Yes (no password needed — DANGEROUS) |

### Example Entries

```
root:$y$j9T$abc...xyz:19500:0:99999:7:::
alice:$y$j9T$def...uvw:19988:1:90:14:30:20088:
locked:!$y$j9T$ghi...rst:19500:0:99999:7:::
svcaccount:*:19500:0:99999:7:::
newuser:!!:19700:0:99999:7:::
insecure::19700:0:99999:7:::
```

**Explanation of `alice`'s entry:**
- `alice` — username
- `$y$j9T$def...uvw` — yescrypt password hash (password is set)
- `19988` — last changed ~September 2024 (19988 days since Jan 1, 1970)
- `1` — must wait at least 1 day between password changes
- `90` — password expires after 90 days
- `14` — warning starts 14 days before expiry
- `30` — account disabled 30 days after password expires
- `20088` — account expires ~December 2025
- (empty) — reserved field

**Explanation of `locked`'s entry:**
- `!$y$j9T$ghi...rst` — the `!` prefix means the password is locked. The hash is preserved, so unlocking restores the original password.

### Why Hashed, Not Encrypted?

- **Encryption** is reversible — if someone gets the key, they can recover the original password.
- **Hashing** is one-way — you cannot derive the password from the hash. You can only verify by hashing a candidate password and comparing.
- This means even if an attacker obtains `/etc/shadow`, they must crack the hashes (computationally expensive with modern algorithms like yescrypt).

### Why Restricted Permissions?

If users could read `/etc/shadow`, they could:
1. Copy the hashes
2. Run offline brute-force attacks against them
3. Potentially crack weak passwords

The restricted permissions ensure only root (and authentication services running as root) can read the hashes.

---

## /etc/group — Group Database

### Purpose

Stores group names, GIDs, and supplementary group membership.

### File Properties

| Property | Value |
|----------|-------|
| Permissions | `-rw-r--r--` (644) |
| Owner | `root:root` |
| Modified by | `groupadd`, `groupmod`, `groupdel`, `gpasswd`, `usermod` |

### Format

```
groupname:password:GID:member_list
```

### Field-by-Field Explanation

| # | Field | Description | Example |
|---|-------|-------------|---------|
| 1 | Group name | Name of the group | `developers` |
| 2 | Password | `x` or empty (group passwords are rarely used) | `x` |
| 3 | GID | Numeric group ID | `1002` |
| 4 | Members | Comma-separated list of users with this as supplementary group | `alice,bob,charlie` |

### Important Note About Membership

The member list in `/etc/group` only shows users who have this group as a **supplementary group**. Users whose **primary group** matches this GID are NOT listed here — their primary GID is in `/etc/passwd`.

To find ALL members of a group (both primary and supplementary):

```bash
# Find users with "developers" as supplementary group (from /etc/group)
$ getent group developers
developers:x:1002:alice,bob

# Find users with "developers" as primary group (from /etc/passwd)
$ awk -F: '$4 == 1002 {print $1}' /etc/passwd
charlie

# Combined: all members of group "developers"
$ getent group developers | cut -d: -f4 ; awk -F: '$4 == 1002 {print $1}' /etc/passwd
alice,bob
charlie
```

### Example Entries

```
root:x:0:
wheel:x:10:alice,bob
developers:x:1002:alice,charlie
docker:x:1003:alice
nobody:x:65534:
nginx:x:991:
```

---

## /etc/gshadow — Group Shadow Database

### Purpose

Stores group passwords and group administrator information. Rarely used in practice but exists for completeness.

### Format

```
groupname:password:admins:members
```

| # | Field | Description |
|---|-------|-------------|
| 1 | Group name | Name of the group |
| 2 | Password | Hashed group password, `!` (no password), or empty |
| 3 | Admins | Comma-separated list of group administrators |
| 4 | Members | Comma-separated list of group members |

### Group Administrators

A group administrator (set via `gpasswd -A`) can add or remove members from the group without needing root access:

```bash
# Make alice an administrator of the developers group
$ sudo gpasswd -A alice developers

# Now alice can add members without sudo
$ gpasswd -a bob developers
Adding user bob to group developers
```

---

## /etc/login.defs — Login Configuration Defaults

### Purpose

System-wide defaults for user account creation, password policies, and UID/GID ranges. Read by `useradd`, `userdel`, `groupadd`, `login`, and other commands.

### Important Parameters for RHEL 9

```bash
# UID/GID ranges
UID_MIN                  1000    # Minimum UID for regular users
UID_MAX                 60000    # Maximum UID for regular users
SYS_UID_MIN               201    # Minimum UID for system users
SYS_UID_MAX               999    # Maximum UID for system users
GID_MIN                  1000    # Minimum GID for regular groups
GID_MAX                 60000    # Maximum GID for regular groups
SYS_GID_MIN               201    # Minimum GID for system groups
SYS_GID_MAX               999    # Maximum GID for system groups

# Password aging defaults (applied when creating new users)
PASS_MAX_DAYS           99999    # Maximum days before password must change
PASS_MIN_DAYS               0    # Minimum days between password changes
PASS_WARN_AGE               7    # Days before expiry to warn user

# Home directory
CREATE_HOME             yes      # Create home directory by default
HOME_MODE               0700     # Home directory permissions (RHEL 9 default)
UMASK                   022      # Default umask for new accounts

# Password hashing
ENCRYPT_METHOD          YESCRYPT # Password hashing algorithm

# User private groups
USERGROUPS_ENAB         yes      # Create a private group for each user
```

### Key Differences Between Distributions

| Parameter | RHEL 9 | Debian/Ubuntu |
|-----------|--------|---------------|
| `SYS_UID_MIN` | 201 | 100 |
| `HOME_MODE` | 0700 | 0755 |
| `ENCRYPT_METHOD` | YESCRYPT | YESCRYPT (recent) or SHA512 |
| `CREATE_HOME` | yes | no (must use -m or adduser) |

---

## /etc/default/useradd — useradd Defaults

### Purpose

Stores default values used by the `useradd` command when options are not explicitly specified.

### Contents

```bash
$ cat /etc/default/useradd
GROUP=100
HOME=/home
INACTIVE=-1
EXPIRE=
SHELL=/bin/bash
SKEL=/etc/skel
CREATE_MAIL_SPOOL=yes
```

| Field | Description | Default |
|-------|-------------|---------|
| `GROUP` | Default GID (used when `USERGROUPS_ENAB=no`) | 100 |
| `HOME` | Base directory for home directories | `/home` |
| `INACTIVE` | Days after password expiry before account lock (-1 = disabled) | -1 |
| `EXPIRE` | Default account expiration date (empty = never) | (none) |
| `SHELL` | Default login shell | `/bin/bash` |
| `SKEL` | Skeleton directory for new home directories | `/etc/skel` |
| `CREATE_MAIL_SPOOL` | Create a mail spool file | yes |

View with:

```bash
$ useradd -D
```

---

## /etc/skel/ — Skeleton Directory

### Purpose

Contains template files that are copied to a new user's home directory when the account is created (with home directory creation).

### Default Contents on RHEL 9

```bash
$ ls -la /etc/skel/
total 24
drwxr-xr-x.  2 root root   62 Jul 21  2023 .
drwxr-xr-x. 76 root root 8192 Sep 22 10:00 ..
-rw-r--r--.  1 root root   18 Jul 21  2023 .bash_logout
-rw-r--r--.  1 root root  141 Jul 21  2023 .bash_profile
-rw-r--r--.  1 root root  492 Jul 21  2023 .bashrc
```

### Customization

You can add files to `/etc/skel/` that should be present in every new user's home directory:

```bash
# Add a welcome message for all new users
$ sudo cat > /etc/skel/README << 'EOF'
Welcome to the server!
Please review the company policies at https://wiki.example.com/policies
EOF

# Add a custom .vimrc
$ sudo cp /path/to/standard.vimrc /etc/skel/.vimrc

# Now all new users will get these files in their home directory
$ sudo useradd -m newuser
$ ls -la /home/newuser/
# .bash_logout, .bash_profile, .bashrc, README, .vimrc
```

**Note:** Changes to `/etc/skel/` only affect **new** users. Existing users are not modified.

---

## /etc/shells — Valid Login Shells

### Purpose

Lists all valid login shells on the system. The `chsh` command validates against this file. Some services (like FTP servers) refuse access if the user's shell is not in this list.

### Contents

```bash
$ cat /etc/shells
/bin/sh
/bin/bash
/usr/bin/sh
/usr/bin/bash
```

If you install a new shell (like `zsh`), you must add it to this file:

```bash
$ sudo dnf install zsh
$ which zsh
/usr/bin/zsh
# zsh is automatically added to /etc/shells by the package on RHEL
# On some systems you may need to add it manually
```

**Note:** `/sbin/nologin` and `/bin/false` are intentionally NOT in `/etc/shells` — they are used to prevent login, so they should not be considered valid login shells.

---

## /etc/nsswitch.conf — Name Service Switch

### Purpose

Controls where the system looks for various types of information (users, groups, hosts, services, etc.). This is the master switch that determines whether user lookups use local files, LDAP, SSSD, or other sources.

### Relevant Lines for User Management

```bash
$ grep -E '^(passwd|shadow|group)' /etc/nsswitch.conf
passwd:     files sss
shadow:     files sss
group:      files sss
```

### How It Works

For `passwd: files sss`:

1. **files** — First, check `/etc/passwd`
2. **sss** — If not found, query SSSD (which may check LDAP, Active Directory, or FreeIPA)

### Common Configurations

| Configuration | Meaning |
|--------------|---------|
| `passwd: files` | Local files only (standalone server) |
| `passwd: files sss` | Local files, then SSSD (centralized identity) |
| `passwd: files ldap` | Local files, then LDAP directly |
| `passwd: compat` | Compatibility mode (supports NIS +/- syntax) |

### Impact

If NSS is misconfigured, user lookups fail:

```bash
# If SSSD is configured but the service is down:
$ getent passwd ldapuser
# (no output — user not found)

$ id ldapuser
id: 'ldapuser': no such user

# The user cannot log in even though their account exists in LDAP
```

---

## /etc/pam.d/ — PAM Configuration

### Purpose

Contains per-service PAM (Pluggable Authentication Modules) configuration files. Each file defines the authentication stack for a specific service.

### Key Files

| File | Purpose |
|------|---------|
| `system-auth` | System-wide authentication defaults (RHEL) |
| `password-auth` | Password-based authentication (RHEL) |
| `sshd` | SSH authentication |
| `login` | Console login |
| `sudo` | sudo authentication |
| `su` | su authentication |
| `postlogin` | Post-login processing |

### Structure (Example: `/etc/pam.d/system-auth`)

```
auth        required      pam_env.so
auth        required      pam_faillock.so preauth silent audit
auth        sufficient    pam_unix.so try_first_pass
auth        required      pam_faillock.so authfail audit
auth        required      pam_deny.so

account     required      pam_unix.so
account     required      pam_faillock.so

password    requisite     pam_pwquality.so try_first_pass local_users_only retry=3
password    sufficient    pam_unix.so try_first_pass use_authtok yescrypt shadow
password    required      pam_deny.so

session     optional      pam_keyinit.so revoke
session     required      pam_limits.so
session     required      pam_unix.so
```

**Warning:** Incorrectly editing PAM files can lock all users (including root) out of the system. Always keep a root session open when modifying PAM configuration. Use `visudo` for sudoers; for PAM, copy the file first: `cp /etc/pam.d/system-auth /etc/pam.d/system-auth.bak`.

---

## /etc/security/ — Security Configuration

### /etc/security/limits.conf

Controls resource limits for user sessions:

```bash
# Format: domain  type  item  value
# Examples:
alice     hard  nproc     4096    # Max processes for alice
@developers  soft  nofile    8192    # Soft file descriptor limit for developers group
*         hard  core      0       # Disable core dumps for everyone
```

### /etc/security/pwquality.conf

Password quality requirements (used by `pam_pwquality`):

```bash
minlen = 8        # Minimum password length
dcredit = -1      # Require at least 1 digit
ucredit = -1      # Require at least 1 uppercase letter
lcredit = -1      # Require at least 1 lowercase letter
ocredit = -1      # Require at least 1 special character
```

### /etc/security/access.conf

Login access control (used by `pam_access`):

```bash
# Format: permission : users : origins
+ : root : LOCAL              # Allow root from local console
+ : @wheel : ALL              # Allow wheel group from anywhere
- : ALL : ALL                 # Deny everyone else
```

---

## /etc/subuid and /etc/subgid — Subordinate IDs

### Purpose

Define ranges of subordinate UIDs and GIDs allocated to users for use in user namespaces (rootless containers with Podman or Docker).

### Format

```
username:start_id:count
```

### Example

```bash
$ cat /etc/subuid
alice:100000:65536
bob:165536:65536

$ cat /etc/subgid
alice:100000:65536
bob:165536:65536
```

This means user `alice` can map container UIDs 0–65535 to host UIDs 100000–165535, providing isolation between containers and the host system.

These files are managed by `useradd` (which allocates ranges automatically) and `usermod --add-subuids/--add-subgids`.

---

## Safe Editing: vipw and vigr

### Why Not Just Use vim or nano?

Directly editing `/etc/passwd`, `/etc/shadow`, `/etc/group`, or `/etc/gshadow` with a text editor is risky:

1. **No file locking** — Another process (like `useradd`) could modify the file simultaneously, causing corruption.
2. **No syntax validation** — A typo could lock users out or break authentication.
3. **No shadow consistency** — Editing `/etc/passwd` might desync it from `/etc/shadow`.

### vipw and vigr

| Command | Edits | Features |
|---------|-------|----------|
| `vipw` | `/etc/passwd` | File locking, warns about shadow consistency |
| `vipw -s` | `/etc/shadow` | File locking |
| `vigr` | `/etc/group` | File locking, warns about gshadow consistency |
| `vigr -s` | `/etc/gshadow` | File locking |

```bash
# Safely edit /etc/passwd
$ sudo vipw
# Opens in vi with file locking

# After saving, it prompts:
# vipw: /etc/passwd was modified.
# You may need to modify /etc/shadow for consistency.
# Please use the command 'vipw -s' to do so.
```

### pwck and grpck — Integrity Checking

```bash
# Check passwd and shadow consistency
$ sudo pwck
user 'alice': directory '/home/alice' does not exist

# Check group and gshadow consistency
$ sudo grpck
```

These commands find:
- Users in `/etc/passwd` without matching `/etc/shadow` entries
- Invalid UIDs or GIDs
- Missing home directories
- Duplicate usernames or UIDs
- Orphaned group memberships

---

## getent — The Universal Lookup Tool

`getent` is the correct way to look up user and group information because it queries NSS (all configured sources), not just local files.

```bash
# Look up a specific user
$ getent passwd alice
alice:x:1001:1001:Alice Smith:/home/alice:/bin/bash

# Look up a specific group
$ getent group wheel
wheel:x:10:alice,bob

# Look up shadow information (requires root)
$ sudo getent shadow alice
alice:$y$j9T$...:19988:1:90:14:30:20088:

# List all users (from all sources)
$ getent passwd | wc -l
42

# List all groups
$ getent group | wc -l
65

# Look up by UID
$ getent passwd 1001
alice:x:1001:1001:Alice Smith:/home/alice:/bin/bash
```

**Difference from `cat /etc/passwd`:**
- `cat /etc/passwd` — Only shows local accounts from the file
- `getent passwd` — Shows accounts from ALL configured sources (local files, LDAP, SSSD, etc.)

In environments with centralized identity management, always use `getent` instead of reading files directly.

---

## Practical Examples

### Reading and Interpreting /etc/passwd

```bash
# Find all users with UID >= 1000 (regular users)
$ awk -F: '$3 >= 1000 && $3 < 65534 {print $1, $3, $7}' /etc/passwd
alice 1001 /bin/bash
bob 1002 /bin/bash

# Find all users with a real login shell
$ grep -v '/sbin/nologin\|/bin/false' /etc/passwd
root:x:0:0:root:/root:/bin/bash
alice:x:1001:1001:Alice Smith:/home/alice:/bin/bash
bob:x:1002:1002:Bob Jones:/home/bob:/bin/bash

# Find users with UID 0 (should only be root)
$ awk -F: '$3 == 0 {print $1}' /etc/passwd
root
```

### Reading and Interpreting /etc/shadow

```bash
# Check password status for all users (requires root)
$ sudo awk -F: '{
    if ($2 == "!!") status="No password set"
    else if ($2 == "*") status="Disabled"
    else if ($2 ~ /^!/) status="Locked"
    else if ($2 == "") status="EMPTY PASSWORD - DANGER"
    else status="Password set"
    print $1, status
}' /etc/shadow | column -t

root      Password set
alice     Password set
bob       Password set
nginx     Disabled
locked    Locked
newuser   No password set
```

### Converting Epoch Days to Dates

The date fields in `/etc/shadow` are in days since January 1, 1970:

```bash
# Convert days-since-epoch to a readable date
$ date -d "1970-01-01 + 19988 days"
Sun Sep 22 00:00:00 UTC 2024

# Check when alice's password was last changed
$ sudo awk -F: '$1 == "alice" {print $3}' /etc/shadow
19988
$ date -d "1970-01-01 + 19988 days" +%Y-%m-%d
2024-09-22

# Easier method: just use chage
$ sudo chage -l alice
Last password change                    : Sep 22, 2024
```

---

## Troubleshooting

### Problem: User Not Found Despite Being in /etc/passwd

**Symptoms:** `id username` returns "no such user" even though the user exists in `/etc/passwd`.

**Causes:**
- NSS cache is stale: `sudo systemctl restart sssd` or `sudo sss_cache -E`
- `/etc/nsswitch.conf` is not configured to check `files`
- A conflicting entry in LDAP/SSSD overrides the local entry

**Diagnostic:**

```bash
$ getent passwd username    # Check NSS resolution
$ grep username /etc/passwd # Check local file
$ grep '^passwd:' /etc/nsswitch.conf  # Check NSS order
```

### Problem: Authentication Fails for a User Who Exists

**Symptoms:** User exists in `getent passwd` but cannot log in.

**Diagnostic:**

```bash
$ sudo passwd -S username        # Check password status (LK, PS, NP)
$ sudo chage -l username         # Check account/password expiry
$ getent passwd username          # Check shell (is it /sbin/nologin?)
$ sudo faillock --user username   # Check failed login lockout
$ sudo cat /var/log/secure | tail # Check authentication logs (RHEL)
```

### Problem: /etc/passwd and /etc/shadow Are Out of Sync

**Symptoms:** Users exist in one file but not the other.

**Fix:**

```bash
$ sudo pwck    # Reports inconsistencies
$ sudo pwck -s # Fix inconsistencies (sort and remove orphans)
```

---

## Interview Questions

### Q1: What are the 7 fields in /etc/passwd?

**Answer:** Username, password placeholder (x), UID, primary GID, GECOS/comment, home directory, login shell. The fields are colon-separated.

### Q2: Why is /etc/passwd world-readable but /etc/shadow is not?

**Answer:** `/etc/passwd` must be readable by all users because many programs (ls, ps, id) need to convert UIDs to usernames. It does not contain sensitive data — the password field just shows `x`. `/etc/shadow` contains password hashes and aging information, which could be used for offline brute-force attacks. It is only readable by root (mode 0000 or 0640).

### Q3: What does `x` in the password field of `/etc/passwd` mean?

**Answer:** It means the actual password hash is stored in `/etc/shadow`, not in `/etc/passwd`. This is the standard configuration on all modern Linux systems. Historically, the hash was stored directly in `/etc/passwd`, but this was insecure because `/etc/passwd` is world-readable.

### Q4: What is the difference between `!`, `*`, `!!`, and an empty password field in /etc/shadow?

**Answer:**
- `!!` — No password has ever been set. The account was created but `passwd` was never run.
- `!` — The password is locked. A `!` is prepended to the existing hash. Unlocking removes the `!`.
- `!$y$...` — A locked password with the hash preserved. The hash can be restored by unlocking.
- `*` — The account is disabled for password authentication. It was never intended to have a password (common for system accounts).
- Empty — No password is required for login. This is a critical security vulnerability.

### Q5: What password hashing algorithm does RHEL 9 use by default?

**Answer:** RHEL 9 uses **yescrypt** (identified by `$y$` in the hash). Previous versions used SHA-512 (`$6$`). Yescrypt is a memory-hard password hashing function designed to be resistant to GPU-based cracking attacks, making it significantly stronger than SHA-512 for this purpose.

### Q6: How do you safely edit /etc/passwd?

**Answer:** Use `vipw`, which provides file locking (prevents concurrent modifications by tools like `useradd`) and prompts you to edit `/etc/shadow` for consistency. Never edit `/etc/passwd` directly with `vi` or `nano`. After editing, run `pwck` to verify integrity.

### Q7: What is /etc/nsswitch.conf used for?

**Answer:** It controls the Name Service Switch — the system that determines where to look up user, group, host, and service information. For example, `passwd: files sss` means first check `/etc/passwd` (local files), then query SSSD (which may connect to LDAP or Active Directory). This is how Linux supports both local and centralized identity management.

### Q8: What does /etc/skel contain and when is it used?

**Answer:** `/etc/skel` is the skeleton directory containing template files (`.bash_profile`, `.bashrc`, `.bash_logout`) that are copied into a new user's home directory when the account is created with a home directory. Administrators can add custom files to `/etc/skel` to provide a standard environment for all new users. Changes only affect new users, not existing ones.

### Q9: How do you find all users on a system, including those from LDAP?

**Answer:** Use `getent passwd`, which queries all configured NSS sources (local files, SSSD, LDAP, etc.). Simply reading `/etc/passwd` with `cat` only shows local accounts. The order of sources is defined in `/etc/nsswitch.conf`.

### Q10: What are /etc/subuid and /etc/subgid used for?

**Answer:** They define subordinate UID and GID ranges allocated to users for use in user namespaces, primarily for rootless containers (Podman, Docker). For example, `alice:100000:65536` means alice can map container UIDs 0–65535 to host UIDs 100000–165535, providing isolation between containers and the host system. These ranges are automatically allocated by `useradd` on modern distributions.

---

**Next Module:** [Module 5: Linux File Permissions Fundamentals](05-linux-file-permissions-fundamentals.md) — Learn how Linux controls access to files and directories.

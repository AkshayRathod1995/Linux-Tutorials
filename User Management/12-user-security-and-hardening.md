# 12 - User Security and Hardening

## Table of Contents

- [Introduction](#introduction)
- [Core Security Principles](#core-security-principles)
- [Root Account Security](#root-account-security)
- [Sudo Policy Design](#sudo-policy-design)
- [Service Account Hardening](#service-account-hardening)
- [Password Policy Implementation](#password-policy-implementation)
- [Security Audit Commands](#security-audit-commands)
- [Common Security Mistakes](#common-security-mistakes)
- [User Offboarding Process](#user-offboarding-process)
- [Security Hardening Checklist](#security-hardening-checklist)
- [Interview Questions](#interview-questions)

---

## Introduction

User security and hardening is about reducing the attack surface of a Linux system by applying the principles of least privilege, separation of duties, and defense in depth. A single misconfigured user account, an overly permissive sudo rule, or a forgotten service account can lead to a complete system compromise.

This guide covers practical security hardening techniques for **AlmaLinux 9 / RHEL 9**, with notes on Ubuntu/Debian where applicable. It is written for system administrators who need to secure production systems.

---

## Core Security Principles

### Principle of Least Privilege

**Definition:** Every user, process, and program should have ONLY the minimum privileges necessary to accomplish their task -- nothing more.

**In practice:**
- Users get access to only the commands and files they need
- Service accounts run with minimal permissions
- sudo rules allow specific commands, not ALL
- File permissions are restrictive by default
- Users are added to groups only when necessary

**Example:**
```bash
# BAD: Developer gets full root access
developer ALL=(ALL:ALL) ALL

# GOOD: Developer gets only what they need
developer ALL=(root) NOPASSWD: /usr/bin/systemctl restart myapp.service, \
                                /usr/bin/journalctl -u myapp
```

### Separation of Duties

**Definition:** No single person should control all aspects of a critical process. Different people handle different parts.

**In practice:**
- The person who deploys code should not be the same person who approves the deployment
- The DBA should not have access to application servers
- The network administrator should not have access to application code
- System auditors should have read-only access, not write access

**Example organizational model:**
```
System Administrators -- Full sudo, system configuration
Application Developers -- Application deployment only
Database Administrators -- Database tools only, on DB servers
Security Auditors       -- Read-only access to logs and configs
Network Engineers       -- Network tool access only
```

### Defense in Depth

**Definition:** Multiple layers of security, so that if one layer fails, other layers still provide protection.

**In practice:**
- Even if a user gains sudo access, mandatory access control (SELinux) limits what they can do
- Even if a service is compromised, it runs as a non-root user with limited file access
- Even if a password is cracked, SSH key requirements and MFA provide additional barriers
- Even if an attacker is on the network, firewall rules limit what they can reach

**Layers for user security:**
```
Layer 1: Strong authentication (password policy, MFA)
Layer 2: Authorization controls (sudo, file permissions)
Layer 3: Mandatory access control (SELinux/AppArmor)
Layer 4: Network access control (firewall, SSH restrictions)
Layer 5: Monitoring and auditing (logs, alerting)
Layer 6: Incident response (account lockout, forensics)
```

---

## Root Account Security

The root account is the most powerful account on any Linux system. Securing it is the foundation of system security.

### Disable Direct Root SSH Login

By far the most important hardening step for root:

```bash
# Edit sshd_config
sudo vi /etc/ssh/sshd_config

# Change or add:
PermitRootLogin no

# Restart sshd
sudo systemctl restart sshd
```

**Why this matters:**
- root is a known username -- attackers always try root first in brute-force attacks
- Direct root SSH provides no audit trail of who actually logged in
- Requiring sudo means every privileged action is logged with the real user's identity
- If an attacker compromises one user's SSH key, they still need sudo access

**After disabling root SSH:**
- Ensure at least one user has sudo access before disabling root SSH
- Test that sudo works before closing your current session
- Keep root password known for emergency console access

### Restrict Root to Console Only

On physical/virtual servers where you want root available for emergency console access but not for any network login:

```bash
# In /etc/ssh/sshd_config
PermitRootLogin no

# In /etc/securetty (if used -- note: often empty or absent on RHEL 9)
# List only physical console terminals:
tty1
tty2
tty3
tty4
tty5
tty6
```

### Use sudo Instead of su

```bash
# BAD: Using su requires sharing root password
su -
# Who logged in as root? The logs just say "root."

# GOOD: Using sudo with individual accounts
sudo -i
# Logs show: "akshay : TTY=pts/0 ; PWD=/home/akshay ; USER=root ; COMMAND=/bin/bash"
```

### Monitor Root Access

```bash
# Check who has used sudo recently
sudo journalctl -t sudo --since "24 hours ago"

# Check who has sudo access
for u in $(awk -F: '$3 >= 1000 {print $1}' /etc/passwd); do
    result=$(sudo -l -U "$u" 2>/dev/null)
    if echo "$result" | grep -q "may run"; then
        echo "=== $u ==="
        echo "$result" | grep -A100 "may run"
        echo ""
    fi
done

# Check who is in the wheel group
getent group wheel

# Monitor sudo in real-time
sudo tail -f /var/log/secure | grep sudo
```

### Root Password Management

- Store the root password in a secure password manager or vault
- At least two trusted administrators should know it (bus factor)
- Change it when any administrator leaves the team
- Use it ONLY for emergency console access
- Never transmit it in plaintext (email, chat, etc.)

---

## Sudo Policy Design

### Start Restrictive, Add Permissions as Needed

The correct approach to sudo policy is to start with NO access and add specific permissions as requirements emerge:

```bash
# Step 1: Default -- users have no sudo access
# (No rules in sudoers for regular users)

# Step 2: A developer needs to restart a service
# Add ONLY what they need:
sudo visudo -f /etc/sudoers.d/20-developers
# developer ALL=(root) NOPASSWD: /usr/bin/systemctl restart myapp.service

# Step 3: They also need to view logs
# Add the specific log command:
# developer ALL=(root) NOPASSWD: /usr/bin/journalctl -u myapp
```

### Use Command-Specific Rules

Never use `ALL` for commands unless the user genuinely needs full root access:

```bash
# BAD: Overly broad
developer ALL=(ALL) ALL

# BAD: Looks restricted but is bypassable
developer ALL=(ALL) ALL, !/bin/bash, !/bin/sh

# GOOD: Specific commands only
developer ALL=(root) NOPASSWD: /usr/bin/systemctl restart myapp.service, \
                                /usr/bin/systemctl status myapp.service, \
                                /usr/bin/journalctl -u myapp
```

### Audit sudo Permissions Regularly

Create a script to audit sudo access:

```bash
#!/bin/bash
# audit-sudo.sh -- Audit sudo permissions for all users

echo "=============================================="
echo "SUDO ACCESS AUDIT - $(date)"
echo "Host: $(hostname)"
echo "=============================================="

echo ""
echo "--- Users in wheel group ---"
getent group wheel

echo ""
echo "--- Files in /etc/sudoers.d/ ---"
ls -la /etc/sudoers.d/

echo ""
echo "--- Syntax check ---"
visudo -c

echo ""
echo "--- NOPASSWD rules ---"
grep -r "NOPASSWD" /etc/sudoers /etc/sudoers.d/ 2>/dev/null

echo ""
echo "--- Per-user sudo access ---"
for u in $(awk -F: '$3 >= 1000 {print $1}' /etc/passwd); do
    result=$(sudo -l -U "$u" 2>/dev/null)
    if echo "$result" | grep -q "may run"; then
        echo ""
        echo "=== $u ==="
        echo "$result" | tail -n +2
    fi
done

echo ""
echo "--- Users with UID 0 (root equivalents) ---"
awk -F: '$3 == 0 {print $1}' /etc/passwd

echo ""
echo "=============================================="
echo "Audit complete."
```

### Test with sudo -l -U

Always verify effective permissions after making changes:

```bash
# Check what a specific user can do
sudo -l -U deploy

# Verify a specific command is allowed
sudo -l -U deploy /usr/bin/systemctl restart myapp.service
# Returns 0 if allowed, non-zero if denied

# Check in verbose mode
sudo -ll -U deploy
```

### Use Aliases for Manageability

```bash
# Instead of listing users in every rule:
alice ALL=(root) /usr/bin/systemctl restart myapp
bob ALL=(root) /usr/bin/systemctl restart myapp
charlie ALL=(root) /usr/bin/systemctl restart myapp

# Use aliases:
User_Alias DEPLOYERS = alice, bob, charlie
Cmnd_Alias DEPLOY_CMDS = /usr/bin/systemctl restart myapp.service, \
                          /usr/bin/systemctl status myapp.service

DEPLOYERS ALL=(root) NOPASSWD: DEPLOY_CMDS

# Adding a new deployer = add one name to User_Alias
# Adding a new command = add one line to Cmnd_Alias
```

---

## Service Account Hardening

Service accounts are user accounts that run system services and applications. They should never be used for interactive login.

### Create Properly Hardened Service Accounts

```bash
# Create a service account with all hardening applied
sudo useradd \
    --system \
    --shell /sbin/nologin \
    --home-dir /opt/myapp \
    --no-create-home \
    --comment "MyApp Service Account" \
    myapp

# Explanation of flags:
# --system         : Use a UID from the system range (< 1000 on RHEL)
# --shell /sbin/nologin : Prevent interactive login
# --home-dir       : Set home directory (usually the app directory)
# --no-create-home : Don't create the home directory (app installer should do this)
# --comment        : Description of the account's purpose
```

### Service Account Security Checklist

| Setting | Correct Value | Why |
|---------|---------------|-----|
| Shell | `/sbin/nologin` | Prevents interactive login |
| Password | Locked (`!!` or `!`) | No password authentication |
| Home directory | Application directory or `/nonexistent` | Minimal footprint |
| UID | System range (< 1000) | Distinguishes from human users |
| Group | Dedicated group | Principle of least privilege |
| SSH keys | None | Should not have remote access |
| sudo | None (or very limited) | Should not escalate privileges |
| File ownership | Only files the service needs | Minimal file access |

### Verify Service Account Configuration

```bash
# Check all service account properties
getent passwd myapp
# myapp:x:997:995:MyApp Service Account:/opt/myapp:/sbin/nologin

# Verify password is locked
sudo passwd -S myapp
# myapp LK (Password locked.)

# Verify no SSH keys
ls -la /opt/myapp/.ssh/ 2>/dev/null || echo "No .ssh directory (good)"

# Verify no crontabs
crontab -l -u myapp 2>/dev/null || echo "No crontab (expected)"

# Verify no sudo access
sudo -l -U myapp 2>/dev/null
```

### Examples of Common Service Accounts

```bash
# Default service accounts on RHEL 9
getent passwd | grep nologin | head -20

# Typical entries:
# bin:x:1:1:bin:/bin:/sbin/nologin
# daemon:x:2:2:daemon:/sbin:/sbin/nologin
# nobody:x:65534:65534:Kernel Overflow User:/:/sbin/nologin
# nginx:x:996:993:Nginx web server:/var/lib/nginx:/sbin/nologin
# postgres:x:26:26:PostgreSQL Server:/var/lib/pgsql:/bin/bash  <-- Note: postgres has bash
```

> **Note on postgres:** The postgres user typically has `/bin/bash` as its shell because DBAs need to run interactive psql sessions as postgres via `sudo -u postgres psql`. This is an exception -- it should be tightly controlled via sudo rules.

### Do Not Use Service Accounts for Interactive Work

```bash
# WRONG: Logging in as the service account to debug
sudo su - nginx
# This defeats the purpose of the service account

# RIGHT: Use sudo -u for specific commands
sudo -u nginx nginx -t
sudo -u nginx cat /var/log/nginx/error.log

# RIGHT: Check the service's journal
sudo journalctl -u nginx

# RIGHT: If you need an interactive shell for debugging
sudo -u nginx /bin/bash    # Only if sudoers allows it, and only temporarily
```

---

## Password Policy Implementation

A comprehensive password policy involves multiple components working together.

### Password Complexity with pam_pwquality

Configure `/etc/security/pwquality.conf`:

```bash
# Minimum password length
minlen = 14

# Minimum number of character classes (uppercase, lowercase, digit, special)
minclass = 3

# Require at least 1 digit
dcredit = -1

# Require at least 1 uppercase letter
ucredit = -1

# Require at least 1 lowercase letter
lcredit = -1

# Require at least 1 special character
ocredit = -1

# Maximum consecutive identical characters
maxrepeat = 3

# Maximum sequential characters (abc, 123, cba)
maxsequence = 3

# Check against dictionary
dictcheck = 1

# Check if password contains username
usercheck = 1

# Minimum characters that must differ from old password
difok = 8

# Number of retries
retry = 3
```

### Password Aging with chage

Set organization-wide password aging policy:

```bash
# Configure defaults for new users in /etc/login.defs
sudo vi /etc/login.defs

# Set these values:
PASS_MAX_DAYS   90      # Password expires after 90 days
PASS_MIN_DAYS   7       # Minimum 7 days between changes
PASS_MIN_LEN    14      # Minimum length (also enforced by pam_pwquality)
PASS_WARN_AGE   14      # Warn 14 days before expiry

# Apply to existing users:
for user in $(awk -F: '$3 >= 1000 && $7 !~ /nologin|false/ {print $1}' /etc/passwd); do
    sudo chage -M 90 -m 7 -W 14 -I 30 "$user"
    echo "Updated: $user"
done
```

### Password History with pam_pwhistory

Prevent password reuse:

```bash
# On RHEL 9, enable via authselect
sudo authselect enable-feature with-pwhistory

# Configure the number of remembered passwords
# Edit /etc/security/pwhistory.conf (RHEL 9)
# Or add to PAM config: remember=12
```

The `remember=12` parameter stores the last 12 password hashes in `/etc/security/opasswd`, preventing the user from reusing any of them.

### Account Lockout with pam_faillock

```bash
# On RHEL 9, enable via authselect
sudo authselect enable-feature with-faillock

# Configure /etc/security/faillock.conf
deny = 5              # Lock after 5 failures
unlock_time = 900     # Unlock after 15 minutes
fail_interval = 900   # Count failures within 15 minutes
even_deny_root = false # Don't lock root (prevents DoS)
```

### Complete Password Policy Summary

| Policy | Setting | Value |
|--------|---------|-------|
| Minimum length | pwquality minlen | 14 characters |
| Complexity | pwquality minclass | 3 character classes |
| Maximum age | chage -M / login.defs | 90 days |
| Minimum age | chage -m / login.defs | 7 days |
| Warning | chage -W / login.defs | 14 days |
| History | pam_pwhistory remember | 12 passwords |
| Lockout threshold | faillock deny | 5 attempts |
| Lockout duration | faillock unlock_time | 900 seconds (15 min) |
| Dictionary check | pwquality dictcheck | Enabled |
| Username check | pwquality usercheck | Enabled |

---

## Security Audit Commands

Regular security audits are essential. Here are the key commands and how to interpret their output.

### Find SUID Files

SUID (Set User ID) files execute with the owner's permissions, regardless of who runs them. Unexpected SUID files can be backdoors.

```bash
# Find all SUID files
sudo find / -perm -4000 -type f -ls 2>/dev/null

# Expected SUID files on a clean RHEL 9 system include:
# /usr/bin/sudo
# /usr/bin/passwd
# /usr/bin/chage
# /usr/bin/gpasswd
# /usr/bin/newgrp
# /usr/bin/su
# /usr/bin/mount
# /usr/bin/umount
# /usr/bin/crontab
# /usr/sbin/unix_chkpwd
# /usr/sbin/pam_timestamp_check

# Compare against a known baseline
sudo find / -perm -4000 -type f 2>/dev/null | sort > /tmp/suid_current.txt
diff /opt/baseline/suid_baseline.txt /tmp/suid_current.txt
```

**What to look for:**
- Files in /tmp, /home, or other user-writable directories
- Files that were not in your baseline
- Recently modified SUID files

### Find SGID Files

SGID (Set Group ID) files execute with the group's permissions.

```bash
# Find all SGID files
sudo find / -perm -2000 -type f -ls 2>/dev/null

# Expected SGID files include:
# /usr/bin/write
# /usr/bin/wall
# /usr/sbin/postdrop
# /usr/sbin/postqueue
```

### Find World-Writable Files

World-writable files can be modified by any user -- potential security risks outside of /tmp.

```bash
# Find world-writable files (excluding /proc, /sys, /dev, /tmp)
sudo find / -perm -o+w -type f \
    -not -path "/proc/*" \
    -not -path "/sys/*" \
    -not -path "/dev/*" \
    -not -path "/tmp/*" \
    -not -path "/var/tmp/*" \
    -not -path "/run/*" \
    -ls 2>/dev/null
```

**What to look for:**
- Configuration files that should not be world-writable
- Scripts that are run by privileged processes
- Anything outside of /tmp and /var/tmp

### Find World-Writable Directories

World-writable directories without the sticky bit can allow file replacement attacks.

```bash
# Find world-writable directories WITHOUT sticky bit
sudo find / -perm -o+w -type d \
    -not -perm -1000 \
    -not -path "/proc/*" \
    -not -path "/sys/*" \
    -not -path "/dev/*" \
    -ls 2>/dev/null

# The sticky bit (1000) on /tmp ensures only file owners can delete their files
# World-writable directories WITHOUT sticky bit are dangerous
```

### Find Files with No Owner

Files with no owner might indicate deleted user accounts or security issues.

```bash
# Find files with no owner
sudo find / -nouser \
    -not -path "/proc/*" \
    -not -path "/sys/*" \
    -ls 2>/dev/null

# Find files with no group
sudo find / -nogroup \
    -not -path "/proc/*" \
    -not -path "/sys/*" \
    -ls 2>/dev/null
```

**Action:** Investigate these files. They may belong to:
- A deleted user account (orphaned files)
- A package that was improperly removed
- Files from a mounted external filesystem

### Check Home Directory Permissions

```bash
# List all home directories with permissions
ls -la /home/

# Expected: drwx------  (mode 700) for each user
# Or at most: drwxr-x---  (mode 750)

# Find home directories with incorrect permissions
for dir in /home/*/; do
    user=$(basename "$dir")
    perms=$(stat -c "%a" "$dir")
    owner=$(stat -c "%U" "$dir")
    if [ "$perms" != "700" ] && [ "$perms" != "750" ]; then
        echo "WARNING: $dir has permissions $perms (owner: $owner)"
    fi
done
```

### Check Sensitive File Permissions

```bash
# Check critical system file permissions
echo "=== Sensitive File Permissions ==="
for file in /etc/shadow /etc/gshadow /etc/sudoers /etc/ssh/sshd_config; do
    if [ -f "$file" ]; then
        stat -c "%a %U:%G %n" "$file"
    fi
done

# Expected:
# 000 root:root /etc/shadow        (RHEL 9)
# 000 root:root /etc/gshadow       (RHEL 9)
# 440 root:root /etc/sudoers
# 600 root:root /etc/ssh/sshd_config

# On Ubuntu/Debian, shadow is typically 640 root:shadow
```

### List Users with UID 0

Only root should have UID 0. Any other user with UID 0 has full root privileges.

```bash
# List all users with UID 0
awk -F: '$3 == 0 {print $1}' /etc/passwd

# Expected output: root
# If anything else appears, investigate immediately
```

### List Users with Empty Passwords

Users with empty passwords can log in without any authentication.

```bash
# Find users with empty password field in shadow
sudo awk -F: '$2 == "" {print $1}' /etc/shadow

# This should return NOTHING in a properly secured system
# If any users appear, set a password immediately:
# sudo passwd <username>
```

### List Users with Login Shells

Identify all accounts that can get an interactive shell:

```bash
# Users with login shells (excluding nologin and false)
grep -v '/sbin/nologin\|/bin/false' /etc/passwd

# Just usernames
grep -v '/sbin/nologin\|/bin/false' /etc/passwd | cut -d: -f1

# Only regular users (UID >= 1000) with login shells
awk -F: '$3 >= 1000 && $7 !~ /nologin|false/ {print $1, $7}' /etc/passwd
```

### Comprehensive sudo Audit

```bash
# Audit sudo privileges for all regular users
for u in $(awk -F: '$3 >= 1000 {print $1}' /etc/passwd | sort); do
    result=$(sudo -l -U "$u" 2>/dev/null)
    if echo "$result" | grep -q "may run"; then
        echo "=== $u ==="
        sudo -l -U "$u" 2>/dev/null | grep -A 100 "may run"
        echo ""
    fi
done

# Check for NOPASSWD rules
echo "=== NOPASSWD Rules ==="
grep -rn "NOPASSWD" /etc/sudoers /etc/sudoers.d/ 2>/dev/null

# Check for ALL access
echo "=== Users with ALL access ==="
grep -rn "ALL=(ALL" /etc/sudoers /etc/sudoers.d/ 2>/dev/null
```

### Complete Security Audit Script

```bash
#!/bin/bash
# user-security-audit.sh
# Comprehensive user security audit for RHEL 9 / AlmaLinux 9

REPORT_FILE="/tmp/security-audit-$(hostname)-$(date +%Y%m%d).txt"

{
echo "============================================================"
echo "USER SECURITY AUDIT REPORT"
echo "Host: $(hostname)"
echo "Date: $(date)"
echo "OS: $(cat /etc/redhat-release 2>/dev/null || cat /etc/os-release | head -1)"
echo "============================================================"

echo ""
echo "=== 1. USERS WITH UID 0 ==="
awk -F: '$3 == 0 {print $1}' /etc/passwd

echo ""
echo "=== 2. USERS WITH EMPTY PASSWORDS ==="
sudo awk -F: '$2 == "" {print $1}' /etc/shadow || echo "(requires root)"

echo ""
echo "=== 3. USERS WITH LOGIN SHELLS ==="
awk -F: '$3 >= 1000 && $7 !~ /nologin|false/ {printf "%-20s UID=%-6s Shell=%s\n", $1, $3, $7}' /etc/passwd

echo ""
echo "=== 4. LOCKED ACCOUNTS ==="
for u in $(awk -F: '$3 >= 1000 {print $1}' /etc/passwd); do
    status=$(sudo passwd -S "$u" 2>/dev/null | awk '{print $2}')
    if [ "$status" = "LK" ]; then
        echo "$u (LOCKED)"
    fi
done

echo ""
echo "=== 5. EXPIRED ACCOUNTS ==="
for u in $(awk -F: '$3 >= 1000 {print $1}' /etc/passwd); do
    expire=$(sudo chage -l "$u" 2>/dev/null | grep "Account expires" | cut -d: -f2 | xargs)
    if [ "$expire" != "never" ] && [ -n "$expire" ]; then
        echo "$u -- expires: $expire"
    fi
done

echo ""
echo "=== 6. WHEEL GROUP MEMBERS ==="
getent group wheel

echo ""
echo "=== 7. SUDO NOPASSWD RULES ==="
grep -rn "NOPASSWD" /etc/sudoers /etc/sudoers.d/ 2>/dev/null

echo ""
echo "=== 8. HOME DIRECTORY PERMISSIONS ==="
for dir in /home/*/; do
    stat -c "%a %U:%G %n" "$dir" 2>/dev/null
done

echo ""
echo "=== 9. SUID FILES (non-standard) ==="
sudo find / -perm -4000 -type f \
    -not -path "/proc/*" \
    -not -path "/sys/*" \
    2>/dev/null | sort

echo ""
echo "=== 10. WORLD-WRITABLE FILES (excluding tmp) ==="
sudo find / -perm -o+w -type f \
    -not -path "/proc/*" \
    -not -path "/sys/*" \
    -not -path "/dev/*" \
    -not -path "/tmp/*" \
    -not -path "/var/tmp/*" \
    -not -path "/run/*" \
    2>/dev/null | head -50

echo ""
echo "=== 11. FILES WITH NO OWNER ==="
sudo find / -nouser \
    -not -path "/proc/*" \
    -not -path "/sys/*" \
    2>/dev/null | head -20

echo ""
echo "=== 12. SSH CONFIGURATION ==="
sudo grep -E "PermitRootLogin|PasswordAuthentication|AllowUsers|DenyUsers|AllowGroups|DenyGroups" /etc/ssh/sshd_config 2>/dev/null | grep -v "^#"

echo ""
echo "=== 13. FAILLOCK STATUS ==="
faillock 2>/dev/null || echo "faillock not available"

echo ""
echo "=== 14. PASSWORD AGING DEFAULTS ==="
grep -E "^PASS_" /etc/login.defs

echo ""
echo "============================================================"
echo "AUDIT COMPLETE"
echo "============================================================"
} | tee "$REPORT_FILE"

echo ""
echo "Report saved to: $REPORT_FILE"
```

---

## Common Security Mistakes

### Mistake 1: chmod 777 as a Troubleshooting Shortcut

**What people do:**
```bash
# "The application can't access the file, let me fix permissions"
chmod 777 /var/www/html/config.php
chmod -R 777 /opt/myapp/
```

**Why it is dangerous:**
- `777` means EVERYONE can read, write, AND execute the file
- Any user on the system can modify application code
- Any user can read sensitive configuration (database passwords, API keys)
- Web-accessible files with 777 permissions can be modified by web exploits
- It often masks the real problem (wrong owner, wrong group, SELinux context)

**The correct approach:**
```bash
# 1. Identify the actual problem
ls -la /var/www/html/config.php
# Check: who owns it? what group? what are the current permissions?

# 2. Fix ownership if needed
sudo chown apache:apache /var/www/html/config.php

# 3. Set appropriate permissions
sudo chmod 640 /var/www/html/config.php
# Owner (apache) can read+write, group can read, others get nothing

# 4. Check SELinux context if on RHEL
ls -laZ /var/www/html/config.php
sudo restorecon -v /var/www/html/config.php
```

### Mistake 2: Unrestricted sudo for Everyone

**What people do:**
```bash
# "Just add everyone to wheel, it's easier"
sudo usermod -aG wheel developer1
sudo usermod -aG wheel developer2
sudo usermod -aG wheel contractor
sudo usermod -aG wheel intern
```

**Why it is dangerous:**
- Every user has full root access
- A compromised account means the entire system is compromised
- No accountability -- everyone can do everything
- Violates the principle of least privilege
- Interns and contractors should NOT have full root access

**The correct approach:**
```bash
# Give specific, minimal permissions to each role
sudo visudo -f /etc/sudoers.d/20-developers
# DEVELOPERS ALL=(root) NOPASSWD: /usr/bin/systemctl restart myapp.service

sudo visudo -f /etc/sudoers.d/30-contractors
# contractor ALL=(root) NOPASSWD: /usr/bin/journalctl -u myapp

# Only trusted senior admins get full sudo
sudo visudo -f /etc/sudoers.d/10-admins
# SENIOR_ADMINS ALL=(ALL:ALL) ALL
```

### Mistake 3: Sharing Root Credentials

**What people do:**
- Email the root password to the team
- Put root password in a shared document
- Post root password in a chat channel
- Use the same root password on all servers

**Why it is dangerous:**
- No accountability -- anyone could have done anything
- Password in email/chat is stored in multiple places forever
- When someone leaves, you must change every shared password
- The password can be intercepted or leaked
- Violates every security compliance framework

**The correct approach:**
- Use sudo with individual accounts -- root password is rarely needed
- Store root password in a proper secrets manager (HashiCorp Vault, CyberArk, etc.)
- Limit root password knowledge to 2-3 senior administrators
- Use different root passwords on different servers
- Rotate root password when any knowledgeable admin leaves
- Enable MFA for any remaining direct root access

### Mistake 4: Using Service Accounts for Interactive Login

**What people do:**
```bash
# "I need to debug the app, let me log in as the app user"
sudo su - nginx
sudo su - postgres
ssh appuser@server
```

**Why it is dangerous:**
- Service accounts often have access to sensitive data and configuration
- Actions performed as a service account are not attributable to a specific person
- Service accounts may have broader file access than any individual should
- Attackers who compromise a service account credential get service-level access
- It blurs the line between human and automated actions in logs

**The correct approach:**
```bash
# Use sudo -u for specific commands (maintains audit trail)
sudo -u nginx nginx -t
sudo -u postgres psql -c "SELECT count(*) FROM users;"

# View service logs through sudo
sudo journalctl -u myapp

# If interactive debugging is necessary, use a time-limited exception
# and document why it was needed
```

### Mistake 5: Unnecessary Group Memberships

**What people do:**
```bash
# "Just add them to all the groups, it's faster"
sudo usermod -aG wheel,docker,developers,dbteam,netadmins username
```

**Why it is dangerous:**
- User gets access to files and resources they do not need
- If the user account is compromised, the attacker gets all those access levels
- Group-based file permissions become meaningless
- Violates the principle of least privilege

**The correct approach:**
```bash
# Add users to ONLY the groups they need
sudo usermod -aG developers username    # They are a developer

# Regularly audit group memberships
for user in $(awk -F: '$3 >= 1000 {print $1}' /etc/passwd); do
    groups=$(groups "$user" 2>/dev/null)
    echo "$groups"
done

# Remove unnecessary group memberships
sudo gpasswd -d username unnecessarygroup
```

### Mistake 6: Leaving Old Accounts Active

**What people do:**
- Employee leaves, account stays active for months
- Contractor finishes, account is forgotten
- Temporary accounts become permanent

**Why it is dangerous:**
- Dormant accounts are targets for attackers
- The former employee may still have credentials (SSH keys, passwords)
- Compliance requirements mandate timely account deactivation
- Old accounts increase attack surface

**The correct approach:**
- Implement an offboarding process (see next section)
- Set account expiry dates for contractors and temporary accounts
- Review user accounts regularly (monthly or quarterly)
- Automate account lifecycle management where possible

### Mistake 7: Running Applications as Root

**What people do:**
```bash
# "The app needs port 80, so it needs root"
sudo ./myapp
# Or in systemd: User=root
```

**Why it is dangerous:**
- If the application has a vulnerability, the attacker gets root access
- The application can access ANY file on the system
- The application can modify system configuration
- Application bugs can damage the entire system

**The correct approach:**
```bash
# Create a dedicated service account
sudo useradd --system --shell /sbin/nologin --home-dir /opt/myapp myapp

# Use capabilities instead of root for privileged ports
sudo setcap cap_net_bind_service=+ep /opt/myapp/bin/myapp

# Or use a reverse proxy (nginx/Apache) on port 80 that proxies to the app on a high port
# App runs on port 8080 as non-root, nginx forwards port 80 to 8080

# In systemd service file:
# [Service]
# User=myapp
# Group=myapp
# AmbientCapabilities=CAP_NET_BIND_SERVICE
```

### Mistake 8: Incorrect Credential Handling

**What people do:**
- Store passwords in plain text in scripts
- Hardcode API keys in source code
- Put credentials in environment variables visible to all processes
- Use world-readable configuration files with passwords

```bash
# BAD: Password in a script
#!/bin/bash
mysql -u root -pMySecretPassword123 mydb < backup.sql

# BAD: Credentials in a world-readable file
echo "DB_PASSWORD=secret123" > /opt/myapp/.env
chmod 644 /opt/myapp/.env
```

**Why it is dangerous:**
- Plaintext passwords in scripts end up in version control, backups, and logs
- Anyone who can read the script or file gets the credentials
- Process listings (ps aux) can show command-line passwords
- Environment variables are visible in /proc/PID/environ

**The correct approach:**
```bash
# Use a secrets manager (HashiCorp Vault, AWS Secrets Manager, etc.)

# If files must contain secrets, restrict permissions:
chmod 600 /opt/myapp/.env
chown myapp:myapp /opt/myapp/.env

# Use credential files instead of command-line passwords:
# MySQL: ~/.my.cnf with mode 600
# PostgreSQL: ~/.pgpass with mode 600

# Use systemd credential loading:
# [Service]
# EnvironmentFile=/etc/myapp/credentials
# (with the credentials file mode 600, owned by root)

# Never pass passwords as command-line arguments
# BAD:  mysql -u root -pPassword
# GOOD: mysql -u root -p < input_file
# GOOD: mysql --defaults-file=/root/.my.cnf
```

---

## User Offboarding Process

When a user leaves the organization (or changes roles), follow this systematic process to ensure complete access removal.

### Step 1: Lock the Account

```bash
# Lock the password (immediately prevents password login)
sudo usermod -L username

# Verify
sudo passwd -S username
# username LK ...
```

### Step 2: Set Account Expiry

```bash
# Expire the account (prevents ALL login methods, including SSH keys)
sudo usermod -e 1 username
# Note: -e 1 sets expiry to Jan 2, 1970, which is in the past

# Verify
sudo chage -l username
# Account expires: Jan 02, 1970
```

### Step 3: Kill Running Processes

```bash
# List user's running processes
ps -u username

# Kill all processes owned by the user
sudo pkill -u username

# If processes refuse to die
sudo pkill -9 -u username

# Verify no processes remain
ps -u username
```

### Step 4: Backup Home Directory

```bash
# Create a dated backup
sudo tar czf /backup/offboarded/username-$(date +%Y%m%d).tar.gz \
    /home/username/

# Verify the backup
tar tzf /backup/offboarded/username-$(date +%Y%m%d).tar.gz | head
```

### Step 5: Review and Remove Crontabs

```bash
# List user's crontabs
sudo crontab -l -u username

# Save for reference
sudo crontab -l -u username > /backup/offboarded/username-crontab.txt 2>/dev/null

# Remove the crontab
sudo crontab -r -u username
```

### Step 6: Review at Jobs

```bash
# List user's at jobs
sudo atq | grep username

# Remove at jobs
sudo at -d <job_number>

# Or remove all at jobs for the user
for job in $(sudo atq | awk '{print $1}'); do
    if sudo at -c "$job" 2>/dev/null | head -1 | grep -q username; then
        sudo at -d "$job"
    fi
done
```

### Step 7: Remove from Groups

```bash
# List current groups
groups username

# Remove from each supplementary group
sudo gpasswd -d username wheel
sudo gpasswd -d username developers
sudo gpasswd -d username docker
# ... etc for each group

# Verify
groups username
```

### Step 8: Transfer File Ownership

```bash
# Find all files owned by the user outside their home directory
sudo find / -user username \
    -not -path "/home/username/*" \
    -not -path "/proc/*" \
    -not -path "/sys/*" \
    -ls 2>/dev/null

# Transfer ownership of shared files to an appropriate owner
sudo chown -R newowner:newgroup /shared/projects/username-project/

# For system files that the user owned
sudo find /var -user username -exec chown root:root {} \;
```

### Step 9: Revoke SSH Keys

```bash
# Remove authorized_keys
sudo rm -f /home/username/.ssh/authorized_keys

# If the user had SSH keys for other hosts, those should be removed
# from the authorized_keys on those hosts as well

# If the user had SSH access to service accounts
sudo grep -rl "username\|user_email" /home/*/.ssh/authorized_keys \
    /root/.ssh/authorized_keys 2>/dev/null
# Review and remove their key from each file found
```

### Step 10: Remove from sudo/sudoers.d

```bash
# Check if user has specific sudoers rules
sudo grep -r username /etc/sudoers /etc/sudoers.d/ 2>/dev/null

# Remove their sudoers file if they had one
sudo rm /etc/sudoers.d/username 2>/dev/null

# If they were in a User_Alias, remove them from it
sudo visudo -f /etc/sudoers.d/20-developers
# Remove username from User_Alias
```

### Step 11: Archive and Delete Account

```bash
# After a retention period (per your organization's policy), delete the account
# The retention period allows for data recovery if needed

# Delete the account and home directory
sudo userdel -r username

# Verify
getent passwd username    # Should return nothing
ls /home/username         # Should not exist
```

### Step 12: Document the Offboarding

Create a record of all actions taken:

```
Offboarding Record
==================
User: username
Date: 2024-01-15
Reason: Employment terminated
Performed by: admin_name

Actions:
[x] Account locked (usermod -L)
[x] Account expired (usermod -e 1)
[x] Running processes killed (pkill -u)
[x] Home directory backed up to /backup/offboarded/username-20240115.tar.gz
[x] Crontab reviewed and removed
[x] At jobs reviewed and removed
[x] Removed from groups: wheel, developers, docker
[x] File ownership transferred: /shared/projects/ -> newowner
[x] SSH authorized_keys removed
[x] sudoers entry in /etc/sudoers.d/20-developers updated
[x] Account deletion scheduled for 2024-04-15 (90-day retention)

Notes:
- User had active cron job for daily backup; transferred to backup_user
- Found 3 files owned by user in /var/www; transferred to www-data
```

### Offboarding Script

```bash
#!/bin/bash
# offboard-user.sh -- Offboard a user from the system
# Usage: sudo ./offboard-user.sh <username>

USERNAME="$1"
BACKUP_DIR="/backup/offboarded"
DATE=$(date +%Y%m%d)

if [ -z "$USERNAME" ]; then
    echo "Usage: $0 <username>"
    exit 1
fi

if ! id "$USERNAME" &>/dev/null; then
    echo "User $USERNAME does not exist."
    exit 1
fi

echo "=== Offboarding user: $USERNAME ==="
echo "Date: $(date)"
echo ""

# Step 1: Lock account
echo "[1/10] Locking account..."
usermod -L "$USERNAME"
passwd -S "$USERNAME"

# Step 2: Expire account
echo "[2/10] Setting account expiry..."
usermod -e 1 "$USERNAME"
chage -l "$USERNAME" | grep "Account expires"

# Step 3: Kill processes
echo "[3/10] Killing user processes..."
pkill -u "$USERNAME" 2>/dev/null
sleep 2
pkill -9 -u "$USERNAME" 2>/dev/null
echo "Remaining processes: $(ps -u "$USERNAME" --no-headers 2>/dev/null | wc -l)"

# Step 4: Backup home
echo "[4/10] Backing up home directory..."
mkdir -p "$BACKUP_DIR"
tar czf "${BACKUP_DIR}/${USERNAME}-${DATE}.tar.gz" "/home/${USERNAME}/" 2>/dev/null
echo "Backup: ${BACKUP_DIR}/${USERNAME}-${DATE}.tar.gz"

# Step 5: Save and remove crontab
echo "[5/10] Handling crontab..."
crontab -l -u "$USERNAME" > "${BACKUP_DIR}/${USERNAME}-crontab-${DATE}.txt" 2>/dev/null
crontab -r -u "$USERNAME" 2>/dev/null

# Step 6: Remove from groups
echo "[6/10] Removing from supplementary groups..."
GROUPS=$(id -nG "$USERNAME" | tr ' ' '\n' | grep -v "^${USERNAME}$")
for group in $GROUPS; do
    gpasswd -d "$USERNAME" "$group" 2>/dev/null
    echo "  Removed from: $group"
done

# Step 7: Remove SSH keys
echo "[7/10] Removing SSH keys..."
rm -f "/home/${USERNAME}/.ssh/authorized_keys" 2>/dev/null

# Step 8: Remove sudoers
echo "[8/10] Removing sudoers entries..."
rm -f "/etc/sudoers.d/${USERNAME}" 2>/dev/null
echo "  Check and manually update alias-based rules if needed."

# Step 9: Find owned files outside home
echo "[9/10] Finding files owned by user outside home..."
find / -user "$USERNAME" \
    -not -path "/home/${USERNAME}/*" \
    -not -path "/proc/*" \
    -not -path "/sys/*" \
    -not -path "/run/*" \
    2>/dev/null | head -20

# Step 10: Summary
echo "[10/10] Offboarding summary:"
echo "  Account: LOCKED and EXPIRED"
echo "  Processes: KILLED"
echo "  Backup: ${BACKUP_DIR}/${USERNAME}-${DATE}.tar.gz"
echo "  Account deletion should be scheduled per retention policy."
echo ""
echo "=== Offboarding complete ==="
echo "REMINDER: Review files owned by user and transfer ownership as needed."
echo "REMINDER: Check other systems for this user's access."
echo "REMINDER: Schedule account deletion after retention period."
```

---

## Security Hardening Checklist

Use this checklist for hardening a new RHEL 9 / AlmaLinux 9 system:

### Authentication and Passwords

- [ ] Password complexity configured in /etc/security/pwquality.conf (minlen >= 14)
- [ ] Password aging configured in /etc/login.defs (PASS_MAX_DAYS <= 90)
- [ ] Password history enabled (pam_pwhistory, remember >= 12)
- [ ] Account lockout enabled (faillock, deny=5)
- [ ] Default algorithm is yescrypt (RHEL 9 default)

### Root and sudo

- [ ] Direct root SSH login disabled (PermitRootLogin no)
- [ ] Root password stored securely (password manager/vault)
- [ ] sudo used instead of su for privilege escalation
- [ ] sudo rules use specific commands, not ALL (except senior admins)
- [ ] NOPASSWD used only where necessary (automation, specific commands)
- [ ] sudoers files managed with visudo
- [ ] sudo permissions audited regularly

### User Accounts

- [ ] All regular users have unique accounts (no shared accounts)
- [ ] Service accounts use /sbin/nologin shell
- [ ] Service accounts have locked passwords
- [ ] No users have empty passwords
- [ ] Only root has UID 0
- [ ] Home directory permissions are 700 or 750
- [ ] Offboarding process documented and followed

### SSH

- [ ] Root login disabled
- [ ] Password authentication disabled for users with SSH keys
- [ ] SSH key permissions correct (700 for .ssh, 600 for authorized_keys)
- [ ] AllowUsers or AllowGroups configured
- [ ] Idle timeout configured (ClientAliveInterval, ClientAliveCountMax)
- [ ] SSH protocol version 2 only (default on modern systems)

### File Permissions

- [ ] No unnecessary SUID/SGID files
- [ ] No world-writable files outside /tmp
- [ ] No files with no owner or no group
- [ ] Sensitive files (/etc/shadow, /etc/sudoers) have correct permissions
- [ ] Application configuration files are not world-readable

### Monitoring and Auditing

- [ ] sudo logging enabled and monitored
- [ ] Failed login attempts logged and alerted
- [ ] User account changes logged (useradd, usermod, userdel)
- [ ] Audit rules configured for sensitive files (auditd)
- [ ] Regular security audits performed

---

## Interview Questions

### Q1: What is the principle of least privilege and how does it apply to Linux user management?

**Answer:** The principle of least privilege states that every user, process, and program should have only the minimum privileges necessary to accomplish their task. In Linux user management, this means: (1) Users only get sudo access for commands they specifically need, not ALL commands. (2) Users are only added to groups they need for their role. (3) Service accounts use /sbin/nologin and have no sudo access. (4) File permissions are restrictive by default (e.g., home directories are 700, not 755). (5) When using sudo, specific commands are whitelisted rather than giving blanket ALL access. This limits the damage that can occur from a compromised account, an insider threat, or an accidental mistake.

### Q2: Describe the complete process for offboarding a user from a Linux system.

**Answer:** A thorough offboarding process involves: (1) Lock the account immediately with `usermod -L`. (2) Expire the account with `usermod -e 1` to block ALL authentication methods including SSH keys. (3) Kill all running processes with `pkill -u username`. (4) Back up the home directory. (5) Review and remove crontabs with `crontab -l -u` and `crontab -r -u`. (6) Review and remove at jobs. (7) Remove from all supplementary groups. (8) Transfer ownership of shared files. (9) Remove SSH authorized_keys. (10) Remove from sudoers and sudoers.d. (11) After a retention period, delete the account with `userdel -r`. (12) Document everything. Key points: locking the password alone is NOT sufficient -- SSH key login still works unless the account is expired or keys are removed.

### Q3: Why is `chmod 777` dangerous and what should you do instead?

**Answer:** `chmod 777` gives read, write, and execute permissions to the owner, group, AND all other users on the system. This means any user (or any compromised process) can read sensitive data, modify application code, or execute files. It is commonly used as a lazy troubleshooting shortcut when the real issue is wrong ownership, wrong group, or an SELinux context problem. The correct approach is to: (1) Identify the actual problem with `ls -la` and `ls -laZ`. (2) Fix ownership with `chown`. (3) Set minimal permissions (e.g., 640 for config files, 750 for directories). (4) Fix SELinux context with `restorecon` if on RHEL. Typical secure permissions: config files = 640, scripts = 750, web files = 644, directories = 755 or 750.

### Q4: How would you design a sudo policy for an organization with system admins, developers, and contractors?

**Answer:** Use a layered approach with separate sudoers.d files: (1) `/etc/sudoers.d/10-admins`: Full sudo for trusted senior admins (`SENIOR_ADMINS ALL=(ALL:ALL) ALL`). (2) `/etc/sudoers.d/20-developers`: Developers get only application-related commands (`DEVELOPERS ALL=(root) NOPASSWD: /usr/bin/systemctl restart myapp.service, /usr/bin/journalctl -u myapp`). (3) `/etc/sudoers.d/30-contractors`: Contractors get minimal, read-only access (`CONTRACTORS ALL=(root) NOPASSWD: NOEXEC: /usr/bin/journalctl -u myapp`). Use User_Alias and Cmnd_Alias for maintainability. Add NOEXEC for commands with shell escape features. Set account expiry for contractors. Audit regularly with `sudo -l -U`. Review and remove access when roles change.

### Q5: What are SUID files and why should you audit them regularly?

**Answer:** SUID (Set User ID) files execute with the permissions of the file owner, regardless of who runs them. For example, `/usr/bin/passwd` has SUID set so that any user can change their own password (which requires writing to /etc/shadow, a root-owned file). While some SUID files are necessary (sudo, passwd, mount), unexpected SUID files can be backdoors planted by attackers -- a SUID shell in /tmp gives root access to anyone who runs it. Audit with: `find / -perm -4000 -type f -ls`. Compare against a baseline of known-good SUID files. Investigate any new or unexpected entries. Remove SUID from files that do not need it: `chmod u-s /path/to/file`. Regular SUID audits are required by most security compliance frameworks (CIS Benchmarks, STIG).

### Q6: Explain the difference between locking a password and expiring an account.

**Answer:** Locking a password (`passwd -l` or `usermod -L`) prepends `!` to the password hash in /etc/shadow, which disables password authentication. However, SSH key authentication, Kerberos, and other non-password methods STILL WORK. The user can still log in via SSH with a key. Expiring an account (`usermod -e 1` or `chage -E 1`) sets the account expiration date to the past, which causes PAM's account module to reject ALL authentication -- password, SSH keys, everything. The user gets "Your account has expired." For complete access removal, you should do both: lock the password AND expire the account. This is why simply locking a password is insufficient for offboarding.

### Q7: How do you harden a service account on Linux?

**Answer:** A properly hardened service account has: (1) Shell set to `/sbin/nologin` -- prevents interactive login. (2) Password locked (the default `!!` for new system accounts or explicitly with `passwd -l`). (3) UID in the system range (below 1000 on RHEL) -- created with `useradd --system`. (4) A dedicated group for file access. (5) No SSH keys (no .ssh/authorized_keys). (6) No sudo access (unless specific commands are absolutely necessary). (7) Minimal file ownership -- only files the service needs to access. (8) Home directory set to the application directory or /nonexistent. Create with: `useradd --system --shell /sbin/nologin --home-dir /opt/app --no-create-home appuser`.

### Q8: What security risks exist with running applications as root?

**Answer:** Running applications as root means: (1) If the application has any vulnerability (remote code execution, path traversal, injection), the attacker gets ROOT access -- full control of the system. (2) The application can read ANY file, including /etc/shadow, private keys, and other applications' data. (3) Application bugs can damage system configuration and other services. (4) It violates the principle of least privilege. Alternatives: (1) Run as a dedicated non-root user. (2) For privileged ports (below 1024), use Linux capabilities (`setcap cap_net_bind_service`) or a reverse proxy. (3) Use systemd's User= directive. (4) Use containers with user namespaces. Most modern applications are designed to run as non-root.

### Q9: How would you conduct a security audit of user accounts on a production system?

**Answer:** A comprehensive user account audit checks: (1) Users with UID 0: `awk -F: '$3 == 0' /etc/passwd` -- only root should appear. (2) Users with empty passwords: `awk -F: '$2 == ""' /etc/shadow`. (3) Users with login shells who should not have them. (4) Users in the wheel/sudo group -- verify each needs admin access. (5) sudo permissions per user: `sudo -l -U username`. (6) NOPASSWD rules: `grep -r NOPASSWD /etc/sudoers*`. (7) SUID/SGID files: `find / -perm -4000 -type f`. (8) World-writable files. (9) Files with no owner. (10) Home directory permissions. (11) SSH configuration. (12) Password aging compliance. (13) Service account configurations. (14) Inactive accounts that should be disabled. This should be performed regularly (monthly or quarterly) and after any security incident.

### Q10: What is defense in depth and how does it apply to user security?

**Answer:** Defense in depth means using multiple independent security layers so that if one fails, others still provide protection. For user security: (1) Authentication layer -- strong passwords, MFA, SSH keys. (2) Authorization layer -- sudo rules, file permissions, ACLs. (3) Mandatory access control -- SELinux/AppArmor restricts even root-level processes. (4) Network layer -- firewalls, SSH AllowUsers, VPN requirements. (5) Monitoring layer -- audit logs, failed login alerts, anomaly detection. (6) Response layer -- account lockout, incident response procedures. Example: even if an attacker guesses a user's password (layer 1 fails), sudo restrictions limit what they can do (layer 2), SELinux prevents accessing sensitive files (layer 3), the firewall blocks lateral movement (layer 4), and monitoring detects the intrusion (layer 5).

### Q11: How do you prevent password reuse in Linux?

**Answer:** Password reuse is prevented using pam_pwhistory. On RHEL 9, enable it via `authselect enable-feature with-pwhistory`. Configure the `remember` parameter to store N previous password hashes in `/etc/security/opasswd` -- for example, `remember=12` prevents reusing the last 12 passwords. This must be combined with a minimum password age (`chage -m 7`) to prevent users from rapidly changing their password 12 times to cycle back to their preferred password. Without the minimum age restriction, a user could change their password 12 times in a row and return to their original password. Both controls together ensure genuine password rotation.

### Q12: Explain how you would set up account lockout on RHEL 9 and how to manage locked accounts.

**Answer:** On RHEL 9, account lockout is implemented using pam_faillock. Enable it with `authselect enable-feature with-faillock`. Configure `/etc/security/faillock.conf` with: `deny=5` (lock after 5 failures), `unlock_time=900` (auto-unlock after 15 minutes), `fail_interval=900` (count failures within 15 minutes). By default, root is not locked (even_deny_root=false) to prevent denial-of-service attacks. To manage locked accounts: `faillock --user username` shows failed attempts and lock status, `faillock --user username --reset` clears the counter and unlocks the account. If a user reports being locked out, check with `faillock --user`, verify the failure source (legitimate user or attack), reset if appropriate, and investigate the cause of the failures.

---

## Summary

| Topic | Key Points |
|-------|------------|
| Least privilege | Users get ONLY what they need -- specific sudo commands, minimal groups |
| Root security | Disable SSH root login, use sudo, do not share root password |
| Sudo policy | Whitelist commands, use aliases, audit regularly, last matching rule wins |
| Service accounts | /sbin/nologin shell, locked password, system UID, no SSH keys |
| Password policy | minlen 14+, aging (90-day max), history (12), lockout (5 attempts) |
| Security audits | Regular checks for SUID files, world-writable files, orphaned files |
| Common mistakes | chmod 777, unrestricted sudo, shared root password, running apps as root |
| Offboarding | Lock + expire + kill processes + backup + remove access + document |
| Defense in depth | Multiple security layers: authentication, authorization, MAC, network, monitoring |

---

*Previous: [11 - Linux Authentication and Password Management](11-linux-authentication-and-password-management.md)*

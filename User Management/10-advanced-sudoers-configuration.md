# 10 - Advanced sudoers Configuration

## Table of Contents

- [Introduction](#introduction)
- [Sudoers Rule Structure Deep Dive](#sudoers-rule-structure-deep-dive)
- [Aliases](#aliases)
- [Tags](#tags)
- [Command Exclusions and Negation](#command-exclusions-and-negation)
- [Important Defaults](#important-defaults)
- [Rule Precedence](#rule-precedence)
- [Production Scenarios](#production-scenarios)
- [Privilege Escalation Through sudo - Critical Security Section](#privilege-escalation-through-sudo---critical-security-section)
- [Session Recording with sudo](#session-recording-with-sudo)
- [Sudoers Best Practices Summary](#sudoers-best-practices-summary)
- [Interview Questions](#interview-questions)

---

## Introduction

The previous chapter covered sudo fundamentals -- what sudo is, how it works, and basic sudoers syntax. This chapter goes deep into **advanced sudoers configuration**, including aliases, tags, Defaults, production scenarios, and critically important security considerations.

Understanding advanced sudoers configuration is essential for building secure, maintainable privilege management in production environments. This guide uses **AlmaLinux 9 / RHEL 9** as the primary reference, with notes for Ubuntu/Debian where applicable.

---

## Sudoers Rule Structure Deep Dive

Every sudoers rule follows this structure:

```
user_spec    host_spec = (runas_spec)    tag_spec: command_spec
```

Let's examine each component in detail:

### user_spec -- Who Is Allowed

Specifies which user(s) this rule applies to:

| Format | Meaning | Example |
|--------|---------|---------|
| `username` | A specific user | `akshay` |
| `%groupname` | All members of a Unix group | `%wheel` |
| `%:nonunix_group` | A non-Unix group (e.g., LDAP) | `%:ldap_admins` |
| `+netgroup` | An NIS netgroup | `+sysadmins` |
| `User_Alias` | A defined alias (see Aliases section) | `ADMINS` |
| `ALL` | All users | `ALL` |
| `!user` | Negation (all except this user) | `ALL, !guest` |

### host_spec -- Where It Applies

Specifies which host(s) this rule is valid on:

| Format | Meaning | Example |
|--------|---------|---------|
| `hostname` | A specific hostname | `server01` |
| `IP address` | A specific IP | `192.168.1.10` |
| `network/mask` | A network range | `192.168.1.0/24` |
| `Host_Alias` | A defined alias | `WEBSERVERS` |
| `ALL` | All hosts | `ALL` |

On single-server setups, `ALL` is typically used. Host specifications become important when you manage a centralized sudoers file distributed to multiple servers (via configuration management like Ansible, Puppet, or Chef) and want different rules on different hosts.

### runas_spec -- Run As Whom

Specifies which target user(s) and group(s) the command can be run as:

```
(runas_user:runas_group)
```

| Format | Meaning | Example |
|--------|---------|---------|
| `(ALL)` | Any user (no group specified) | `sudo -u anyone command` |
| `(ALL:ALL)` | Any user and any group | `sudo -u anyone -g anygroup command` |
| `(root)` | Only as root | `sudo command` (default) |
| `(postgres)` | Only as postgres user | `sudo -u postgres command` |
| `(root:root)` | As root user with root group | Explicit user and group |
| Omitted entirely | Defaults to root | Same as `(root)` |

If the runas_spec is omitted, commands can only be run as root:
```
akshay ALL= /usr/bin/systemctl restart httpd
# Equivalent to:
akshay ALL=(root) /usr/bin/systemctl restart httpd
```

### tag_spec -- Modifiers

Tags modify how the command is handled (covered in detail in the Tags section):

```
NOPASSWD:     -- do not prompt for password
PASSWD:       -- prompt for password (overrides NOPASSWD)
NOEXEC:       -- prevent command from spawning sub-processes
EXEC:         -- allow command to spawn sub-processes (overrides NOEXEC)
SETENV:       -- allow user to set environment variables
NOSETENV:     -- prevent user from setting environment variables
LOG_INPUT:    -- log stdin
NOLOG_INPUT:  -- do not log stdin
LOG_OUTPUT:   -- log stdout/stderr
NOLOG_OUTPUT: -- do not log stdout/stderr
```

### command_spec -- What Can Be Run

Specifies which commands are allowed:

| Format | Meaning | Example |
|--------|---------|---------|
| `ALL` | Any command | Full access |
| `/full/path/to/command` | A specific command | `/usr/bin/systemctl` |
| `/full/path/to/command args` | A command with specific arguments | `/usr/bin/systemctl restart httpd` |
| `/full/path/to/command *` | A command with any arguments | `/usr/bin/systemctl *` |
| `/full/path/to/command ""` | A command with NO arguments allowed | `/usr/bin/systemctl ""` |
| `Cmnd_Alias` | A defined alias | `SERVICES` |
| `!command` | Negation (not this command) | `ALL, !/bin/bash` |
| `sudoedit /path` | Edit a file with sudoedit | `sudoedit /etc/hosts` |

> **IMPORTANT:** Always use **full paths** for commands. sudo will not search PATH for commands in the sudoers file.

### Complete Rule Examples

```
# User akshay on any host, run any command as any user/group
akshay ALL=(ALL:ALL) ALL

# Members of wheel group on any host, run any command as any user/group
%wheel ALL=(ALL:ALL) ALL

# User deploy on web servers, run service commands as root, no password
deploy WEBSERVERS=(root) NOPASSWD: /usr/bin/systemctl restart httpd

# Multiple commands with different tags
admin ALL=(ALL) PASSWD: ALL, NOPASSWD: /usr/bin/systemctl status *
```

---

## Aliases

Aliases group users, hosts, run-as targets, and commands into named collections. They make complex sudoers configurations much more readable and maintainable.

### User_Alias

Groups users together under a meaningful name:

```
# Basic user alias
User_Alias ADMINS = alice, bob, charlie

# Including groups (with % prefix)
User_Alias SENIOR_STAFF = alice, bob, %managers

# Including other aliases
User_Alias ALL_ADMINS = ADMINS, SENIOR_STAFF

# Excluding users
User_Alias ADMINS = alice, bob, charlie, !dave
```

Usage in a rule:
```
ADMINS ALL=(ALL:ALL) ALL
```

This is much cleaner than:
```
alice ALL=(ALL:ALL) ALL
bob ALL=(ALL:ALL) ALL
charlie ALL=(ALL:ALL) ALL
```

### Runas_Alias

Groups target users (the users you can run commands as):

```
# Web application users
Runas_Alias WEBUSERS = www-data, nginx, apache

# Database users
Runas_Alias DBUSERS = postgres, mysql, mongod

# Application service accounts
Runas_Alias APPUSERS = tomcat, wildfly, nodeapp
```

Usage in a rule:
```
# Developers can run commands as any web or app user
DEVELOPERS ALL=(WEBUSERS) ALL
DEVELOPERS ALL=(APPUSERS) /opt/app/bin/deploy.sh
```

### Host_Alias

Groups hosts together:

```
# Web server farm
Host_Alias WEBSERVERS = web01, web02, web03, web04

# Database servers
Host_Alias DBSERVERS = db01, db02

# By IP address
Host_Alias INTERNAL = 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16

# By network
Host_Alias DMZ = 203.0.113.0/24
```

Usage in a rule:
```
# DBAs only get database privileges on database servers
DBADMINS DBSERVERS=(postgres) /usr/bin/psql, /usr/bin/pg_dump, /usr/bin/pg_restore
```

### Cmnd_Alias

Groups commands together:

```
# Service management commands
Cmnd_Alias SERVICES = /usr/bin/systemctl start *, \
                       /usr/bin/systemctl stop *, \
                       /usr/bin/systemctl restart *, \
                       /usr/bin/systemctl reload *, \
                       /usr/bin/systemctl status *

# Network diagnostic commands
Cmnd_Alias NETWORKING = /usr/sbin/ip, \
                        /usr/bin/ss, \
                        /usr/sbin/tcpdump, \
                        /usr/bin/traceroute, \
                        /usr/sbin/iptables -L *

# Package management
Cmnd_Alias PACKAGES = /usr/bin/dnf install *, \
                      /usr/bin/dnf update *, \
                      /usr/bin/dnf remove *, \
                      /usr/bin/rpm -i *, \
                      /usr/bin/rpm -U *

# Log viewing commands
Cmnd_Alias LOGS = /usr/bin/journalctl, \
                  /usr/bin/tail -f /var/log/*, \
                  /usr/bin/less /var/log/*, \
                  /usr/bin/cat /var/log/*

# User management
Cmnd_Alias USERMGMT = /usr/sbin/useradd, \
                      /usr/sbin/usermod, \
                      /usr/sbin/userdel, \
                      /usr/bin/passwd, \
                      /usr/sbin/groupadd, \
                      /usr/sbin/groupmod, \
                      /usr/sbin/groupdel
```

### Complete Alias-Based Configuration Example

```
# ============ ALIASES ============

# User Aliases
User_Alias FULLADMINS = alice, bob
User_Alias JURADMINS = charlie, dave
User_Alias DEVELOPERS = eve, frank, grace
User_Alias DBAS = henry, iris

# Host Aliases
Host_Alias WEBSERVERS = web01, web02, web03
Host_Alias DBSERVERS = db01, db02
Host_Alias ALLSERVERS = WEBSERVERS, DBSERVERS

# Runas Aliases
Runas_Alias WEBACCTS = apache, nginx
Runas_Alias DBACCTS = postgres, mysql

# Command Aliases
Cmnd_Alias SERVICES = /usr/bin/systemctl start *, /usr/bin/systemctl stop *, \
                       /usr/bin/systemctl restart *, /usr/bin/systemctl status *
Cmnd_Alias WEBDEPLOY = /opt/deploy/web-deploy.sh, /usr/bin/rsync
Cmnd_Alias DBTOOLS = /usr/bin/psql, /usr/bin/pg_dump, /usr/bin/pg_restore
Cmnd_Alias LOGS = /usr/bin/journalctl, /usr/bin/tail -f /var/log/*

# ============ RULES ============

# Full admins get full access everywhere
FULLADMINS ALLSERVERS=(ALL:ALL) ALL

# Junior admins can manage services and view logs
JURADMINS ALLSERVERS=(root) SERVICES, LOGS

# Developers can deploy to web servers
DEVELOPERS WEBSERVERS=(root) NOPASSWD: WEBDEPLOY
DEVELOPERS WEBSERVERS=(WEBACCTS) NOPASSWD: ALL

# DBAs can use database tools on database servers
DBAS DBSERVERS=(DBACCTS) DBTOOLS
```

---

## Tags

Tags modify how sudo handles commands. They apply to all commands that follow them in the rule, until overridden by another tag.

### NOPASSWD / PASSWD

```
# No password required for these commands
deploy ALL=(root) NOPASSWD: /usr/bin/systemctl restart myapp

# Password required (default -- used to override NOPASSWD)
deploy ALL=(root) NOPASSWD: /usr/bin/systemctl restart myapp, \
                  PASSWD: /usr/sbin/reboot
```

In the above example, `deploy` can restart myapp without a password, but rebooting requires a password.

**Combining NOPASSWD and PASSWD in the same rule:**
```
admin ALL=(ALL) ALL
admin ALL=(ALL) NOPASSWD: /usr/bin/systemctl status *
```

The first rule requires a password for everything. The second rule (which comes later and therefore takes precedence for matching commands) allows status checks without a password.

### NOEXEC

Prevents the command from executing child processes. This is a security feature that mitigates the risk of allowing programs that can spawn shells:

```
# Allow less but prevent shell escapes
akshay ALL=(root) NOEXEC: /usr/bin/less
```

With NOEXEC, even if a user types `!bash` inside less, the shell will not execute. However, NOEXEC is not foolproof -- it works by preventing shared library calls to `exec()` functions, but statically compiled binaries are not affected.

**How NOEXEC works internally:**
- sudo sets the `LD_PRELOAD` environment variable to load a library that overrides exec-family functions
- When the command tries to call exec(), the preloaded library returns an error
- This only works on dynamically linked binaries

### SETENV / NOSETENV

Controls whether the user can pass environment variables through sudo:

```
# Allow setting environment variables
deploy ALL=(root) SETENV: /opt/deploy/run.sh

# Usage:
sudo JAVA_HOME=/usr/lib/jvm/java-17 /opt/deploy/run.sh

# Prevent setting environment variables (default when env_reset is on)
deploy ALL=(root) NOSETENV: /opt/deploy/run.sh
```

**Security concern:** SETENV can be dangerous because it allows setting variables like `LD_PRELOAD`, `LD_LIBRARY_PATH`, and `PATH`, which can be used for privilege escalation. Only use SETENV when absolutely necessary and only for specific commands.

### LOG_INPUT / LOG_OUTPUT

Enable session recording for specific rules:

```
# Record all input and output for admin commands
ADMINS ALL=(ALL) LOG_INPUT: LOG_OUTPUT: ALL
```

This records the complete terminal session (all keystrokes and output) for forensic analysis. Logs are stored in a configurable directory (default: `/var/log/sudo-io/`).

---

## Command Exclusions and Negation

### Using ! for Negation

The `!` operator negates a command, user, host, or alias:

```
# Allow all commands EXCEPT shells
akshay ALL=(ALL) ALL, !/bin/bash, !/bin/sh, !/bin/zsh, !/bin/csh, !/bin/tcsh
```

### Why Command Blacklists Are Almost Always Insecure

The above rule **looks** restrictive -- it prevents running shells directly. However, it is **trivially bypassable**:

**Ways to get a shell despite the blacklist:**

1. **Use another shell not in the list:**
   ```bash
   sudo /bin/dash          # Different shell not blocked
   sudo /bin/fish          # Fish shell not blocked
   sudo /usr/bin/bash      # Different path to bash (symlink)
   ```

2. **Use an interpreter:**
   ```bash
   sudo python3 -c 'import os; os.system("/bin/bash")'
   sudo perl -e 'exec "/bin/bash"'
   sudo ruby -e 'exec "/bin/bash"'
   ```

3. **Use an editor:**
   ```bash
   sudo vi
   # Then type :!/bin/bash
   ```

4. **Use find:**
   ```bash
   sudo find / -name "anything" -exec /bin/bash \;
   ```

5. **Use awk:**
   ```bash
   sudo awk 'BEGIN {system("/bin/bash")}'
   ```

6. **Use less or more:**
   ```bash
   sudo less /etc/passwd
   # Then type !/bin/bash
   ```

7. **Copy a shell:**
   ```bash
   sudo cp /bin/bash /tmp/notbash
   sudo /tmp/notbash
   ```

8. **Use nmap:**
   ```bash
   sudo nmap --interactive
   nmap> !bash
   ```

9. **Use expect:**
   ```bash
   sudo expect -c 'spawn /bin/bash; interact'
   ```

10. **Use git:**
    ```bash
    sudo git help status
    # Then type !/bin/bash at the pager prompt
    ```

### The Correct Approach: Whitelisting

Instead of trying to block specific commands (blacklisting), only allow the specific commands needed (whitelisting):

```
# WRONG: Blacklist approach (insecure)
deploy ALL=(ALL) ALL, !/bin/bash, !/bin/sh

# RIGHT: Whitelist approach (secure)
deploy ALL=(root) NOPASSWD: /usr/bin/systemctl restart myapp.service, \
                             /usr/bin/systemctl status myapp.service
```

**The principle:** If you cannot enumerate exactly what commands a user needs, you probably should not be giving them sudo access at all. Command blacklists provide a false sense of security.

---

## Important Defaults

The `Defaults` directive in sudoers sets global options that affect how sudo behaves. These are critical for security and usability.

### secure_path

```
Defaults secure_path="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
```

**What it does:** Sets a fixed PATH for all commands run through sudo, regardless of the invoking user's PATH.

**Why it matters:** Without secure_path, a malicious user could:
1. Create `/home/attacker/bin/systemctl` (a malicious script)
2. Set their PATH to `/home/attacker/bin:/usr/bin:/bin`
3. Run `sudo systemctl restart httpd`
4. The malicious systemctl would execute instead of the real one

With secure_path, sudo ignores the user's PATH entirely and uses only trusted system directories.

> **Note on RHEL 9/AlmaLinux 9:** secure_path is set by default in the sudoers file.

### env_reset

```
Defaults env_reset
```

**What it does:** Clears the environment when running commands through sudo, keeping only a minimal set of safe variables.

**Why it matters:** Environment variables can be attack vectors:
- `LD_PRELOAD` -- load malicious shared libraries
- `LD_LIBRARY_PATH` -- redirect library loading
- `PYTHONPATH` / `PERL5LIB` -- inject malicious modules
- `PATH` -- redirect command execution

With env_reset, these dangerous variables are removed before the command executes.

The variables that ARE kept are controlled by `env_keep`:
```
Defaults env_keep += "COLORS DISPLAY HOSTNAME HISTSIZE KDEDIR LS_COLORS"
Defaults env_keep += "MAIL PS1 PS2 QTDIR USERNAME LANG LC_ADDRESS"
```

### timestamp_timeout

```
Defaults timestamp_timeout=5
```

**What it does:** Sets how long (in minutes) sudo caches credentials before requiring re-authentication.

| Value | Behavior |
|-------|----------|
| `5` (default) | Cache for 5 minutes |
| `0` | Always prompt for password |
| `-1` | Never expire (not recommended) |
| `15` | Cache for 15 minutes |

```
# Always require password (most secure)
Defaults timestamp_timeout=0

# Cache for 15 minutes (more convenient)
Defaults timestamp_timeout=15

# Per-user timeout
Defaults:deploy timestamp_timeout=0
Defaults:admin timestamp_timeout=10
```

### passwd_timeout

```
Defaults passwd_timeout=5
```

**What it does:** Sets how long (in minutes) sudo waits for the user to enter their password before timing out.

```
# Wait 2 minutes for password entry
Defaults passwd_timeout=2

# No timeout (wait forever)
Defaults passwd_timeout=0
```

### logfile

```
Defaults logfile="/var/log/sudo.log"
```

**What it does:** Sends sudo logs to a dedicated file in addition to syslog.

This creates a separate, dedicated sudo log that is easy to monitor, grep, and archive. The file is created with restricted permissions.

### log_input and log_output

```
Defaults log_input, log_output
Defaults iolog_dir="/var/log/sudo-io/%{user}"
```

**What it does:** Records complete terminal sessions -- all keystrokes (input) and all screen output. This creates a complete replay of every sudo session.

**Replay a recorded session:**
```bash
sudo sudoreplay -l                    # List recorded sessions
sudo sudoreplay -l user=akshay        # List sessions by user
sudo sudoreplay <session_id>          # Replay a specific session
```

This is used in high-security environments for compliance and forensic analysis.

### requiretty

```
Defaults requiretty
```

**What it does:** Requires that sudo be run from a terminal (TTY). Prevents sudo from being run from scripts, cron jobs, or other non-interactive contexts.

**History on RHEL:**
- RHEL 6 and earlier: `requiretty` was enabled by default
- RHEL 7+: `requiretty` was **removed** from defaults because it interfered with automation tools (Ansible, Puppet, etc.)

If you need to allow specific users to run sudo without a TTY (e.g., for automation):
```
Defaults requiretty
Defaults:deploy !requiretty
```

### visiblepw

```
Defaults !visiblepw
```

**What it does:** Prevents sudo from running if the password would be displayed on screen. This can happen when sudo is invoked from a terminal that does not support password hiding.

### Other Useful Defaults

```
# Set the maximum number of password attempts
Defaults passwd_tries=3

# Custom insult messages on wrong password (for fun, not production)
Defaults insults

# Mail alerts for unauthorized sudo attempts
Defaults mailto="admin@example.com"
Defaults mail_always

# Require password for sudo -l
Defaults listpw=always

# Set default target user (already root by default)
Defaults runas_default=root

# Set umask for commands run through sudo
Defaults umask=0022

# Custom password prompt
Defaults passprompt="[sudo] Enter YOUR password for %u: "
```

### Per-User, Per-Host, Per-Command Defaults

Defaults can be scoped:

```
# Per-user defaults
Defaults:deploy timestamp_timeout=0
Defaults:admin timestamp_timeout=15

# Per-host defaults
Defaults@WEBSERVERS log_output

# Per-command defaults
Defaults!/usr/bin/passwd passwd_timeout=1

# Per-runas-user defaults
Defaults>postgres log_input, log_output
```

---

## Rule Precedence

### Last Matching Rule Wins

This is the **most critical** concept in sudoers configuration:

> When multiple rules match a user/command combination, the **last matching rule wins**.

This means rule ORDER matters enormously.

### Example: Rule Ordering Problem

```
# Rule 1: deploy gets NOPASSWD for systemctl
deploy ALL=(root) NOPASSWD: /usr/bin/systemctl restart myapp

# Rule 2: deploy gets general access (with password)
deploy ALL=(ALL) ALL
```

**Result:** deploy is prompted for a password even when running `systemctl restart myapp` because Rule 2 matches and it comes LAST. Rule 2 overrides Rule 1's NOPASSWD.

### Correct Ordering

```
# Rule 1: General access (with password) -- FIRST
deploy ALL=(ALL) ALL

# Rule 2: NOPASSWD for specific commands -- LAST (overrides Rule 1)
deploy ALL=(root) NOPASSWD: /usr/bin/systemctl restart myapp
```

**Result:** deploy can run `systemctl restart myapp` without a password, and all other commands require a password.

### File Ordering in sudoers.d

Files in `/etc/sudoers.d/` are processed in **lexicographic (sorted) order**:

```
/etc/sudoers            # Processed first
/etc/sudoers.d/10-admins       # Then these in order
/etc/sudoers.d/20-developers
/etc/sudoers.d/30-deploy
/etc/sudoers.d/90-overrides    # Processed last
```

Rules in later files override rules in earlier files. This is why numeric prefixes are important for managing precedence.

### Viewing Effective Permissions

```bash
# See what the current user can do
sudo -l

# See what a specific user can do
sudo -l -U deploy

# Verbose output showing which rules matched
sudo -ll -U deploy
```

---

## Production Scenarios

### Scenario 1: Application Deployment User

**Requirement:** A deploy user needs to restart a specific application service without a password.

```
# /etc/sudoers.d/30-deploy

# Deploy user can manage the myapp service
deploy ALL=(root) NOPASSWD: /usr/bin/systemctl restart myapp.service, \
                             /usr/bin/systemctl start myapp.service, \
                             /usr/bin/systemctl stop myapp.service, \
                             /usr/bin/systemctl status myapp.service, \
                             /usr/bin/systemctl reload myapp.service
```

**Why this is secure:**
- Only the specific service is allowed
- The full unit name (`myapp.service`) is specified -- not a wildcard
- NOPASSWD is appropriate because this is likely used by automation
- No shell access, no privilege escalation path

### Scenario 2: Deployment Automation Pipeline

**Requirement:** A CI/CD pipeline needs to deploy code and manage services.

```
# /etc/sudoers.d/30-deploy-pipeline

Cmnd_Alias DEPLOY_CMDS = /usr/bin/systemctl restart myapp.service, \
                          /usr/bin/systemctl start myapp.service, \
                          /usr/bin/systemctl stop myapp.service, \
                          /usr/bin/rsync -a --delete /opt/staging/ /var/www/myapp/

deploy ALL=(root) NOPASSWD: DEPLOY_CMDS
```

**Note:** The rsync command has specific arguments that must match exactly. `sudo rsync -a /other/path /somewhere` would NOT be allowed.

### Scenario 3: Log Viewer Role

**Requirement:** Operations team needs to view application logs without full root access.

```
# /etc/sudoers.d/40-logviewer

Cmnd_Alias VIEW_LOGS = /usr/bin/journalctl -u myapp*, \
                        /usr/bin/journalctl -u nginx*, \
                        /usr/bin/tail -f /var/log/myapp/*, \
                        /usr/bin/less /var/log/myapp/*, \
                        /usr/bin/cat /var/log/myapp/*, \
                        /usr/bin/zcat /var/log/myapp/*

User_Alias LOGVIEWERS = ops_alice, ops_bob, ops_charlie

LOGVIEWERS ALL=(root) NOPASSWD: NOEXEC: VIEW_LOGS
```

**Important:** NOEXEC is used here because `less` and `journalctl` have shell escape features (type `!bash`). NOEXEC prevents those escapes from working.

### Scenario 4: Running Commands as a Service Account

**Requirement:** Deploy user needs to run a deployment script as the application user.

```
# /etc/sudoers.d/30-deploy-appuser

deploy ALL=(appuser) NOPASSWD: /opt/app/bin/deploy.sh, \
                                /opt/app/bin/healthcheck.sh
```

Usage:
```bash
sudo -u appuser /opt/app/bin/deploy.sh
```

**Security note:** The scripts must NOT be writable by the deploy user. If deploy could modify the script, they could put anything in it and then run it as appuser.

### Scenario 5: Developer Access with Aliases

**Requirement:** Developers need Docker and Git access through sudo.

```
# /etc/sudoers.d/20-developers

User_Alias DEVELOPERS = alice, bob, charlie, dave
Cmnd_Alias DEV_CMDS = /usr/bin/docker, \
                       /usr/bin/docker-compose, \
                       /usr/bin/git

DEVELOPERS ALL=(root) DEV_CMDS
```

**Security warning:** Allowing `docker` via sudo is essentially equivalent to giving full root access. Docker can mount the host filesystem, access the Docker socket, and run containers as root. See the security section below.

### Scenario 6: Mixed Password Requirements

**Requirement:** An admin needs full sudo access with a password, but checking service status should not require a password.

```
# /etc/sudoers.d/10-admin

# General access requires password (this must come FIRST)
admin ALL=(ALL) ALL

# Service status does not require password (this comes LAST, overrides)
admin ALL=(ALL) NOPASSWD: /usr/bin/systemctl status *
```

**Remember:** Last matching rule wins. The NOPASSWD rule must come after the general rule.

### Scenario 7: Database Administrator Role

**Requirement:** DBA needs to run psql as the postgres user, with a password.

```
# /etc/sudoers.d/50-dba

User_Alias DBAS = henry, iris
Cmnd_Alias DB_CMDS = /usr/bin/psql, \
                      /usr/bin/pg_dump, \
                      /usr/bin/pg_restore, \
                      /usr/bin/createdb, \
                      /usr/bin/dropdb

DBAS ALL=(postgres) PASSWD: DB_CMDS
```

Usage:
```bash
sudo -u postgres psql -d production_db
```

### Scenario 8: Read-Only System Monitoring

**Requirement:** Monitoring tool needs read-only access to system information.

```
# /etc/sudoers.d/50-monitoring

Cmnd_Alias MONITORING = /usr/bin/ss -tulnp, \
                         /usr/bin/df -h, \
                         /usr/sbin/fdisk -l, \
                         /usr/bin/free -m, \
                         /usr/bin/dmidecode, \
                         /usr/sbin/pvs, \
                         /usr/sbin/vgs, \
                         /usr/sbin/lvs, \
                         /usr/bin/lscpu

monitoring ALL=(root) NOPASSWD: NOEXEC: MONITORING
```

---

## Privilege Escalation Through sudo - Critical Security Section

This section covers common mistakes in sudoers configuration that allow privilege escalation. Understanding these attack vectors is essential for any system administrator.

### 1. Text Editors (vim, nano, vi, emacs)

**The seemingly restrictive rule:**
```
developer ALL=(root) /usr/bin/vim
```

**How it is exploited:**
```bash
sudo vim /etc/hostname
# Inside vim, type:
:!/bin/bash
# You now have a root shell

# Or:
:set shell=/bin/bash
:shell
# Root shell again

# Or with nano:
sudo nano /etc/hostname
# Press Ctrl+T (execute command)
# Type: /bin/bash
# Root shell
```

**Why it works:** Text editors have features to execute external commands ("shell escape"). When running as root, those shell commands run as root too.

**The secure alternative:**
```
# Use sudoedit instead of allowing editors
developer ALL=(root) sudoedit /etc/myapp/config.yaml
```

`sudoedit` (also invoked as `sudo -e`) copies the file to a temporary location, lets the user edit it with their own editor (running as themselves, NOT as root), and then copies it back. No shell escape is possible because the editor never runs with root privileges.

### 2. Scripting Language Interpreters (python, perl, ruby, lua)

**The seemingly restrictive rule:**
```
developer ALL=(root) /usr/bin/python3
```

**How it is exploited:**
```bash
sudo python3 -c 'import os; os.system("/bin/bash")'
# Root shell

sudo python3 -c 'import subprocess; subprocess.call(["/bin/bash"])'
# Root shell

# With perl:
sudo perl -e 'exec "/bin/bash";'
# Root shell

# With ruby:
sudo ruby -e 'exec "/bin/bash"'
# Root shell
```

**Why it works:** Interpreters can execute arbitrary system commands. Giving root access to an interpreter is identical to giving a root shell.

**The secure alternative:**
```
# Run a specific script, not the interpreter itself
developer ALL=(root) NOPASSWD: /opt/scripts/generate_report.py

# Make sure the script is NOT writable by the developer:
# chown root:root /opt/scripts/generate_report.py
# chmod 755 /opt/scripts/generate_report.py
```

### 3. Shells (bash, sh, zsh, csh, tcsh, fish, dash)

**The seemingly restrictive rule:**
```
developer ALL=(root) /bin/bash
```

**How it is exploited:**
```bash
sudo /bin/bash
# You now have a root shell. This is not even an "exploit" -- it is the direct effect.
```

**The secure alternative:**
There is no secure way to give someone shell access with sudo while restricting what they can do in that shell. If someone needs a root shell, they need full sudo access and should be trusted with it.

### 4. Package Managers (dnf, yum, apt, rpm)

**The seemingly restrictive rule:**
```
developer ALL=(root) /usr/bin/dnf
```

**How it is exploited:**

```bash
# Method 1: Install a package with a malicious post-install script
# Create an RPM with a %post script that adds your SSH key to root's authorized_keys

# Method 2: Use dnf's built-in plugin system
sudo dnf install backdoor-package

# Method 3: rpm can run scripts
sudo rpm -i malicious.rpm
# The RPM's %post script runs as root

# Method 4: dnf has a shell mode
sudo dnf shell
# Provides an interactive dnf shell that can do anything
```

**Why it works:** Package managers run pre/post-install scripts as root. They can also download and install arbitrary software, including backdoors.

**The secure alternative:**
```
# Only allow specific, read-only package operations
developer ALL=(root) /usr/bin/dnf list *, /usr/bin/dnf info *, /usr/bin/rpm -qa

# Or better: don't give package management access through sudo at all.
# Use a CI/CD pipeline or configuration management tool for package installation.
```

### 5. Broad Service Management

**The seemingly restrictive rule:**
```
developer ALL=(root) /usr/bin/systemctl *
```

**How it is exploited:**
```bash
# Create a malicious service unit
# (if developer can write to any directory systemd reads)
cat > /tmp/evil.service << 'EOF'
[Service]
Type=oneshot
ExecStart=/bin/bash -c 'cp /bin/bash /tmp/rootbash && chmod +s /tmp/rootbash'
EOF

sudo systemctl link /tmp/evil.service
sudo systemctl start evil.service
/tmp/rootbash -p
# Root shell

# Or even simpler with systemctl edit:
sudo systemctl edit someservice
# Modify ExecStart to run a malicious command
```

**The secure alternative:**
```
# Allow only specific service names and specific actions
developer ALL=(root) NOPASSWD: /usr/bin/systemctl restart myapp.service, \
                                /usr/bin/systemctl status myapp.service
```

### 6. User-Writable Scripts

**The seemingly restrictive rule:**
```
developer ALL=(root) NOPASSWD: /home/developer/scripts/deploy.sh
```

**How it is exploited:**
```bash
# The developer can edit the script!
echo '#!/bin/bash' > /home/developer/scripts/deploy.sh
echo '/bin/bash' >> /home/developer/scripts/deploy.sh
chmod +x /home/developer/scripts/deploy.sh
sudo /home/developer/scripts/deploy.sh
# Root shell
```

**Why it works:** If the user can modify the script that sudo allows them to run, they can put anything in it.

**The secure alternative:**
```
# Store scripts in a root-owned directory with proper permissions
developer ALL=(root) NOPASSWD: /opt/scripts/deploy.sh

# The script must be:
# - Owned by root: chown root:root /opt/scripts/deploy.sh
# - Not writable by others: chmod 755 /opt/scripts/deploy.sh
# - In a directory not writable by the user: chown root:root /opt/scripts
```

### 7. Wildcard Arguments

**The seemingly restrictive rule:**
```
developer ALL=(root) /usr/bin/chown developer *
```

**How it is exploited:**
```bash
# The wildcard matches ANY arguments after "developer"
sudo chown developer /etc/shadow
# Developer now owns /etc/shadow and can read all password hashes

sudo chown developer /etc/sudoers
# Developer now owns sudoers and can give themselves full root access
```

**The secure alternative:**
```
# Restrict to specific directory
developer ALL=(root) /usr/bin/chown developer /var/www/myapp/*

# But even this can be escaped with .. paths:
# sudo chown developer /var/www/myapp/../../etc/shadow
# Better: use a wrapper script with input validation
developer ALL=(root) NOPASSWD: /opt/scripts/fix-ownership.sh
```

### 8. Environment Manipulation (SETENV, env)

**The seemingly restrictive rule:**
```
developer ALL=(root) SETENV: /opt/app/run.sh
```

**How it is exploited:**
```bash
# Inject a malicious shared library via LD_PRELOAD
# First, create a malicious library:
gcc -shared -o /tmp/evil.so evil.c
sudo LD_PRELOAD=/tmp/evil.so /opt/app/run.sh
# The malicious library code runs as root BEFORE the script

# Or manipulate PATH:
sudo PATH=/home/developer/bin:$PATH /opt/app/run.sh
# If the script calls any command without a full path,
# the developer's version in ~/bin runs first
```

**The secure alternative:**
```
# Don't use SETENV unless absolutely necessary
developer ALL=(root) NOSETENV: /opt/app/run.sh

# Ensure env_reset is enabled (it is by default on RHEL 9)
Defaults env_reset
```

### 9. Docker Access

**The seemingly restrictive rule:**
```
developer ALL=(root) /usr/bin/docker
```

**How it is exploited:**
```bash
# Mount the host filesystem inside a container
sudo docker run -v /:/hostfs -it alpine /bin/sh
# Inside the container:
chroot /hostfs /bin/bash
# You now have a root shell on the host

# Or directly:
sudo docker run -v /etc/shadow:/mnt/shadow -it alpine cat /mnt/shadow
# Read all password hashes

# Or add your SSH key:
sudo docker run -v /root/.ssh:/mnt/ssh -it alpine \
  sh -c 'echo "ssh-rsa AAAA..." >> /mnt/ssh/authorized_keys'
```

**Why it works:** Docker requires root access and can mount any part of the host filesystem. Giving sudo access to docker is functionally equivalent to giving full root access.

**The secure alternative:**
- Add the developer to the `docker` group instead (still has security implications, but does not require sudo)
- Use Podman in rootless mode (no sudo needed)
- Use a restricted container runtime
- Use a CI/CD pipeline for container operations

### Summary Table of Dangerous Commands

| Command | Risk Level | Escalation Method | Secure Alternative |
|---------|-----------|-------------------|-------------------|
| vim, nano, vi, emacs | CRITICAL | Shell escape (:!/bin/bash) | Use sudoedit |
| python, perl, ruby | CRITICAL | Execute arbitrary code | Allow specific scripts only |
| bash, sh, zsh | CRITICAL | IS a shell | Full sudo or nothing |
| dnf, yum, apt, rpm | CRITICAL | Post-install scripts | Read-only operations only |
| systemctl (broad) | HIGH | Link/edit malicious units | Specific service + action |
| docker | CRITICAL | Mount host filesystem | Podman rootless, CI/CD |
| User-writable scripts | CRITICAL | Modify script contents | Root-owned scripts |
| Wildcards in args | HIGH | Match unintended paths | Specific arguments |
| SETENV | HIGH | LD_PRELOAD, PATH injection | NOSETENV, env_reset |
| find | HIGH | -exec flag | Specific find arguments |
| awk, nawk | HIGH | system() function | Do not allow |
| less, more | MEDIUM | Shell escape (!) | Use NOEXEC tag |
| git | MEDIUM | Pager shell escape | Specific git commands |
| env | HIGH | Set any variable | Do not allow |

---

## Session Recording with sudo

For high-security environments, sudo can record complete terminal sessions.

### Configuration

```
# In sudoers or a drop-in file
Defaults log_input, log_output
Defaults iolog_dir="/var/log/sudo-io/%{seq}"
Defaults iolog_file="%{user}"

# Or for specific users/groups:
Defaults:ADMINS log_input, log_output
```

### Viewing Recorded Sessions

```bash
# List all recorded sessions
sudo sudoreplay -l

# Filter by user
sudo sudoreplay -l user=akshay

# Filter by date
sudo sudoreplay -l fromdate="2024-01-01" todate="2024-12-31"

# Filter by command
sudo sudoreplay -l command=/usr/bin/systemctl

# Replay a specific session (shows it in real-time)
sudo sudoreplay <TSID>

# Replay at 2x speed
sudo sudoreplay -s 2 <TSID>
```

### Use Cases

- Compliance requirements (PCI-DSS, SOX, HIPAA)
- Forensic investigation after security incidents
- Training and review of junior administrators
- Auditing third-party contractor access

---

## Sudoers Best Practices Summary

1. **Always use visudo** to edit sudoers files
2. **Use whitelists, not blacklists** -- specify exactly which commands are allowed
3. **Use full paths** for all commands in sudoers rules
4. **Use aliases** to group users, hosts, and commands for maintainability
5. **Use drop-in files** in /etc/sudoers.d/ for modular configuration
6. **Use numeric prefixes** on drop-in files (10-admins, 20-developers) for clear ordering
7. **Remember: last matching rule wins** -- put specific rules after general rules
8. **Use NOEXEC** for commands with shell escape features (less, vim, etc.) when possible, but prefer sudoedit for file editing
9. **Never allow editors, interpreters, or shells** through sudo if you don't intend to give full root access
10. **Never allow user-writable scripts** through sudo -- scripts must be root-owned and not writable by the user
11. **Use NOPASSWD sparingly** -- only for automation and specific low-risk commands
12. **Ensure env_reset and secure_path** are enabled (they are by default on RHEL 9)
13. **Test changes** with `sudo -l -U username` before and after modifications
14. **Keep backups** of working sudoers configurations
15. **Audit regularly** -- review who has sudo access and what they can do

---

## Interview Questions

### Q1: Explain the sudoers rule structure and what each field means.

**Answer:** A sudoers rule has the format: `user host=(runas_user:runas_group) tag: commands`. The `user` field specifies who the rule applies to (username, %group, or alias). The `host` field specifies which hostname this rule is valid on (usually ALL for single servers). The `runas_user:runas_group` in parentheses specifies which user and group the commands can be run as. The `tag` field (like NOPASSWD, NOEXEC) modifies behavior. The `commands` field lists which commands are allowed, always with full paths. Example: `deploy ALL=(root) NOPASSWD: /usr/bin/systemctl restart myapp` means user deploy on any host can run systemctl restart myapp as root without a password.

### Q2: What are sudoers aliases and why are they useful?

**Answer:** Sudoers aliases group related items under a meaningful name. There are four types: User_Alias (groups users), Host_Alias (groups hostnames), Runas_Alias (groups target users), and Cmnd_Alias (groups commands). They improve maintainability by allowing you to define groups once and reference them in rules. For example, `User_Alias ADMINS = alice, bob` and `Cmnd_Alias SERVICES = /usr/bin/systemctl start *, /usr/bin/systemctl restart *` can be combined as `ADMINS ALL=(root) SERVICES`. When you need to add a new admin or a new command, you only change the alias definition, not every rule.

### Q3: Explain why "last matching rule wins" is critical in sudoers configuration.

**Answer:** When multiple sudoers rules match a user/command combination, sudo applies the LAST matching rule. This means rule ordering is crucial. For example, if you have `admin ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart myapp` followed by `admin ALL=(ALL) ALL`, the admin will be prompted for a password even for systemctl restart myapp because the second rule (which requires a password) is the last match. The correct ordering puts the general rule first and the specific NOPASSWD override second. This also applies across files in sudoers.d/ -- files are processed in lexicographic order, so rules in later files override earlier ones.

### Q4: Why is allowing vim/vi through sudo a security risk?

**Answer:** Text editors like vim, vi, nano, and emacs have "shell escape" features that allow executing system commands from within the editor. When running under sudo (as root), typing `:!/bin/bash` in vim gives a root shell. Similarly, `:set shell=/bin/bash` followed by `:shell` achieves the same result. The user effectively has full root access, not just the ability to edit files. The secure alternative is `sudoedit` (or `sudo -e`), which copies the file to a temp location, lets the user edit it with their own editor (running as themselves), and copies it back. The editor never runs as root.

### Q5: What is the difference between NOPASSWD and NOEXEC tags?

**Answer:** NOPASSWD controls authentication -- it tells sudo to skip the password prompt for the specified commands. The user does not need to enter their password. NOEXEC controls execution -- it prevents the command from spawning child processes (subshells, sub-commands). NOEXEC works by preloading a library that intercepts exec() system calls. This is useful for programs like `less` or `vim` that have shell escape features. With NOEXEC, `:!/bin/bash` in vim would fail. However, NOEXEC only works on dynamically linked binaries and is not a guarantee against all forms of privilege escalation. They serve different purposes and can be combined.

### Q6: What is `secure_path` and why is it important?

**Answer:** `Defaults secure_path` sets a fixed, hardcoded PATH for all commands run through sudo. Without it, sudo would inherit the user's PATH, which could include directories under the user's control (like ~/bin). A malicious user could create a trojan command in their PATH that would execute instead of the real system command. For example, creating `~/bin/systemctl` as a script that gives them a shell. With secure_path, sudo ignores the user's PATH and only looks in trusted system directories (/usr/sbin, /usr/bin, etc.). On RHEL 9/AlmaLinux 9, secure_path is configured by default.

### Q7: Explain the SETENV tag and its security implications.

**Answer:** The SETENV tag allows a user to pass environment variables through to commands run via sudo. Normally, with `env_reset` enabled (the default on RHEL 9), sudo clears the environment for security. SETENV overrides this for specific rules. The security risk is significant: a user with SETENV can set `LD_PRELOAD` to load a malicious shared library that runs as root before the actual command, set `PATH` to redirect command execution, or set language-specific variables (PYTHONPATH, PERL5LIB) to inject malicious code. SETENV should only be used when absolutely necessary and only for specific commands, not broadly.

### Q8: How would you design a sudoers configuration for a team of 5 developers who need to deploy applications?

**Answer:** I would create a drop-in file `/etc/sudoers.d/30-deploy` with:
1. A User_Alias for the developers: `User_Alias DEPLOYERS = dev1, dev2, dev3, dev4, dev5`
2. A Cmnd_Alias for deployment commands: `Cmnd_Alias DEPLOY_CMDS = /usr/bin/systemctl restart myapp.service, /opt/deploy/deploy.sh`
3. The rule: `DEPLOYERS ALL=(root) NOPASSWD: DEPLOY_CMDS`
4. Ensure deploy scripts are root-owned and not writable by developers
5. Use NOPASSWD since deployment may be automated
6. Log all sudo activity for accountability
7. Do NOT give broad access like ALL commands or allow shells/editors/interpreters

### Q9: What is command negation (!) in sudoers and why is it generally insecure?

**Answer:** Command negation uses `!` to exclude specific commands from an otherwise broad rule, like `ALL, !/bin/bash, !/bin/sh`. While this looks like it restricts shell access, it is almost always bypassable. Users can: use a different shell not in the exclusion list (dash, fish), use an interpreter (python3 -c 'import os; os.system("/bin/bash")'), use a text editor's shell escape, use find with -exec, use awk's system() function, or copy a shell to a different path. This "blacklist" approach provides a false sense of security. The correct approach is whitelisting -- only allowing the specific commands needed. If you cannot enumerate what commands someone needs, you probably should not be giving them sudo access.

### Q10: How do you audit and verify sudo permissions across all users on a system?

**Answer:** Use `sudo -l -U username` to check each user's effective sudo permissions. For a comprehensive audit: `for u in $(awk -F: '$3 >= 1000 {print $1}' /etc/passwd); do echo "=== $u ==="; sudo -l -U $u 2>/dev/null; done`. Also check: (1) which users are in the wheel group: `grep wheel /etc/group`, (2) all files in /etc/sudoers.d/: `ls -la /etc/sudoers.d/`, (3) syntax validation: `visudo -c`, (4) search for NOPASSWD rules: `grep -r NOPASSWD /etc/sudoers /etc/sudoers.d/`. Regular auditing should be part of security operations, checking for overly broad rules, unnecessary NOPASSWD, dangerous command allowances (editors, interpreters), and users who no longer need sudo access.

### Q11: Explain how sudo session recording works and when you would use it.

**Answer:** sudo can record complete terminal sessions with `Defaults log_input, log_output`. This captures all keystrokes (stdin) and all terminal output (stdout/stderr), stored in a configurable directory (default: /var/log/sudo-io/). Sessions can be replayed with `sudoreplay`, which plays back the session in real-time or at an adjustable speed. Use cases include: compliance requirements (PCI-DSS, SOX, HIPAA mandate privileged access logging), forensic investigation after security incidents, auditing third-party contractor access, and training review. Configuration can be targeted to specific users or groups: `Defaults:CONTRACTORS log_input, log_output` to record only contractor sessions.

### Q12: How do wildcard arguments in sudoers rules create security risks?

**Answer:** Wildcards in sudoers match any argument, which can lead to unintended command execution. For example, `user ALL=(root) /usr/bin/chown user *` was intended to let a user change ownership of their files, but the wildcard matches ANY path -- including `/etc/shadow`, `/etc/sudoers`, or any system file. Similarly, `user ALL=(root) /usr/bin/systemctl restart *` allows restarting ANY service, not just the intended one. Path traversal is also a risk: `/usr/bin/cat /var/log/myapp/*` could be exploited with `sudo cat /var/log/myapp/../../etc/shadow`. The fix is to specify exact arguments: `/usr/bin/systemctl restart myapp.service` instead of using wildcards.

### Q13: What is the difference between `Defaults requiretty` and `Defaults !requiretty`?

**Answer:** `Defaults requiretty` requires that sudo be run from an actual terminal (TTY), preventing execution from cron jobs, scripts run without a terminal, and certain automation tools. This was the default on RHEL 6 and earlier but was removed from defaults on RHEL 7+ because it interfered with automation tools like Ansible and Puppet, which run commands over SSH without an interactive terminal. `Defaults !requiretty` (the current default) allows sudo from any context, including non-interactive sessions. If you need requiretty as a baseline but need to exempt certain users for automation, use: `Defaults requiretty` followed by `Defaults:ansible !requiretty`.

### Q14: How do per-user Defaults work in sudoers?

**Answer:** Defaults can be scoped to specific users, hosts, commands, or runas users using special syntax. `Defaults:username` applies to a specific user: `Defaults:deploy timestamp_timeout=0` makes deploy always enter a password. `Defaults@hostname` applies on specific hosts: `Defaults@WEBSERVERS log_output`. `Defaults!command` applies to specific commands: `Defaults!/usr/bin/passwd passwd_timeout=1`. `Defaults>runas_user` applies when running as a specific user: `Defaults>postgres log_input, log_output`. These scoped Defaults allow fine-grained configuration without affecting all users.

### Q15: Explain the security model difference between blacklisting and whitelisting in sudo.

**Answer:** Blacklisting (using negation like `ALL, !/bin/bash`) starts with giving everything and trying to remove dangerous items. This is fundamentally insecure because: (1) you cannot anticipate every possible way to bypass the blacklist, (2) new tools and methods are constantly being discovered, (3) it requires perfect knowledge of all dangerous commands on the system, and (4) a single oversight gives full root access. Whitelisting (specifying only allowed commands like `/usr/bin/systemctl restart myapp.service`) starts with giving nothing and adding only what is needed. This follows the principle of least privilege. Even if you miss a command the user needs, the failure mode is "user cannot do something they need" (fixed by adding a rule) rather than "user has full root access" (a security breach). Whitelisting is always the recommended approach.

### Q16: How would you handle a situation where sudo rules break and no one can use sudo?

**Answer:** This typically happens when someone edits /etc/sudoers directly (without visudo) and introduces a syntax error. Recovery options, in order of preference: (1) Use `pkexec visudo` if PolicyKit is installed -- pkexec provides an alternative privilege escalation path. (2) If you have an active root shell open anywhere, use it to fix the file. (3) Log in as root at the physical console (or IPMI/iDRAC/iLO). (4) Boot into single-user/rescue mode from GRUB (append `rd.break` to the kernel line), remount the filesystem read-write, and run visudo. (5) On cloud instances, use the provider's serial console. Prevention is better: always use visudo, keep a backup of working sudoers, and test changes with `visudo -c` before relying on them.

---

## Summary

| Topic | Key Points |
|-------|------------|
| Rule structure | `user host=(runas) tag: commands` -- each field controls a different aspect |
| Aliases | Group users, hosts, runas, and commands for maintainability |
| Tags | NOPASSWD, NOEXEC, SETENV modify rule behavior |
| Negation (!) | Almost always insecure for commands -- use whitelists instead |
| Defaults | secure_path, env_reset, timestamp_timeout are critical for security |
| Precedence | Last matching rule wins -- order matters enormously |
| Dangerous commands | Editors, interpreters, shells, package managers, docker give root shells |
| Whitelisting | Only allow specific commands needed -- never use ALL with restrictions |
| Session recording | log_input/log_output for compliance and forensics |

---

*Previous: [09 - sudo and sudoers Fundamentals](09-sudo-and-sudoers-fundamentals.md)*
*Next: [11 - Linux Authentication and Password Management](11-linux-authentication-and-password-management.md)*

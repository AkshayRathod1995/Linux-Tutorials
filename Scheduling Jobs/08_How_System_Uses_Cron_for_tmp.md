# How Linux Uses Cron to Maintain `/tmp`

## Index

1. [What Is `/tmp` and Why Does It Need Cleaning?](#what-is-tmp-and-why-does-it-need-cleaning)
2. [The Problem: `/tmp` Grows Forever](#the-problem-tmp-grows-forever)
3. [Method 1: `tmpwatch` / `tmpreaper` (The Classic Cron Approach)](#method-1-tmpwatch--tmpreaper-the-classic-cron-approach)
4. [Method 2: `systemd-tmpfiles` (The Modern Approach)](#method-2-systemd-tmpfiles-the-modern-approach)
5. [How `systemd-tmpfiles-clean.timer` Works](#how-systemd-tmpfiles-clean-timer-works)
6. [The Configuration Files: `/etc/tmpfiles.d/` and `/usr/lib/tmpfiles.d/`](#the-configuration-files-etctmpfilesd-and-usrlibtmpfilesd)
7. [Reading tmpfiles.d Configuration Lines](#reading-tmpfilesd-configuration-lines)
8. [Real-World Example: What Happens on Boot and Daily](#real-world-example-what-happens-on-boot-and-daily)
9. [Customizing `/tmp` Cleanup Behavior](#customizing-tmp-cleanup-behavior)
10. [Inspecting Your System's Current Setup](#inspecting-your-systems-current-setup)
11. [Summary: The Full Picture](#summary-the-full-picture)

---

## What Is `/tmp` and Why Does It Need Cleaning?

The `/tmp` directory is Linux's **scratch pad**. Any program, any user, any process can create files here for temporary use.

Examples of what ends up in `/tmp`:

| What Creates It                    | Example Files                          |
| ---------------------------------- | -------------------------------------- |
| A text editor saving a crash recovery file | `/tmp/vi_recovery_akshay_8372` |
| A web browser downloading a file   | `/tmp/firefox_download_abc123.pdf`    |
| A software installer unpacking itself | `/tmp/install_pkg_9281/`           |
| A script using a temporary file     | `/tmp/report_output.csv`             |
| The system itself                   | `/tmp/systemd-private-*`             |

The key rule of `/tmp`: **files here are temporary and disposable.** Programs should never assume that a file in `/tmp` will survive a reboot or even last more than a few hours.

---

## The Problem: `/tmp` Grows Forever

Left unchecked, `/tmp` can fill up your disk. Here's why:

1. **Programs crash** — they create temp files but never clean them up.
2. **Users forget** — they extract files to `/tmp` and walk away.
3. **Long-running processes** — a server running for months accumulates temp files from hundreds of processes.

If `/tmp` fills up and your disk has no space left, things break badly:
- Programs can't write temp files, so they crash
- Databases can't create temporary tables
- Logging stops because there's no room to write
- Login may even fail on some systems

**This is why Linux automates `/tmp` cleanup — you should never have to do it manually.**

---

## Method 1: `tmpwatch` / `tmpreaper` (The Classic Cron Approach)

Older Linux systems (and some current ones) use a tool called `tmpwatch` (CentOS/RHEL) or `tmpreaper` (Ubuntu/Debian) that runs as a **daily cron job**.

### How It Works

A script in `/etc/cron.daily/` runs once per day (via cron + anacron) that deletes files in `/tmp` older than a specified age.

### On CentOS/RHEL (tmpwatch)

```bash
# Check if tmpwatch is installed
rpm -q tmpwatch

# View the cron script that runs it
cat /etc/cron.daily/tmpwatch
```

A typical `tmpwatch` cron script:

```bash
#!/bin/bash
/usr/sbin/tmpwatch -x /tmp/.X11-unix -x /tmp/.XIM-unix \
    -x /tmp/.font-unix -x /tmp/.ICE-unix -x /tmp/.Test-unix \
    240 /tmp
```

**Breaking it down:**

| Part                          | Meaning                                                |
| ----------------------------- | ------------------------------------------------------ |
| `/usr/sbin/tmpwatch`          | The cleanup tool                                       |
| `-x /tmp/.X11-unix`           | **Exclude** this directory (used by the display server) |
| `-x /tmp/.font-unix`          | Exclude font sockets                                   |
| `240`                         | Delete files older than **240 hours** (10 days)        |
| `/tmp`                        | The directory to clean                                 |

So this script says: "Every day, delete everything in `/tmp` that is older than 10 days — except for X11 display sockets and font sockets (which are needed by the graphical interface)."

### On Ubuntu/Debian (tmpreaper)

```bash
# Check if tmpreaper is installed
dpkg -l | grep tmpreaper

# View the cron script
cat /etc/cron.daily/tmpreaper
```

The configuration is in `/etc/tmpreaper.conf`:

```bash
cat /etc/tmpreaper.conf
```

```
TMPREAPER_TIME=7d
TMPREAPER_PROTECT_EXTRA='/tmp/.X11-unix /tmp/.ICE-unix'
TMPREAPER_DIRS='/tmp/.'
TMPREAPER_DELAY='256'
```

| Setting                    | Meaning                                    |
| -------------------------- | ------------------------------------------ |
| `TMPREAPER_TIME=7d`        | Delete files older than 7 days             |
| `TMPREAPER_PROTECT_EXTRA`  | Directories to never delete                |
| `TMPREAPER_DIRS`           | Which directories to clean                 |
| `TMPREAPER_DELAY`          | Random delay (in seconds) before starting  |

### The Cron Chain for tmpwatch/tmpreaper

```
crond (running 24/7)
  │
  ▼
Triggers anacron (once per day)
  │
  ▼
anacron runs: run-parts /etc/cron.daily/
  │
  ▼
Executes /etc/cron.daily/tmpwatch (or tmpreaper)
  │
  ▼
tmpwatch scans /tmp, deletes old files
```

---

## Method 2: `systemd-tmpfiles` (The Modern Approach)

Modern Linux distributions (Ubuntu 16.04+, CentOS 7+, Fedora, Arch) use **systemd-tmpfiles** instead of tmpwatch/tmpreaper. This is a `systemd` service + timer combo — not a traditional cron job.

### The Key Components

```
┌──────────────────────────────────────────────────────┐
│  systemd-tmpfiles-clean.timer                        │
│  (WHEN: daily, and 15 minutes after boot)            │
│                                                      │
│         triggers                                     │
│            │                                         │
│            ▼                                         │
│  systemd-tmpfiles-clean.service                      │
│  (WHAT: runs "systemd-tmpfiles --clean")             │
│                                                      │
│         reads rules from                             │
│            │                                         │
│            ▼                                         │
│  /usr/lib/tmpfiles.d/*.conf   (system defaults)      │
│  /etc/tmpfiles.d/*.conf       (admin overrides)      │
│  /run/tmpfiles.d/*.conf       (runtime overrides)    │
└──────────────────────────────────────────────────────┘
```

---

## How `systemd-tmpfiles-clean.timer` Works

Let's look at the actual timer:

```bash
systemctl cat systemd-tmpfiles-clean.timer
```

```ini
[Unit]
Description=Daily Cleanup of Temporary Directories

[Timer]
OnBootSec=15min
OnUnitActiveSec=1d
```

| Directive             | Meaning                                                |
| --------------------- | ------------------------------------------------------ |
| `OnBootSec=15min`     | First run: 15 minutes after the system boots           |
| `OnUnitActiveSec=1d`  | Then repeat: once every day after that                 |

**No `[Install]` section** — this timer is always enabled by default. You don't need to `systemctl enable` it.

Check its status:

```bash
systemctl status systemd-tmpfiles-clean.timer
```

```
● systemd-tmpfiles-clean.timer - Daily Cleanup of Temporary Directories
     Loaded: loaded (/usr/lib/systemd/system/systemd-tmpfiles-clean.timer; static)
     Active: active (waiting)
    Trigger: Fri 2026-09-19 07:45:32 UTC; 23h left
   Triggers: systemd-tmpfiles-clean.service
```

```bash
systemctl list-timers | grep tmpfiles
```

This shows you when the next cleanup will happen and when the last one ran.

---

## The Configuration Files: `/etc/tmpfiles.d/` and `/usr/lib/tmpfiles.d/`

The rules for what to create, clean, and delete live in `.conf` files across three directories:

| Directory                  | Purpose                                                  | Who edits it? |
| -------------------------- | -------------------------------------------------------- | ------------- |
| `/usr/lib/tmpfiles.d/`     | **System defaults** — installed by packages              | Never (managed by packages) |
| `/etc/tmpfiles.d/`         | **Admin overrides** — your customizations go here        | You (the admin) |
| `/run/tmpfiles.d/`         | **Runtime overrides** — generated on the fly             | System processes |

**Override rule:** A file in `/etc/tmpfiles.d/` with the **same name** as one in `/usr/lib/tmpfiles.d/` completely replaces it. This lets you customize without editing system files.

### The Main `/tmp` Configuration File

```bash
cat /usr/lib/tmpfiles.d/tmp.conf
```

```
# See tmpfiles.d(5) for details

# Clear tmp directories separately, to make them easier to override
q /tmp 1777 root root 10d
q /var/tmp 1777 root root 30d
```

**This is the file that controls `/tmp` cleanup.** Let's decode it.

---

## Reading tmpfiles.d Configuration Lines

The format of each line is:

```
TYPE    PATH    MODE    USER    GROUP    AGE    ARGUMENT
```

Let's decode the two lines from `tmp.conf`:

### Line 1: `q /tmp 1777 root root 10d`

| Field      | Value    | Meaning                                                    |
| ---------- | -------- | ---------------------------------------------------------- |
| **Type**   | `q`      | Create the directory if missing AND clean up old files in it. Also set the right permissions and manage subvolumes if applicable. |
| **Path**   | `/tmp`   | The directory to manage                                    |
| **Mode**   | `1777`   | Permissions: everyone can read/write, but the **sticky bit** is set (users can only delete their own files) |
| **User**   | `root`   | Owner of the directory                                     |
| **Group**  | `root`   | Group owner of the directory                               |
| **Age**    | `10d`    | Delete files older than **10 days**                        |

### Line 2: `q /var/tmp 1777 root root 30d`

Same as above, but for `/var/tmp` — and files are kept for **30 days** instead of 10.

**Why is `/var/tmp` different from `/tmp`?**

| Directory   | Cleanup Age | Purpose                                                  |
| ----------- | ----------- | -------------------------------------------------------- |
| `/tmp`      | 10 days     | Short-lived temporary files; may be cleared on reboot    |
| `/var/tmp`  | 30 days     | Longer-lived temporary files; should survive reboots     |

### Common Type Codes

| Type | Meaning                                                        |
| ---- | -------------------------------------------------------------- |
| `d`  | Create a directory if it doesn't exist                         |
| `D`  | Create a directory; remove all contents if it already exists   |
| `q`  | Create directory (with subvolume support) + clean old files    |
| `r`  | Remove a file or directory                                     |
| `e`  | Clean up (remove old files) in an existing directory           |
| `x`  | Exclude a path from cleanup (ignore it)                        |
| `X`  | Exclude a path and all its contents from cleanup               |

### Age Suffixes

| Suffix | Meaning     | Example |
| ------ | ----------- | ------- |
| `s`    | Seconds     | `300s`  |
| `m`    | Minutes     | `30m`   |
| `h`    | Hours       | `12h`   |
| `d`    | Days        | `10d`   |
| `w`    | Weeks       | `2w`    |

---

## Real-World Example: What Happens on Boot and Daily

### At Boot Time (systemd-tmpfiles --create)

When the system boots, `systemd` runs:

```bash
systemd-tmpfiles --create --remove
```

This:
1. **Creates** directories listed in the config files (like `/tmp`, `/run/lock`)
2. **Sets permissions** on those directories
3. **Removes** any files marked for removal

### Daily (systemd-tmpfiles --clean)

The timer triggers the service daily, which runs:

```bash
systemd-tmpfiles --clean
```

This:
1. Scans `/tmp` and finds files that haven't been accessed in 10+ days
2. Scans `/var/tmp` and finds files older than 30+ days
3. Deletes those files
4. Leaves recent files untouched

**How does it determine file age?** It checks the file's **atime** (access time), **mtime** (modification time), and **ctime** (change time). The file must be older than the specified age on ALL three timestamps to be deleted.

### Seeing It in Action

```bash
# Check when tmpfiles cleanup last ran
systemctl status systemd-tmpfiles-clean.service
```

```
● systemd-tmpfiles-clean.service - Cleanup of Temporary Directories
     Loaded: loaded (/usr/lib/systemd/system/systemd-tmpfiles-clean.service; static)
     Active: inactive (dead) since Thu 2026-09-18 07:45:32 UTC; 16h ago
   Main PID: 8421 (code=exited, status=0/SUCCESS)
```

`status=0/SUCCESS` means the cleanup completed without errors.

```bash
# View logs from the cleanup
journalctl -u systemd-tmpfiles-clean.service
```

---

## Customizing `/tmp` Cleanup Behavior

### Change How Long Files Are Kept

Create an override file (don't edit the system default):

```bash
sudo nano /etc/tmpfiles.d/tmp.conf
```

```
# Keep /tmp files for 3 days instead of 10
q /tmp 1777 root root 3d

# Keep /var/tmp files for 14 days instead of 30
q /var/tmp 1777 root root 14d
```

Since this file has the same name (`tmp.conf`) as the one in `/usr/lib/tmpfiles.d/`, it **overrides** the system default completely.

### Exclude Specific Files or Directories

Create a new config file:

```bash
sudo nano /etc/tmpfiles.d/my-exclusions.conf
```

```
# Never delete files in /tmp/important_data
x /tmp/important_data

# Never delete files matching this pattern
x /tmp/keep-*
```

### Add a Custom Directory to Clean

```bash
sudo nano /etc/tmpfiles.d/app-temp.conf
```

```
# Clean /opt/myapp/temp — delete files older than 7 days
e /opt/myapp/temp - - - 7d
```

### Verify Your Configuration

After making changes, test them:

```bash
# Dry run — show what WOULD be cleaned (without actually deleting)
sudo systemd-tmpfiles --clean --dry-run

# Validate all config files for syntax errors
sudo systemd-tmpfiles --verify
```

---

## Inspecting Your System's Current Setup

Run these commands to understand exactly how your system handles `/tmp`:

```bash
# 1. Check if the systemd timer is active
systemctl status systemd-tmpfiles-clean.timer

# 2. See when the next cleanup will run
systemctl list-timers | grep tmpfiles

# 3. View the default /tmp rules
cat /usr/lib/tmpfiles.d/tmp.conf

# 4. Check for any overrides you (or your distro) have set
ls /etc/tmpfiles.d/

# 5. View ALL tmpfiles rules that apply
systemd-tmpfiles --cat-config

# 6. Check if the old-style tmpwatch/tmpreaper is also installed
ls /etc/cron.daily/tmpwatch 2>/dev/null && echo "tmpwatch found" || echo "No tmpwatch"
ls /etc/cron.daily/tmpreaper 2>/dev/null && echo "tmpreaper found" || echo "No tmpreaper"

# 7. See what's currently in /tmp and how old it is
ls -la --time=atime /tmp/
```

---

## Summary: The Full Picture

```
┌──────────────────────────────────────────────────────────────┐
│                    HOW /tmp GETS CLEANED                      │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  OLDER SYSTEMS (CentOS 6, Ubuntu 14.04 and earlier):         │
│                                                              │
│    crond → anacron → /etc/cron.daily/tmpwatch                │
│                          │                                   │
│                          ▼                                   │
│                  tmpwatch deletes files in /tmp               │
│                  older than 240 hours (10 days)               │
│                                                              │
│  ─────────────────────────────────────────────────────────── │
│                                                              │
│  MODERN SYSTEMS (CentOS 7+, Ubuntu 16.04+, Fedora, Arch):   │
│                                                              │
│    systemd-tmpfiles-clean.timer (daily + 15min after boot)   │
│                          │                                   │
│                          ▼                                   │
│    systemd-tmpfiles-clean.service                            │
│                          │                                   │
│                          ▼                                   │
│    Reads /usr/lib/tmpfiles.d/tmp.conf                        │
│    (overridden by /etc/tmpfiles.d/tmp.conf if present)       │
│                          │                                   │
│                          ▼                                   │
│    Deletes files in /tmp older than 10 days                  │
│    Deletes files in /var/tmp older than 30 days              │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

**Key takeaway:** You never have to manually clean `/tmp`. Linux handles it automatically through either cron (older systems) or systemd timers (modern systems). As an admin, you can customize the age threshold and exclusion rules, but the automation is built in from day one.

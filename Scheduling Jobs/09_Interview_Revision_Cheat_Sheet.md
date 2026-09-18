# Interview Revision Cheat Sheet: Linux Job Scheduling

> One file. Everything you need. Read this the night before your interview.

## Index

1. [Key Vocabulary](#1-key-vocabulary)
2. [The 5 Scheduling Tools at a Glance](#2-the-5-scheduling-tools-at-a-glance)
3. [Daemons & Services](#3-daemons--services)
4. [`at` — One-Time Jobs](#4-at--one-time-jobs)
5. [`batch` — Run When Idle](#5-batch--run-when-idle)
6. [`cron` — Recurring Jobs](#6-cron--recurring-jobs)
7. [Crontab Syntax — The Five Fields](#7-crontab-syntax--the-five-fields)
8. [Crontab Special Symbols](#8-crontab-special-symbols)
9. [Crontab Shortcut Strings](#9-crontab-shortcut-strings)
10. [Crontab Examples — Know These Cold](#10-crontab-examples--know-these-cold)
11. [Crontab Management Commands](#11-crontab-management-commands)
12. [Cron Environment Variables](#12-cron-environment-variables)
13. [User Crontab vs System Crontab](#13-user-crontab-vs-system-crontab)
14. [`anacron` — The Catch-Up Scheduler](#14-anacron--the-catch-up-scheduler)
15. [`/etc/anacrontab` Syntax](#15-etcanacrontab-syntax)
16. [`anacron` Commands](#16-anacron-commands)
17. [`cron` vs `anacron` — Differences](#17-cron-vs-anacron--differences)
18. [How `cron` and `anacron` Work Together](#18-how-cron-and-anacron-work-together)
19. [`systemd` Timers](#19-systemd-timers)
20. [`OnCalendar` Syntax](#20-oncalendar-syntax)
21. [`systemd` Timer Commands](#21-systemd-timer-commands)
22. [`cron` vs `systemd` Timers — Differences](#22-cron-vs-systemd-timers--differences)
23. [All Configuration Files — Master List](#23-all-configuration-files--master-list)
24. [All Log Files — Where to Look](#24-all-log-files--where-to-look)
25. [Access Control Files](#25-access-control-files)
26. [Output Handling — Where Does Script Output Go?](#26-output-handling--where-does-script-output-go)
27. [Output Redirection Patterns](#27-output-redirection-patterns)
28. [The `/tmp` Cleanup System](#28-the-tmp-cleanup-system)
29. [System Cron Directories — What Linux Schedules Itself](#29-system-cron-directories--what-linux-schedules-itself)
30. [Troubleshooting — The 8 Common Pitfalls](#30-troubleshooting--the-8-common-pitfalls)
31. [Troubleshooting — Quick Diagnosis Guide](#31-troubleshooting--quick-diagnosis-guide)
32. [Preventing Overlapping Jobs — `flock`](#32-preventing-overlapping-jobs--flock)
33. [All Scheduling Commands — Master Reference](#33-all-scheduling-commands--master-reference)
34. [General Linux Commands Used with Scheduling](#34-general-linux-commands-used-with-scheduling)
35. [Best Practices — The Golden Rules](#35-best-practices--the-golden-rules)
36. [When to Use vs. Avoid `@reboot`](#36-when-to-use-vs-avoid-reboot)
37. [Incident Response — Failed Critical Cron Job](#37-incident-response--failed-critical-cron-job)
38. [Interview Tips — How to Deliver Answers](#38-interview-tips--how-to-deliver-answers)
39. [Interview Power Answers — One-Liners](#39-interview-power-answers--one-liners)

---

## 1. Key Vocabulary

| Term                | Definition                                                         |
| ------------------- | ------------------------------------------------------------------ |
| **Daemon**          | Background process that runs continuously (e.g., `crond`, `atd`). Pronounced "dee-mon" |
| **Chronos**         | Greek word meaning "time" — origin of the name `cron`              |
| **Job/Task**        | A command or script scheduled to run automatically                 |
| **Crontab**         | "Cron table" — the file where cron jobs are defined                |
| **Shebang**         | `#!/bin/bash` — first line of a script, tells OS which shell to use |
| **run-parts**       | Utility that executes all scripts inside a directory. `--report` flag prints each script name as it runs |
| **MTA**             | Mail Transfer Agent (e.g., `postfix`) — delivers cron output as email |
| **Sticky bit**      | Permission bit (`1777`) — users can only delete their own files in a shared directory |
| **Idempotent**      | A job that produces the same result even if run multiple times — critical for cron jobs that may overlap or retry |
| **logwatch**        | A tool that summarizes system activity into daily reports, typically run via a scheduled cron job |

### Why Schedule Jobs? — Common Use Cases

| Use Case                    | Example                                                    |
| --------------------------- | ---------------------------------------------------------- |
| Automated backups           | Back up database every night at 2 AM                       |
| Scheduled reports           | Generate sales PDF every Sunday at 11 PM                   |
| Disk cleanup                | Delete temp files older than 30 days to prevent disk full  |
| Health monitoring           | Check if website is up every 5 minutes, email alert if down |
| Security patching           | Apply OS patches during maintenance window (Saturday 3 AM) |
| Log management              | Rotate and compress logs daily via `logrotate`             |

### Four Types of Execution Time

| Type               | Tool      | Example                                         |
| ------------------ | --------- | ----------------------------------------------- |
| Specific time      | `at`      | "Run at 11 PM tonight"                          |
| Recurring time     | `cron`    | "Run every day at 2 AM"                         |
| Relative time      | `at`      | "Run 30 minutes from now"                       |
| Condition-based    | `batch`   | "Run when system load drops below 0.8"          |

---

## 2. The 5 Scheduling Tools at a Glance

| Tool            | Type       | Frequency     | Daemon?    | Best For                          |
| --------------- | ---------- | ------------- | ---------- | --------------------------------- |
| `at`            | One-time   | Specific time | `atd`      | "Restart at 11 PM tonight"        |
| `batch`         | One-time   | When idle     | `atd`      | "Run when CPU is free"            |
| `cron`          | Recurring  | Min to yearly | `crond`    | "Every day at 2 AM"               |
| `anacron`       | Recurring  | Daily+        | Not a daemon | "Run daily, even if machine was off" |
| `systemd.timer` | Recurring  | Sec to yearly | `systemd`  | "Modern, logged, reliable"        |

---

## 3. Daemons & Services

```bash
# Check status
systemctl status atd          # at/batch daemon
systemctl status crond        # cron daemon (CentOS/RHEL)
systemctl status cron         # cron daemon (Ubuntu/Debian)

# Start and enable
sudo systemctl start atd && sudo systemctl enable atd
sudo systemctl start crond && sudo systemctl enable crond
```

---

## 4. `at` — One-Time Jobs

```bash
# Interactive — type commands, then press Ctrl+D to save
at 2:00 AM                     # 12-hour format
at 15:00                       # 24-hour format (same as 3:00 PM)
at now + 30 minutes            # Relative time
at now + 2 hours               # Relative — hours
at now + 3 days                # Relative — days
at now + 1 week                # Relative — weeks
at 4pm + 3 days                # Combine time + offset
at 3:00 PM tomorrow            # Tomorrow keyword
at midnight                    # Keyword (12:00 AM)
at noon                        # Keyword (12:00 PM)
at teatime                     # Keyword (4:00 PM — yes, it's real)
at 10:00 AM 07/04/2027         # Date in MM/DD/YYYY
at 9:00 AM December 25         # Date by month name
at 10:00 AM Jul 4              # Abbreviated month name

# Non-interactive methods
echo "command" | at 2:00 AM    # Pipe
at 2:00 AM <<< "/path/script.sh"  # Here-string
at 2:00 AM -f /path/script.sh  # Read from file

# Job management
atq                            # View pending jobs (same as at -l)
at -c JOB_NUMBER               # View full job details (script + env)
atrm JOB_NUMBER                # Remove a job (same as at -d)
at -v 2:00 AM                  # Show the time the job will run
```

**`atq` output format:**

```
3   Thu Sep 18 14:00:00 2026  a  akshay
│        │                    │    │
│        │                    │    └── Username
│        │                    └── Queue letter (a=at, b=batch)
│        └── Scheduled date/time
└── Job number
```

---

## 5. `batch` — Run When Idle

```bash
batch                          # Interactive — same as at, but no time needed
echo "command" | batch         # Non-interactive
```

- Runs when system load drops below **0.8** (on a single-CPU system — this threshold is configurable)
- Uses the same queue as `at` — shows in `atq` with queue letter **`b`** (vs `a` for `at`)

---

## 6. `cron` — Recurring Jobs

Named after **Chronos** (Greek for "time"). `crond` wakes up **every minute**, checks all crontabs, runs due jobs, goes back to sleep.

---

## 7. Crontab Syntax — The Five Fields

```
┌───────────── Minute        (0-59)
│ ┌─────────── Hour          (0-23)
│ │ ┌───────── Day of Month  (1-31)
│ │ │ ┌─────── Month         (1-12)
│ │ │ │ ┌───── Day of Week   (0-6, Sun=0)
│ │ │ │ │
* * * * * command
```

**System crontab (`/etc/crontab`) has a 6th field — the USERNAME:**

```
* * * * * USERNAME command
```

---

## 8. Crontab Special Symbols

| Symbol | Meaning         | Example            | Result                         |
| ------ | --------------- | ------------------ | ------------------------------ |
| `*`    | Every value     | `* * * * *`        | Every minute                   |
| `,`    | List            | `0 9,12,18 * * *`  | 9 AM, 12 PM, 6 PM             |
| `-`    | Range           | `0 9-17 * * *`     | Every hour 9 AM–5 PM          |
| `/`    | Step            | `*/15 * * * *`     | Every 15 minutes               |

---

## 9. Crontab Shortcut Strings

| Shortcut     | Equivalent    | Meaning                     |
| ------------ | ------------- | --------------------------- |
| `@reboot`    | —             | Run once at boot            |
| `@yearly`    | `0 0 1 1 *`  | Jan 1st at midnight         |
| `@annually`  | `0 0 1 1 *`  | Same as `@yearly`           |
| `@monthly`   | `0 0 1 * *`  | 1st of month at midnight    |
| `@weekly`    | `0 0 * * 0`  | Sunday at midnight          |
| `@daily`     | `0 0 * * *`  | Every day at midnight       |
| `@midnight`  | `0 0 * * *`  | Same as `@daily`            |
| `@hourly`    | `0 * * * *`  | Every hour at :00           |

---

## 10. Crontab Examples — Know These Cold

| Schedule                              | Expression               |
| ------------------------------------- | ------------------------ |
| Every day at 2:00 AM                  | `0 2 * * *`              |
| Every Monday at 9:00 AM              | `0 9 * * 1`              |
| Every 15 minutes                      | `*/15 * * * *`           |
| Weekdays at 6:00 PM                  | `0 18 * * 1-5`           |
| 1st and 15th of month at midnight    | `0 0 1,15 * *`           |
| Every 5 min during business hours     | `*/5 9-17 * * *`         |
| 9 AM, 12 PM, and 6 PM daily          | `0 9,12,18 * * *`        |
| Business hours only, weekdays only    | `*/15 9-17 * * 1-5`      |
| Once a year (Jan 1st)                | `0 0 1 1 *`              |

---

## 11. Crontab Management Commands

```bash
crontab -e                     # Edit your crontab
crontab -l                     # List your crontab
crontab -r                     # Remove your crontab (NO confirmation!)
crontab -ri                    # Remove with confirmation prompt
sudo crontab -u john -e        # Edit another user's crontab
sudo crontab -u john -l        # View another user's crontab
```

---

## 12. Cron Environment Variables

Set these at the **top of your crontab** — they apply to all jobs below:

```
SHELL=/bin/bash
PATH=/usr/local/bin:/usr/bin:/bin:/home/akshay/.local/bin
MAILTO=akshay@company.com
HOME=/home/akshay
CRON_TZ=America/New_York
```

| Variable    | Purpose                                                      | Default                |
| ----------- | ------------------------------------------------------------ | ---------------------- |
| `SHELL`     | Which shell runs the job                                     | `/bin/sh`              |
| `PATH`      | Where to find commands (cron's default is very short!)       | `/usr/bin:/bin`         |
| `MAILTO`    | Who receives job output as email. `""` to disable            | Crontab owner          |
| `HOME`      | Working directory for the job                                | User's home directory  |
| `CRON_TZ`   | Timezone for the schedule (overrides system timezone)        | System timezone        |

**`CRON_TZ` example** — server is UTC but you want 9 AM New York time:

```
CRON_TZ=America/New_York
0 9 * * * /home/akshay/scripts/morning_report.sh
```

**Timezone commands:**

```bash
timedatectl                                    # Check system timezone
timedatectl list-timezones | grep America      # List available timezones
cat /etc/timezone                              # Ubuntu
ls -la /etc/localtime                          # CentOS — shows symlink to timezone file
```

### Cron vs Anacron Environment Variables — Comparison

| Variable             | Available in `cron`? | Available in `anacron`? | Notes                              |
| -------------------- | -------------------- | ----------------------- | ---------------------------------- |
| `SHELL`              | Yes                  | Yes                     | Same in both                       |
| `PATH`               | Yes                  | Yes                     | Anacron has a richer default PATH  |
| `HOME`               | Yes                  | Yes                     | Same in both                       |
| `MAILTO`             | Yes                  | Yes                     | Same in both                       |
| `LOGNAME`            | No                   | Yes                     | Anacron only                       |
| `CRON_TZ`            | Yes                  | No                      | Cron only — set timezone per job   |
| `RANDOM_DELAY`       | No                   | Yes                     | Anacron only — random extra delay  |
| `START_HOURS_RANGE`  | No                   | Yes                     | Anacron only — restrict run window |

---

## 13. User Crontab vs System Crontab

| Feature           | User Crontab               | System Crontab (`/etc/crontab`)  |
| ----------------- | -------------------------- | -------------------------------- |
| **Edited with**   | `crontab -e`               | `sudo nano /etc/crontab`        |
| **Stored at**     | `/var/spool/cron/` (CentOS) or `/var/spool/cron/crontabs/` (Ubuntu) | `/etc/crontab` |
| **Runs as**       | The user who created it    | User specified in 6th field      |
| **Fields**        | 5 + command                | 5 + **username** + command       |
| **Extra locations** | —                        | `/etc/cron.d/` directory         |

---

## 14. `anacron` — The Catch-Up Scheduler

- Stands for **"anachronistic cron"**
- **NOT a daemon** — runs once, checks what's overdue, executes it, exits
- Triggered at **boot** and periodically by **cron** (via `/etc/cron.d/anacron`)
- Minimum interval: **1 day** (cannot do minutes/hours)
- Tracks last run in **`/var/spool/anacron/<job-id>`** as `YYYYMMDD`
- Config: **`/etc/anacrontab`**

---

## 15. `/etc/anacrontab` Syntax

```
# /etc/anacrontab: configuration file for anacron

# Environment variables
SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin
HOME=/root
LOGNAME=root
MAILTO=root

# the maximal random delay added to the base delay of the jobs
RANDOM_DELAY=45

# the jobs will be started during the following hours only
START_HOURS_RANGE=3-22

# Jobs: PERIOD  DELAY  JOB-ID  COMMAND
1        5      cron.daily     run-parts --report /etc/cron.daily
7        10     cron.weekly    run-parts --report /etc/cron.weekly
@monthly 15     cron.monthly   run-parts --report /etc/cron.monthly
```

### Anacrontab Environment Variables

| Variable             | Purpose                                                              | Example          |
| -------------------- | -------------------------------------------------------------------- | ---------------- |
| `SHELL`              | Which shell runs the commands                                        | `/bin/sh`        |
| `PATH`               | Where to find commands                                               | `/usr/local/bin:/usr/bin:/bin` |
| `HOME`               | Working directory for jobs                                           | `/root`          |
| `LOGNAME`            | Username logged in system records                                    | `root`           |
| `MAILTO`             | Who receives output via email. `""` to disable                       | `root`           |
| `RANDOM_DELAY`       | Max **random minutes** added to base DELAY (prevents all machines running at once) | `45` |
| `START_HOURS_RANGE`  | Hours during which jobs are **allowed to start** (jobs wait if outside window) | `3-22` (3 AM–10 PM) |

**How delays add up:**

```
Total wait = Base DELAY + Random(0 to RANDOM_DELAY)
             ...but only within START_HOURS_RANGE

Example: cron.daily (DELAY=5, RANDOM_DELAY=45, START_HOURS_RANGE=3-22)
  Minimum wait: 5 + 0  = 5 minutes   (if within 3 AM–10 PM)
  Maximum wait: 5 + 45 = 50 minutes   (if within 3 AM–10 PM)
  If anacron starts at 11 PM → waits until 3 AM to run
```

### Anacrontab Job Fields

| Field      | Meaning                                        | Values                         |
| ---------- | ---------------------------------------------- | ------------------------------ |
| `PERIOD`   | How often (in days)                            | `1`, `7`, `30`, `@monthly`, `@yearly` |
| `DELAY`    | Base minutes to wait after anacron starts       | `5`, `10`, `15`                |
| `JOB-ID`   | Unique name, **no spaces** (creates timestamp file) | `cron.daily`, `my-backup`  |
| `COMMAND`  | What to run                                    | Any command/script             |

**`run-parts --report`** — the `--report` flag prints the name of each script as it runs (useful for logging). `run-parts --test` shows what scripts _would_ run without executing them.

**Last-run timestamps:**

```bash
cat /var/spool/anacron/cron.daily     # Output: 20260918
cat /var/spool/anacron/cron.weekly    # Output: 20260914
cat /var/spool/anacron/cron.monthly   # Output: 20260915
```

---

## 16. `anacron` Commands

```bash
sudo anacron -T          # Test anacrontab for syntax errors
sudo anacron -n          # Run now, skip delay
sudo anacron -f          # Force run, ignore timestamps
sudo anacron -fn         # Force + immediate (most aggressive)
sudo anacron -u          # Update timestamps without running
sudo anacron -d          # Debug mode (foreground, verbose)
```

---

## 17. `cron` vs `anacron` — Differences

| Feature              | `cron`                   | `anacron`                     |
| -------------------- | ------------------------ | ----------------------------- |
| Min interval         | 1 minute                 | 1 day                         |
| Runs missed jobs     | No                       | Yes                           |
| Is a daemon          | Yes (runs 24/7)          | No (runs once and exits)      |
| Precision            | Exact time               | Approximate (sometime today)  |
| User crontabs        | Yes                      | No (system-wide only)         |
| Runs as              | The owning user          | root (system-wide only)       |
| Output handling      | Mail to user or `MAILTO` | Mail to `MAILTO` or root      |
| Best for             | Servers (always-on)      | Laptops/desktops (on/off)     |
| Config               | `crontab -e`             | `/etc/anacrontab`             |
| Tracking             | None                     | `/var/spool/anacron/`         |

---

## 18. How `cron` and `anacron` Work Together

```
BOOT → crond starts (daemon)    +    anacron runs (one-shot)
         │                                │
         ▼                                ▼
  Runs user/hourly jobs           Checks /var/spool/anacron/
         │                        Runs overdue daily/weekly/monthly
         ▼
  Also triggers anacron daily
  via /etc/cron.d/anacron
```

The trigger entry (in `/etc/cron.d/anacron`):

```
30 7 * * * root test -x /etc/init.d/anacron && /usr/sbin/invoke-rc.d anacron start
```

---

## 19. `systemd` Timers

**Two files required (same base name):**

```ini
# /etc/systemd/system/my-job.service  (WHAT to run)
[Unit]
Description=My Scheduled Job

[Service]
Type=oneshot
ExecStart=/path/to/script.sh

# /etc/systemd/system/my-job.timer  (WHEN to run)
[Unit]
Description=Run my job daily at 2 AM

[Timer]
OnCalendar=*-*-* 02:00:00
Persistent=true

[Install]
WantedBy=timers.target
```

**Supported distros:** Ubuntu 16.04+, CentOS/RHEL 7+, Fedora, Arch Linux — any distro using `systemd` as init.

| Key Directive      | Meaning                                              |
| ------------------ | ---------------------------------------------------- |
| `Type=oneshot`     | Run once and exit (not a long-running service)       |
| `User=`            | Run as a specific user (default: root)               |
| `OnCalendar=`      | Calendar-based schedule                              |
| `Persistent=true`  | Catch up missed runs (like anacron)                  |
| `RandomizedDelaySec=` | Add random jitter to prevent thundering herd      |
| `OnBootSec=`       | Run X time after boot                                |
| `OnStartupSec=`    | Run X time after systemd itself started              |
| `OnActiveSec=`     | Run X time after the timer unit is activated         |
| `OnUnitActiveSec=` | Repeat every X time after last activation            |
| `WantedBy=timers.target` | Activate when system reaches timer target      |

**Combined monotonic timer example:**

```ini
[Timer]
OnBootSec=2min
OnUnitActiveSec=10min
```

This means: "Run 2 minutes after boot, then repeat every 10 minutes after that."

---

## 20. `OnCalendar` Syntax

Format: `DayOfWeek Year-Month-Day Hour:Minute:Second`

| Schedule                | Expression                  |
| ----------------------- | --------------------------- |
| Every day at 2 AM       | `*-*-* 02:00:00`            |
| Every Monday at 9 AM    | `Mon *-*-* 09:00:00`       |
| 1st of every month      | `*-*-01 00:00:00`           |
| Weekdays at 6 PM        | `Mon..Fri *-*-* 18:00:00`  |
| Every hour              | `*-*-* *:00:00`             |
| Every 15 minutes        | `*-*-* *:00/15:00`          |
| Every 3 hours           | `*-*-* 00/3:00:00`          |
| Jan 1st each year       | `*-01-01 00:00:00`          |

```bash
# Test your expression
systemd-analyze calendar "*-*-* 02:00:00"
systemd-analyze calendar "Mon..Fri *-*-* 09:00:00"
```

---

## 21. `systemd` Timer Commands

```bash
sudo systemctl daemon-reload                  # Reload after creating/editing files
sudo systemctl enable --now my-job.timer      # Enable + start in one command
sudo systemctl start my-job.timer             # Start timer
sudo systemctl stop my-job.timer              # Stop timer
sudo systemctl disable my-job.timer           # Disable from boot
sudo systemctl status my-job.timer            # Check timer status
sudo systemctl status my-job.service          # Check last run result
sudo systemctl start my-job.service           # Manually trigger (for testing)
systemctl list-timers                         # List all active timers
systemctl list-timers --all                   # Include inactive timers
journalctl -u my-job.service                  # View logs
journalctl -u my-job.service -n 20            # Last 20 lines
journalctl -u my-job.service --since today   # Today's logs only
journalctl -u my-job.service -f              # Follow logs in real-time
systemd-analyze verify my-job.service        # Validate service file syntax
```

**`systemctl list-timers` output columns:**

```
NEXT                        LEFT     LAST                        PASSED   UNIT              ACTIVATES
Fri 2026-09-19 02:00:00 UTC 7h left  Thu 2026-09-18 02:00:00 UTC 17h ago my-job.timer      my-job.service
│                           │        │                           │        │                 │
│                           │        │                           │        │                 └── Service it triggers
│                           │        │                           │        └── Timer unit name
│                           │        │                           └── How long since last run
│                           │        └── Last time it ran
│                           └── Time until next run
└── Next scheduled run
```

**`systemd-analyze calendar` sample output:**

```bash
$ systemd-analyze calendar "Mon..Fri *-*-* 09:00:00"
  Original form: Mon..Fri *-*-* 09:00:00
Normalized form: Mon..Fri *-*-* 09:00:00
    Next elapse: Mon 2026-09-21 09:00:00 UTC
```

---

## 22. `cron` vs `systemd` Timers — Differences

| Feature              | `cron`                      | `systemd` Timer                    |
| -------------------- | --------------------------- | ---------------------------------- |
| Setup                | 1 line in crontab           | 2 files (.service + .timer)        |
| Learning curve       | Low (one line)              | Medium (two files)                 |
| Logging              | Scattered in syslog         | Centralized (`journalctl`)         |
| Missed jobs          | No (needs anacron)          | Yes (`Persistent=true`)           |
| Dependencies         | None                        | Can require network, mounts, etc.  |
| Error handling       | No built-in                 | Automatic restarts, failure detection |
| Status checking      | Hard (parse logs)           | Easy (`systemctl status`)          |
| Environment          | Minimal, bare               | Full service environment with control |
| Resource limits      | None                        | CPU/memory via cgroups             |
| Management           | Separate from system        | Managed alongside all services     |
| Min interval         | 1 minute                    | 1 second                           |

**Bottom line:** `cron` is simpler for quick, basic tasks. `systemd` timers are better for production systems where you need reliability, logging, and control.

---

## 23. All Configuration Files — Master List

| File / Directory                   | Purpose                                           |
| ---------------------------------- | ------------------------------------------------- |
| `crontab -e`                       | Edit current user's crontab                       |
| `/var/spool/cron/`                 | User crontab storage (CentOS)                     |
| `/var/spool/cron/crontabs/`        | User crontab storage (Ubuntu)                     |
| `/etc/crontab`                     | System crontab (has USERNAME field)               |
| `/etc/cron.d/`                     | Drop-in system cron files                         |
| `/etc/cron.hourly/`               | Scripts run every hour                            |
| `/etc/cron.daily/`                | Scripts run once per day                          |
| `/etc/cron.weekly/`               | Scripts run once per week                         |
| `/etc/cron.monthly/`              | Scripts run once per month                        |
| `/etc/anacrontab`                  | Anacron configuration                             |
| `/var/spool/anacron/`              | Anacron last-run timestamps (`YYYYMMDD`)          |
| `/etc/cron.d/anacron`              | Cron entry that triggers anacron daily            |
| `/etc/systemd/system/*.timer`      | Custom systemd timer files                        |
| `/etc/systemd/system/*.service`    | Custom systemd service files                      |
| `/usr/lib/tmpfiles.d/tmp.conf`     | Default `/tmp` cleanup rules                      |
| `/etc/tmpfiles.d/`                 | Admin overrides for tmpfiles (same-name file replaces system default) |
| `/run/tmpfiles.d/`                 | Runtime overrides generated on the fly by system processes |
| `/etc/tmpreaper.conf`              | tmpreaper config (Ubuntu, older systems)          |
| `/etc/logrotate.conf`              | Main logrotate configuration                      |
| `/etc/logrotate.d/`               | Per-application logrotate drop-in configs          |

---

## 24. All Log Files — Where to Look

| What                | Ubuntu/Debian                   | CentOS/RHEL              | systemd                         |
| ------------------- | ------------------------------- | ------------------------ | ------------------------------- |
| **cron jobs**       | `grep CRON /var/log/syslog`     | `cat /var/log/cron`      | `journalctl -u cron`            |
| **anacron**         | `grep anacron /var/log/syslog`  | `grep anacron /var/log/cron` | `journalctl \| grep anacron` |
| **at jobs**         | `grep atd /var/log/syslog`      | `grep atd /var/log/cron` | `journalctl -u atd`            |
| **systemd timers**  | —                               | —                        | `journalctl -u my-job.service`  |
| **tmpfiles cleanup**| —                               | —                        | `journalctl -u systemd-tmpfiles-clean.service` |

**Real-time monitoring:**

```bash
sudo tail -f /var/log/syslog | grep CRON     # Ubuntu
sudo tail -f /var/log/cron                     # CentOS
journalctl -u cron -f                          # systemd
```

---

## 25. Access Control Files

| File              | Controls access to | Rule                                       |
| ----------------- | ------------------ | ------------------------------------------ |
| `/etc/cron.allow` | `cron`             | If exists, ONLY listed users can use cron  |
| `/etc/cron.deny`  | `cron`             | If allow doesn't exist, listed users blocked |
| `/etc/at.allow`   | `at`               | If exists, ONLY listed users can use at    |
| `/etc/at.deny`    | `at`               | If allow doesn't exist, listed users blocked |

**Priority:** `.allow` always takes precedence over `.deny`. If neither exists, only `root` can use the tool.

---

## 26. Output Handling — Where Does Script Output Go?

| Scenario                           | What Happens                                      |
| ---------------------------------- | ------------------------------------------------- |
| MTA installed (postfix/sendmail)   | Output is emailed to the user (or `MAILTO`)       |
| No MTA installed (most modern)     | Output is **silently lost**                       |
| `MAILTO=admin@company.com`         | Output emailed to that address                    |
| `MAILTO=""`                        | Output is discarded (no mail sent)                |
| `>> /path/log 2>&1`               | Output saved to a log file (recommended)          |
| `> /dev/null 2>&1`                | Output discarded to the black hole                |

**"No MTA installed" diagnostic message** — if you see this in syslog, it means cron tried to mail output but couldn't:

```
CRON: (CRON) info (No MTA installed, discarding output)
```

**Setting up local mail (if needed):**

```bash
# Install
sudo apt install postfix mailutils    # Ubuntu — choose "Local only" during setup
sudo yum install postfix mailx        # CentOS

# Start and enable
sudo systemctl start postfix && sudo systemctl enable postfix

# Read mail
mail                                  # Read current user's mail
sudo mail                             # Read root's mail (cron jobs running as root)
cat /var/mail/username                # Read mail file directly
```

**Rule of thumb:** Always redirect output yourself. Never rely on mail.

---

## 27. Output Redirection Patterns

```bash
# Append stdout AND stderr to same log
command >> /var/log/job.log 2>&1

# Separate stdout and stderr
command >> /var/log/job.log 2>> /var/log/job_errors.log

# Discard everything
command > /dev/null 2>&1

# Overwrite log each run (use > instead of >>)
command > /var/log/job.log 2>&1
```

| Symbol   | Meaning                                |
| -------- | -------------------------------------- |
| `>`      | Redirect stdout (overwrite)            |
| `>>`     | Redirect stdout (append)               |
| `2>`     | Redirect stderr (overwrite)            |
| `2>>`    | Redirect stderr (append)               |
| `2>&1`   | Send stderr to the same place as stdout |
| `/dev/null` | The "black hole" — discards data    |

---

## 28. The `/tmp` Cleanup System

**If `/tmp` fills up:** Programs crash, databases can't create temp tables, logging stops, login may fail.

| System Type     | Tool                        | Config                               | Default Age |
| --------------- | --------------------------- | ------------------------------------ | ----------- |
| Older (CentOS 6)| `tmpwatch`                  | `/etc/cron.daily/tmpwatch`           | 240h (10d)  |
| Older (Ubuntu)  | `tmpreaper`                 | `/etc/tmpreaper.conf`                | 7d          |
| Modern (systemd)| `systemd-tmpfiles-clean`    | `/usr/lib/tmpfiles.d/tmp.conf`       | `/tmp`: 10d, `/var/tmp`: 30d |

```bash
# Check your system's method
systemctl status systemd-tmpfiles-clean.timer    # Modern
ls /etc/cron.daily/tmpwatch 2>/dev/null           # Old CentOS
ls /etc/cron.daily/tmpreaper 2>/dev/null          # Old Ubuntu
rpm -q tmpwatch                                   # Check if tmpwatch installed (CentOS)
dpkg -l | grep tmpreaper                          # Check if tmpreaper installed (Ubuntu)
```

**`/tmp` vs `/var/tmp`:**

| Directory  | Cleanup | Survives Reboot? | Use For                     |
| ---------- | ------- | ---------------- | --------------------------- |
| `/tmp`     | 10 days | No (may clear)   | Short-lived temp files      |
| `/var/tmp` | 30 days | Yes              | Longer-lived temp files     |

### Modern: `systemd-tmpfiles-clean.timer` Internals

```ini
[Timer]
OnBootSec=15min          # First run: 15 minutes after boot
OnUnitActiveSec=1d       # Then repeat: once every day
# No [Install] section — always enabled by default, no need for systemctl enable
```

- **At boot:** `systemd-tmpfiles --create --remove` — creates directories and sets permissions
- **Daily:** `systemd-tmpfiles --clean` — scans and deletes old files
- **File age check:** uses atime, mtime, AND ctime — file must be older than threshold on ALL THREE to be deleted

### `tmpfiles.d` Configuration Line Format

```
TYPE    PATH    MODE    USER    GROUP    AGE    ARGUMENT
```

Example: `q /tmp 1777 root root 10d`

| Type Code | Meaning                                                 |
| --------- | ------------------------------------------------------- |
| `d`       | Create a directory if it doesn't exist                  |
| `D`       | Create directory; remove ALL contents if it exists      |
| `q`       | Create directory (with subvolume support) + clean old files |
| `r`       | Remove a file or directory                              |
| `e`       | Clean up (remove old files) in an existing directory    |
| `x`       | Exclude a path from cleanup                             |
| `X`       | Exclude a path AND all its contents from cleanup        |

**Age suffixes:** `s` (seconds), `m` (minutes), `h` (hours), `d` (days), `w` (weeks)

### `/etc/tmpfiles.d/` Override Rule

A file in `/etc/tmpfiles.d/` with the **same name** as one in `/usr/lib/tmpfiles.d/` **completely replaces** the system default. This lets you customize without editing system files.

```bash
# Change /tmp retention to 3 days (create override with same filename)
sudo nano /etc/tmpfiles.d/tmp.conf
```

```
q /tmp 1777 root root 3d
q /var/tmp 1777 root root 14d
```

```bash
# Exclude a specific path from cleanup
sudo nano /etc/tmpfiles.d/my-exclusions.conf
```

```
x /tmp/important_data
x /tmp/keep-*
```

```bash
# Add a custom directory to cleanup
sudo nano /etc/tmpfiles.d/app-temp.conf
```

```
e /opt/myapp/temp - - - 7d
```

### Older: `tmpreaper.conf` Settings

```
TMPREAPER_TIME=7d                              # Delete files older than 7 days
TMPREAPER_PROTECT_EXTRA='/tmp/.X11-unix /tmp/.ICE-unix'  # Never delete these
TMPREAPER_DIRS='/tmp/.'                        # Directories to clean
TMPREAPER_DELAY='256'                          # Random delay in seconds
```

### Older: `tmpwatch` Exclusion Flags

```bash
/usr/sbin/tmpwatch -x /tmp/.X11-unix -x /tmp/.font-unix 240 /tmp
# -x = exclude (protects X11 display sockets and font sockets)
# 240 = delete files older than 240 hours (10 days)
```

### Cron Chain for `/tmp` Cleanup (Older Systems)

```
crond → anacron → run-parts /etc/cron.daily/ → tmpwatch/tmpreaper → deletes old files
```

### Useful Inspection Commands

```bash
ls -la --time=atime /tmp/                      # View file access times in /tmp
systemd-tmpfiles --cat-config                  # View ALL active tmpfiles rules
sudo systemd-tmpfiles --clean --dry-run        # Dry-run — show what WOULD be deleted
sudo systemd-tmpfiles --verify                 # Validate config files for errors
```

---

## 29. System Cron Directories — What Linux Schedules Itself

```
/etc/cron.daily/    → logrotate, mlocate, apt/yum updates, tmpwatch
/etc/cron.weekly/   → man-db rebuild, fstrim (SSD)
/etc/cron.monthly/  → (varies by distro)
/etc/cron.hourly/   → (usually empty or 0anacron trigger)
```

| Task               | Where It Lives                  | What It Does                             |
| ------------------ | ------------------------------- | ---------------------------------------- |
| Log rotation       | `/etc/cron.daily/logrotate`     | Compress/archive old logs                |
| File database      | `/etc/cron.daily/mlocate`       | Update `locate` database                 |
| Package updates    | `/etc/cron.daily/apt-compat`    | Check for security updates               |
| Temp cleanup       | Systemd timer or `cron.daily`   | Delete old files from `/tmp`             |

---

## 30. Troubleshooting — The 10 Common Pitfalls

| #  | Pitfall               | Fix                                                     |
| -- | --------------------- | ------------------------------------------------------- |
| 1  | **PATH problem**      | Use absolute paths (`/usr/bin/python3` not `python3`)   |
| 2  | **No environment**    | Source `~/.bashrc`, `~/.bash_profile`, or `~/.profile` in script, or set vars explicitly |
| 3  | **Permissions**       | `chmod +x script.sh`, check file/dir ownership          |
| 4  | **No output capture** | Add `>> /path/log 2>&1` to every crontab line           |
| 5  | **Bad syntax**        | 5 fields only; no seconds; Sunday = 0; trailing newline |
| 6  | **Relative paths**    | Use absolute paths or `cd /dir || exit 1` in script     |
| 7  | **Job overlap**       | Use `flock -n /tmp/job.lock script.sh`                  |
| 8  | **Missing newline**   | Crontab MUST end with a blank line or last job is ignored |
| 9  | **`sudo` in cron**    | Cron has no password prompt — use `sudo crontab -e` to schedule as root instead |
| 10 | **Spaces in paths**   | Quote paths: `"/home/user/my scripts/backup.sh"` — unquoted spaces break the command |

**Find command paths:** `which python3`, `which node`, `which aws`

**See cron's environment:** `* * * * * env > /tmp/cron_env.txt` (remove after checking!)

**Compare cron vs terminal environment:**

```bash
# Step 1: Capture cron's environment (temporary cron job)
* * * * * env > /tmp/cron_env.txt

# Step 2: Capture terminal's environment
env > /tmp/terminal_env.txt

# Step 3: Compare the two
diff /tmp/cron_env.txt /tmp/terminal_env.txt
```

**Validate cron expressions online:** Use [crontab.guru](https://crontab.guru) — type an expression and see in plain English what it means.

**Fix output directory permissions:**

```bash
sudo mkdir -p /var/log/myapp
sudo chown akshay:akshay /var/log/myapp
```

---

## 31. Troubleshooting — Quick Diagnosis Guide

```
Job doesn't run at all?
  → systemctl status crond          (is daemon running?)
  → crontab -l                      (is job listed?)
  → cat /etc/cron.allow             (is user allowed?)
  → Check for trailing newline

Job runs but fails?
  → Check log: cat /path/to/your/log
  → which command_name              (find absolute path)
  → diff /tmp/cron_env.txt /tmp/terminal_env.txt  (compare environments)
  → ls -la script.sh                (check permissions)

Job runs but does nothing?
  → Is the script executable?       (chmod +x)
  → Does it have the shebang line?  (#!/bin/bash)
  → Is the working directory correct?
  → Are relative paths the problem?

Job seems to run twice?
  → crontab -l + cat /etc/crontab   (duplicate entries?)
  → ls /etc/cron.daily/             (also in drop-in dir?)
  → Use flock to prevent overlap
```

### The Ultimate Debugging Template

When nothing else works, wrap your cron job in this script:

```bash
#!/bin/bash
LOG="/var/log/my_cron_debug.log"

echo "====================================" >> "$LOG"
echo "Job started at: $(date)" >> "$LOG"
echo "Running as user: $(whoami)" >> "$LOG"
echo "Working directory: $(pwd)" >> "$LOG"
echo "PATH: $PATH" >> "$LOG"
echo "====================================" >> "$LOG"

# Your actual commands go here
/usr/bin/python3 /home/akshay/scripts/report.py >> "$LOG" 2>&1
EXIT_CODE=$?

echo "Job finished at: $(date) with exit code: $EXIT_CODE" >> "$LOG"
echo "" >> "$LOG"
```

This tells you: when it ran, who it ran as, where from, what PATH it had, and whether it succeeded (exit code 0) or failed (non-zero).

---

## 32. Preventing Overlapping Jobs — `flock`

```bash
# In crontab — skip if previous run is still going
*/5 * * * * /usr/bin/flock -n /tmp/myjob.lock /path/to/script.sh
```

| Flag | Meaning                            |
| ---- | ---------------------------------- |
| `-n` | Non-blocking — exit immediately if locked |
| `-w 30` | Wait up to 30 seconds for lock  |
| `-x` | Exclusive lock (default)           |
| `-s` | Shared lock (allow multiple readers) |

---

## 33. All Scheduling Commands — Master Reference

### `at` / `batch`

| Command                              | Purpose                              |
| ------------------------------------ | ------------------------------------ |
| `at TIME`                            | Schedule one-time job interactively  |
| `at TIME -f script.sh`              | Schedule from file                   |
| `echo "cmd" \| at TIME`             | Schedule via pipe                    |
| `at -m TIME`                         | Send mail even if no output          |
| `at -M TIME`                         | Never send mail                      |
| `at -v TIME`                         | Show time the job will run           |
| `atq`                                | List pending jobs (same as `at -l`)  |
| `at -c JOB_NUM`                     | View full job details (script content + env) |
| `atrm JOB_NUM`                      | Remove a job (same as `at -d`)       |
| `batch`                              | Run when load < 0.8                  |

### `cron`

| Command                              | Purpose                              |
| ------------------------------------ | ------------------------------------ |
| `crontab -e`                         | Edit crontab                         |
| `crontab -l`                         | List crontab                         |
| `crontab -r`                         | Remove crontab (dangerous — no undo) |
| `crontab -ri`                        | Remove with confirmation prompt      |
| `crontab -l > backup.txt`           | Backup crontab to a file             |
| `crontab backup.txt`                 | Restore crontab from a file          |
| `sudo crontab -u USER -e`           | Edit another user's crontab          |
| `sudo crontab -u USER -l`           | View another user's crontab          |
| `sudo cat /etc/crontab`             | View the system crontab              |
| `sudo ls /etc/cron.d/`              | List system cron drop-in files       |
| `sudo ls /etc/cron.daily/`          | List daily scheduled scripts         |
| `sudo run-parts --test /etc/cron.daily/` | Dry-run — show what scripts would run |
| `sudo run-parts /etc/cron.daily/`   | Manually trigger all daily scripts   |

### `anacron`

| Command                              | Purpose                              |
| ------------------------------------ | ------------------------------------ |
| `cat /etc/anacrontab`               | View anacron config                  |
| `sudo nano /etc/anacrontab`         | Edit anacron config                  |
| `sudo anacron -T`                   | Test syntax for errors               |
| `sudo anacron -n`                   | Run now, skip delay                  |
| `sudo anacron -f`                   | Force run, ignore timestamps         |
| `sudo anacron -fn`                  | Force + immediate (most aggressive)  |
| `sudo anacron -d`                   | Debug mode (foreground, verbose)     |
| `sudo anacron -s`                   | Serialize — run one job at a time    |
| `sudo anacron -u`                   | Update timestamps only (don't run)   |
| `cat /var/spool/anacron/cron.daily` | Check last run date (`YYYYMMDD`)     |
| `ls -la /var/spool/anacron/`        | List all timestamp files             |

### `systemd` Timers

| Command                                    | Purpose                         |
| ------------------------------------------ | ------------------------------- |
| `sudo systemctl daemon-reload`             | Reload after creating/editing files |
| `sudo systemctl enable --now name.timer`   | Enable + start in one command   |
| `sudo systemctl start name.timer`          | Start timer                     |
| `sudo systemctl stop name.timer`           | Stop timer                      |
| `sudo systemctl disable name.timer`        | Disable from boot               |
| `sudo systemctl restart name.timer`        | Restart timer (apply new schedule) |
| `sudo systemctl status name.timer`         | Check timer status              |
| `sudo systemctl status name.service`       | Check last run result           |
| `sudo systemctl start name.service`        | Manually trigger service (for testing) |
| `systemctl list-timers`                    | List all active timers          |
| `systemctl list-timers --all`              | Include inactive timers         |
| `systemctl cat name.timer`                 | View timer file contents        |
| `systemctl cat name.service`               | View service file contents      |
| `systemctl show name.timer`                | Show all timer properties       |
| `systemd-analyze calendar "EXPRESSION"`    | Test OnCalendar syntax          |
| `systemd-analyze verify name.service`      | Validate service file syntax    |
| `journalctl -u name.service`              | View all logs for service       |
| `journalctl -u name.service -n 20`        | View last 20 log lines          |
| `journalctl -u name.service --since today` | View today's logs only          |
| `journalctl -u name.service -f`           | Follow logs in real-time        |

### `/tmp` Cleanup

| Command                                         | Purpose                                  |
| ------------------------------------------------ | ---------------------------------------- |
| `systemctl status systemd-tmpfiles-clean.timer`  | Check if cleanup timer is active         |
| `systemctl list-timers \| grep tmpfiles`         | When next cleanup runs                   |
| `cat /usr/lib/tmpfiles.d/tmp.conf`               | View default `/tmp` cleanup rules        |
| `ls /etc/tmpfiles.d/`                            | List admin override rules                |
| `systemd-tmpfiles --cat-config`                  | View ALL active tmpfiles rules           |
| `sudo systemd-tmpfiles --clean --dry-run`        | Dry-run — show what WOULD be deleted     |
| `sudo systemd-tmpfiles --clean`                  | Manually trigger cleanup now             |
| `sudo systemd-tmpfiles --create`                 | Create/fix directories listed in configs |
| `sudo systemd-tmpfiles --verify`                 | Validate config files for errors         |

---

## 34. General Linux Commands Used with Scheduling

These commands come up constantly when working with scheduled jobs:

### Finding & Verifying

| Command                              | Purpose                              |
| ------------------------------------ | ------------------------------------ |
| `which python3`                      | Find absolute path of a command      |
| `type -a python3`                    | Find all locations of a command      |
| `whereis cron`                       | Find binary, source, and man page    |
| `file /path/to/script.sh`           | Check file type (script, binary, etc.) |
| `echo $PATH`                         | View current PATH variable           |
| `env`                                | View all environment variables       |
| `whoami`                             | Check which user you are             |
| `id`                                 | Show user ID, group ID, and groups   |
| `date`                               | Show current date/time               |
| `uptime`                             | System uptime + load average         |
| `cat /proc/loadavg`                  | Current system load (for `batch`)    |
| `find /tmp -mtime +7 -delete`       | Delete files older than 7 days (cleanup scripts) |
| `find ~/dir -name "*.tmp" -mtime +7 -delete` | Delete matching files older than 7 days |
| `ls -la --time=atime /tmp/`         | View file access times in `/tmp`     |
| `nice command`                       | Run command with lower CPU priority (used in some anacrontabs) |

### File Permissions & Ownership

| Command                              | Purpose                              |
| ------------------------------------ | ------------------------------------ |
| `chmod +x script.sh`                | Make script executable               |
| `chmod 755 script.sh`               | Owner: rwx, Group/Others: r-x       |
| `chmod 700 script.sh`               | Owner only: rwx                      |
| `chown user:group script.sh`        | Change file ownership                |
| `ls -la script.sh`                  | View permissions and ownership       |
| `stat script.sh`                     | Detailed file info (permissions, timestamps) |
| `namei -l /path/to/script.sh`       | Check permissions of every directory in the path |

### Logs & Monitoring

| Command                              | Purpose                              |
| ------------------------------------ | ------------------------------------ |
| `grep CRON /var/log/syslog`         | Search cron entries (Ubuntu)         |
| `cat /var/log/cron`                  | View cron log (CentOS)              |
| `tail -f /var/log/syslog`           | Follow log in real-time              |
| `tail -n 50 /var/log/cron`          | Last 50 lines of cron log           |
| `journalctl -u cron`                | Cron logs via systemd journal        |
| `journalctl -u cron --since "1 hour ago"` | Cron logs from last hour      |
| `journalctl --since today -u crond` | Today's cron activity               |
| `dmesg`                              | Kernel messages (hardware issues)    |
| `last reboot`                        | Show reboot history                  |

### Process Management

| Command                              | Purpose                              |
| ------------------------------------ | ------------------------------------ |
| `ps aux \| grep cron`               | Check if cron process is running     |
| `pgrep -l cron`                      | Find cron process ID                 |
| `kill PID`                           | Stop a process by PID                |
| `killall script.sh`                  | Stop all instances of a script       |
| `nohup command &`                    | Run command immune to hangups        |

### Mail & Notifications

| Command                              | Purpose                              |
| ------------------------------------ | ------------------------------------ |
| `mail`                               | Read local mail (cron output)        |
| `cat /var/mail/username`             | Read mail file directly              |
| `sudo postfix status`               | Check if mail system is running      |
| `dpkg -l \| grep postfix`           | Check if postfix is installed (Ubuntu) |
| `rpm -q postfix`                     | Check if postfix is installed (CentOS) |

### Installation

| Command                              | Purpose                              |
| ------------------------------------ | ------------------------------------ |
| `sudo apt install at`               | Install `at` (Ubuntu/Debian)         |
| `sudo yum install at`               | Install `at` (CentOS/RHEL)          |
| `sudo dnf install at`               | Install `at` (Fedora/newer RHEL)    |
| `sudo apt install cron`             | Install cron (Ubuntu — usually pre-installed) |
| `sudo yum install cronie`           | Install cron (CentOS/RHEL)          |
| `sudo apt install anacron`          | Install anacron (Ubuntu)             |
| `sudo yum install cronie-anacron`   | Install anacron (CentOS)             |
| `sudo apt install postfix mailutils` | Install mail system (Ubuntu)        |
| `sudo yum install postfix mailx`   | Install mail system (CentOS)         |

### Audit All Scheduled Jobs (Comprehensive Script)

```bash
# List ALL user crontabs
for user in $(cut -d: -f1 /etc/passwd); do
    crontab_output=$(sudo crontab -u "$user" -l 2>/dev/null)
    if [ -n "$crontab_output" ]; then
        echo "=== Crontab for $user ==="
        echo "$crontab_output"
    fi
done

# System crontab and drop-ins
echo "=== /etc/crontab ===" && cat /etc/crontab
echo "=== /etc/cron.d/ ===" && ls /etc/cron.d/

# System cron directories
echo "=== Cron directories ==="
ls /etc/cron.daily/ /etc/cron.weekly/ /etc/cron.monthly/ /etc/cron.hourly/ 2>/dev/null

# Systemd timers
echo "=== Active systemd timers ==="
systemctl list-timers --all

# Pending at jobs
echo "=== Pending at jobs ==="
atq
```

---

## 35. Best Practices — The Golden Rules

### Scripting Best Practices

| #  | Rule                                      | Why                                                       |
| -- | ----------------------------------------- | --------------------------------------------------------- |
| 1  | **Always use absolute paths**             | Cron has a minimal PATH — relative/short paths will fail  |
| 2  | **Start every script with a shebang**     | `#!/bin/bash` — tells the OS which interpreter to use     |
| 3  | **Make scripts executable**               | `chmod +x script.sh` — or cron cannot run it              |
| 4  | **Test the script manually first**        | Run it from your terminal before adding it to crontab     |
| 5  | **Use `set -e` in scripts**              | Script exits immediately on any error (fail fast)          |
| 6  | **Use `set -o pipefail` in scripts**     | Catches errors in piped commands that `set -e` misses      |
| 7  | **Add `cd /dir || exit 1` at the top**   | Ensures the script works from the correct directory        |
| 8  | **Don't hardcode secrets in crontab**     | Use environment files or vaults — crontab is readable      |

**Recommended script header:**

```bash
#!/bin/bash
set -euo pipefail

cd /home/akshay/project || exit 1

# Your commands here
```

### Crontab Best Practices

| #  | Rule                                      | Why                                                       |
| -- | ----------------------------------------- | --------------------------------------------------------- |
| 1  | **Always redirect output**                | `>> /var/log/job.log 2>&1` — without this, output is lost |
| 2  | **Always end crontab with a blank line**  | Missing newline at end silently skips the last job         |
| 3  | **Backup your crontab regularly**         | `crontab -l > ~/crontab_backup_$(date +%F).txt`          |
| 4  | **Comment every job**                     | `# Daily DB backup at 2 AM` above the entry               |
| 5  | **Use `flock` for long-running jobs**     | Prevents the same job from overlapping itself              |
| 6  | **Use `MAILTO=""` if no mail needed**     | Prevents mail queue buildup on systems with MTA           |
| 7  | **Never use `crontab -r` carelessly**     | It deletes ALL your jobs instantly — use `crontab -ri`     |
| 8  | **Avoid `%` in crontab commands**         | Cron treats `%` as newline — escape it with `\%`          |

**Example of a well-written crontab:**

```bash
MAILTO=""
PATH=/usr/local/bin:/usr/bin:/bin

# Daily database backup at 2:00 AM
0 2 * * * /usr/bin/flock -n /tmp/backup.lock /home/akshay/scripts/backup.sh >> /var/log/backup.log 2>&1

# Weekly log cleanup every Sunday at 3:00 AM
0 3 * * 0 /home/akshay/scripts/cleanup_logs.sh >> /var/log/cleanup.log 2>&1

# Disk usage report every 6 hours
0 */6 * * * /home/akshay/scripts/disk_report.sh >> /var/log/disk.log 2>&1
```

### Logging Best Practices

| #  | Rule                                      | Why                                                       |
| -- | ----------------------------------------- | --------------------------------------------------------- |
| 1  | **Log start and end times**               | Know how long a job takes and if it actually completed     |
| 2  | **Log exit codes**                        | `$?` after a command tells you pass (0) or fail (non-zero)|
| 3  | **Use `>>` (append), not `>` (overwrite)** | Overwrite destroys the previous run's log                 |
| 4  | **Rotate your log files**                 | Use `logrotate` — a log that grows forever fills the disk |
| 5  | **Separate stdout and stderr when needed** | `>> job.log 2>> job_errors.log` — easier to find errors  |

### Security Best Practices

| #  | Rule                                      | Why                                                       |
| -- | ----------------------------------------- | --------------------------------------------------------- |
| 1  | **Restrict cron access**                  | Use `/etc/cron.allow` to whitelist only authorized users   |
| 2  | **Don't run jobs as root unless needed**  | Least-privilege principle — a buggy root job can destroy everything |
| 3  | **Protect script files**                  | `chmod 700` — owner only, no group/other read/write        |
| 4  | **Never store passwords in scripts**      | Use environment files (`source /etc/job-secrets.env`), restricted to `600` permissions |
| 5  | **Audit cron jobs regularly**             | `for user in $(cut -d: -f1 /etc/passwd); do sudo crontab -u $user -l 2>/dev/null; done` |
| 6  | **Monitor for unauthorized cron jobs**    | Check `/var/spool/cron/` periodically for unexpected entries |

### `systemd` Timer Best Practices

| #  | Rule                                      | Why                                                       |
| -- | ----------------------------------------- | --------------------------------------------------------- |
| 1  | **Always run `daemon-reload` after edits** | systemd caches unit files — edits are invisible until reload |
| 2  | **Use `Persistent=true`**                 | Catches up missed runs — like anacron but built-in         |
| 3  | **Test with `systemctl start name.service`** | Manually trigger the service before trusting the timer   |
| 4  | **Validate with `systemd-analyze calendar`** | Verify your OnCalendar expression before deploying       |
| 5  | **Check status after deployment**         | `systemctl status name.timer` — confirm it says "active (waiting)" |
| 6  | **Use `RandomizedDelaySec=`**             | Adds jitter to prevent thundering herd (many jobs at same time) |
| 7  | **Add `After=network-online.target`**     | If your script needs network — prevents running before network is ready |

### Production / Real-World Best Practices

| #  | Rule                                      | Why                                                       |
| -- | ----------------------------------------- | --------------------------------------------------------- |
| 1  | **Set up alerting for critical jobs**      | If a backup job fails at 2 AM, you need to know before morning |
| 2  | **Version-control your scripts**          | Store in Git — know who changed what and when              |
| 3  | **Document your crontab in a shared wiki** | When you're on vacation, your team needs to know what's running |
| 4  | **Use consistent naming conventions**     | `daily_db_backup.sh`, `weekly_log_cleanup.sh` — be obvious |
| 5  | **Schedule heavy jobs during off-peak hours** | Backups at 2 AM, not during business hours             |
| 6  | **Stagger job start times**               | Don't schedule 10 jobs at `0 2 * * *` — spread them out  |
| 7  | **Test on staging before production**     | A broken cron job in prod at 2 AM is a very bad night     |
| 8  | **Have a rollback plan**                  | What happens if the script goes wrong? Can you undo it?    |
| 9  | **Monitor disk space**                    | Logs, backups, temp files — scheduled jobs create output that fills disks |
| 10 | **Keep jobs idempotent**                  | Running the same job twice should be safe — no duplicate data, no errors |

---

## 36. When to Use vs. Avoid `@reboot`

| Use It For                                    | Avoid It For                                          |
| --------------------------------------------- | ----------------------------------------------------- |
| Starting a background app without a systemd service | Production services (no restart-on-failure)        |
| One-time initialization script after reboot   | Anything needing dependency ordering                  |
| Quick dev/test convenience                    | Jobs that need network/mount/service guarantees       |

**For production:** Use a proper systemd service with `WantedBy=multi-user.target` and `After=network-online.target` instead of `@reboot`.

---

## 37. Incident Response — Failed Critical Cron Job

If a critical nightly job didn't run, follow this process:

```
1. Check logs       → grep CRON /var/log/syslog  (did cron try to run it?)
2. Check daemon     → systemctl status crond      (is cron running?)
3. Check crontab    → crontab -l                  (is the job still listed?)
4. Check system     → uptime, dmesg, journalctl --since yesterday
                      (was the server rebooted? resource issue?)
5. Run manually     → Execute the script and capture output
6. Check permissions→ Did a recent change break file/user access?
7. Remediate        → Run the job if safe, fix root cause,
                      add monitoring/alerting to prevent recurrence
```

---

## 38. Interview Tips — How to Deliver Answers

1. **Start with the "what"** — define the concept in one sentence
2. **Give the "why"** — explain when and why you'd use it
3. **Show the "how"** — mention the specific command or file
4. **Mention trade-offs** — interviewers love hearing you weigh options
5. **Use real examples** — "In my last role, I used this for..." beats textbook answers
6. **It's okay to say "I'd look it up"** — for exact syntax, admitting you'd check the man page is honest and professional

**Two-crontab-entry rule:** When asked to schedule different actions at different times (e.g., "weekdays at 8:30 AM and Saturdays at noon"), remember this requires **two separate crontab entries** — one schedule per line.

---

## 39. Interview Power Answers — One-Liners

Use these when the interviewer asks a short-answer question:

| Question                                         | Power Answer                                                                                     |
| ------------------------------------------------ | ------------------------------------------------------------------------------------------------ |
| What is `cron`?                                  | A daemon that runs scheduled jobs repeatedly based on a 5-field time expression in a crontab file. |
| What is `at`?                                    | A command to schedule a one-time job at a specific time in the future.                            |
| `at` vs `cron`?                                  | `at` = one-time. `cron` = recurring.                                                             |
| `cron` vs `anacron`?                             | `cron` skips missed jobs. `anacron` catches them up. `cron` = servers. `anacron` = laptops.       |
| `cron` vs `systemd` timer?                       | Timers offer centralized logging, dependency management, resource limits, and catch-up via `Persistent=true`. |
| What is `anacron`?                               | A tool that guarantees daily/weekly/monthly jobs run even if the machine was powered off.          |
| What does `*/15 9-17 * * 1-5` mean?             | Every 15 minutes, during 9 AM–5 PM, Monday through Friday.                                       |
| Where are cron logs?                             | `/var/log/syslog` (Ubuntu), `/var/log/cron` (CentOS), or `journalctl -u cron`.                   |
| Cron job works manually but not in cron?         | Check PATH (use absolute paths), environment vars, permissions, and redirect output to a log.     |
| How to prevent overlapping cron jobs?            | Use `flock -n /tmp/job.lock /path/to/script.sh`.                                                 |
| What is `MAILTO` in crontab?                     | Controls where cron sends script output. Set to `""` to disable mail, or an email to redirect it. |
| What is `run-parts`?                             | A utility that runs all executable scripts inside a directory.                                    |
| Where does anacron store last-run timestamps?    | `/var/spool/anacron/` — one file per job, contains date in `YYYYMMDD` format.                    |
| How does Linux clean `/tmp`?                     | Modern: `systemd-tmpfiles-clean.timer` (10d for `/tmp`, 30d for `/var/tmp`). Older: `tmpwatch`/`tmpreaper` via cron.daily. |
| What is the sticky bit on `/tmp`?                | Permission `1777` — all users can write, but can only delete their own files.                     |
| User crontab vs `/etc/crontab`?                  | User crontab has 5 fields + command. System crontab has 5 fields + USERNAME + command.            |
| What is `@reboot` in cron?                       | A shortcut that runs a command once when the cron daemon starts (at system boot).                 |
| What does `2>&1` mean?                           | Redirect standard error (fd 2) to the same destination as standard output (fd 1).                |
| How to check if `crond` is running?              | `systemctl status crond` (CentOS) or `systemctl status cron` (Ubuntu).                           |
| What file controls who can use cron?             | `/etc/cron.allow` (whitelist) and `/etc/cron.deny` (blacklist). Allow takes priority.             |
| What is `RANDOM_DELAY` in anacron?               | Max random minutes added on top of the base delay — prevents all machines from running at the exact same time. |
| What is `START_HOURS_RANGE` in anacron?           | Restricts which hours jobs can start. `3-22` means jobs only run between 3 AM and 10 PM.         |
| What is `CRON_TZ`?                               | A crontab variable that sets the timezone for job schedules, overriding the system timezone.      |
| What is idempotent and why does it matter for cron? | A job is idempotent if running it twice produces the same result. Important because cron jobs can accidentally run twice (overlap, retry). |
| How to backup and restore a crontab?             | Backup: `crontab -l > backup.txt`. Restore: `crontab backup.txt`.                                |
| What does `set -euo pipefail` do in a script?    | `set -e` exits on error, `-u` treats unset vars as errors, `-o pipefail` catches pipe failures.   |
| What is `Persistent=true` in systemd timers?     | Catches up missed timer runs — like anacron behavior but built into systemd.                      |
| What is `run-parts --test`?                      | Dry-run mode — shows which scripts would execute in a directory without actually running them.    |
| What is job scheduling?                          | The process of automating command/script execution at a specific time or on a recurring basis, without manual intervention. |
| Why avoid `@reboot` in production?               | No restart-on-failure, no dependency ordering, no guarantee about network/mounts. Use a systemd service instead. |
| What happens if `/tmp` fills up?                 | Programs crash, databases fail, logging stops, login may fail. Linux auto-cleans via systemd-tmpfiles or tmpwatch. |
| What is `logrotate`?                             | A tool that rotates, compresses, and archives log files. Runs daily via `/etc/cron.daily/logrotate`. Config in `/etc/logrotate.d/`. |
| Why use `%` carefully in crontab?                | Cron treats `%` as a newline. Escape it with `\%` or put the command in a script.                 |
| What does `nice` do in anacrontab?               | Runs the command with lower CPU priority so scheduled jobs don't slow down the system.            |
| What is `OnUnitActiveSec` in systemd timers?     | Repeats the timer X time after the service last ran. Example: `OnUnitActiveSec=10min` = every 10 minutes after last run. |
| How to see all scheduled jobs on a system?       | Combine: `crontab -l`, `cat /etc/crontab`, `ls /etc/cron.d/`, `systemctl list-timers`, and `atq`. |

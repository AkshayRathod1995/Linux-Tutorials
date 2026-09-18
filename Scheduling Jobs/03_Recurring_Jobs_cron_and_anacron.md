# Recurring Jobs: `cron` and `anacron`

## Index

### Part 1: The `cron` System
1. [What Is `cron`?](#what-is-cron)
2. [The `crond` Daemon](#the-crond-daemon)
3. [The Crontab: Your Schedule File](#the-crontab-your-schedule-file)
4. [Crontab Examples (From Simple to Complex)](#crontab-examples-from-simple-to-complex)
5. [Special Symbols Cheat Sheet](#special-symbols-cheat-sheet)
6. [Special Shortcut Strings](#special-shortcut-strings)
7. [Managing Your Crontab](#managing-your-crontab)
8. [Cron Environment Variables](#cron-environment-variables)
9. [User Cron Jobs vs. System Cron Jobs](#user-cron-jobs-vs-system-cron-jobs)

### Part 2: The `anacron` System
10. [The Problem `anacron` Solves](#the-problem-anacron-solves)
11. [What Is `anacron`?](#enter-anacron)
12. [How `anacron` Works — Step by Step](#how-anacron-works--step-by-step)
13. [The `/etc/anacrontab` File — Complete Breakdown](#the-etcanacrontab-file--complete-breakdown)
14. [Anacron File Syntax — Field by Field](#anacron-file-syntax--field-by-field)
15. [Practical `/etc/anacrontab` Examples](#practical-etcanacrontab-examples)
16. [Where `anacron` Tracks Last Execution — `/var/spool/anacron/`](#where-anacron-tracks-last-execution--varspoolanacron)
17. [Anacron Log Files — Where to Find Them](#anacron-log-files--where-to-find-them)
18. [Where Does Script Output Go? (Mail Configuration)](#where-does-script-output-go-mail-configuration)
19. [Running `anacron` Manually](#running-anacron-manually)
20. [`cron` vs `anacron`: The Complete Comparison](#cron-vs-anacron-the-complete-comparison)
21. [How `cron` and `anacron` Work Together](#how-cron-and-anacron-work-together)
22. [Quick Reference Summary](#quick-reference-summary)
23. [What's Next?](#whats-next)

---

## Part 1: The `cron` System

### What Is `cron`?

`cron` is the **king of job scheduling** in Linux. While `at` runs a job once, `cron` runs jobs **on a repeating schedule** — every minute, every hour, every day, every month, or any combination you can think of.

The name "cron" comes from the Greek word **Chronos**, meaning **time**.

**Use cases:**
- Back up the database every night at 2 AM
- Clear temporary files every Sunday
- Check disk space every 30 minutes
- Send a weekly summary email every Friday at 5 PM

### The `crond` Daemon

Just like `at` has `atd`, `cron` has its own daemon called `crond` (or simply `cron` on some systems).

```bash
# Check if the cron daemon is running
sudo systemctl status crond     # CentOS/RHEL
sudo systemctl status cron      # Ubuntu/Debian
```

`crond` wakes up **every minute**, checks if any jobs are due, runs them, and goes back to sleep. It does this 24/7/365.

---

## The Crontab: Your Schedule File

A **crontab** (short for "cron table") is a file where you write your scheduled jobs. Each line in the file represents one scheduled job.

### The Crontab Syntax

Every line in a crontab follows this format:

```
* * * * * command_to_run
```

Those five stars represent **when** the job should run. Here's the visual guide:

```
┌───────────── Minute        (0 - 59)
│ ┌─────────── Hour          (0 - 23)
│ │ ┌───────── Day of Month  (1 - 31)
│ │ │ ┌─────── Month         (1 - 12)
│ │ │ │ ┌───── Day of Week   (0 - 6, where 0 = Sunday)
│ │ │ │ │
* * * * * command_to_run
```

### How to Memorize It

Think of it as answering five questions in order:

1. **What minute?** (0-59)
2. **What hour?** (0-23, using 24-hour clock)
3. **What day of the month?** (1-31)
4. **What month?** (1-12)
5. **What day of the week?** (0-6, Sunday = 0)

A `*` means **"every"** — so `* * * * *` means "every minute of every hour of every day of every month on every day of the week."

---

## Crontab Examples (From Simple to Complex)

### Example 1: Every day at 2:00 AM

```
0 2 * * * /home/akshay/scripts/backup.sh
```

**Reading it:** "At minute **0**, at hour **2**, on every day, every month, every day of the week."

### Example 2: Every Monday at 9:00 AM

```
0 9 * * 1 /home/akshay/scripts/weekly_report.sh
```

**Reading it:** "At minute 0, at hour 9, every day of month, every month, on day **1** (Monday)."

### Example 3: Every 15 minutes

```
*/15 * * * * /home/akshay/scripts/check_disk.sh
```

**Reading it:** "Every 15 minutes (`*/15`), every hour, every day."

### Example 4: Every weekday (Mon-Fri) at 6:00 PM

```
0 18 * * 1-5 /home/akshay/scripts/end_of_day.sh
```

### Example 5: The 1st and 15th of every month at midnight

```
0 0 1,15 * * /home/akshay/scripts/payroll.sh
```

### Example 6: Every 5 minutes, but only during business hours (9 AM - 5 PM)

```
*/5 9-17 * * * /home/akshay/scripts/monitor.sh
```

### Example 7: Once a year — midnight on January 1st

```
0 0 1 1 * /home/akshay/scripts/happy_new_year.sh
```

---

## Special Symbols Cheat Sheet

| Symbol   | Meaning              | Example          | Explanation                     |
| -------- | -------------------- | ---------------- | ------------------------------- |
| `*`      | Every possible value | `* * * * *`      | Every minute                    |
| `,`      | List of values       | `0 9,12,18 * * *`| At 9 AM, 12 PM, and 6 PM       |
| `-`      | Range of values      | `0 9-17 * * *`   | Every hour from 9 AM to 5 PM   |
| `/`      | Step/interval        | `*/10 * * * *`   | Every 10 minutes                |

---

## Special Shortcut Strings

Instead of writing five numbers, cron supports these handy shortcuts:

| Shortcut     | Equivalent         | Meaning                       |
| ------------ | ------------------ | ----------------------------- |
| `@reboot`    | *(runs at startup)*| Run once when the system boots |
| `@yearly`    | `0 0 1 1 *`        | Once a year (Jan 1st)         |
| `@annually`  | `0 0 1 1 *`        | Same as @yearly               |
| `@monthly`   | `0 0 1 * *`        | First day of every month      |
| `@weekly`    | `0 0 * * 0`        | Every Sunday at midnight      |
| `@daily`     | `0 0 * * *`        | Every day at midnight         |
| `@midnight`  | `0 0 * * *`        | Same as @daily                |
| `@hourly`    | `0 * * * *`        | Every hour at minute 0        |

Example:

```
@daily /home/akshay/scripts/backup.sh
@reboot /home/akshay/scripts/start_services.sh
```

---

## Managing Your Crontab

### Edit your crontab

```bash
crontab -e
```

This opens your personal crontab in your default text editor. Add your schedule lines, save, and exit. The jobs are installed immediately.

### View your crontab

```bash
crontab -l
```

This lists all the cron jobs you have scheduled.

### Remove your entire crontab (CAUTION!)

```bash
crontab -r
```

This **deletes all** your cron jobs without asking for confirmation. Use with extreme care.

To remove with a confirmation prompt:

```bash
crontab -ri
```

### Edit another user's crontab (requires root)

```bash
sudo crontab -u john -e
```

### View another user's crontab

```bash
sudo crontab -u john -l
```

---

## Cron Environment Variables

You can set environment variables at the **top of any crontab** file. These apply to all jobs below them.

```bash
crontab -e
```

```
SHELL=/bin/bash
PATH=/usr/local/bin:/usr/bin:/bin:/home/akshay/.local/bin
MAILTO=akshay@company.com
HOME=/home/akshay
CRON_TZ=America/New_York
```

| Variable    | Purpose                                                            | Default                |
| ----------- | ------------------------------------------------------------------ | ---------------------- |
| `SHELL`     | Which shell runs the job                                           | `/bin/sh`              |
| `PATH`      | Where to find commands — cron's default PATH is very short!        | `/usr/bin:/bin`         |
| `MAILTO`    | Who receives the job's output as email. `""` to disable mail       | The user who owns the crontab |
| `HOME`      | Working directory for the job                                      | User's home directory  |
| `CRON_TZ`   | **Timezone** for the schedule. Jobs run according to THIS timezone, not the system timezone | System timezone |

### `CRON_TZ` — Running Jobs in a Different Timezone

If your server is in UTC but you want a job to run at 9 AM New York time:

```
CRON_TZ=America/New_York
0 9 * * * /home/akshay/scripts/morning_report.sh
```

Without `CRON_TZ`, cron uses the server's timezone. This is important when your team is in a different timezone from your server.

**To check your system's current timezone:**

```bash
timedatectl
# or
cat /etc/timezone          # Ubuntu
ls -la /etc/localtime      # CentOS — shows a link to the timezone file
```

**To list all available timezones:**

```bash
timedatectl list-timezones
timedatectl list-timezones | grep America
```

---

## User Cron Jobs vs. System Cron Jobs

There are two types of cron jobs in Linux:

### 1. User Cron Jobs

- Created with `crontab -e`
- Stored in `/var/spool/cron/` (CentOS) or `/var/spool/cron/crontabs/` (Ubuntu)
- Each user has their own crontab file
- Jobs run as the user who created them

### 2. System Cron Jobs

- Edited directly in `/etc/crontab`
- Have an **extra field** — the username of who should run the command

The system crontab format:

```
* * * * * USERNAME command_to_run
```

Example:

```
0 2 * * * root /usr/local/bin/system_backup.sh
0 3 * * * www-data /var/www/cleanup.sh
```

Notice the `root` and `www-data` between the schedule and the command — this is the **user** the job runs as.

### System Cron Directories

Linux also provides special directories for dropping scripts into:

```
/etc/cron.hourly/    - Scripts here run every hour
/etc/cron.daily/     - Scripts here run once a day
/etc/cron.weekly/    - Scripts here run once a week
/etc/cron.monthly/   - Scripts here run once a month
```

To use them, simply place your executable script in the appropriate directory:

```bash
sudo cp /home/akshay/scripts/cleanup.sh /etc/cron.daily/
sudo chmod +x /etc/cron.daily/cleanup.sh
```

No crontab editing needed — anything in `/etc/cron.daily/` will automatically run once a day.

### Controlling Who Can Use `cron`

| File              | Purpose                                                       |
| ----------------- | ------------------------------------------------------------- |
| `/etc/cron.allow` | If exists, **only** users listed here can use cron             |
| `/etc/cron.deny`  | If exists, users listed here are **blocked** from using cron   |

The rules work the same way as `/etc/at.allow` and `/etc/at.deny`.

---

## Part 2: `anacron` — The Catch-Up Scheduler

### The Problem `anacron` Solves

`cron` has one major weakness: **it assumes your computer is always on.**

If you schedule a backup for 2 AM every night, but your laptop was turned off at 2 AM, `cron` simply **skips that job**. It doesn't say "Oh, I missed that — let me run it now." It just forgets about it.

This is fine for servers that run 24/7. But for:

- Laptops that get shut down at night
- Desktop computers that get turned off over weekends
- Any machine that doesn't run continuously

...missed jobs can be a real problem.

### Enter `anacron`

`anacron` (which stands for **"anachronistic cron"**) was built to solve exactly this problem. Its key difference:

> **`anacron` guarantees that a job will run — even if it has to wait until the next time the computer is turned on.**

Unlike `cron`, `anacron` is **NOT a daemon** — it doesn't run in the background constantly. Instead, it is **invoked** (triggered) at boot time and/or by `cron` itself, checks what's overdue, runs it, and exits.

---

### How `anacron` Works — Step by Step

Here's exactly what happens when `anacron` runs:

```
1. anacron starts (triggered at boot or by cron)
        │
        ▼
2. Reads /etc/anacrontab to find all configured jobs
        │
        ▼
3. For EACH job, checks /var/spool/anacron/<job-id>
   to find the LAST RUN DATE (stored as YYYYMMDD)
        │
        ▼
4. Compares: Has <period> days passed since last run?
        │
   ┌────┴────┐
   │ NO      │ YES
   │         │
   ▼         ▼
5. Skip it  6. Wait <delay> minutes, then RUN the job
                    │
                    ▼
              7. Update the timestamp in /var/spool/anacron/<job-id>
                    │
                    ▼
              8. Move to the next job
```

**Key takeaway:** `anacron` doesn't care what time it is. It only cares: "Has enough time passed since this job last ran?"

---

### The `/etc/anacrontab` File — Complete Breakdown

This is the **heart** of `anacron`. Let's look at a real one:

```bash
sudo cat /etc/anacrontab
```

Here's what a typical `/etc/anacrontab` looks like:

```
# /etc/anacrontab: configuration file for anacron

# See anacron(8) and anacrontab(5) for details.

SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin
HOME=/root
LOGNAME=root
MAILTO=root

# the maximal random delay added to the base delay of the jobs
RANDOM_DELAY=45

# the jobs will be started during the following hours only
START_HOURS_RANGE=3-22

# These replace cron's entries
1	5	cron.daily	run-parts --report /etc/cron.daily
7	10	cron.weekly	run-parts --report /etc/cron.weekly
@monthly	15	cron.monthly	run-parts --report /etc/cron.monthly
```

Let's break down **every section**:

#### Environment Variables at the Top

```
SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin
HOME=/root
LOGNAME=root
MAILTO=root
RANDOM_DELAY=45
START_HOURS_RANGE=3-22
```

| Variable             | Purpose                                                              |
| -------------------- | -------------------------------------------------------------------- |
| `SHELL`              | Which shell to use when running commands (default: `/bin/sh`)        |
| `PATH`               | Where to find commands — notice this is much richer than cron's PATH |
| `HOME`               | The home directory for the jobs (commands run "from" here)           |
| `LOGNAME`            | The username logged in system records                                |
| `MAILTO`             | Who receives the output via email. Set to `""` to disable mail       |
| `RANDOM_DELAY`       | **Maximum random minutes** added on top of the base DELAY. Prevents all machines in a fleet from running at the exact same time. Example: `RANDOM_DELAY=45` means anacron adds a random delay between 0 and 45 minutes to each job's base delay |
| `START_HOURS_RANGE`  | **Hours during which jobs are allowed to start.** Jobs will NOT start outside this window. Example: `START_HOURS_RANGE=3-22` means jobs can only start between 3:00 AM and 10:00 PM. If anacron runs at 11 PM and the window is `3-22`, the job will wait until 3:00 AM the next day |

**How the delays add up:**

```
Total wait before a job runs = Base DELAY + Random(0 to RANDOM_DELAY)

Example for cron.daily (DELAY=5, RANDOM_DELAY=45):
  Minimum wait: 5 + 0  = 5 minutes
  Maximum wait: 5 + 45 = 50 minutes

BUT it also must fall within START_HOURS_RANGE (3-22).
If anacron starts at 2:30 AM with START_HOURS_RANGE=3-22,
it will wait until 3:00 AM before starting any job.
```

**Important:** Unlike cron, anacron lets you set `PATH` right in the config file. This means the "PATH problem" that plagues cron jobs is less of an issue with anacron.

#### Job Lines

```
1	5	cron.daily	run-parts --report /etc/cron.daily
7	10	cron.weekly	run-parts --report /etc/cron.weekly
@monthly	15	cron.monthly	run-parts --report /etc/cron.monthly
```

Each line defines one scheduled job. Let's decode them field by field.

---

### Anacron File Syntax — Field by Field

```
PERIOD    DELAY    JOB-ID    COMMAND
```

| Field       | What It Means                                     | Valid Values                          |
| ----------- | ------------------------------------------------- | ------------------------------------- |
| `PERIOD`    | How often the job should run (in **days**)        | A number (`1`, `7`, `30`) or `@monthly`, `@yearly` |
| `DELAY`     | Minutes to **wait** before running after anacron starts | A number (`5`, `10`, `15`, `45`)  |
| `JOB-ID`    | A unique name to **identify and track** this job  | Any string (no spaces), e.g. `cron.daily` |
| `COMMAND`   | The actual command or script to execute            | Any valid shell command              |

#### Field 1: PERIOD (How Often)

| Value       | Meaning                                            |
| ----------- | -------------------------------------------------- |
| `1`         | Run every day (once per day)                       |
| `7`         | Run every 7 days (once per week)                   |
| `30`        | Run every 30 days (roughly monthly)                |
| `@monthly`  | Run once per month (calendar month, not 30 days)   |
| `@yearly`   | Run once per year                                  |

**Important:** The minimum period is **1 day**. You cannot use anacron for hourly or per-minute jobs — use `cron` for those.

#### Field 2: DELAY (Stagger to Avoid Overload)

Why does anacron have a delay? Imagine your laptop boots up after being off all weekend. **Three** jobs are overdue — daily, weekly, and monthly. If they all start at once, your laptop would be slow for minutes.

The delay **staggers** them:

```
1	5	cron.daily	...      ← Starts 5 minutes after boot
7	10	cron.weekly	...      ← Starts 10 minutes after boot
@monthly	15	cron.monthly	...  ← Starts 15 minutes after boot
```

This way, your daily job finishes first, then the weekly kicks in, and the monthly starts last.

#### Field 3: JOB-ID (The Tracking Name)

The JOB-ID is critical because anacron uses it to create a **timestamp file** in `/var/spool/anacron/`. This is how it remembers when a job was last run.

```
JOB-ID: cron.daily  →  Timestamp file: /var/spool/anacron/cron.daily
JOB-ID: cron.weekly →  Timestamp file: /var/spool/anacron/cron.weekly
JOB-ID: my-backup   →  Timestamp file: /var/spool/anacron/my-backup
```

Rules for JOB-ID:
- Must be unique
- No spaces allowed
- Keep it descriptive

#### Field 4: COMMAND

The most common command you'll see is `run-parts`:

```
run-parts --report /etc/cron.daily
```

`run-parts` is a utility that **runs every executable script** inside a directory. So this single command runs every script in `/etc/cron.daily/` — one after another.

The `--report` flag makes it print the name of each script as it runs (useful for logging).

---

### Practical `/etc/anacrontab` Examples

#### Example 1: Adding Your Own Daily Backup

```
# Period  Delay  Job-ID           Command
1         5      cron.daily       run-parts --report /etc/cron.daily
7         10     cron.weekly      run-parts --report /etc/cron.weekly
@monthly  15     cron.monthly     run-parts --report /etc/cron.monthly
1         20     daily-backup     /home/akshay/scripts/backup.sh
```

**Reading the last line:** "Run `backup.sh` every **1 day**. Wait **20 minutes** after anacron starts (so it doesn't conflict with cron.daily). Track it with the ID `daily-backup`."

#### Example 2: Weekly Log Cleanup

```
7    25    weekly-log-cleanup    /usr/local/bin/clean_old_logs.sh
```

"Run every **7 days**. Wait **25 minutes** after anacron starts. Track with ID `weekly-log-cleanup`."

#### Example 3: Monthly Security Audit

```
@monthly    30    monthly-audit    /opt/scripts/security_audit.sh
```

"Run once per **calendar month**. Wait **30 minutes** delay. Track with ID `monthly-audit`."

---

### Where `anacron` Tracks Last Execution — `/var/spool/anacron/`

This directory is how anacron has a **memory**. Each job has a corresponding file that contains one line: the date it last ran.

```bash
ls -la /var/spool/anacron/
```

```
-rw-------  1 root root  9 Sep 18 07:35 cron.daily
-rw-------  1 root root  9 Sep 15 07:40 cron.monthly
-rw-------  1 root root  9 Sep 14 07:45 cron.weekly
```

```bash
cat /var/spool/anacron/cron.daily
```

```
20260918
```

```bash
cat /var/spool/anacron/cron.weekly
```

```
20260914
```

```bash
cat /var/spool/anacron/cron.monthly
```

```
20260915
```

**What does this tell us?**

| File            | Contents     | Meaning                                   |
| --------------- | ------------ | ----------------------------------------- |
| `cron.daily`    | `20260918`   | Daily jobs last ran on September 18, 2026 |
| `cron.weekly`   | `20260914`   | Weekly jobs last ran on September 14, 2026 |
| `cron.monthly`  | `20260915`   | Monthly jobs last ran on September 15, 2026 |

**How anacron uses this:**

When anacron starts, it reads `cron.daily` and sees `20260918`. Today is September 18, so `today - 20260918 = 0 days`. The period is `1` day. Since 0 < 1, the daily job is **not yet due** — skip it.

If the machine was off yesterday and today is September 19, then `20260919 - 20260918 = 1 day`. Since 1 >= 1 (the period), the daily job **is due** — run it!

#### Manually Checking Execution History

You can always check when each job last ran:

```bash
# View all timestamps at once
for f in /var/spool/anacron/*; do
    echo "$(basename $f): $(cat $f)"
done
```

Output:

```
cron.daily: 20260918
cron.monthly: 20260915
cron.weekly: 20260914
```

---

### Anacron Log Files — Where to Find Them

Anacron itself doesn't create a separate log file. Instead, it logs to the **system log**, just like cron.

#### Where to Check

| Distribution       | Log File                   | Command                               |
| ------------------ | -------------------------- | ------------------------------------- |
| **Ubuntu/Debian**  | `/var/log/syslog`          | `grep anacron /var/log/syslog`        |
| **CentOS/RHEL**    | `/var/log/cron`            | `grep anacron /var/log/cron`          |
| **Any systemd**    | Journal                    | `journalctl | grep anacron`           |

#### Viewing Anacron Logs

```bash
# Ubuntu/Debian — search syslog for anacron entries
grep anacron /var/log/syslog
```

```
Sep 18 07:30:01 myserver anacron[12345]: Anacron started on 2026-09-18
Sep 18 07:30:01 myserver anacron[12345]: Will run job `cron.daily' in 5 min.
Sep 18 07:30:01 myserver anacron[12345]: Jobs will be executed sequentially
Sep 18 07:35:01 myserver anacron[12345]: Job `cron.daily' started
Sep 18 07:36:42 myserver anacron[12345]: Job `cron.daily' terminated
Sep 18 07:36:42 myserver anacron[12345]: Normal exit (1 job run)
```

```bash
# CentOS/RHEL — search the cron log
grep anacron /var/log/cron
```

```bash
# Real-time monitoring
sudo tail -f /var/log/syslog | grep anacron     # Ubuntu
sudo tail -f /var/log/cron | grep anacron        # CentOS
```

**What the log tells you:**
- When anacron started
- Which jobs it determined were due
- How long it waited (the delay)
- When each job started and finished
- Whether it exited normally or with errors

---

### Where Does Script Output Go? (Mail Configuration)

This is a question that catches many beginners: **"My anacron/cron job runs, but where does the output go?"**

#### Default Behavior: Mail

By default, both `cron` and `anacron` try to **email** any output (stdout and stderr) from the job to the user who owns the job (typically `root` for anacron).

This means:
1. Your script runs and prints "Backup completed successfully"
2. `anacron` captures that output
3. `anacron` tries to send it as an email to `root`

#### If No Mail System Is Installed (Most Common Today)

On most modern systems, there's **no mail server** (like `postfix` or `sendmail`) installed. When this happens:

- The output is **silently lost** — gone forever
- You'll see warnings in the logs like: `CRON: (CRON) info (No MTA installed, discarding output)`
- You'll never know if your job succeeded or failed

**This is the #1 reason to always redirect output yourself.**

#### Setting Up Mail Delivery

If you want the mail-based approach to work:

**Step 1:** Install a basic mail system:

```bash
# Ubuntu/Debian
sudo apt install postfix mailutils -y

# CentOS/RHEL
sudo yum install postfix mailx -y
```

During `postfix` setup, choose "Local only" if you just want mail delivered to local mailboxes.

**Step 2:** Start the mail service:

```bash
sudo systemctl start postfix
sudo systemctl enable postfix
```

**Step 3:** Check for mail:

```bash
# Check root's mail
sudo mail

# Or check a specific user's mail
mail
```

#### Controlling Where Mail Goes

In `/etc/anacrontab`, you can set `MAILTO` to control email delivery:

```
# Send output to a specific email address
MAILTO=admin@company.com

# Send output to root's local mailbox
MAILTO=root

# Disable mail entirely (discard all output)
MAILTO=""
```

In a **user crontab** (`crontab -e`), the same variable works:

```
MAILTO=akshay@company.com
0 2 * * * /home/akshay/scripts/backup.sh
```

#### The Recommended Approach: Redirect Output to a Log File

Instead of relying on mail (which may not be set up), **always redirect output in the command itself**:

```
# In /etc/anacrontab — capture output to a log file
1    5    cron.daily    run-parts --report /etc/cron.daily >> /var/log/anacron-daily.log 2>&1

# In your own scripts — log everything inside the script
#!/bin/bash
LOG="/var/log/my-backup.log"
echo "=== Backup started at $(date) ===" >> "$LOG"
tar -czf /backups/home_$(date +%F).tar.gz /home/ >> "$LOG" 2>&1
echo "=== Backup finished at $(date) ===" >> "$LOG"
```

#### Quick Comparison: Output Handling Options

| Method                    | Pros                              | Cons                                |
| ------------------------- | --------------------------------- | ----------------------------------- |
| **Default (mail)**        | Automatic, no setup in scripts    | Requires MTA, easy to miss emails   |
| **MAILTO=""**             | Suppresses unwanted mail          | Output is lost entirely             |
| **Redirect to log file**  | Always works, easy to check       | Must set up in each script/command  |
| **MAILTO + redirect**     | Belt and suspenders               | More setup                          |

---

### Running `anacron` Manually

You can invoke anacron yourself to test or force jobs:

```bash
# Test the anacrontab for syntax errors (recommended before editing)
sudo anacron -T
```

If there are no errors, it prints nothing. If there's a problem, it tells you which line is broken.

```bash
# Run all jobs now, ignoring the delay (don't wait 5/10/15 minutes)
sudo anacron -n
```

```bash
# Force-run ALL jobs, ignoring timestamps (pretend nothing ever ran)
sudo anacron -f
```

```bash
# Force-run immediately (combine -f and -n)
sudo anacron -fn
```

```bash
# Update all timestamps to TODAY without actually running jobs
# Useful if you want to "reset" anacron's memory
sudo anacron -u
```

```bash
# Run anacron in the foreground (don't fork to background)
# Useful for debugging — you see all output in your terminal
sudo anacron -d
```

```bash
# Serialize jobs to run one at a time (default behavior)
sudo anacron -s
```

| Flag | What It Does                                        |
| ---- | --------------------------------------------------- |
| `-T` | Test anacrontab for syntax errors                   |
| `-n` | Run now, skip the delay                             |
| `-f` | Force run, ignore timestamps                        |
| `-fn`| Force run immediately (most aggressive)             |
| `-u` | Update timestamps to today without running anything |
| `-d` | Debug mode — run in foreground with verbose output  |
| `-s` | Serialize — run jobs one after another (default)    |

---

## `cron` vs `anacron`: The Complete Comparison

| Feature                  | `cron`                      | `anacron`                    |
| ------------------------ | --------------------------- | ---------------------------- |
| **Minimum interval**     | 1 minute                    | 1 day                        |
| **Runs missed jobs?**    | No                          | Yes                          |
| **Best for**             | Servers (always-on)         | Laptops/desktops (on/off)    |
| **Precision**            | Exact time (2:00 AM sharp)  | Approximate (sometime today) |
| **User crontabs?**       | Yes                         | No (system-wide only)        |
| **Is it a daemon?**      | Yes (`crond` runs 24/7)     | No (runs once and exits)     |
| **Config file**          | `crontab -e` or `/etc/crontab` | `/etc/anacrontab`        |
| **Tracking**             | None (relies on being awake)| Timestamp files in `/var/spool/anacron/` |
| **Output handling**      | Mail to user or MAILTO      | Mail to MAILTO or root       |
| **Runs as**              | The owning user             | root (system-wide only)      |

---

### How `cron` and `anacron` Work Together

In most modern Linux distributions, `cron` and `anacron` work **together** in a chain. Understanding this chain is important:

```
                                    BOOT
                                      │
                ┌─────────────────────┤
                ▼                     ▼
        cron starts (daemon)    anacron runs (one-shot)
                │                     │
                ▼                     ▼
        Checks /etc/crontab     Checks /etc/anacrontab
        and /etc/cron.d/        and /var/spool/anacron/
                │                     │
                ▼                     ▼
        Runs per-minute/        Runs any overdue
        per-hour user jobs      daily/weekly/monthly jobs
                │
                ▼
        Also triggers anacron
        periodically via
        /etc/cron.d/anacron
```

**The typical flow:**

1. System boots up
2. `crond` starts as a daemon and keeps running
3. `anacron` also runs at boot (via systemd or a script in `/etc/init.d/`)
4. `anacron` checks which daily/weekly/monthly jobs are overdue and runs them
5. Later, `cron` also triggers `anacron` on a schedule (typically once a day) via an entry in `/etc/cron.d/anacron`

You can see the cron-to-anacron trigger:

```bash
cat /etc/cron.d/anacron
```

```
# /etc/cron.d/anacron: crontab entries for the anacron package

SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

30 7    * * *   root    test -x /etc/init.d/anacron && /usr/sbin/invoke-rc.d anacron start >/dev/null
```

This says: "Every day at 7:30 AM, if anacron is installed, trigger it." This ensures that even on a server that never reboots, anacron still gets a chance to check for overdue jobs.

---

## Quick Reference Summary

```bash
# === CRON ===

# Edit your personal crontab
crontab -e

# View your crontab
crontab -l

# Remove your crontab
crontab -r

# Edit another user's crontab (as root)
sudo crontab -u USERNAME -e

# Check cron daemon status
sudo systemctl status crond

# View system crontab
cat /etc/crontab

# View cron logs
grep CRON /var/log/syslog        # Ubuntu
cat /var/log/cron                 # CentOS

# === ANACRON ===

# View anacron config
cat /etc/anacrontab

# Test anacrontab for syntax errors
sudo anacron -T

# Force anacron to run all jobs now
sudo anacron -fn

# Run anacron in debug/foreground mode
sudo anacron -d

# Update timestamps without running jobs
sudo anacron -u

# Check anacron timestamps (last run dates)
cat /var/spool/anacron/cron.daily
cat /var/spool/anacron/cron.weekly
cat /var/spool/anacron/cron.monthly

# View all timestamps at once
for f in /var/spool/anacron/*; do echo "$(basename $f): $(cat $f)"; done

# View anacron logs
grep anacron /var/log/syslog      # Ubuntu
grep anacron /var/log/cron        # CentOS
journalctl | grep anacron         # systemd-based
```

---

## What's Next?

You now know the classic tools: `at` for one-time jobs and `cron`/`anacron` for recurring jobs. In the next lesson, we'll explore the **modern approach** — `systemd` timers, which are replacing cron on many systems.

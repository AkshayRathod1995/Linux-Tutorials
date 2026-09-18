# Recurring Jobs: `cron` and `anacron`

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

### How `anacron` Works

`anacron` keeps a record of the **last time** each job ran. When the system starts up, it checks:

1. "When was this job last run?"
2. "Has enough time passed since then?"
3. If yes: "Run it now (after a short delay)."

### The `anacron` Configuration File

`anacron` is configured through `/etc/anacrontab`:

```bash
cat /etc/anacrontab
```

The format:

```
PERIOD   DELAY   JOB_ID   COMMAND
```

| Field     | Meaning                                                    |
| --------- | ---------------------------------------------------------- |
| `PERIOD`  | How often the job should run (in **days**)                 |
| `DELAY`   | Minutes to wait after boot before running (prevents overload) |
| `JOB_ID`  | A unique name for this job (used for tracking)             |
| `COMMAND` | The command or script to run                               |

### Example `/etc/anacrontab`:

```
# Period  Delay  Job ID           Command
1         5      daily-backup     /home/akshay/scripts/backup.sh
7         10     weekly-report    /home/akshay/scripts/report.sh
30        15     monthly-cleanup  /home/akshay/scripts/cleanup.sh
```

**Reading the first line:** "Run `backup.sh` every **1 day**. If the computer was off, run it **5 minutes** after boot. Track it with the ID `daily-backup`."

### Where `anacron` Tracks Last Run Times

`anacron` stores timestamps in `/var/spool/anacron/`:

```bash
ls /var/spool/anacron/
```

```
cron.daily
cron.monthly
cron.weekly
```

```bash
cat /var/spool/anacron/cron.daily
```

```
20260917
```

This tells `anacron`: "The daily jobs were last run on September 17, 2026."

### Running `anacron` Manually

```bash
# Check what anacron would do (dry run)
sudo anacron -T        # Test the anacrontab for errors
sudo anacron -n        # Run all jobs now, ignoring delays
sudo anacron -f        # Force run all jobs, ignoring timestamps
sudo anacron -u        # Update timestamps without running jobs
```

---

## `cron` vs `anacron`: The Complete Comparison

| Feature                  | `cron`                      | `anacron`                    |
| ------------------------ | --------------------------- | ---------------------------- |
| **Minimum interval**     | 1 minute                    | 1 day                        |
| **Runs missed jobs?**    | No                          | Yes                          |
| **Best for**             | Servers (always-on)         | Laptops/desktops (on/off)    |
| **Precision**            | Exact time (2:00 AM sharp)  | Approximate (sometime today) |
| **User crontabs?**       | Yes                         | No (system-wide only)        |
| **Daemon**               | `crond` (runs constantly)   | Triggered at boot/by cron    |
| **Config file**          | `crontab -e` or `/etc/crontab` | `/etc/anacrontab`        |

### How They Work Together

In most modern Linux distributions, `cron` and `anacron` work **together**:

1. `cron` triggers `anacron` (usually via a script in `/etc/cron.d/`)
2. `anacron` checks if daily/weekly/monthly jobs need to run
3. If they do, `anacron` runs them
4. This way, even if the machine was off, the jobs get caught up

---

## Quick Reference Summary

```bash
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

# View anacron config
cat /etc/anacrontab

# Test anacrontab for errors
sudo anacron -T

# Force anacron to run all jobs now
sudo anacron -fn

# Check anacron timestamps
cat /var/spool/anacron/cron.daily
```

---

## What's Next?

You now know the classic tools: `at` for one-time jobs and `cron`/`anacron` for recurring jobs. In the next lesson, we'll explore the **modern approach** — `systemd` timers, which are replacing cron on many systems.

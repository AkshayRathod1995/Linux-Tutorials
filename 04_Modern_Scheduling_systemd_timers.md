# Modern Scheduling: `systemd` Timers

## Why Are We Moving Beyond `cron`?

`cron` has been around since the 1970s. It works, and it works well. But it has some limitations:

| Limitation of `cron`                     | How `systemd` timers fix it                         |
| ---------------------------------------- | --------------------------------------------------- |
| Hard to debug (logs are scattered)       | Full logging via `journalctl`                       |
| No dependency management                 | Can depend on other services/conditions             |
| No built-in error handling               | Automatic restarts, failure detection               |
| Can't easily track if a job is running   | `systemctl status` shows current state              |
| Runs in a minimal environment            | Full service environment with control               |
| No centralized management                | Managed alongside all other system services          |

Modern distributions like **Ubuntu**, **CentOS/RHEL 7+**, **Fedora**, and **Arch Linux** all use `systemd` as their init system. Since `systemd` is already managing your services, it makes sense to let it manage your scheduled tasks too.

---

## The Core Concept: Two Files Working Together

A `systemd` timer requires **two files**:

```
1. A .service file  →  WHAT to run (the actual task)
2. A .timer file    →  WHEN to run it (the schedule)
```

Think of it like a restaurant:

- The **`.service` file** is the recipe (what to cook)
- The **`.timer` file** is the reservation (when to cook it)

Both files must have the **same base name**. For example:

```
my-backup.service    ← What: runs the backup script
my-backup.timer      ← When: every day at 2 AM
```

---

## Step-by-Step: Creating Your First Timer

Let's create a timer that runs a backup script every day at 2:00 AM.

### Step 1: Create the script you want to run

```bash
sudo mkdir -p /opt/scripts
sudo nano /opt/scripts/backup.sh
```

```bash
#!/bin/bash
echo "Backup started at $(date)" >> /var/log/my-backup.log
tar -czf /backups/home_$(date +%F).tar.gz /home/
echo "Backup completed at $(date)" >> /var/log/my-backup.log
```

Make it executable:

```bash
sudo chmod +x /opt/scripts/backup.sh
```

### Step 2: Create the `.service` file

Service files live in `/etc/systemd/system/`:

```bash
sudo nano /etc/systemd/system/my-backup.service
```

```ini
[Unit]
Description=Daily Home Directory Backup

[Service]
Type=oneshot
ExecStart=/opt/scripts/backup.sh
```

**What each line means:**

| Line                        | Meaning                                             |
| --------------------------- | --------------------------------------------------- |
| `[Unit]`                    | General information section                         |
| `Description=...`           | A human-readable name for this service              |
| `[Service]`                 | The service configuration section                   |
| `Type=oneshot`              | Run the command once and exit (not a long-running service) |
| `ExecStart=...`             | The command or script to execute                    |

### Step 3: Create the `.timer` file

```bash
sudo nano /etc/systemd/system/my-backup.timer
```

```ini
[Unit]
Description=Run backup every day at 2 AM

[Timer]
OnCalendar=*-*-* 02:00:00
Persistent=true

[Install]
WantedBy=timers.target
```

**What each line means:**

| Line                           | Meaning                                                        |
| ------------------------------ | -------------------------------------------------------------- |
| `[Unit]`                       | General information section                                    |
| `Description=...`              | A human-readable name for this timer                           |
| `[Timer]`                      | The timer configuration section                                |
| `OnCalendar=*-*-* 02:00:00`   | The schedule: every day at 02:00:00 (2 AM)                     |
| `Persistent=true`              | If the system was off at 2 AM, run the job when it boots up    |
| `[Install]`                    | How to enable this timer                                       |
| `WantedBy=timers.target`       | "Activate this timer when the system reaches the timers target" |

### Step 4: Enable and start the timer

```bash
# Reload systemd to pick up the new files
sudo systemctl daemon-reload

# Enable the timer (so it starts on every boot)
sudo systemctl enable my-backup.timer

# Start the timer right now
sudo systemctl start my-backup.timer
```

### Step 5: Verify it's working

```bash
# Check the timer's status
sudo systemctl status my-backup.timer
```

Output:

```
● my-backup.timer - Run backup every day at 2 AM
     Loaded: loaded (/etc/systemd/system/my-backup.timer; enabled)
     Active: active (waiting)
    Trigger: Fri 2026-09-19 02:00:00 UTC; 8h left
```

The key information: **Active: active (waiting)** means the timer is running and waiting for the next trigger time.

---

## Understanding `OnCalendar` Syntax

The `OnCalendar` line uses this format:

```
DayOfWeek Year-Month-Day Hour:Minute:Second
```

Here are practical examples:

| Schedule                  | `OnCalendar` Value                | Meaning                            |
| ------------------------- | --------------------------------- | ---------------------------------- |
| Every day at 2 AM         | `*-*-* 02:00:00`                  | Any year, any month, any day at 2  |
| Every Monday at 9 AM      | `Mon *-*-* 09:00:00`             | Every Monday at 9 AM               |
| First of every month      | `*-*-01 00:00:00`                | Day 1 of every month at midnight   |
| Every 15 minutes          | `*-*-* *:00/15:00`               | Every 15 minutes                   |
| Every hour                | `*-*-* *:00:00`                  | At minute 0 of every hour          |
| Weekdays at 6 PM          | `Mon..Fri *-*-* 18:00:00`        | Monday through Friday at 6 PM     |
| Jan 1st every year        | `*-01-01 00:00:00`               | New Year's Day at midnight         |
| Every 3 hours             | `*-*-* 00/3:00:00`               | At 00:00, 03:00, 06:00, 09:00...  |

### Testing Your `OnCalendar` Syntax

Use `systemd-analyze calendar` to test whether your schedule is correct:

```bash
systemd-analyze calendar "*-*-* 02:00:00"
```

Output:

```
  Original form: *-*-* 02:00:00
Normalized form: *-*-* 02:00:00
    Next elapse: Fri 2026-09-19 02:00:00 UTC
       (in UTC): Fri 2026-09-19 02:00:00 UTC
```

This tells you **exactly when** the next run will be. Incredibly useful for verifying your schedule.

```bash
systemd-analyze calendar "Mon *-*-* 09:00:00"
```

```
    Next elapse: Mon 2026-09-21 09:00:00 UTC
```

---

## Other Timer Types (Not Just Calendar-Based)

Besides `OnCalendar`, `systemd` timers support **relative/monotonic** timers:

| Directive              | Meaning                                          | Example            |
| ---------------------- | ------------------------------------------------ | ------------------ |
| `OnBootSec=`           | Run X time after system boot                     | `OnBootSec=5min`   |
| `OnUnitActiveSec=`     | Run X time after the timer was last activated     | `OnUnitActiveSec=1h` |
| `OnStartupSec=`        | Run X time after `systemd` started               | `OnStartupSec=10min` |
| `OnActiveSec=`         | Run X time after the timer unit is activated      | `OnActiveSec=30s`  |

Example — run a health check every 10 minutes:

```ini
[Timer]
OnBootSec=2min
OnUnitActiveSec=10min
```

This means: "Run 2 minutes after boot, then every 10 minutes after that."

---

## Managing `systemd` Timers

### List all active timers

```bash
systemctl list-timers
```

Output:

```
NEXT                         LEFT        LAST                         PASSED    UNIT                      ACTIVATES
Fri 2026-09-19 02:00:00 UTC  8h left     Thu 2026-09-18 02:00:00 UTC  15h ago  my-backup.timer           my-backup.service
Fri 2026-09-19 00:00:00 UTC  6h left     Thu 2026-09-18 00:00:00 UTC  17h ago  logrotate.timer           logrotate.service
```

This shows:
- **NEXT**: When the timer will fire next
- **LEFT**: How much time until it fires
- **LAST**: When it last ran
- **UNIT**: The timer's name
- **ACTIVATES**: The service it triggers

### Include inactive timers too

```bash
systemctl list-timers --all
```

### Check a timer's status

```bash
sudo systemctl status my-backup.timer
```

### Check the service's status (to see if the last run succeeded)

```bash
sudo systemctl status my-backup.service
```

### View logs for the service

```bash
journalctl -u my-backup.service
```

To see only the most recent logs:

```bash
journalctl -u my-backup.service -n 20
```

### Stop a timer

```bash
sudo systemctl stop my-backup.timer
```

### Disable a timer (prevent it from starting on boot)

```bash
sudo systemctl disable my-backup.timer
```

### Manually trigger the service (for testing)

```bash
sudo systemctl start my-backup.service
```

---

## `cron` vs `systemd` Timers: The Comparison

| Feature                    | `cron`                         | `systemd` Timer                       |
| -------------------------- | ------------------------------ | ------------------------------------- |
| **Setup complexity**       | One line in crontab            | Two files (.service + .timer)         |
| **Logging**                | Scattered in syslog            | Centralized via `journalctl`          |
| **Missed jobs (catch-up)** | No (need anacron)              | Yes (`Persistent=true`)              |
| **Dependencies**           | None                           | Can depend on network, mounts, etc.   |
| **Status checking**        | Hard (`pgrep`, log parsing)    | Easy (`systemctl status`)             |
| **Resource control**        | None                           | CPU/memory limits via cgroups         |
| **Minimum interval**       | 1 minute                       | 1 second                              |
| **Learning curve**         | Low (one line)                 | Medium (two files)                    |

**Bottom line:** `cron` is simpler for quick, basic tasks. `systemd` timers are better for production systems where you need reliability, logging, and control.

---

## Quick Reference Summary

```bash
# Create service file
sudo nano /etc/systemd/system/my-job.service

# Create timer file
sudo nano /etc/systemd/system/my-job.timer

# Reload after creating/editing files
sudo systemctl daemon-reload

# Enable and start a timer
sudo systemctl enable --now my-job.timer

# List all active timers
systemctl list-timers

# Check timer status
sudo systemctl status my-job.timer

# Check service status (last run result)
sudo systemctl status my-job.service

# View service logs
journalctl -u my-job.service

# Test your OnCalendar expression
systemd-analyze calendar "Mon *-*-* 09:00:00"

# Manually trigger the service
sudo systemctl start my-job.service

# Stop and disable
sudo systemctl stop my-job.timer
sudo systemctl disable my-job.timer
```

---

## What's Next?

You now know three ways to schedule jobs in Linux: `at`/`batch`, `cron`/`anacron`, and `systemd` timers. But what happens when things go wrong? In the next lesson, we'll learn how to **troubleshoot** failed scheduled jobs.

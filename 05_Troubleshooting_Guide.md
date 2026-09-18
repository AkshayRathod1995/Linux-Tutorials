# Troubleshooting Scheduled Jobs

## Why Do Scheduled Jobs Fail?

You've written the perfect cron job. The syntax is correct. You tested the script manually and it works. Then you schedule it, go to bed... and wake up to find it never ran, or it ran but produced errors.

This is **extremely common**, and it happens to everyone — even experienced admins. This guide covers the most common reasons scheduled jobs fail and exactly how to fix them.

---

## Pitfall #1: The PATH Problem (Most Common!)

### The Problem

When you type a command in your terminal, your shell knows where to find it because of the `PATH` environment variable. Your terminal might have a `PATH` like this:

```bash
echo $PATH
```

```
/usr/local/bin:/usr/bin:/bin:/home/akshay/.local/bin:/snap/bin
```

But when `cron` runs your job, it uses a **much shorter PATH**:

```
/usr/bin:/bin
```

So a command like `node`, `python3`, `docker`, or `aws` that works perfectly in your terminal might **not be found** when cron runs it.

### The Fix: Always Use Absolute Paths

Instead of writing this in your crontab:

```
# BAD - might not find python3
0 2 * * * python3 /home/akshay/scripts/report.py
```

Use the full path to the command:

```
# GOOD - uses the absolute path
0 2 * * * /usr/bin/python3 /home/akshay/scripts/report.py
```

**How to find the full path of any command:**

```bash
which python3       # Output: /usr/bin/python3
which node          # Output: /usr/local/bin/node
which docker        # Output: /usr/bin/docker
```

### Alternative Fix: Set PATH in Your Crontab

You can set the PATH at the top of your crontab:

```bash
crontab -e
```

```
PATH=/usr/local/bin:/usr/bin:/bin:/home/akshay/.local/bin

0 2 * * * python3 /home/akshay/scripts/report.py
```

---

## Pitfall #2: The Environment Problem

### The Problem

Your terminal session loads a rich environment from files like `~/.bashrc`, `~/.bash_profile`, and `~/.profile`. These set up:

- Environment variables (`JAVA_HOME`, `NODE_ENV`, `DATABASE_URL`)
- Aliases (`ll` for `ls -la`)
- Functions

**Cron loads NONE of these.** It runs in a bare, minimal environment.

### The Fix: Source Your Profile in the Script

Add this to the top of your script:

```bash
#!/bin/bash
source /home/akshay/.bashrc
# OR
source /home/akshay/.bash_profile

# Now your environment variables are available
echo "Database is at: $DATABASE_URL"
```

Or set the variables directly in the script:

```bash
#!/bin/bash
export DATABASE_URL="postgresql://localhost:5432/mydb"
export NODE_ENV="production"

/usr/local/bin/node /home/akshay/app/generate_report.js
```

### How to See Cron's Actual Environment

Create a diagnostic cron job:

```
* * * * * env > /tmp/cron_env.txt
```

Wait a minute, then compare:

```bash
# What cron sees
cat /tmp/cron_env.txt

# What your terminal sees
env > /tmp/terminal_env.txt

# Compare them
diff /tmp/cron_env.txt /tmp/terminal_env.txt
```

Remove the diagnostic job after you're done!

---

## Pitfall #3: Permission Issues

### The Problem

Your script works when you run it as yourself, but cron runs it and it fails because:

- The script file isn't executable
- The script tries to write to a directory the user doesn't own
- The script needs `sudo` but cron doesn't have a password prompt

### The Fix

**Make the script executable:**

```bash
chmod +x /home/akshay/scripts/backup.sh
```

**Check file ownership:**

```bash
ls -la /home/akshay/scripts/backup.sh
```

**Ensure the output directory is writable:**

```bash
# If the script writes to /var/log/myapp/
sudo mkdir -p /var/log/myapp
sudo chown akshay:akshay /var/log/myapp
```

**For tasks that need root privileges:**

Edit root's crontab instead of yours:

```bash
sudo crontab -e
```

```
0 2 * * * /usr/local/bin/system_backup.sh
```

---

## Pitfall #4: Forgetting to Redirect Output

### The Problem

By default, cron tries to **email** the output of your job to the user. If no mail system is configured (which is the case on most modern systems), the output is simply **lost**. You'll never know if your job printed errors.

### The Fix: Redirect Output to a Log File

**Capture both normal output AND errors:**

```
0 2 * * * /home/akshay/scripts/backup.sh >> /var/log/backup.log 2>&1
```

**Breaking this down:**

| Part                  | Meaning                                              |
| --------------------- | ---------------------------------------------------- |
| `>>`                  | Append standard output (stdout) to the log file      |
| `2>&1`                | Also send standard error (stderr) to the same file   |
| `> /var/log/backup.log` | Overwrite the log each time (use `>` instead of `>>`) |

**Capture output separately:**

```
0 2 * * * /home/akshay/scripts/backup.sh >> /var/log/backup.log 2>> /var/log/backup_errors.log
```

**Discard output entirely (when you truly don't care):**

```
0 2 * * * /home/akshay/scripts/backup.sh > /dev/null 2>&1
```

`/dev/null` is Linux's "black hole" — anything sent there disappears forever.

---

## Pitfall #5: Incorrect Crontab Syntax

### The Problem

A tiny syntax error can prevent your entire crontab from loading — or worse, make a job run at the wrong time.

### The Fix: Double-Check Your Syntax

Common mistakes:

```
# WRONG - using seconds (cron doesn't support seconds)
0 0 2 * * * /script.sh

# CORRECT - five fields only
0 2 * * * /script.sh
```

```
# WRONG - Day of week: 7 is not valid on all systems
0 9 * * 7 /script.sh

# CORRECT - Sunday is 0
0 9 * * 0 /script.sh
```

```
# WRONG - space in the path, unquoted
0 2 * * * /home/akshay/my scripts/backup.sh

# CORRECT - quote the path
0 2 * * * "/home/akshay/my scripts/backup.sh"
```

**Use online validators:** Websites like crontab.guru let you type a cron expression and see in plain English what it means.

---

## Pitfall #6: The Script Works Manually but Not in Cron

### The Debugging Checklist

When your script works in the terminal but not in cron, walk through this checklist:

```
[ ] 1. Are all paths ABSOLUTE? (not relative like ./file.txt)
[ ] 2. Is the script executable? (chmod +x)
[ ] 3. Does the script have the shebang line? (#!/bin/bash)
[ ] 4. Is the PATH set correctly? (use absolute command paths)
[ ] 5. Are required environment variables set?
[ ] 6. Does the user have permission to run the command?
[ ] 7. Is cron output being captured? (>> log 2>&1)
[ ] 8. Is the cron daemon running? (systemctl status crond)
[ ] 9. Is the user allowed to use cron? (check /etc/cron.allow)
[ ]10. Is there a newline at the end of the crontab?
```

That last one catches people all the time: **cron requires a newline at the end of the crontab file.** If your last line doesn't end with a newline, it may be silently ignored.

---

## Where to Check the Logs

### Cron Logs

Depending on your distribution, cron logs are in different places:

| Distribution           | Log Location                    | Command                        |
| ---------------------- | ------------------------------- | ------------------------------ |
| **Ubuntu/Debian**      | `/var/log/syslog`               | `grep CRON /var/log/syslog`    |
| **CentOS/RHEL**        | `/var/log/cron`                 | `cat /var/log/cron`            |
| **systemd-based**      | Journal                         | `journalctl -u cron`           |

### Viewing Cron Logs in Real Time

```bash
# Ubuntu/Debian
sudo tail -f /var/log/syslog | grep CRON

# CentOS/RHEL
sudo tail -f /var/log/cron
```

### `systemd` Timer Logs

```bash
# View logs for a specific timer/service
journalctl -u my-backup.service

# View only the last 50 lines
journalctl -u my-backup.service -n 50

# View logs since today
journalctl -u my-backup.service --since today

# Follow logs in real time
journalctl -u my-backup.service -f
```

### `at` Job Logs

`at` jobs are logged in the same syslog/cron log:

```bash
grep atd /var/log/syslog     # Ubuntu
grep atd /var/log/cron        # CentOS
```

---

## Pitfall #7: The Overlap Problem

### The Problem

Your cron job takes 10 minutes to run, but it's scheduled every 5 minutes. Now you have **two copies** running at the same time, fighting over the same files.

### The Fix: Use a Lock File (flock)

```bash
# In your crontab
*/5 * * * * /usr/bin/flock -n /tmp/backup.lock /home/akshay/scripts/backup.sh
```

**How it works:**
- `flock` tries to acquire a lock on `/tmp/backup.lock`
- `-n` means "don't wait" — if the lock is already held, just exit
- If a previous run is still going, the new one quietly skips

---

## Pitfall #8: Working Directory Confusion

### The Problem

Your script uses relative paths like `./data/input.csv` or `../config/settings.ini`. When you run it from the terminal, you're in the right directory. But cron runs it from a different directory (usually the user's home directory).

### The Fix

**Option A:** Use only absolute paths in your script:

```bash
#!/bin/bash
INPUT="/home/akshay/project/data/input.csv"
OUTPUT="/home/akshay/project/reports/output.pdf"
```

**Option B:** Change directory at the start of your script:

```bash
#!/bin/bash
cd /home/akshay/project || exit 1

# Now relative paths work
./process_data.sh
```

The `|| exit 1` means "if the `cd` fails, stop the script immediately."

---

## The Ultimate Debugging Template

When a cron job isn't working, add this wrapper to capture everything:

```bash
#!/bin/bash

# Log file for debugging
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

This tells you exactly:
- When the job ran
- Who it ran as
- Where it ran from
- What the PATH was
- Whether it succeeded (exit code 0) or failed (non-zero)

---

## Quick Reference: Troubleshooting Checklist

```
SYMPTOM: Job doesn't run at all
→ Is the daemon running? (systemctl status crond / atd)
→ Is the user allowed? (check /etc/cron.allow, /etc/cron.deny)
→ Is the crontab syntax correct? (crontab -l to verify)
→ Is there a newline at the end of the crontab?

SYMPTOM: Job runs but produces errors
→ Check the log file you redirected output to
→ Is the PATH correct? (use absolute paths)
→ Are environment variables set?
→ Does the user have permission to access the files?

SYMPTOM: Job runs but does nothing
→ Is the script executable? (chmod +x)
→ Does it have the shebang line? (#!/bin/bash)
→ Is the working directory correct?
→ Are relative paths the problem?

SYMPTOM: Job seems to run twice
→ Is it scheduled in both user crontab AND /etc/cron.d/?
→ Is it in both crontab and a cron.daily directory?
→ Use flock to prevent overlap
```

---

## What's Next?

Now that you can build and debug scheduled jobs, let's prepare you for the real world — **job interviews**. The next lesson covers the most common interview questions about Linux scheduling.

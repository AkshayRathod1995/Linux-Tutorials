# 30 Practical Exercises: Scheduling Jobs in Linux

## Instructions

These exercises are designed to be performed on a live Linux system (a virtual machine, cloud instance, or WSL works great). Work through them in order — they build on each other.

**Before you start:**

```bash
# Make a directory for exercise scripts
mkdir -p ~/scheduling-lab/scripts
mkdir -p ~/scheduling-lab/logs

# Ensure at and cron are installed and running
sudo apt install at cron -y          # Ubuntu/Debian
# OR
sudo yum install at cronie -y        # CentOS/RHEL

sudo systemctl start atd
sudo systemctl start cron            # or crond on CentOS
```

**Difficulty Legend:**

- Easy: Straightforward, single command
- Medium: Requires combining concepts
- Challenging: Requires problem-solving or multi-step setup

---

## Exercises 1-5: `at` and `batch`

### Exercise 1 (Easy)

**Task:** Schedule a one-time job using `at` that writes "Hello from the future!" to the file `~/scheduling-lab/logs/at_test.log` — set it to run 2 minutes from now.

After scheduling, verify the job is in the queue. Then wait for it to run and confirm the file was created.

---

### Exercise 2 (Easy)

**Task:** Schedule a job using `at` that runs at **5:00 PM tomorrow**. The job should write the current date and the system uptime to `~/scheduling-lab/logs/system_info.log`.

After scheduling, view the full details of the job (including the commands it will run) using its job number.

---

### Exercise 3 (Medium)

**Task:** Schedule three separate `at` jobs:
1. One to run **1 minute from now**
2. One to run **3 minutes from now**
3. One to run **5 minutes from now**

Each should append a line with its number and the current timestamp to `~/scheduling-lab/logs/sequence.log` (e.g., "Job 1 ran at Thu Sep 18 14:30:00 UTC 2026").

After scheduling all three, list the queue with `atq`. Then remove the **second** job from the queue and verify it's gone.

---

### Exercise 4 (Medium)

**Task:** Create a script at `~/scheduling-lab/scripts/disk_report.sh` that outputs:
- The current date
- The output of `df -h` (disk usage)
- The output of `free -m` (memory usage)

All output should go to `~/scheduling-lab/logs/disk_report.log`.

Then schedule this script with `at` to run at **midnight tonight** using the `-f` flag (reading commands from a file).

---

### Exercise 5 (Medium)

**Task:** Use the `batch` command to schedule a job that writes "Batch job executed when load was low at: [current date/time]" to `~/scheduling-lab/logs/batch_test.log`.

After scheduling, use `atq` to view the batch job in the queue. What letter designates batch jobs in the queue (as opposed to regular `at` jobs)?

---

## Exercises 6-15: `cron` (Writing Specific Schedules)

### Exercise 6 (Easy)

**Task:** Write a cron expression that runs a job **every day at 3:30 AM**. Add this to your crontab — the job should append the current date to `~/scheduling-lab/logs/daily.log`.

---

### Exercise 7 (Easy)

**Task:** Write a cron expression that runs a job **every 10 minutes**. Add this to your crontab — the job should append "Health check OK" and the current date to `~/scheduling-lab/logs/healthcheck.log`.

After adding it, list your crontab to confirm it was saved.

---

### Exercise 8 (Easy)

**Task:** Write cron expressions for each of the following schedules (you can write them in a text file or directly in your crontab — don't worry about the actual command, use `/bin/true` as a placeholder):

1. Every Sunday at 6:00 AM
2. The 1st of every month at midnight
3. Every weekday (Monday through Friday) at 8:45 AM
4. Every 30 minutes

---

### Exercise 9 (Medium)

**Task:** Write a single cron expression that runs a job at **9:00 AM, 12:00 PM, and 6:00 PM every day**. Add it to your crontab to write "Break time!" and the current time to `~/scheduling-lab/logs/breaks.log`.

---

### Exercise 10 (Medium)

**Task:** Write a cron expression that runs **every 15 minutes, but only during business hours (9 AM to 5 PM), and only on weekdays (Monday through Friday)**. Use it to append "Monitoring active" to `~/scheduling-lab/logs/monitor.log`.

---

### Exercise 11 (Medium)

**Task:** Create a script `~/scheduling-lab/scripts/cleanup.sh` that deletes all `.tmp` files from `~/scheduling-lab/logs/` that are older than 7 days. Schedule it with cron to run **every Sunday at 2:00 AM**.

Hint: Use the `find` command with the `-mtime` and `-delete` options.

---

### Exercise 12 (Medium)

**Task:** Use the `@reboot` cron shortcut to schedule a job that writes "System booted at: [date/time]" to `~/scheduling-lab/logs/boot.log` every time the system starts.

---

### Exercise 13 (Medium)

**Task:** You currently have several test entries in your crontab from the previous exercises. List your crontab, then carefully remove only the **every-10-minutes health check** from Exercise 7 (keep all other entries).

---

### Exercise 14 (Challenging)

**Task:** Create a cron job that runs every day at **1:00 AM** and:
1. Uses `flock` to prevent overlapping runs
2. Runs the script `~/scheduling-lab/scripts/disk_report.sh` (from Exercise 4)
3. Redirects both stdout and stderr to `~/scheduling-lab/logs/disk_report_cron.log`

Write the complete crontab line.

---

### Exercise 15 (Challenging)

**Task:** Examine the system crontab file at `/etc/crontab`. Note the format difference from user crontabs. Then look at what scripts exist in `/etc/cron.daily/` and pick one — read it to understand what daily maintenance Linux does automatically. Write a brief summary of what you found.

---

## Exercises 16-20: `anacron` and System Cron Directories

### Exercise 16 (Easy)

**Task:** View the current `anacron` configuration file. Identify:
1. What is the period for each configured job?
2. What is the delay for each job?
3. What are the job IDs?

---

### Exercise 17 (Easy)

**Task:** Check when `anacron` last ran its daily, weekly, and monthly jobs by examining the timestamp files in `/var/spool/anacron/`. What date does each file contain?

---

### Exercise 18 (Medium)

**Task:** Create a simple script `~/scheduling-lab/scripts/hello_anacron.sh` that appends "Anacron says hello at [date/time]" to `~/scheduling-lab/logs/anacron_test.log`. Make it executable.

Now, copy this script into the `/etc/cron.daily/` directory (you'll need sudo). Verify it's executable in that directory. This script will now run once daily via anacron.

---

### Exercise 19 (Medium)

**Task:** Test your anacrontab configuration for syntax errors using `anacron -T`. Then perform a dry run to see what anacron **would** do without actually running any jobs.

Hint: Check the `anacron` man page for the test and dry-run flags.

---

### Exercise 20 (Challenging)

**Task:** Trace how `cron` and `anacron` work together on your system:

1. Look in `/etc/cron.d/` or `/etc/crontab` for the entry that triggers `anacron`
2. Read the anacron trigger script to understand what it does
3. Check `/etc/anacrontab` to see what jobs anacron manages

Write a brief explanation of the chain: "cron triggers X, which runs Y, which checks Z..."

---

## Exercises 21-25: `systemd` Timers

### Exercise 21 (Easy)

**Task:** List all currently active `systemd` timers on your system. Identify:
1. How many timers are active?
2. Which timer will fire next?
3. Which timer ran most recently?

---

### Exercise 22 (Medium)

**Task:** Use `systemd-analyze calendar` to verify the following `OnCalendar` expressions. For each one, note when the next trigger would be:

1. `*-*-* 06:00:00`
2. `Mon *-*-* 09:30:00`
3. `*-*-01 00:00:00`
4. `Mon..Fri *-*-* 17:00:00`

---

### Exercise 23 (Medium)

**Task:** Create a complete `systemd` timer that runs every hour. You need to:

1. Create a script: `~/scheduling-lab/scripts/hourly_log.sh` that appends the current date and load average (`uptime`) to `~/scheduling-lab/logs/hourly.log`
2. Create a service file: `/etc/systemd/system/hourly-log.service`
3. Create a timer file: `/etc/systemd/system/hourly-log.timer`
4. Reload `systemd`, enable and start the timer
5. Verify it appears in `systemctl list-timers`

---

### Exercise 24 (Medium)

**Task:** Check the status of the timer you created in Exercise 23. Then manually trigger the associated service (without waiting for the timer) and verify that the log file was written to. Check the journal logs for the service.

---

### Exercise 25 (Challenging)

**Task:** Create a `systemd` timer with the following requirements:
- Runs every weekday (Monday through Friday) at 8:00 AM
- If the system was powered off at 8:00 AM, it should catch up when it boots
- The service should run as your user (not root)
- The service should execute `~/scheduling-lab/scripts/disk_report.sh`

Create both the `.service` and `.timer` files, enable the timer, and verify its next trigger time.

---

## Exercises 26-30: Troubleshooting and Log Checking

### Exercise 26 (Easy)

**Task:** Check the cron logs on your system. Find evidence that your cron jobs from the earlier exercises actually ran (or attempted to run). Which log file did you check, and what did you find?

---

### Exercise 27 (Medium)

**Task:** Create a script `~/scheduling-lab/scripts/broken.sh` that intentionally fails (for example, it tries to read a file that doesn't exist). Schedule it with cron to run every minute.

**Then troubleshoot it:**
1. Add proper output redirection to capture the error
2. Check the log to see the error message
3. Fix the script
4. Verify the fix works
5. Remove the cron job when done

---

### Exercise 28 (Medium)

**Task:** Create a cron job that demonstrates the PATH problem:
1. Write a crontab entry that tries to use a command **without** its full path (e.g., just `python3` instead of `/usr/bin/python3`)
2. Redirect the output and errors to a log file
3. Check the log — what error do you see?
4. Fix it by using the absolute path
5. Verify the fix

---

### Exercise 29 (Challenging)

**Task:** Create the "ultimate debugging wrapper" — a script at `~/scheduling-lab/scripts/debug_wrapper.sh` that:

1. Logs the start time
2. Logs the current user (`whoami`)
3. Logs the current working directory (`pwd`)
4. Logs the current PATH
5. Runs a command passed as an argument (`$1`)
6. Logs whether the command succeeded or failed (exit code)
7. Logs the end time

Test it by scheduling it with cron to run your `disk_report.sh` script:

```
* * * * * ~/scheduling-lab/scripts/debug_wrapper.sh ~/scheduling-lab/scripts/disk_report.sh
```

Check the debug output after it runs. Then remove the cron job.

---

### Exercise 30 (Challenging)

**Task:** Perform a full audit of all scheduled jobs on your system:

1. List all user cron jobs for every user on the system (hint: loop through `/var/spool/cron/` or use `sudo crontab -u USER -l` for key users)
2. List all system cron jobs (`/etc/crontab` and `/etc/cron.d/`)
3. List all scripts in `/etc/cron.hourly/`, `/etc/cron.daily/`, `/etc/cron.weekly/`, `/etc/cron.monthly/`
4. List all active `systemd` timers
5. List all pending `at` jobs

Combine all of this into a single audit script at `~/scheduling-lab/scripts/audit_all_jobs.sh` that outputs a clean summary.

---

---

# Answer Key

## Exercise 1

```bash
# Schedule the job (2 minutes from now)
at now + 2 minutes
at> echo "Hello from the future!" > ~/scheduling-lab/logs/at_test.log
at> <Ctrl+D>

# Verify it's in the queue
atq

# After 2 minutes, check the file
cat ~/scheduling-lab/logs/at_test.log
```

---

## Exercise 2

```bash
# Schedule the job
at 5:00 PM tomorrow
at> echo "Date: $(date)" > ~/scheduling-lab/logs/system_info.log
at> echo "Uptime: $(uptime)" >> ~/scheduling-lab/logs/system_info.log
at> <Ctrl+D>

# View the job details (replace 1 with your actual job number from atq)
atq
at -c 1
```

---

## Exercise 3

```bash
# Schedule three jobs
echo 'echo "Job 1 ran at $(date)" >> ~/scheduling-lab/logs/sequence.log' | at now + 1 minute
echo 'echo "Job 2 ran at $(date)" >> ~/scheduling-lab/logs/sequence.log' | at now + 3 minutes
echo 'echo "Job 3 ran at $(date)" >> ~/scheduling-lab/logs/sequence.log' | at now + 5 minutes

# List the queue
atq

# Remove the second job (replace 2 with its actual job number)
atrm 2

# Verify it's gone
atq
```

---

## Exercise 4

```bash
# Create the script
cat > ~/scheduling-lab/scripts/disk_report.sh << 'EOF'
#!/bin/bash
echo "=== Disk Report: $(date) ===" > ~/scheduling-lab/logs/disk_report.log
echo "" >> ~/scheduling-lab/logs/disk_report.log
echo "--- Disk Usage ---" >> ~/scheduling-lab/logs/disk_report.log
df -h >> ~/scheduling-lab/logs/disk_report.log
echo "" >> ~/scheduling-lab/logs/disk_report.log
echo "--- Memory Usage ---" >> ~/scheduling-lab/logs/disk_report.log
free -m >> ~/scheduling-lab/logs/disk_report.log
EOF

# Make it executable
chmod +x ~/scheduling-lab/scripts/disk_report.sh

# Schedule with at using -f
at midnight -f ~/scheduling-lab/scripts/disk_report.sh
```

---

## Exercise 5

```bash
# Schedule a batch job
batch
at> echo "Batch job executed when load was low at: $(date)" >> ~/scheduling-lab/logs/batch_test.log
at> <Ctrl+D>

# Check the queue
atq
```

**Answer:** Batch jobs show the queue letter `b` in the `atq` output, while regular `at` jobs show `a`.

---

## Exercise 6

```bash
crontab -e
# Add this line:
30 3 * * * echo "Daily job ran at $(date)" >> ~/scheduling-lab/logs/daily.log
```

---

## Exercise 7

```bash
crontab -e
# Add this line:
*/10 * * * * echo "Health check OK - $(date)" >> ~/scheduling-lab/logs/healthcheck.log

# Verify
crontab -l
```

---

## Exercise 8

```bash
crontab -e
# Add these lines:

# 1. Every Sunday at 6:00 AM
0 6 * * 0 /bin/true

# 2. The 1st of every month at midnight
0 0 1 * * /bin/true

# 3. Every weekday (Mon-Fri) at 8:45 AM
45 8 * * 1-5 /bin/true

# 4. Every 30 minutes
*/30 * * * * /bin/true
```

---

## Exercise 9

```bash
crontab -e
# Add this line:
0 9,12,18 * * * echo "Break time! $(date +%H:%M)" >> ~/scheduling-lab/logs/breaks.log
```

---

## Exercise 10

```bash
crontab -e
# Add this line:
*/15 9-17 * * 1-5 echo "Monitoring active - $(date)" >> ~/scheduling-lab/logs/monitor.log
```

---

## Exercise 11

```bash
# Create the cleanup script
cat > ~/scheduling-lab/scripts/cleanup.sh << 'EOF'
#!/bin/bash
find ~/scheduling-lab/logs/ -name "*.tmp" -mtime +7 -delete
echo "Cleanup completed at $(date)" >> ~/scheduling-lab/logs/cleanup.log
EOF

chmod +x ~/scheduling-lab/scripts/cleanup.sh

# Add to crontab
crontab -e
# Add this line:
0 2 * * 0 ~/scheduling-lab/scripts/cleanup.sh
```

---

## Exercise 12

```bash
crontab -e
# Add this line:
@reboot echo "System booted at: $(date)" >> ~/scheduling-lab/logs/boot.log
```

---

## Exercise 13

```bash
# View current crontab
crontab -l

# Edit and remove only the health check line
crontab -e
# Delete the line: */10 * * * * echo "Health check OK...
# Save and exit

# Verify
crontab -l
```

---

## Exercise 14

```bash
crontab -e
# Add this line:
0 1 * * * /usr/bin/flock -n /tmp/disk_report.lock ~/scheduling-lab/scripts/disk_report.sh >> ~/scheduling-lab/logs/disk_report_cron.log 2>&1
```

---

## Exercise 15

```bash
# View the system crontab
cat /etc/crontab

# List daily cron scripts
ls -la /etc/cron.daily/

# Read one (e.g., logrotate)
cat /etc/cron.daily/logrotate
# OR
cat /etc/cron.daily/apt-compat    # Ubuntu
```

**Example summary:** "The `logrotate` script in `/etc/cron.daily/` rotates, compresses, and removes old log files to prevent them from filling up the disk. It reads its configuration from `/etc/logrotate.conf` and files in `/etc/logrotate.d/`."

---

## Exercise 16

```bash
cat /etc/anacrontab
```

**Example output and answers:**
```
# Period  Delay  Job-ID          Command
1         5      cron.daily      nice run-parts /etc/cron.daily
7         25     cron.weekly     nice run-parts /etc/cron.weekly
@monthly  45     cron.monthly    nice run-parts /etc/cron.monthly
```

- Period: 1 day, 7 days, monthly
- Delay: 5 min, 25 min, 45 min
- Job IDs: cron.daily, cron.weekly, cron.monthly

---

## Exercise 17

```bash
cat /var/spool/anacron/cron.daily
cat /var/spool/anacron/cron.weekly
cat /var/spool/anacron/cron.monthly
```

Each file contains a date in `YYYYMMDD` format (e.g., `20260918`).

---

## Exercise 18

```bash
# Create the script
cat > ~/scheduling-lab/scripts/hello_anacron.sh << 'EOF'
#!/bin/bash
echo "Anacron says hello at $(date)" >> ~/scheduling-lab/logs/anacron_test.log
EOF

chmod +x ~/scheduling-lab/scripts/hello_anacron.sh

# Copy to cron.daily
sudo cp ~/scheduling-lab/scripts/hello_anacron.sh /etc/cron.daily/hello_anacron

# Verify it's executable
sudo chmod +x /etc/cron.daily/hello_anacron
ls -la /etc/cron.daily/hello_anacron
```

---

## Exercise 19

```bash
# Test anacrontab for syntax errors
sudo anacron -T

# Dry run — see what would run without executing
sudo anacron -t -n
# OR check the man page:
man anacron
```

Note: The `-T` flag tests for syntax errors. For a "what would happen" view, you can check timestamps and periods manually or use `-t` (some distributions support a test/dry-run mode — check your version's man page).

---

## Exercise 20

```bash
# Step 1: Find the cron entry that triggers anacron
cat /etc/cron.d/anacron
# OR look in /etc/crontab for anacron references

# Step 2: Read the trigger script (location varies by distro)
cat /etc/cron.d/anacron
# OR
cat /etc/anacrontab

# Step 3: Check what anacron manages
cat /etc/anacrontab
```

**Explanation chain:** "Cron runs a trigger script (typically at boot or on a schedule via `/etc/cron.d/anacron`) which invokes `anacron`. Anacron then checks its own config (`/etc/anacrontab`) and compares each job's period against the last-run timestamps in `/var/spool/anacron/`. If enough time has passed, it waits the configured delay, then executes the job (typically `run-parts /etc/cron.daily/` etc.)."

---

## Exercise 21

```bash
systemctl list-timers
```

Count the rows for the number of active timers. The **NEXT** column (earliest date) tells you which fires next. The **LAST** column (most recent date) tells you which ran most recently.

---

## Exercise 22

```bash
systemd-analyze calendar "*-*-* 06:00:00"
systemd-analyze calendar "Mon *-*-* 09:30:00"
systemd-analyze calendar "*-*-01 00:00:00"
systemd-analyze calendar "Mon..Fri *-*-* 17:00:00"
```

Each command outputs the **Next elapse** line showing when it would next trigger.

---

## Exercise 23

```bash
# Step 1: Create the script
cat > ~/scheduling-lab/scripts/hourly_log.sh << 'EOF'
#!/bin/bash
echo "$(date) - Load: $(uptime)" >> ~/scheduling-lab/logs/hourly.log
EOF
chmod +x ~/scheduling-lab/scripts/hourly_log.sh

# Step 2: Create the service file
sudo tee /etc/systemd/system/hourly-log.service > /dev/null << 'EOF'
[Unit]
Description=Hourly System Log Entry

[Service]
Type=oneshot
ExecStart=/home/akshay/scheduling-lab/scripts/hourly_log.sh
User=akshay
EOF

# Step 3: Create the timer file
sudo tee /etc/systemd/system/hourly-log.timer > /dev/null << 'EOF'
[Unit]
Description=Run hourly log every hour

[Timer]
OnCalendar=*-*-* *:00:00
Persistent=true

[Install]
WantedBy=timers.target
EOF

# Step 4: Reload, enable, and start
sudo systemctl daemon-reload
sudo systemctl enable hourly-log.timer
sudo systemctl start hourly-log.timer

# Step 5: Verify
systemctl list-timers | grep hourly
```

Note: Replace `akshay` with your actual username in the service file.

---

## Exercise 24

```bash
# Check timer status
sudo systemctl status hourly-log.timer

# Manually trigger the service
sudo systemctl start hourly-log.service

# Check the log file
cat ~/scheduling-lab/logs/hourly.log

# Check journal logs
journalctl -u hourly-log.service -n 10
```

---

## Exercise 25

```bash
# Create the service file
sudo tee /etc/systemd/system/weekday-report.service > /dev/null << 'EOF'
[Unit]
Description=Weekday Disk Report

[Service]
Type=oneshot
ExecStart=/home/akshay/scheduling-lab/scripts/disk_report.sh
User=akshay
EOF

# Create the timer file
sudo tee /etc/systemd/system/weekday-report.timer > /dev/null << 'EOF'
[Unit]
Description=Run disk report on weekday mornings

[Timer]
OnCalendar=Mon..Fri *-*-* 08:00:00
Persistent=true

[Install]
WantedBy=timers.target
EOF

# Reload, enable, and start
sudo systemctl daemon-reload
sudo systemctl enable weekday-report.timer
sudo systemctl start weekday-report.timer

# Verify the next trigger time
systemctl list-timers | grep weekday
systemd-analyze calendar "Mon..Fri *-*-* 08:00:00"
```

Note: Replace `akshay` with your actual username.

---

## Exercise 26

```bash
# Ubuntu/Debian
grep CRON /var/log/syslog | tail -20

# CentOS/RHEL
cat /var/log/cron | tail -20

# Systemd-based (any distro)
journalctl -u cron --since today
# OR
journalctl -u crond --since today
```

Look for lines showing your username and the commands from your crontab entries.

---

## Exercise 27

```bash
# Step 1: Create a broken script
cat > ~/scheduling-lab/scripts/broken.sh << 'EOF'
#!/bin/bash
cat /nonexistent/file/that/doesnt/exist.txt
echo "This should not appear if the above fails"
EOF
chmod +x ~/scheduling-lab/scripts/broken.sh

# Step 2: Schedule it (every minute, with output capture)
crontab -e
# Add:
* * * * * ~/scheduling-lab/scripts/broken.sh >> ~/scheduling-lab/logs/broken.log 2>&1

# Step 3: Wait 1 minute, then check the log
cat ~/scheduling-lab/logs/broken.log
# You should see: "cat: /nonexistent/file/that/doesnt/exist.txt: No such file or directory"

# Step 4: Fix the script
cat > ~/scheduling-lab/scripts/broken.sh << 'EOF'
#!/bin/bash
FILE="/etc/hostname"
if [ -f "$FILE" ]; then
    cat "$FILE"
else
    echo "ERROR: File $FILE not found"
fi
echo "Script completed successfully"
EOF

# Step 5: Wait 1 minute, check the log again for success, then remove the cron job
crontab -e
# Delete the line with broken.sh
```

---

## Exercise 28

```bash
# Step 1: Add a cron job WITHOUT full path
crontab -e
# Add:
* * * * * python3 -c "print('Hello from Python')" >> ~/scheduling-lab/logs/path_test.log 2>&1

# Step 2: Wait 1 minute, check the log
cat ~/scheduling-lab/logs/path_test.log
# Expected error: "/bin/sh: python3: not found" (or similar)

# Step 3: Find the full path
which python3

# Step 4: Fix it with the absolute path
crontab -e
# Change to:
* * * * * /usr/bin/python3 -c "print('Hello from Python')" >> ~/scheduling-lab/logs/path_test.log 2>&1

# Step 5: Wait 1 minute, verify it works, then remove the cron job
cat ~/scheduling-lab/logs/path_test.log
crontab -e
# Delete the line
```

---

## Exercise 29

```bash
# Create the debug wrapper script
cat > ~/scheduling-lab/scripts/debug_wrapper.sh << 'OUTER'
#!/bin/bash
LOG=~/scheduling-lab/logs/debug.log

echo "==========================================" >> "$LOG"
echo "DEBUG: Job started at: $(date)" >> "$LOG"
echo "DEBUG: Running as user: $(whoami)" >> "$LOG"
echo "DEBUG: Working directory: $(pwd)" >> "$LOG"
echo "DEBUG: PATH: $PATH" >> "$LOG"
echo "DEBUG: Running command: $1" >> "$LOG"
echo "==========================================" >> "$LOG"

# Run the command passed as argument
bash "$1" >> "$LOG" 2>&1
EXIT_CODE=$?

echo "DEBUG: Exit code: $EXIT_CODE" >> "$LOG"
echo "DEBUG: Job finished at: $(date)" >> "$LOG"
echo "" >> "$LOG"
OUTER

chmod +x ~/scheduling-lab/scripts/debug_wrapper.sh

# Schedule it
crontab -e
# Add:
* * * * * ~/scheduling-lab/scripts/debug_wrapper.sh ~/scheduling-lab/scripts/disk_report.sh

# Wait 1 minute, then check
cat ~/scheduling-lab/logs/debug.log

# Remove the cron job when done
crontab -e
# Delete the line
```

---

## Exercise 30

```bash
cat > ~/scheduling-lab/scripts/audit_all_jobs.sh << 'EOF'
#!/bin/bash

echo "============================================"
echo "  FULL SCHEDULED JOBS AUDIT"
echo "  Generated: $(date)"
echo "============================================"

echo ""
echo "--- 1. USER CRON JOBS ---"
for user in $(cut -f1 -d: /etc/passwd); do
    crontab_content=$(sudo crontab -u "$user" -l 2>/dev/null)
    if [ -n "$crontab_content" ] && ! echo "$crontab_content" | grep -q "no crontab"; then
        echo ""
        echo "  User: $user"
        echo "$crontab_content" | sed 's/^/    /'
    fi
done

echo ""
echo "--- 2. SYSTEM CRONTAB (/etc/crontab) ---"
grep -v '^#' /etc/crontab | grep -v '^$' | sed 's/^/    /'

echo ""
echo "--- 3. SYSTEM CRON.D (/etc/cron.d/) ---"
for file in /etc/cron.d/*; do
    echo "  File: $file"
    grep -v '^#' "$file" | grep -v '^$' | sed 's/^/    /'
done

echo ""
echo "--- 4. CRON DIRECTORY SCRIPTS ---"
echo "  Hourly:  $(ls /etc/cron.hourly/ 2>/dev/null | tr '\n' ', ')"
echo "  Daily:   $(ls /etc/cron.daily/ 2>/dev/null | tr '\n' ', ')"
echo "  Weekly:  $(ls /etc/cron.weekly/ 2>/dev/null | tr '\n' ', ')"
echo "  Monthly: $(ls /etc/cron.monthly/ 2>/dev/null | tr '\n' ', ')"

echo ""
echo "--- 5. ACTIVE SYSTEMD TIMERS ---"
systemctl list-timers --no-pager 2>/dev/null | sed 's/^/    /'

echo ""
echo "--- 6. PENDING AT JOBS ---"
atq 2>/dev/null | sed 's/^/    /'
if [ -z "$(atq 2>/dev/null)" ]; then
    echo "    (none)"
fi

echo ""
echo "============================================"
echo "  AUDIT COMPLETE"
echo "============================================"
EOF

chmod +x ~/scheduling-lab/scripts/audit_all_jobs.sh

# Run it
sudo ~/scheduling-lab/scripts/audit_all_jobs.sh
```

---

## Cleanup

When you're done with all exercises, clean up your test cron jobs and timers:

```bash
# Remove all your test cron jobs
crontab -r

# Stop and remove systemd timers
sudo systemctl stop hourly-log.timer weekday-report.timer
sudo systemctl disable hourly-log.timer weekday-report.timer
sudo rm /etc/systemd/system/hourly-log.service /etc/systemd/system/hourly-log.timer
sudo rm /etc/systemd/system/weekday-report.service /etc/systemd/system/weekday-report.timer
sudo systemctl daemon-reload

# Remove the test script from cron.daily
sudo rm /etc/cron.daily/hello_anacron

# Optionally remove the lab directory
# rm -rf ~/scheduling-lab
```

---

**Congratulations!** You've completed all 30 exercises. You now have hands-on experience with every major scheduling tool in Linux.

# One-Time Scheduling: `at` and `batch`

## Index

1. [What Is the `at` Command?](#what-is-the-at-command)
2. [Step 1: Install and Start the `atd` Daemon](#step-1-install-and-start-the-atd-daemon)
3. [Step 2: Scheduling Jobs with `at`](#step-2-scheduling-jobs-with-at)
4. [Step 3: Time Format Cheat Sheet](#step-3-time-format-cheat-sheet)
5. [Step 4: Managing Your `at` Jobs](#step-4-managing-your-at-jobs)
6. [Step 5: Controlling Who Can Use `at`](#step-5-controlling-who-can-use-at)
7. [The `batch` Command](#the-batch-command)
8. [Quick Reference Summary](#quick-reference-summary)
9. [What's Next?](#whats-next)

---

## What Is the `at` Command?

The `at` command lets you schedule a job to run **once** at a specific time in the future.

Think of it like setting a single alarm on your phone — it goes off once and then it's done.

**Use case:** "Restart the web server tonight at 11 PM after everyone has gone home."

---

## Step 1: Install and Start the `atd` Daemon

The `at` command needs its daemon (`atd`) to be running in the background, listening for your scheduled jobs.

### On Ubuntu/Debian:

```bash
sudo apt update
sudo apt install at -y
```

### On CentOS/RHEL/Fedora:

```bash
sudo yum install at -y
# or on newer versions:
sudo dnf install at -y
```

### Start and enable the daemon:

```bash
sudo systemctl start atd
sudo systemctl enable atd    # Makes it start automatically on boot
```

### Verify it's running:

```bash
sudo systemctl status atd
```

You should see **active (running)** in green.

---

## Step 2: Scheduling Jobs with `at`

### Basic Syntax

```
at [TIME]
```

When you type this command, `at` opens an interactive prompt where you type the commands you want to run. When you're done, press **Ctrl+D** to save and exit.

### Example 1: Run a command at a specific time today

```bash
at 10:30 PM
```

You'll see a prompt like this:

```
at> echo "Good night! Server maintenance starting." >> /tmp/maintenance.log
at> sudo systemctl restart nginx
at> <press Ctrl+D>
job 1 at Fri Sep 18 22:30:00 2026
```

**What happened?** You told Linux: "At 10:30 PM tonight, write a message to a log file and then restart Nginx."

### Example 2: Schedule for a specific date

```bash
at 9:00 AM December 25
```

```
at> echo "Merry Christmas!" | mail -s "Holiday Greetings" team@company.com
at> <press Ctrl+D>
```

### Example 3: Use relative time (from now)

```bash
at now + 30 minutes
```

```
at> /home/akshay/scripts/cleanup.sh
at> <press Ctrl+D>
```

Other relative time examples:

```bash
at now + 2 hours
at now + 3 days
at now + 1 week
```

### Example 4: Using keywords

```bash
at midnight              # 12:00 AM tonight
at noon                  # 12:00 PM today
at teatime               # 4:00 PM today (yes, this is real!)
at noon + 2 days         # Noon, two days from now
```

### Example 5: Feed a command without the interactive prompt

If you don't want to use the interactive prompt, you can pipe a command:

```bash
echo "/home/akshay/scripts/backup.sh" | at 2:00 AM
```

Or use a **here-string**:

```bash
at 2:00 AM <<< "/home/akshay/scripts/backup.sh"
```

Or read commands from a file:

```bash
at 2:00 AM -f /home/akshay/scripts/backup.sh
```

---

## Step 3: Time Format Cheat Sheet

Here's a quick reference for all the ways you can specify time with `at`:

| Format                      | Meaning                            |
| --------------------------- | ---------------------------------- |
| `at 3:00 PM`                | Today at 3:00 PM                   |
| `at 15:00`                  | Today at 3:00 PM (24-hour format)  |
| `at 3:00 PM tomorrow`       | Tomorrow at 3:00 PM               |
| `at 3:00 PM + 3 days`       | Three days from now at 3:00 PM    |
| `at 4pm + 3 days`           | Three days from now at 4:00 PM    |
| `at now + 1 hour`           | One hour from right now            |
| `at now + 45 minutes`       | 45 minutes from now                |
| `at midnight`               | Tonight at 12:00 AM               |
| `at noon`                   | Today at 12:00 PM                 |
| `at teatime`                | Today at 4:00 PM                  |
| `at 10:00 AM Jul 4`         | July 4th at 10:00 AM              |
| `at 10:00 AM 07/04/2027`    | July 4th, 2027 at 10:00 AM        |

---

## Step 4: Managing Your `at` Jobs

### View the job queue: `atq`

To see all your pending (waiting) jobs:

```bash
atq
```

Output looks like this:

```
2   Fri Sep 18 22:30:00 2026 a akshay
3   Sun Dec 25 09:00:00 2026 a akshay
5   Fri Sep 18 18:15:00 2026 a akshay
```

**Reading the output:**
- First column (`2`, `3`, `5`): The **job number**
- Date and time: When the job will run
- `a`: The queue name (default is `a`)
- `akshay`: The user who created the job

### View the actual commands in a job: `at -c`

```bash
at -c 2
```

This prints out everything that job #2 will do — including the full environment it will run in.

### Remove a job: `atrm`

Changed your mind? Remove a job by its number:

```bash
atrm 2
```

Verify it's gone:

```bash
atq
```

---

## Step 5: Controlling Who Can Use `at`

System administrators can control which users are allowed to schedule `at` jobs:

| File              | Purpose                                            |
| ----------------- | -------------------------------------------------- |
| `/etc/at.allow`   | If this file exists, **only** users listed in it can use `at` |
| `/etc/at.deny`    | If this file exists, users listed in it are **blocked** from using `at` |

**Rules:**
1. If `/etc/at.allow` exists, only users listed in it can use `at` (everyone else is denied).
2. If `/etc/at.allow` does NOT exist, then `/etc/at.deny` is checked. Users listed there are denied.
3. If neither file exists, only `root` can use `at`.

---

## The `batch` Command

### What Is `batch`?

`batch` is the polite cousin of `at`. Instead of running a job at a specific time, it says:

> "Run this job whenever the system isn't busy."

Technically, `batch` runs the job when the **system load average** drops below **0.8** (on a single-CPU system) or a configured threshold.

### Why Use `batch`?

Imagine you have a CPU-intensive report to generate. You don't want it to slow down the server while 100 users are logged in. With `batch`, you say:

> "Generate this report, but wait until the server has some free time."

### Syntax

`batch` works exactly like `at`, except you don't specify a time:

```bash
batch
```

```
at> /home/akshay/scripts/generate_big_report.sh
at> <press Ctrl+D>
```

Or with a pipe:

```bash
echo "/home/akshay/scripts/generate_big_report.sh" | batch
```

### Key Differences: `at` vs `batch`

| Feature            | `at`                              | `batch`                                   |
| ------------------ | --------------------------------- | ----------------------------------------- |
| **When it runs**   | At the exact time you specify     | When system load is low enough            |
| **Time required?** | Yes, you must give a time         | No, the system decides when               |
| **Best for**       | Time-sensitive one-time tasks     | CPU-heavy tasks that can wait             |
| **Example**        | "Restart the server at 11 PM"     | "Generate this report when the server is free" |

---

## Quick Reference Summary

```bash
# Schedule a job at a specific time
at 2:00 AM

# Schedule a job with relative time
at now + 30 minutes

# Schedule from a file
at 4pm -f /path/to/script.sh

# Pipe a command into at
echo "command" | at midnight

# View all pending jobs
atq

# View details of a specific job
at -c JOB_NUMBER

# Remove a pending job
atrm JOB_NUMBER

# Run a job when system load is low
batch

# Check if atd daemon is running
sudo systemctl status atd
```

---

## What's Next?

The `at` command is great for **one-time** tasks. But what if you need a job to run **every day at 2 AM** or **every Monday at 9 AM**? That's where `cron` comes in — and it's the most powerful scheduling tool in Linux.

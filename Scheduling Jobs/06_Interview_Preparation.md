# Job Interview Preparation: Linux Scheduling

## Index

1. [How to Use This Guide](#how-to-use-this-guide)
2. [Basic Questions (Entry-Level / Junior Sysadmin)](#basic-questions-entry-level--junior-sysadmin)
   - [Q1: What is job scheduling in Linux?](#q1-what-is-job-scheduling-in-linux)
   - [Q2: What is the difference between `at` and `cron`?](#q2-what-is-the-difference-between-at-and-cron)
   - [Q3: Explain the five fields in a crontab entry](#q3-explain-the-five-fields-in-a-crontab-entry)
   - [Q4: How do you list, edit, and remove a user's crontab?](#q4-how-do-you-list-edit-and-remove-a-users-crontab)
   - [Q5: What is a daemon?](#q5-what-is-a-daemon-name-the-daemons-related-to-job-scheduling)
3. [Intermediate Questions (Mid-Level Sysadmin / DevOps)](#intermediate-questions-mid-level-sysadmin--devops)
   - [Q6: What is `anacron` and how does it differ from `cron`?](#q6-what-is-anacron-and-how-does-it-differ-from-cron)
   - [Q7: Troubleshooting a cron job](#q7-your-cron-job-works-when-you-run-it-manually-but-fails-when-cron-runs-it-how-do-you-troubleshoot)
   - [Q8: User crontab vs system crontab](#q8-what-is-the-difference-between-a-user-crontab-and-the-system-crontab-etccrontab)
   - [Q9: Preventing overlapping cron jobs](#q9-how-do-you-prevent-two-instances-of-the-same-cron-job-from-overlapping)
   - [Q10: Restricting cron and at access](#q10-how-do-you-restrict-which-users-can-use-cron-and-at)
4. [Advanced Questions (Senior Sysadmin / DevOps / SRE)](#advanced-questions-senior-sysadmin--devops--sre)
   - [Q11: `systemd` timers vs `cron`](#q11-explain-systemd-timers-why-would-you-use-them-instead-of-cron)
   - [Q12: Writing complex cron schedules](#q12-write-a-cron-job-that-runs-every-weekday-at-830-am-and-every-saturday-at-noon)
   - [Q13: Scheduling in containers](#q13-how-would-you-schedule-a-job-in-a-containerized-dockerkubernetes-environment)
   - [Q14: Incident response for failed cron jobs](#q14-a-critical-cron-job-that-runs-nightly-didnt-execute-last-night-walk-me-through-your-incident-response)
   - [Q15: The `@reboot` directive](#q15-explain-the-reboot-directive-in-cron-when-would-you-use-it-and-when-would-you-avoid-it)
5. [Tips for Interview Success](#tips-for-interview-success)

---

## How to Use This Guide

These are the most commonly asked interview questions about Linux job scheduling, organized by difficulty. For each question, we provide:

- The **question** as an interviewer would ask it
- The **ideal answer** — concise, confident, and technically accurate

Practice saying these answers out loud. Interviewers want clear, structured responses — not rambling explanations.

---

## Basic Questions (Entry-Level / Junior Sysadmin)

### Q1: What is job scheduling in Linux?

**Answer:** Job scheduling is the process of automating the execution of commands or scripts at a specific time or on a recurring basis, without manual intervention. Linux provides tools like `at` for one-time jobs, `cron` for recurring jobs, and `systemd` timers as a modern alternative to cron.

---

### Q2: What is the difference between `at` and `cron`?

**Answer:** `at` is used to schedule a job that runs **once** at a specific time in the future — like a single alarm. `cron` is used for jobs that need to run **repeatedly** on a schedule — like a repeating alarm. For example, I'd use `at` to restart a server tonight at 11 PM, and `cron` to back up a database every night at 2 AM.

---

### Q3: Explain the five fields in a crontab entry.

**Answer:** The five fields represent, in order: **minute** (0-59), **hour** (0-23), **day of month** (1-31), **month** (1-12), and **day of week** (0-6, where 0 is Sunday). An asterisk (`*`) means "every." For example, `0 9 * * 1` means "at minute 0, hour 9, every day of the month, every month, on Monday."

---

### Q4: How do you list, edit, and remove a user's crontab?

**Answer:**
- **List:** `crontab -l` shows all cron jobs for the current user.
- **Edit:** `crontab -e` opens the crontab in the default editor.
- **Remove:** `crontab -r` deletes the entire crontab. I'd use `crontab -ri` for an interactive prompt before deleting.
- To manage another user's crontab as root: `sudo crontab -u username -e`.

---

### Q5: What is a daemon? Name the daemons related to job scheduling.

**Answer:** A daemon is a background process that runs continuously, waiting to perform tasks. The key scheduling daemons are: `crond` (handles recurring cron jobs), `atd` (handles one-time `at` jobs), and `systemd` (the modern init system that manages timers along with all other services).

---

## Intermediate Questions (Mid-Level Sysadmin / DevOps)

### Q6: What is `anacron` and how does it differ from `cron`?

**Answer:** `anacron` is designed for systems that are **not running 24/7**, like laptops and desktops. While `cron` skips a job if the system was off at the scheduled time, `anacron` tracks when each job was last run and ensures missed jobs are executed when the system next boots. The trade-off is that `anacron` only supports daily or longer intervals (not minutes or hours), and jobs run at approximate times rather than exact times. It's configured via `/etc/anacrontab` with the format: `period delay job-id command`.

---

### Q7: Your cron job works when you run it manually but fails when cron runs it. How do you troubleshoot?

**Answer:** This is almost always an environment issue. I would check these things in order:

1. **PATH** — cron uses a minimal PATH. I'd use absolute paths for all commands (find them with `which`).
2. **Environment variables** — cron doesn't load `.bashrc` or `.bash_profile`. Any required variables need to be set explicitly in the script or crontab.
3. **Permissions** — ensure the script is executable and the user has access to all referenced files and directories.
4. **Output** — redirect stdout and stderr to a log file (`>> /path/to/log 2>&1`) to capture error messages.
5. **Working directory** — cron doesn't run from the script's directory. I'd use `cd` at the top of the script or use absolute paths throughout.
6. **Logs** — check `/var/log/syslog` (Ubuntu) or `/var/log/cron` (CentOS) for cron execution records.

---

### Q8: What is the difference between a user crontab and the system crontab (`/etc/crontab`)?

**Answer:** A user crontab is managed with `crontab -e` and jobs run as that user. The system crontab at `/etc/crontab` has an **extra field** between the time fields and the command — the **username** — which specifies which user the job runs as. For example: `0 2 * * * root /usr/local/bin/backup.sh`. Additionally, system-wide cron jobs can be placed as scripts in `/etc/cron.daily/`, `/etc/cron.weekly/`, and `/etc/cron.monthly/` directories.

---

### Q9: How do you prevent two instances of the same cron job from overlapping?

**Answer:** I would use `flock` (file lock) to ensure only one instance runs at a time. In the crontab: `*/5 * * * * /usr/bin/flock -n /tmp/myjob.lock /path/to/script.sh`. The `-n` flag makes it non-blocking — if the lock is already held by a previous run, the new instance exits immediately instead of waiting.

---

### Q10: How do you restrict which users can use `cron` and `at`?

**Answer:** Both `cron` and `at` use an allow/deny file system:
- `/etc/cron.allow` and `/etc/at.allow` — if these exist, **only** listed users can use the respective tool.
- `/etc/cron.deny` and `/etc/at.deny` — if the allow file doesn't exist, users listed in the deny file are blocked.
- If neither file exists, typically only root can use the tool.

The allow file takes priority over the deny file.

---

## Advanced Questions (Senior Sysadmin / DevOps / SRE)

### Q11: Explain `systemd` timers. Why would you use them instead of `cron`?

**Answer:** `systemd` timers are the modern replacement for cron on `systemd`-based distributions. They consist of two unit files: a `.service` file (what to run) and a `.timer` file (when to run it). I'd choose timers over cron for several reasons:

- **Centralized logging** via `journalctl` instead of scattered syslog entries
- **Dependency management** — a timer can require network access or a mounted filesystem before running
- **Persistent timers** (`Persistent=true`) catch up missed runs without needing anacron
- **Resource control** — you can set CPU/memory limits using cgroup directives
- **Accurate status tracking** — `systemctl status` shows if the last run succeeded or failed

The trade-off is more setup (two files instead of one line), so for simple tasks, cron is still faster to configure.

---

### Q12: Write a cron job that runs every weekday at 8:30 AM and every Saturday at noon.

**Answer:** This requires two separate crontab entries because the schedules are different:

```
30 8 * * 1-5 /path/to/weekday_script.sh
0 12 * * 6 /path/to/saturday_script.sh
```

The first runs at minute 30, hour 8, on days 1-5 (Mon-Fri). The second runs at minute 0, hour 12, on day 6 (Saturday).

---

### Q13: How would you schedule a job in a containerized (Docker/Kubernetes) environment?

**Answer:** In containers, traditional cron is often problematic because containers are typically single-process. There are several approaches:

- **Kubernetes CronJobs** — the preferred method in K8s. You define a CronJob resource that creates a new Pod on the schedule.
- **Sidecar cron container** — run a lightweight cron container alongside your application container.
- **Host-level cron** — schedule `docker exec` or `kubectl exec` commands from the host's crontab.
- **External schedulers** — tools like Airflow, Temporal, or cloud-native options (AWS EventBridge, GCP Cloud Scheduler).

I'd avoid installing cron inside application containers because it adds complexity and goes against the single-process-per-container principle.

---

### Q14: A critical cron job that runs nightly didn't execute last night. Walk me through your incident response.

**Answer:** I'd follow this process:

1. **Check the logs** — `grep CRON /var/log/syslog` or `cat /var/log/cron` to see if cron attempted to run the job.
2. **Check the daemon** — `systemctl status crond` to verify cron is running.
3. **Check the crontab** — `crontab -l` to verify the job is still listed and the syntax is correct.
4. **Check system events** — was the server rebooted? Was there a resource issue? (`uptime`, `dmesg`, `journalctl --since yesterday`).
5. **Run the job manually** — execute the script and capture output to see if it fails.
6. **Check permissions** — did a recent change affect file permissions or user access?
7. **Remediate** — run the job manually if it's safe to do so, fix the root cause, and add monitoring/alerting to prevent recurrence (e.g., check for the expected output file and alert if it's missing).

---

### Q15: Explain the `@reboot` directive in cron. When would you use it and when would you avoid it?

**Answer:** `@reboot` runs a command **once** when the cron daemon starts, which typically coincides with system boot. I'd use it for tasks like:

- Starting a background application that doesn't have a systemd service file
- Running a one-time initialization script after reboot

I'd **avoid** it for production services because:

- It doesn't provide service management (restart on failure, dependency ordering)
- `systemd` services with `WantedBy=multi-user.target` are a better choice for anything that needs to start at boot reliably
- There's no guarantee about ordering relative to network availability, mounted filesystems, or other services

For any production service, I'd create a proper systemd service file with appropriate `After=` and `Requires=` directives instead.

---

## Tips for Interview Success

1. **Start with the "what"** — define the concept in one sentence.
2. **Give the "why"** — explain when and why you'd use it.
3. **Show the "how"** — mention the specific command or file.
4. **Mention trade-offs** — interviewers love hearing you weigh options.
5. **Use real examples** — "In my last role, I used this for..." beats textbook answers.
6. **It's okay to say "I'd look it up"** — for exact syntax, admitting you'd check the man page is honest and professional.

---

## What's Next?

Theory only gets you so far. The next and final lesson contains **30 hands-on exercises** to build muscle memory with every scheduling tool we've covered.

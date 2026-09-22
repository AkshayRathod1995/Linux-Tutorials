# Linux Job Scheduling

A complete guide to automating tasks on Linux. Covers one-time schedulers (`at`, `batch`), recurring jobs (`cron`, `anacron`), and modern systemd timers -- with real-world troubleshooting, hands-on labs, and interview preparation.

Whether you are setting up your first cron job or designing production-grade automation, this guide takes you from fundamentals through advanced scheduling and helps you confidently handle interview questions on the topic.

---

## Modules

| # | Module | Description |
|---|--------|-------------|
| 01 | [Fundamentals & Why Schedule](01_Fundamentals_and_Why_Schedule.md) | What job scheduling is, why it matters, and key vocabulary -- daemons, scripts, and jobs. |
| 02 | [One-Time Scheduling: at & batch](02_One_Time_Scheduling_at_and_batch.md) | Scheduling one-off tasks with `at` and `batch`, including installation, usage, and access control. |
| 03 | [Recurring Jobs: cron & anacron](03_Recurring_Jobs_cron_and_anacron.md) | Crontab syntax, system vs. user cron jobs, anacron for missed jobs, and how the two complement each other. |
| 04 | [Modern Scheduling: systemd Timers](04_Modern_Scheduling_systemd_timers.md) | Using systemd timers as a modern alternative to cron, with comparisons and step-by-step timer creation. |
| 05 | [Troubleshooting Guide](05_Troubleshooting_Guide.md) | Diagnosing common failures -- PATH issues, permissions, environment differences, and overlapping runs. |
| 06 | [Interview Preparation](06_Interview_Preparation.md) | Scheduling-focused Q&A organized by difficulty, from entry-level through senior SRE roles. |
| 07 | [Practical Exercises](07_Practical_Exercises.md) | 30 hands-on labs covering `at`, `cron`, `anacron`, systemd timers, and troubleshooting on a live system. |
| 08 | [How the System Uses Cron for /tmp](08_How_System_Uses_Cron_for_tmp.md) | How Linux cleans `/tmp` automatically using `tmpwatch`/`tmpreaper` and modern `systemd-tmpfiles`. |
| 09 | [Interview Revision Cheat Sheet](09_Interview_Revision_Cheat_Sheet.md) | Condensed single-page reference of all scheduling tools, syntax, commands, and config paths. |

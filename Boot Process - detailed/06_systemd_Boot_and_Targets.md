# Stage 6: systemd -- From initramfs to the Real OS

## Index

1. [What Is systemd?](#what-is-systemd)
2. [systemd as PID 1](#systemd-as-pid-1)
3. [The Two Phases of systemd During Boot](#the-two-phases-of-systemd-during-boot)
   - [Phase 1: systemd in initramfs](#phase-1-systemd-in-initramfs)
   - [Phase 2: systemd on the Real OS](#phase-2-systemd-on-the-real-os)
4. [Understanding systemd Targets](#understanding-systemd-targets)
   - [What Is a Target?](#what-is-a-target)
   - [The Main Boot Targets](#the-main-boot-targets)
   - [Target Dependency Chain](#target-dependency-chain)
5. [The Default Target](#the-default-target)
6. [systemd Unit Types](#systemd-unit-types)
7. [How systemd Resolves Dependencies](#how-systemd-resolves-dependencies)
8. [The Boot Process Through systemd Targets](#the-boot-process-through-systemd-targets)
9. [Switching Targets at Runtime](#switching-targets-at-runtime)
10. [Selecting a Target at Boot Time](#selecting-a-target-at-boot-time)
11. [Comparing systemd Targets to SysV Runlevels](#comparing-systemd-targets-to-sysv-runlevels)
12. [What's Next?](#whats-next)

---

## What Is systemd?

**systemd** is the **init system** and **service manager** for RHEL 7, 8, and 9. It is the first process started by the kernel (PID 1) and is responsible for bringing the entire system to life.

Think of systemd as the **general manager of a hotel**:

- It decides what order things happen (targets and dependencies)
- It starts all the workers (services)
- It monitors them (restarts crashed services)
- It shuts everything down in the right order when the hotel closes (shutdown)

systemd replaced the older **SysV init** system (used in RHEL 5 and earlier) and **Upstart** (briefly used in some distributions).

---

## systemd as PID 1

On every Linux system, the kernel creates exactly one userspace process first. That process gets **PID 1** (Process ID 1) and is responsible for starting everything else. On RHEL, PID 1 is systemd.

```bash
# Verify PID 1 is systemd
ps -p 1 -o comm=
```

```
systemd
```

```bash
# See the link: /sbin/init → systemd
ls -la /sbin/init
```

```
lrwxrwxrwx. 1 root root 22 Sep 15 10:30 /sbin/init -> ../lib/systemd/systemd
```

**PID 1 is special:**

| Property | Why It Matters |
|----------|---------------|
| **Cannot be killed** | If PID 1 dies, the kernel panics and the system crashes |
| **Adopts orphan processes** | When a parent process dies, its children are re-parented to PID 1 |
| **Handles signals differently** | PID 1 only responds to signals it explicitly registers for |
| **Must always be running** | It's the root of the entire process tree |

---

## The Two Phases of systemd During Boot

A key concept: systemd runs **twice** during the boot process -- once in initramfs, and once on the real system.

### Phase 1: systemd in initramfs

| Property | Detail |
|----------|--------|
| **When** | Immediately after the kernel unpacks initramfs |
| **Which systemd** | The systemd binary from inside the initramfs |
| **Default target** | `initrd.target` |
| **Purpose** | Mount the real root filesystem, prepare for switch_root |
| **Ends when** | `switch_root` replaces it with the real system's systemd |

**Targets used in the initramfs:**

```
initramfs systemd target chain:

  sysinit.target
      │
      ├── Load kernel modules (storage drivers, filesystem drivers)
      ├── Start udevd (detect hardware, create device nodes)
      ├── Set up /dev, /proc, /sys virtual filesystems
      └── Parse kernel command line (root=, rd.lvm.lv=, etc.)
      │
      ▼
  initrd-root-device.target
      │
      ├── Activate LVM (lvm2-activation)
      ├── Activate RAID (mdraid)
      ├── Unlock LUKS encryption (if needed)
      └── Wait for root block device to appear in /dev
      │
      ▼
  initrd-root-fs.target
      │
      ├── Run fsck on root device (if needed)
      └── Mount root filesystem on /sysroot (read-only)
      │
      ▼
  initrd-fs.target
      │
      └── Mount other early filesystems (/boot, etc.)
      │
      ▼
  initrd.target
      │
      └── Everything ready → trigger switch_root
      │
      ▼
  initrd-switch-root.target
      │
      └── systemd executes switch_root:
          1. Delete initramfs contents
          2. Make /sysroot the new /
          3. Re-execute systemd from the real OS
```

### Phase 2: systemd on the Real OS

| Property | Detail |
|----------|--------|
| **When** | After switch_root completes |
| **Which systemd** | The systemd binary from the real root filesystem (`/usr/lib/systemd/systemd`) |
| **Default target** | `multi-user.target` or `graphical.target` (configured by admin) |
| **Purpose** | Start all system services and reach the final operational state |
| **Ends when** | The system is fully booted and at the login prompt |

**After switch_root, systemd "re-executes" itself.** This means:

1. The initramfs systemd calls `systemctl switch-root /sysroot`
2. The kernel pivots the root filesystem
3. systemd re-reads its configuration from the **real** `/etc/systemd/`
4. systemd continues booting, now targeting `default.target`

```
systemd transition:

  [initramfs systemd]          [real OS systemd]
       PID 1                        PID 1
         │                            │
    initrd.target               default.target
         │                            │
    switch_root ──────────────►  re-exec systemd
                                      │
                                 basic.target
                                      │
                              multi-user.target
                                      │
                              graphical.target
                              (if configured)
```

---

## Understanding systemd Targets

### What Is a Target?

A **target** is a grouping of systemd units that represents a system state. Think of targets as **milestones** in the boot process:

> Imagine building a house. "Foundation complete" is a target. "Walls up" is a target. "Roof on" is a target. Each target depends on the previous one being complete. You can't put up walls without a foundation.

A target itself doesn't "do" anything -- it's a collection point. When systemd "reaches" a target, it means all the units that target requires or wants have been started.

### The Main Boot Targets

| Target | Purpose | Equivalent SysV Runlevel |
|--------|---------|-------------------------|
| `sysinit.target` | Early system initialization (mount filesystems, start udev) | (none) |
| `basic.target` | Basic system is functional (logging, timers, sockets ready) | (none) |
| `multi-user.target` | Full multi-user system, text-based login, network up | Runlevel 3 |
| `graphical.target` | Full system with graphical login (GUI) | Runlevel 5 |
| `rescue.target` | Single-user mode, minimal services, root password required | Runlevel 1 |
| `emergency.target` | Bare minimum, root filesystem read-only, root password required | Runlevel S |
| `reboot.target` | System is rebooting | Runlevel 6 |
| `poweroff.target` | System is shutting down | Runlevel 0 |

### Target Dependency Chain

Targets depend on each other in a chain. When you boot to `graphical.target`, systemd automatically pulls in all its dependencies:

```
                    graphical.target
                         │
                    Requires:
                         │
                  multi-user.target
                    │         │
              Requires:    Wants:
                    │         │
              basic.target  NetworkManager.service
                    │        sshd.service
              Requires:      firewalld.service
                    │        chronyd.service
              sysinit.target  crond.service
                    │         (and many more...)
              Requires:
                    │
              local-fs.target    swap.target    cryptsetup.target
                    │
              (mount all        (enable all     (unlock encrypted
               filesystems       swap devices)   volumes)
               from /etc/fstab)
```

```bash
# View the dependency tree of a target
systemctl list-dependencies graphical.target

# View only the target dependencies (filter)
systemctl list-dependencies graphical.target | grep target
```

```
graphical.target
● └─multi-user.target
●   ├─basic.target
●   │ ├─paths.target
●   │ ├─slices.target
●   │ ├─sockets.target
●   │ ├─sysinit.target
●   │ │ ├─cryptsetup.target
●   │ │ ├─local-fs.target
●   │ │ └─swap.target
●   │ └─timers.target
●   └─getty.target
```

---

## The Default Target

The **default target** is what systemd aims for during a normal boot. It's configured as a symbolic link:

```bash
# View the current default target
systemctl get-default
```

```
multi-user.target
```

```bash
# See the symlink
ls -la /etc/systemd/system/default.target
```

```
lrwxrwxrwx. 1 root root 41 Sep 15 10:30 /etc/systemd/system/default.target -> /usr/lib/systemd/system/multi-user.target
```

```bash
# Change the default target to graphical
sudo systemctl set-default graphical.target
```

```
Removed /etc/systemd/system/default.target.
Created symlink /etc/systemd/system/default.target -> /usr/lib/systemd/system/graphical.target.
```

```bash
# Change back to multi-user (text mode)
sudo systemctl set-default multi-user.target
```

| Default Target | What You Get at Boot |
|---------------|---------------------|
| `multi-user.target` | Text-based login prompt (common on servers) |
| `graphical.target` | Graphical login screen -- GDM/GNOME (common on desktops) |

---

## systemd Unit Types

systemd manages many types of units, not just services:

| Unit Type | Extension | Purpose | Example |
|-----------|-----------|---------|---------|
| **Service** | `.service` | A daemon or one-shot process | `sshd.service`, `httpd.service` |
| **Target** | `.target` | A group of units (a milestone) | `multi-user.target` |
| **Socket** | `.socket` | Socket-based activation | `sshd.socket` |
| **Mount** | `.mount` | Filesystem mount point | `boot.mount` (mounts `/boot`) |
| **Timer** | `.timer` | Scheduled execution (like cron) | `logrotate.timer` |
| **Device** | `.device` | A hardware device | `sys-subsystem-net-devices-eth0.device` |
| **Slice** | `.slice` | Resource control (cgroups) | `user.slice` |
| **Swap** | `.swap` | Swap partition | `dev-mapper-rhel\x2dswap.swap` |
| **Path** | `.path` | File/directory watch trigger | `cups.path` |
| **Scope** | `.scope` | External process group | `session-1.scope` |
| **Automount** | `.automount` | Auto-mount on access | `proc-sys-fs-binfmt_misc.automount` |

---

## How systemd Resolves Dependencies

systemd uses several dependency keywords:

| Keyword | Meaning |
|---------|---------|
| `Requires=` | Hard dependency -- if the required unit fails to start, this unit also fails |
| `Wants=` | Soft dependency -- if the wanted unit fails, this unit still starts |
| `After=` | Ordering -- start this unit AFTER the named unit |
| `Before=` | Ordering -- start this unit BEFORE the named unit |
| `Conflicts=` | Mutually exclusive -- stop the conflicting unit when this one starts |
| `BindsTo=` | Like Requires, but also stops this unit if the bound unit stops |

**Important distinction:**

- `Requires=` and `Wants=` control **what** gets started (dependency)
- `After=` and `Before=` control **when** things start (ordering)
- These are independent! `Requires=B` without `After=B` means A and B start **simultaneously**

```bash
# View a unit's dependencies
systemctl show sshd.service | grep -E "^(Requires|Wants|After|Before)="

# View what other units pull in a specific unit
systemctl show sshd.service | grep "WantedBy"
```

---

## The Boot Process Through systemd Targets

Here's the complete sequence after switch_root:

```
systemd (PID 1, from real OS) starts
│
├── Reads default.target (e.g., multi-user.target)
│
├── Resolves ALL dependencies recursively
│
├── Builds a transaction (ordered list of units to start/stop)
│
└── Begins parallel execution:
    │
    ├── local-fs-pre.target
    │   └── (prerequisites for mounting local filesystems)
    │
    ├── local-fs.target
    │   ├── Mount all filesystems from /etc/fstab
    │   ├── Start quota checks
    │   └── Enable swap
    │
    ├── sysinit.target
    │   ├── Set hostname
    │   ├── Load kernel modules from /etc/modules-load.d/
    │   ├── Apply sysctl settings from /etc/sysctl.d/
    │   ├── Set up LVM, RAID, encrypted volumes
    │   ├── Start udevd (hardware detection)
    │   ├── Start systemd-journald (logging)
    │   └── Generate machine-id if needed
    │
    ├── basic.target
    │   ├── Start systemd-logind (user session manager)
    │   ├── Start dbus (inter-process communication)
    │   ├── Start timers.target (scheduled tasks)
    │   ├── Start sockets.target (socket activation)
    │   └── Start paths.target (file watches)
    │
    ├── network-pre.target
    │   └── Firewall rules loaded (firewalld)
    │
    ├── network.target / network-online.target
    │   └── NetworkManager brings up network interfaces
    │
    ├── multi-user.target
    │   ├── sshd.service (SSH server)
    │   ├── crond.service (cron daemon)
    │   ├── chronyd.service (NTP time sync)
    │   ├── rsyslog.service (syslog daemon)
    │   ├── tuned.service (performance tuning)
    │   ├── getty@tty1.service (text login on tty1)
    │   └── (all other enabled services)
    │
    └── graphical.target (if default)
        └── display-manager.service (GDM → GNOME login screen)
```

**systemd starts units in parallel wherever possible.** If two units have no ordering dependency between them, systemd starts them at the same time. This is a major reason modern Linux boots faster than older SysV init systems.

---

## Switching Targets at Runtime

On a running system, you can switch to a different target:

```bash
# Switch to multi-user target (stop GUI, go to text mode)
sudo systemctl isolate multi-user.target

# Switch to graphical target (start GUI)
sudo systemctl isolate graphical.target

# Switch to rescue mode (minimal services, root password required)
sudo systemctl isolate rescue.target

# Switch to emergency mode (almost nothing running)
sudo systemctl isolate emergency.target
```

> **`isolate` means:** Start this target AND stop everything not needed by it. Only targets with `AllowIsolate=yes` in their unit file can be isolated.

```bash
# Check if a target allows isolation
systemctl cat rescue.target | grep AllowIsolate
```

```
AllowIsolate=yes
```

---

## Selecting a Target at Boot Time

To boot into a specific target **once** (without changing the default), modify the kernel command line in GRUB2:

1. Reboot → interrupt GRUB2 → press `e`
2. Find the `linux` line
3. Append `systemd.unit=rescue.target` (or any other target)
4. Press `Ctrl+x` to boot

**Common boot-time targets:**

| Append This | To Do This |
|-------------|-----------|
| `systemd.unit=rescue.target` | Rescue mode -- root password required, basic services running |
| `systemd.unit=emergency.target` | Emergency mode -- root password required, root FS read-only, almost nothing running |
| `rd.break` | Break into initramfs before switch_root (for password reset) |
| `init=/bin/bash` | Skip systemd entirely, drop to a root shell (dangerous, no services at all) |

**Rescue vs Emergency:**

| Feature | rescue.target | emergency.target |
|---------|--------------|-----------------|
| Root filesystem | Read-write | **Read-only** (must remount) |
| Services running | sysinit.target completed (logging, basic fs) | Almost none |
| Network | Not available | Not available |
| Use case | Fix service issues, broken configs | Fix `/etc/fstab`, corrupted filesystems |

---

## Comparing systemd Targets to SysV Runlevels

Older RHEL versions (5 and earlier) used "runlevels" instead of targets:

| SysV Runlevel | systemd Target | Purpose |
|--------------|---------------|---------|
| 0 | `poweroff.target` | Halt/Power off |
| 1 | `rescue.target` | Single-user/rescue mode |
| 2 | `multi-user.target` | Multi-user, no NFS (Debian-specific) |
| 3 | `multi-user.target` | Multi-user with networking, text mode |
| 4 | `multi-user.target` | Custom (unused by default) |
| 5 | `graphical.target` | Multi-user with GUI |
| 6 | `reboot.target` | Reboot |

```bash
# Legacy compatibility -- these commands still work on RHEL
runlevel               # Shows current and previous runlevel
who -r                 # Shows current runlevel
init 3                 # Switch to multi-user (legacy way)
init 5                 # Switch to graphical (legacy way)
telinit 3              # Same as init 3

# But the proper way is:
systemctl isolate multi-user.target
systemctl isolate graphical.target
```

---

## What's Next?

At this point, systemd has:

1. Re-executed itself on the real root filesystem
2. Read the default target
3. Resolved all dependencies
4. Started units in parallel toward the default target
5. Reached `basic.target`, then `multi-user.target`

In the next module, we'll look at how individual services are initialized, how the login prompt appears, and the difference between text-based and graphical login.

> **Remember:** systemd runs twice during boot -- once in the initramfs (to mount the real root) and once on the real OS (to start all services). The initramfs systemd's job is done once switch_root happens. The real OS systemd takes over and brings the system to its final state.

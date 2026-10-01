# Stage 7: Services, Targets & the Login Screen

## Index

1. [How Services Are Initialized](#how-services-are-initialized)
2. [Service Unit Files Explained](#service-unit-files-explained)
3. [Enabling and Disabling Services](#enabling-and-disabling-services)
4. [How Enabling a Service Works Internally](#how-enabling-a-service-works-internally)
5. [The Path to the Text Login (multi-user.target)](#the-path-to-the-text-login-multi-usertarget)
   - [getty -- The Text Login Prompt](#getty----the-text-login-prompt)
   - [How Login Authentication Works](#how-login-authentication-works)
6. [The Path to the Graphical Login (graphical.target)](#the-path-to-the-graphical-login-graphicaltarget)
   - [GDM -- The GNOME Display Manager](#gdm----the-gnome-display-manager)
7. [Boot Complete -- What's Running?](#boot-complete----whats-running)
8. [Monitoring Boot Performance](#monitoring-boot-performance)
9. [Essential Service Management Commands](#essential-service-management-commands)
10. [What's Next?](#whats-next)

---

## How Services Are Initialized

When systemd reaches a target like `multi-user.target`, it starts all services that are "wanted by" or "required by" that target. But how does systemd know which services to start?

**The answer is symlinks.** When you "enable" a service, systemd creates a symbolic link in the target's `.wants/` directory:

```
/etc/systemd/system/multi-user.target.wants/
├── sshd.service → /usr/lib/systemd/system/sshd.service
├── crond.service → /usr/lib/systemd/system/crond.service
├── chronyd.service → /usr/lib/systemd/system/chronyd.service
├── firewalld.service → /usr/lib/systemd/system/firewalld.service
├── NetworkManager.service → /usr/lib/systemd/system/NetworkManager.service
├── rsyslog.service → /usr/lib/systemd/system/rsyslog.service
├── tuned.service → /usr/lib/systemd/system/tuned.service
└── ...
```

When systemd processes `multi-user.target`, it:

1. Looks at all the symlinks in `multi-user.target.wants/`
2. Resolves each symlink to the actual unit file
3. Reads the unit file for dependencies and ordering
4. Starts the services (in parallel where possible)

```bash
# See what's "wanted by" multi-user.target
ls /etc/systemd/system/multi-user.target.wants/

# See what's wanted by graphical.target
ls /etc/systemd/system/graphical.target.wants/
```

---

## Service Unit Files Explained

A service unit file tells systemd HOW to start, stop, and manage a service. Let's look at the `sshd` service:

```bash
systemctl cat sshd.service
```

```ini
# /usr/lib/systemd/system/sshd.service
[Unit]
Description=OpenSSH server daemon
Documentation=man:sshd(8) man:sshd_config(5)
After=network.target sshd-keygen.target
Wants=sshd-keygen.target

[Service]
Type=notify
EnvironmentFile=-/etc/sysconfig/sshd
ExecStart=/usr/sbin/sshd -D $OPTIONS
ExecReload=/bin/kill -HUP $MAINPID
KillMode=process
Restart=on-failure
RestartSec=42s

[Install]
WantedBy=multi-user.target
```

| Section | Directive | Meaning |
|---------|-----------|---------|
| **[Unit]** | `Description=` | Human-readable name |
| | `After=network.target` | Start AFTER the network is up |
| | `Wants=sshd-keygen.target` | Also start SSH key generation (soft dependency) |
| **[Service]** | `Type=notify` | Service notifies systemd when it's ready |
| | `ExecStart=` | The command to start the service |
| | `ExecReload=` | The command to reload configuration (without restarting) |
| | `Restart=on-failure` | Automatically restart if the process crashes |
| | `RestartSec=42s` | Wait 42 seconds before restarting |
| **[Install]** | `WantedBy=multi-user.target` | When enabled, add this service to `multi-user.target` |

### Service Types

| Type | Behavior | Example |
|------|----------|---------|
| `simple` | Default. systemd considers it started immediately after `ExecStart` | `httpd` (in some configs) |
| `forking` | Process forks into the background. systemd waits for the fork | Traditional daemons |
| `oneshot` | Runs a command to completion, then exits. Used for setup tasks | `systemd-tmpfiles-clean` |
| `notify` | Service explicitly tells systemd when it's ready (via `sd_notify`) | `sshd`, `httpd` |
| `dbus` | Service is ready when it acquires a D-Bus name | `NetworkManager` |
| `idle` | Waits until all active jobs are done before starting | `getty` (login prompt) |

---

## Enabling and Disabling Services

```bash
# Enable a service (start at boot)
sudo systemctl enable httpd.service

# Disable a service (don't start at boot)
sudo systemctl disable httpd.service

# Enable AND start immediately
sudo systemctl enable --now httpd.service

# Disable AND stop immediately
sudo systemctl disable --now httpd.service

# Check if a service is enabled
systemctl is-enabled httpd.service
```

```
enabled       ← Will start at boot
disabled      ← Will NOT start at boot
static        ← Cannot be enabled/disabled directly (no [Install] section)
masked        ← Completely blocked, cannot be started at all
```

```bash
# Mask a service (prevent it from starting, even manually)
sudo systemctl mask httpd.service

# Unmask a service
sudo systemctl unmask httpd.service
```

---

## How Enabling a Service Works Internally

When you run `systemctl enable sshd.service`:

```
Before enable:
  /etc/systemd/system/multi-user.target.wants/
  (no sshd.service symlink)

Command:
  sudo systemctl enable sshd.service

After enable:
  /etc/systemd/system/multi-user.target.wants/
  └── sshd.service → /usr/lib/systemd/system/sshd.service
```

The `WantedBy=multi-user.target` line in the `[Install]` section tells systemd **where** to create the symlink. It's just a symlink -- nothing more.

When you `disable`:

```bash
sudo systemctl disable sshd.service
```

The symlink is simply **removed**. The service unit file itself is untouched.

---

## The Path to the Text Login (multi-user.target)

When the default target is `multi-user.target`, the system ends at a **text-based login prompt** on the console.

### getty -- The Text Login Prompt

The login prompt is provided by `getty` (Get TTY). On RHEL, systemd uses `agetty` (an alternative getty):

```bash
systemctl cat getty@tty1.service
```

The `getty@.service` is a **template unit** (note the `@` in the name). It spawns one instance per virtual terminal:

```
getty@tty1.service → Login prompt on virtual terminal 1 (Ctrl+Alt+F1)
getty@tty2.service → Login prompt on virtual terminal 2 (Ctrl+Alt+F2)
getty@tty3.service → Login prompt on virtual terminal 3 (Ctrl+Alt+F3)
...
getty@tty6.service → Login prompt on virtual terminal 6 (Ctrl+Alt+F6)
```

These are controlled by `getty.target`, which is pulled in by `multi-user.target`:

```
multi-user.target
    └── Wants: getty.target
                 └── Wants: getty@tty1.service
                            getty@tty2.service
                            ...
```

**What the login prompt looks like:**

```
Red Hat Enterprise Linux 9.2 (Plow)
Kernel 5.14.0-284.el9.x86_64 on an x86_64

servera login: _
```

### How Login Authentication Works

When you type a username and password at the login prompt:

```
1. getty displays "servera login: "
2. You type your username → getty spawns /bin/login
3. login displays "Password: "
4. You type your password
5. login calls PAM (Pluggable Authentication Modules)
6. PAM checks:
   ├── /etc/passwd (does user exist?)
   ├── /etc/shadow (is the password correct?)
   ├── /etc/pam.d/login (PAM rules for login)
   ├── /etc/security/ (security policies)
   └── SELinux (is the user authorized?)
7. If authentication succeeds:
   ├── login sets up the user's environment
   ├── login reads /etc/profile and ~/.bash_profile
   ├── login starts the user's shell (/bin/bash)
   └── You get your shell prompt: [user@servera ~]$
```

```bash
# View the PAM configuration for login
cat /etc/pam.d/login

# View all virtual consoles
systemctl list-units 'getty@*'
```

---

## The Path to the Graphical Login (graphical.target)

When the default target is `graphical.target`, the system boots to a **graphical login screen**.

### GDM -- The GNOME Display Manager

On RHEL, the graphical login is provided by **GDM** (GNOME Display Manager):

```
graphical.target
    └── Wants: display-manager.service
                    │
                    └── (which is a symlink to gdm.service)
```

```bash
# Check which display manager is configured
ls -la /etc/systemd/system/display-manager.service
```

```
lrwxrwxrwx. 1 root root 35 ... /etc/systemd/system/display-manager.service -> /usr/lib/systemd/system/gdm.service
```

**GDM startup sequence:**

```
1. systemd starts gdm.service
2. GDM starts the X server (Xorg) or Wayland compositor
3. GDM displays the graphical login screen
4. User selects their name and types password
5. GDM authenticates via PAM (same as text login)
6. GDM starts the GNOME session (or other desktop)
7. User sees their desktop
```

```bash
# Check GDM status
systemctl status gdm.service

# Switch between graphical and text mode at runtime
sudo systemctl isolate multi-user.target    # Go to text mode
sudo systemctl isolate graphical.target     # Go back to GUI
```

**Virtual terminal layout with graphical target:**

| Terminal | Accessed Via | What's There |
|----------|-------------|-------------|
| tty1 | Ctrl+Alt+F1 | GDM graphical login (or GNOME session) |
| tty2 | Ctrl+Alt+F2 | Text login prompt (getty) |
| tty3 | Ctrl+Alt+F3 | Text login prompt (getty) |
| tty4 | Ctrl+Alt+F4 | Text login prompt (getty) |
| tty5 | Ctrl+Alt+F5 | Text login prompt (getty) |
| tty6 | Ctrl+Alt+F6 | Text login prompt (getty) |

> On RHEL 9 with Wayland, the graphical session typically runs on **tty2**, with GDM on tty1. This may vary.

---

## Boot Complete -- What's Running?

Once the system reaches its default target, you can see what's running:

```bash
# List all running services
systemctl list-units --type=service --state=running
```

```
UNIT                        LOAD   ACTIVE SUB     DESCRIPTION
auditd.service              loaded active running Security Auditing Service
chronyd.service             loaded active running NTP client/server
crond.service               loaded active running Command Scheduler
dbus-broker.service         loaded active running D-Bus System Message Bus
firewalld.service           loaded active running firewalld - dynamic firewall daemon
NetworkManager.service      loaded active running Network Manager
polkit.service              loaded active running Authorization Manager
rsyslog.service             loaded active running System Logging Service
sshd.service                loaded active running OpenSSH server daemon
systemd-journald.service    loaded active running Journal Service
systemd-logind.service      loaded active running User Login Management
systemd-udevd.service       loaded active running Rule-based Manager for Device Events
tuned.service               loaded active running Dynamic System Tuning Daemon
```

```bash
# List all active targets
systemctl list-units --type=target --state=active
```

```bash
# Check which target was reached
systemctl get-default
```

```bash
# View boot log
journalctl -b
```

---

## Monitoring Boot Performance

systemd tracks how long each unit takes to start. This is incredibly useful for diagnosing slow boots:

```bash
# Overall boot timing
systemd-analyze
```

```
Startup finished in 1.512s (kernel) + 2.345s (initrd) + 8.234s (userspace) = 12.091s
graphical.target reached after 7.890s in userspace.
```

| Phase | What It Covers |
|-------|---------------|
| **kernel** | From kernel start to initramfs handoff |
| **initrd** | initramfs: load drivers, mount root, switch_root |
| **userspace** | systemd on real OS: start services to default target |

```bash
# Show which services took the longest to start
systemd-analyze blame
```

```
         5.234s NetworkManager-wait-online.service
         2.123s firewalld.service
         1.456s tuned.service
         0.987s sshd.service
         0.654s chronyd.service
...
```

```bash
# Generate an SVG diagram of the boot process (visual timeline)
systemd-analyze plot > /tmp/boot-plot.svg

# Show the critical chain (the longest dependency path)
systemd-analyze critical-chain
```

```
graphical.target @7.890s
└─multi-user.target @7.889s
  └─tuned.service @6.433s +1.456s
    └─polkit.service @5.678s +0.123s
      └─basic.target @5.677s
        └─sockets.target @5.676s
          └─dbus.socket @5.675s
            └─sysinit.target @5.674s
              └─systemd-journal-flush.service @4.123s +1.234s
```

The critical chain shows the **longest path** through the dependency graph -- the bottleneck of your boot process.

---

## Essential Service Management Commands

| Command | What It Does |
|---------|-------------|
| `systemctl start sshd` | Start a service now |
| `systemctl stop sshd` | Stop a service now |
| `systemctl restart sshd` | Stop and start a service |
| `systemctl reload sshd` | Reload config without restart (if supported) |
| `systemctl status sshd` | View service status, recent logs, PID |
| `systemctl enable sshd` | Start at boot (create symlink) |
| `systemctl disable sshd` | Don't start at boot (remove symlink) |
| `systemctl enable --now sshd` | Enable AND start immediately |
| `systemctl is-active sshd` | Check if running (`active` or `inactive`) |
| `systemctl is-enabled sshd` | Check if enabled (`enabled` or `disabled`) |
| `systemctl is-failed sshd` | Check if in failed state |
| `systemctl mask sshd` | Completely prevent starting |
| `systemctl unmask sshd` | Remove the mask |
| `systemctl list-units --failed` | List all units that failed to start |
| `systemctl daemon-reload` | Reload unit files after editing them |
| `systemctl cat sshd` | View the unit file contents |
| `systemctl show sshd` | Show all properties of a unit |
| `systemctl edit sshd` | Create an override file (drop-in) |
| `journalctl -u sshd` | View logs for a specific service |
| `journalctl -u sshd -f` | Follow logs in real-time |

---

## What's Next?

The system is now fully booted:

1. Firmware initialized hardware (POST)
2. GRUB2 loaded the kernel and initramfs
3. The kernel started, initramfs mounted the root filesystem
4. systemd transitioned from initramfs to the real OS
5. Services started, network came up
6. Login prompt appeared (text or graphical)

In the next module, we'll see a complete **visual flow diagram** of the entire boot process -- from power button to login screen -- for quick revision.

> **Remember:** The login screen is just another systemd service. `getty@tty1.service` for text login, `gdm.service` for graphical login. They're no different from any other service -- systemd starts them, monitors them, and restarts them if they crash.

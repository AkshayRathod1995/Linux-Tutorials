# Stage 6: systemd -- From initramfs to the Real OS

## What Is systemd?

The **init system** for RHEL 7+. It runs as **PID 1** and manages all processes, services, and the boot sequence. It replaces SysV init.

---

## Two Phases of systemd During Boot

systemd starts **twice** during a single boot:

| Phase | Where | Goal Target | What It Does |
|-------|-------|-------------|-------------|
| **Phase 1** | initramfs (in RAM) | `initrd.target` | Load drivers, activate LVM, mount root on /sysroot, `switch_root` |
| **Phase 2** | Real OS (on disk) | `default.target` | Mount all filesystems, start services, reach login |

After `switch_root`, systemd re-executes itself from the real OS's `/usr/lib/systemd/systemd`.

---

## Understanding Targets

A **target** = a group of units (services, mounts, etc.) that should be active together. Think of it as a milestone.

### Main Boot Targets (ordered)

```
sysinit.target          ← Mount /proc, /sys, udev, swap, load modules
       ↓
basic.target            ← Timers, paths, sockets, firewall
       ↓
multi-user.target       ← Networking, SSH, cron, all non-GUI services
       ↓
graphical.target        ← Display manager (GDM), desktop environment
```

### Special Targets

| Target | When to Use |
|--------|------------|
| `rescue.target` | Single-user root shell, minimal services, all filesystems mounted |
| `emergency.target` | Root shell, root filesystem mounted READ-ONLY, almost nothing started |
| `reboot.target` | Reboot |
| `poweroff.target` | Shutdown |

---

## Default Target

```bash
systemctl get-default                          # View current default
sudo systemctl set-default multi-user.target   # Server (no GUI)
sudo systemctl set-default graphical.target    # Desktop (GUI)
```

---

## Switching Targets

```bash
# At runtime (immediately switches)
sudo systemctl isolate rescue.target
sudo systemctl isolate multi-user.target

# At boot time (temporary, one boot only)
# GRUB2 → press 'e' → append to linux line:
systemd.unit=rescue.target
systemd.unit=emergency.target
```

---

## systemd Unit Types

| Type | Extension | Example |
|------|-----------|---------|
| Service | `.service` | `sshd.service` |
| Target | `.target` | `multi-user.target` |
| Mount | `.mount` | `boot.mount` |
| Socket | `.socket` | `sshd.socket` (on-demand activation) |
| Timer | `.timer` | `logrotate.timer` |

---

## SysV Runlevels → systemd Targets

| Runlevel | systemd Target | Meaning |
|----------|---------------|---------|
| 0 | `poweroff.target` | Halt |
| 1 | `rescue.target` | Single user |
| 3 | `multi-user.target` | Multi-user, no GUI |
| 5 | `graphical.target` | Multi-user + GUI |
| 6 | `reboot.target` | Reboot |

---

## Key Commands

```bash
systemctl list-units --type=target             # Active targets
systemctl list-dependencies multi-user.target  # What a target pulls in
systemd-analyze                                # Total boot time
systemd-analyze blame                          # Slowest units
systemd-analyze critical-chain                 # Boot-blocking dependency chain
journalctl -b                                  # Logs from current boot
journalctl -b -1                               # Logs from previous boot
```

**Next:** Services start and you reach the login screen.

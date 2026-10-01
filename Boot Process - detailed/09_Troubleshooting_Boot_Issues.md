# Troubleshooting Boot Issues on RHEL

## Index

1. [Why Do Boots Fail?](#why-do-boots-fail)
2. [Identifying Which Stage Failed](#identifying-which-stage-failed)
3. [Problem 1: GRUB2 Is Missing or Corrupted](#problem-1-grub2-is-missing-or-corrupted)
   - [Symptom: grub> or grub rescue> Prompt](#symptom-grub-or-grub-rescue-prompt)
   - [Symptom: "No bootable device" or "Operating system not found"](#symptom-no-bootable-device-or-operating-system-not-found)
4. [Problem 2: Kernel Panic -- Unable to Mount Root FS](#problem-2-kernel-panic----unable-to-mount-root-fs)
5. [Problem 3: initramfs Is Corrupted or Missing Drivers](#problem-3-initramfs-is-corrupted-or-missing-drivers)
6. [Problem 4: Bad /etc/fstab Entry](#problem-4-bad-etcfstab-entry)
7. [Problem 5: Forgotten Root Password](#problem-5-forgotten-root-password)
8. [Problem 6: System Drops to Emergency Shell](#problem-6-system-drops-to-emergency-shell)
9. [Problem 7: Service Failures Blocking Boot](#problem-7-service-failures-blocking-boot)
10. [Problem 8: SELinux Blocking Boot](#problem-8-selinux-blocking-boot)
11. [Problem 9: Filesystem Corruption](#problem-9-filesystem-corruption)
12. [Problem 10: Boot Is Extremely Slow](#problem-10-boot-is-extremely-slow)
13. [Using Rescue Mode and Emergency Mode](#using-rescue-mode-and-emergency-mode)
14. [The Debug Shell (debug-shell.service)](#the-debug-shell-debug-shellservice)
15. [Inspecting Previous Boot Logs](#inspecting-previous-boot-logs)
16. [Rescue from a Live CD / USB](#rescue-from-a-live-cd--usb)
17. [What to Prepare: Boot Troubleshooting Readiness](#what-to-prepare-boot-troubleshooting-readiness)
18. [Quick Reference: Troubleshooting Decision Tree](#quick-reference-troubleshooting-decision-tree)

---

## Why Do Boots Fail?

Boot failures can happen at any stage. The key to troubleshooting is identifying **which stage** failed:

| Stage | Possible Failures |
|-------|------------------|
| **Firmware/POST** | Bad RAM, dead disk, no power, hardware failure |
| **MBR/GRUB2** | Corrupted MBR, missing GRUB2, wrong boot order, deleted `/boot` |
| **Kernel** | Missing vmlinuz, wrong kernel parameters, hardware not supported |
| **initramfs** | Missing/corrupted initramfs, missing drivers, can't find root device |
| **Root filesystem** | Corrupted FS, bad `/etc/fstab`, wrong `root=` parameter |
| **systemd** | Failed services, dependency loops, bad unit files |
| **Login** | PAM misconfiguration, expired passwords, SELinux denials |

---

## Identifying Which Stage Failed

**What you see tells you where the failure is:**

| What You See | Failure Stage |
|-------------|--------------|
| No display, beep codes | Firmware/POST (hardware) |
| "No bootable device found" | Firmware can't find MBR/ESP |
| `grub>` or `grub rescue>` prompt | GRUB2 can't find its config or modules |
| "error: file not found" in GRUB2 | Missing kernel or initramfs in `/boot` |
| `Kernel panic - not syncing: VFS: Unable to mount root fs` | Kernel can't find root filesystem (missing initramfs drivers or wrong `root=`) |
| `dracut Warning: Could not boot` | initramfs can't find/mount root device |
| `dracut:/#` prompt (dracut emergency shell) | initramfs failed, gives you a shell to debug |
| `Give root password for maintenance` | Emergency/rescue mode (systemd detected a problem) |
| `A start job is running for...` (stuck) | systemd waiting for a device/mount that won't appear |
| `Failed to start [service]` messages | Individual service failures |
| Login prompt but can't log in | PAM, password, or SELinux issue |

---

## Problem 1: GRUB2 Is Missing or Corrupted

### Symptom: grub> or grub rescue> Prompt

**`grub>` prompt** means GRUB2's core is loaded but it can't find `grub.cfg`.

**Quick fix -- boot manually from grub> prompt:**

```
grub> ls
(hd0) (hd0,msdos1) (hd0,msdos2)

grub> ls (hd0,msdos1)/
vmlinuz-5.14.0-284.el9.x86_64  initramfs-5.14.0-284.el9.x86_64.img  grub2/

grub> set root=(hd0,msdos1)
grub> linux /vmlinuz-5.14.0-284.el9.x86_64 root=/dev/mapper/rhel-root ro
grub> initrd /initramfs-5.14.0-284.el9.x86_64.img
grub> boot
```

**Once booted, permanently fix GRUB2:**

```bash
# Reinstall GRUB2 to the MBR (BIOS system)
sudo grub2-install /dev/sda

# Regenerate grub.cfg
sudo grub2-mkconfig -o /boot/grub2/grub.cfg
```

**`grub rescue>` prompt** means GRUB2 can't even find its modules (more serious).

```
grub rescue> ls
(hd0) (hd0,msdos1) (hd0,msdos2)

grub rescue> set prefix=(hd0,msdos1)/grub2
grub rescue> set root=(hd0,msdos1)
grub rescue> insmod normal
grub rescue> normal
```

This loads the `normal` module and drops you into the regular `grub>` prompt. Then follow the manual boot steps above.

### Symptom: "No bootable device" or "Operating system not found"

**Possible causes:**

| Cause | Fix |
|-------|-----|
| Boot order wrong | Enter BIOS setup, set correct disk as first boot device |
| MBR overwritten | Boot from rescue media, reinstall GRUB2 |
| Disk not detected | Check SATA/NVMe cables, check BIOS storage settings |
| GPT disk in BIOS mode | Switch to UEFI mode in firmware, or recreate MBR |

---

## Problem 2: Kernel Panic -- Unable to Mount Root FS

```
Kernel panic - not syncing: VFS: Unable to mount root fs on unknown-block(0,0)
```

**This means:** The kernel booted but can't find or mount the root filesystem.

**Possible causes and fixes:**

| Cause | Fix |
|-------|-----|
| Wrong `root=` in kernel command line | Edit in GRUB2 (`e` key), fix the `root=` parameter |
| initramfs missing LVM drivers | Rebuild: `dracut --force --add lvm` |
| initramfs missing storage drivers | Rebuild: `dracut --force --add-drivers "driver_name"` |
| LVM not activated | Add `rd.lvm.lv=vgname/lvname` to kernel command line |
| initramfs corrupted | Rebuild: `dracut --force` |
| Root device UUID changed | Update `/etc/fstab` and `root=` parameter, then rebuild GRUB config |

**How to identify the root device:**

```bash
# From a rescue environment
lsblk
pvs
vgs
lvs
blkid
```

---

## Problem 3: initramfs Is Corrupted or Missing Drivers

**Symptom:** `dracut` emergency shell, or kernel panic about missing root.

**Fix from rescue mode:**

```bash
# Boot from a rescue target or live CD

# Rebuild initramfs for the specific kernel
sudo dracut --force /boot/initramfs-5.14.0-284.el9.x86_64.img 5.14.0-284.el9.x86_64

# Rebuild with ALL drivers (larger but guaranteed to work on any hardware)
sudo dracut --force --no-hostonly /boot/initramfs-5.14.0-284.el9.x86_64.img 5.14.0-284.el9.x86_64

# Rebuild for all installed kernels
sudo dracut --regenerate-all --force
```

**Check what modules are in the current initramfs:**

```bash
lsinitrd /boot/initramfs-$(uname -r).img | grep -i "xfs\|lvm\|nvme\|ahci"
```

---

## Problem 4: Bad /etc/fstab Entry

**Symptom:** Boot hangs with `A start job is running for /mount-point...` or system drops to emergency shell.

**Common `/etc/fstab` errors:**

| Error | What Happens |
|-------|-------------|
| Nonexistent UUID | systemd waits for the device (timeout → emergency shell) |
| Wrong mount point | Emergency shell immediately |
| Wrong filesystem type | Mount fails → emergency shell |
| Wrong mount options | Mount fails → emergency shell |

**Fix procedure:**

```bash
# Step 1: Boot into emergency mode
# (Append systemd.unit=emergency.target to kernel command line in GRUB2)

# Step 2: Enter root password

# Step 3: Remount root filesystem read-write
mount -o remount,rw /

# Step 4: Try to mount everything (identify the failing entry)
mount -a

# Step 5: Edit /etc/fstab and fix or comment out the bad line
vim /etc/fstab

# Step 6: Reload systemd's view of fstab
systemctl daemon-reload

# Step 7: Verify the fix
mount -a

# Step 8: Reboot
systemctl reboot
```

> **Critical:** Always run `systemctl daemon-reload` after editing `/etc/fstab` in emergency mode. Without this, systemd continues using the old version.

**Prevention tip:** Use the `nofail` mount option in `/etc/fstab` for non-critical filesystems. This lets the system boot even if that mount fails (but use with caution -- your application might start without its storage).

---

## Problem 5: Forgotten Root Password

This is one of the most common tasks for a RHEL admin. Here's the complete procedure:

**Step 1: Interrupt GRUB2**

1. Reboot the system
2. When the GRUB2 menu appears, press any key to stop the countdown
3. Select the kernel entry and press `e`

**Step 2: Add rd.break**

4. Find the line starting with `linux`
5. Append `rd.break` at the end
6. Press `Ctrl+x` to boot

**Step 3: Reset the password**

```bash
# You're now in the initramfs switch_root prompt
# /sysroot has the real OS mounted read-only

# Remount /sysroot as read-write
mount -o remount,rw /sysroot

# Enter the real OS environment
chroot /sysroot

# Set the new root password
passwd root
# Enter new password twice

# Create the SELinux autorelabel trigger
touch /.autorelabel

# Exit chroot
exit

# Exit the initramfs shell (system continues booting)
exit
```

**Step 4: Wait for relabel**

The system boots, performs a full SELinux relabel (which can take several minutes on large filesystems), and then reboots automatically. After the second reboot, log in with the new password.

**Why each step matters:**

| Step | Why |
|------|-----|
| `rd.break` | Pauses before switch_root -- SELinux is not active yet, so we can modify files |
| `mount -o remount,rw /sysroot` | Root is mounted read-only by default; we need write access |
| `chroot /sysroot` | Makes `/sysroot` act as `/` so `passwd` modifies the real `/etc/shadow` |
| `touch /.autorelabel` | `passwd` creates a new `/etc/shadow` without SELinux labels. Without relabeling, SELinux will block login |

---

## Problem 6: System Drops to Emergency Shell

```
You are in emergency mode. After logging in, type "journalctl -xb" to view
system logs, "systemctl reboot" to reboot, "systemctl default" to try again
to boot into default mode.
Give root password for maintenance
(or press Control-D to continue):
```

**This usually means:** A critical filesystem failed to mount or a required service failed.

**Diagnostic steps:**

```bash
# Step 1: Log in with root password

# Step 2: Check what failed
journalctl -xb --priority=err

# Step 3: Check filesystem mounts
mount
mount -a

# Step 4: If / is read-only, remount it
mount -o remount,rw /

# Step 5: Check /etc/fstab for errors
cat /etc/fstab

# Step 6: Check systemd for stuck jobs
systemctl list-jobs

# Step 7: Check failed units
systemctl --failed

# Step 8: After fixing, reload and try to continue
systemctl daemon-reload
systemctl default
# OR reboot:
systemctl reboot
```

---

## Problem 7: Service Failures Blocking Boot

**Symptom:** Boot takes a very long time, certain services show `[FAILED]`.

```bash
# Check which services failed
systemctl --failed

# View logs for a specific failed service
journalctl -u failed-service.service -b

# If a service is blocking boot (stuck in a start job)
systemctl list-jobs
```

**Common culprits:**

| Service | Why It Blocks | Fix |
|---------|--------------|-----|
| `NetworkManager-wait-online.service` | Waits for network to be fully up | Disable if not needed: `systemctl disable NetworkManager-wait-online` |
| A service requiring a missing device | Waits for device timeout (90s default) | Fix `/etc/fstab` or service config |
| A custom service with a bug | Crashes and restarts in a loop | `systemctl disable problematic.service`, fix the bug, re-enable |

---

## Problem 8: SELinux Blocking Boot

**Symptom:** System boots but services fail with "Permission denied" or "Access denied" in logs, even though file permissions look correct.

```bash
# Check for SELinux denials
ausearch -m AVC -ts boot

# Check SELinux mode
getenforce

# Temporarily set SELinux to permissive (allows everything but logs violations)
# Add to kernel command line in GRUB2:
enforcing=0

# Once booted, fix SELinux contexts
sudo restorecon -Rv /path/to/problematic/files

# Or relabel the entire filesystem
sudo touch /.autorelabel
sudo reboot

# Re-enable SELinux enforcing
sudo setenforce 1
```

> **Never disable SELinux permanently** (by setting `selinux=0` in kernel command line or `SELINUX=disabled` in `/etc/selinux/config`). Instead, fix the labels or write a custom SELinux policy. Disabling SELinux is a red flag in RHCSA/RHCE exams.

---

## Problem 9: Filesystem Corruption

**Symptom:** System drops to emergency shell with messages about filesystem errors.

**For XFS (RHEL default):**

```bash
# XFS replays its journal automatically at mount time
# If that fails, use xfs_repair (must be run on unmounted filesystem)

# Boot into emergency mode
# Do NOT mount the filesystem

# Run repair
xfs_repair /dev/mapper/rhel-root

# If xfs_repair complains about a dirty log:
xfs_repair -L /dev/mapper/rhel-root
# WARNING: -L discards the log. You may lose recent data.
```

**For ext4:**

```bash
# Run filesystem check (must be unmounted or mounted read-only)
fsck.ext4 -y /dev/mapper/rhel-root
# -y answers "yes" to all repair questions

# Force a check even if the filesystem appears clean
fsck.ext4 -f /dev/mapper/rhel-root
```

---

## Problem 10: Boot Is Extremely Slow

```bash
# After booting, analyze boot timing
systemd-analyze

# See which services took the longest
systemd-analyze blame

# See the critical chain (bottleneck path)
systemd-analyze critical-chain

# Generate a visual timeline
systemd-analyze plot > /tmp/boot.svg
```

**Common slow-boot causes:**

| Cause | How to Identify | Fix |
|-------|----------------|-----|
| `NetworkManager-wait-online` | Shows 30-90s in `blame` | Disable if not needed |
| Waiting for missing devices | `A start job is running for dev-xxx.device` | Fix `/etc/fstab`, remove stale entries |
| Disk performance issues | Long `local-fs.target` time | Check disk health: `smartctl -a /dev/sda` |
| Too many enabled services | Many services in `blame` output | Audit and disable unnecessary services |
| Slow DNS resolution | Network-dependent services take long | Check `/etc/resolv.conf`, fix DNS |

---

## Using Rescue Mode and Emergency Mode

### Rescue Mode (systemd.unit=rescue.target)

```
Use when:
├── Services fail but you need basic services (logging, filesystem)
├── You need network? → NO (network not available)
├── You need read-write root? → YES (root is mounted read-write)
└── You need to investigate service issues

How to enter:
├── At GRUB2: append systemd.unit=rescue.target to kernel command line
├── From running system: sudo systemctl isolate rescue.target
└── Requires: root password
```

### Emergency Mode (systemd.unit=emergency.target)

```
Use when:
├── /etc/fstab is broken
├── Root filesystem is corrupted
├── You need the absolute minimum environment
├── Rescue mode itself fails
├── You need read-write root? → NO (mounted read-only, must remount)
└── Almost nothing is running

How to enter:
├── At GRUB2: append systemd.unit=emergency.target to kernel command line
├── System drops here automatically when critical failures occur
└── Requires: root password

First things to do:
├── mount -o remount,rw /          ← Make root writable
├── mount -a                        ← Try mounting other filesystems
├── journalctl -xb --priority=err   ← Check errors
└── systemctl daemon-reload         ← After fixing files
```

### Comparison

| Feature | rescue.target | emergency.target | rd.break |
|---------|--------------|-----------------|----------|
| Root filesystem | Read-write | **Read-only** | Read-only at `/sysroot` |
| Services running | sysinit + basic | Almost none | None (still in initramfs) |
| systemd version | Real OS | Real OS | initramfs |
| Network | No | No | No |
| When to use | Service issues | Filesystem/fstab issues | Password reset |
| Requires root password | Yes | Yes | No |

---

## The Debug Shell (debug-shell.service)

For persistent boot debugging, enable the debug shell:

```bash
# Enable the debug shell -- spawns a root shell on TTY9 during early boot
sudo systemctl enable debug-shell.service

# During boot, press Ctrl+Alt+F9 to access the debug shell
# You get a root shell WITHOUT needing a password

# CRITICAL: Disable after debugging!
sudo systemctl disable debug-shell.service
```

> **Security warning:** The debug shell gives unauthenticated root access on TTY9. **Never** leave it enabled on a production system. Always disable it when done.

---

## Inspecting Previous Boot Logs

By default, RHEL stores journals in `/run/log/journal/` (volatile -- lost on reboot). To enable persistent logging:

```bash
# Check if persistent logging is enabled
cat /etc/systemd/journald.conf | grep Storage

# Enable persistent logging
sudo mkdir -p /var/log/journal
sudo vim /etc/systemd/journald.conf
# Set: Storage=persistent

sudo systemctl restart systemd-journald
```

Once enabled:

```bash
# List available boots
journalctl --list-boots

# View logs from the previous boot
journalctl -b -1

# View only errors from the previous boot
journalctl -b -1 -p err

# View kernel messages from the previous boot
journalctl -b -1 -k
```

---

## Rescue from a Live CD / USB

When nothing else works, boot from a RHEL installation ISO or live USB:

```bash
# Step 1: Boot from the ISO/USB
# Select "Troubleshooting" → "Rescue a Red Hat Enterprise Linux system"

# Step 2: The rescue environment detects and mounts your installed system
# at /mnt/sysimage

# Step 3: Enter the installed system
chroot /mnt/sysimage

# Step 4: Fix the problem
# Examples:
grub2-install /dev/sda                            # Fix GRUB2
grub2-mkconfig -o /boot/grub2/grub.cfg            # Rebuild GRUB config
dracut --force                                     # Rebuild initramfs
vim /etc/fstab                                     # Fix filesystem table
passwd root                                        # Reset password
restorecon -Rv /etc/                               # Fix SELinux labels

# Step 5: Exit and reboot
exit
reboot
```

---

## What to Prepare: Boot Troubleshooting Readiness

**Before a system breaks, prepare these:**

| Preparation | How to Do It | Why It Matters |
|------------|-------------|----------------|
| **Know your root password** | Store it securely in a password manager | Rescue/emergency mode require it |
| **Enable persistent journal** | `Storage=persistent` in `/etc/systemd/journald.conf` | Inspect previous boot logs |
| **Backup your MBR** | `dd if=/dev/sda of=/safe/location/mbr.bak bs=512 count=1` | Restore a corrupted MBR |
| **Backup /boot** | `tar czf /safe/location/boot-backup.tar.gz /boot/` | Recover missing kernel/initramfs |
| **Backup /etc/fstab** | `cp /etc/fstab /etc/fstab.bak` | Quick restore of broken fstab |
| **Keep a rescue USB/ISO** | Download RHEL ISO, write to USB with `dd` | Boot and repair from external media |
| **Document your LVM layout** | `pvs; vgs; lvs; lsblk` -- save the output | Know what to mount during recovery |
| **Know your disk device names** | `lsblk`, `blkid` -- save the output | Reference during GRUB2 manual boot |
| **Keep GRUB2 config backed up** | `cp /boot/grub2/grub.cfg /root/grub.cfg.bak` | Restore corrupted GRUB config |

---

## Quick Reference: Troubleshooting Decision Tree

```
System won't boot. What do you see?

├── Nothing on screen (no POST, no beeps)
│   └── HARDWARE: Check power cable, PSU, RAM seating, motherboard
│
├── BIOS shows "No bootable device"
│   ├── Check boot order in BIOS/UEFI
│   ├── Check disk cables/connections
│   └── Boot from rescue USB → reinstall GRUB2
│
├── grub> or grub rescue> prompt
│   ├── grub> : Boot manually (ls, set root, linux, initrd, boot)
│   ├── grub rescue> : insmod normal → normal → then manual boot
│   └── After booting: grub2-install + grub2-mkconfig
│
├── "error: file '/vmlinuz-...' not found" in GRUB2
│   ├── Kernel deleted? → Boot older kernel from GRUB2 menu
│   └── /boot unmounted? → Check /etc/fstab
│
├── Kernel panic - VFS: Unable to mount root fs
│   ├── Wrong root= → Fix in GRUB2 (press e)
│   ├── Missing LVM/RAID in initramfs → dracut --force
│   └── Missing rd.lvm.lv= → Add to kernel command line
│
├── dracut emergency shell (dracut:/#)
│   ├── Type: journalctl to see what failed
│   ├── Root device not found → Check blkid, lvm, mdadm
│   └── Rebuild initramfs from rescue media
│
├── "Give root password for maintenance" (emergency shell)
│   ├── Enter root password
│   ├── mount -o remount,rw /
│   ├── journalctl -xb -p err
│   ├── mount -a (find broken /etc/fstab entry)
│   ├── Fix → systemctl daemon-reload → systemctl reboot
│   └── Forgot root password? → rd.break procedure
│
├── "A start job is running for..." (hangs)
│   ├── Wait for timeout (usually 90s) → emergency shell
│   ├── Or Ctrl+Alt+Del → boot to emergency.target
│   ├── Find the waiting device/mount
│   ├── Fix /etc/fstab or remove the stale entry
│   └── systemctl daemon-reload → reboot
│
├── Boot completes but services fail
│   ├── systemctl --failed
│   ├── journalctl -u failed-service -b
│   ├── Fix the service or disable it
│   └── systemctl daemon-reload → systemctl restart service
│
└── Boot completes but can't log in
    ├── Wrong password → rd.break password reset
    ├── Account locked → faillock --reset --user username
    ├── SELinux denial → Boot with enforcing=0, fix labels, re-enable
    └── PAM misconfigured → Fix from rescue/emergency mode
```

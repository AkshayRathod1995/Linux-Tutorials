# The RHEL Boot Process -- Complete Single-Page Reference

> Everything from power button to login screen on one page.

---

## The Flow at a Glance

```
Power → BIOS/UEFI → POST → MBR(boot.img) → diskboot.img → core.img →
GRUB2 → vmlinuz + initramfs → Kernel → systemd(initramfs) →
mount /sysroot → switch_root → systemd(real OS) → services → LOGIN
```

---

## Stage 1: Firmware (BIOS/UEFI) & POST

You press the power button. The PSU stabilizes and sends a **"Power Good"** signal. The CPU wakes up and jumps to firmware code stored on a chip on the motherboard.

The firmware runs **POST (Power-On Self-Test)** -- checking CPU, RAM, video, keyboard, and storage controllers. If a critical check fails, you get beep codes (1 short = OK, continuous = RAM failure, 1 long + 2 short = video failure).

After POST, the firmware looks for a **bootable device** in the configured boot order.

| | BIOS (Legacy) | UEFI (Modern) |
|---|---|---|
| **How it finds the bootloader** | Reads first 512 bytes (MBR) of each disk, checks for `0x55AA` signature | Reads EFI System Partition (FAT32), loads `.efi` files |
| **Partition table** | MBR (max 4 partitions, 2 TB limit) | GPT (128+ partitions, no practical size limit) |
| **Secure Boot** | No | Yes -- verifies bootloader signatures |
| **RHEL boot chain** | MBR → boot.img → core.img → GRUB2 | UEFI → `shimx64.efi` → `grubx64.efi` → GRUB2 |
| **Check which you're using** | `[ -d /sys/firmware/efi ] && echo UEFI \|\| echo BIOS` | |

---

## Stage 2: MBR & Getting to GRUB2

### The MBR -- First 512 Bytes of the Disk

```
┌──────────────────────────────────────┐
│  Bootstrap Code (boot.img)    446 B  │  ← Tiny program that loads the next stage
│  Partition Table               64 B  │  ← 4 entries × 16 bytes (why MBR has max 4 partitions)
│  Boot Signature (0x55AA)        2 B  │  ← "This disk is bootable"
└──────────────────────────────────────┘
```

**446 bytes is not enough** for a real bootloader. GRUB2 solves this with a multi-stage chain:

| Stage | Image | Location | What It Does |
|-------|-------|----------|-------------|
| **1** | `boot.img` | MBR (446 bytes) | Loads the first sector of core.img from MBR gap |
| **1.5** | `diskboot.img` + `core.img` | MBR gap (~1 MB between MBR and first partition, sectors 1-2047) | `diskboot.img` loads the rest of `core.img`. `core.img` contains filesystem drivers (XFS, ext4) so GRUB2 can read `/boot/grub2/` |
| **2** | `/boot/grub2/` | `/boot` partition | Full GRUB2 -- reads `grub.cfg`, shows menu, loads kernel |

Each stage loads the next, progressively larger one. `boot.img` → `diskboot.img` → `core.img` → full GRUB2.

> **On UEFI systems:** There is no boot.img/diskboot.img/core.img chain. GRUB2 lives as a single `.efi` file on the EFI System Partition at `/boot/efi/EFI/redhat/grubx64.efi`.

---

## Stage 3: GRUB2 -- The Bootloader

GRUB2 reads `/boot/grub2/grub.cfg`, displays a boot menu, and waits for your selection (or timeout).

**When you select a kernel, GRUB2 does two things:**
1. **Loads `vmlinuz`** (compressed kernel) into RAM
2. **Loads `initramfs`** (initial RAM filesystem) into RAM
3. Passes the **kernel command line** and hands control to the kernel

```
Key kernel command line parameters (from /etc/default/grub → GRUB_CMDLINE_LINUX):

  root=/dev/mapper/rhel-root     Where the root filesystem is
  rd.lvm.lv=rhel/root            Activate this LVM volume in initramfs
  rhgb quiet                     Graphical boot, suppress messages
  rd.break                       ← Pause before switch_root (password reset)
  systemd.unit=rescue.target     ← Boot into rescue mode
```

**Config flow:** `/etc/default/grub` + `/etc/grub.d/` → `grub2-mkconfig` → `/boot/grub2/grub.cfg`

> Never edit `grub.cfg` directly. Edit `/etc/default/grub`, then run `grub2-mkconfig -o /boot/grub2/grub.cfg`.

**Editing boot parameters at boot time:** Interrupt GRUB2 menu → press `e` → edit the `linux` line → `Ctrl+x` to boot. Changes are temporary (one boot only).

---

## Stage 4: Kernel Loading (vmlinuz)

**vmlinuz** = **v**irtual **m**emory **linu**x compresse**d** -- the compressed Linux kernel (~11 MB, decompresses to ~40 MB).

**What happens:**

1. The real-mode setup code runs -- parses boot parameters, detects CPU, queries memory map
2. CPU switches from **Real Mode → Protected Mode → Long Mode** (16-bit → 32-bit → 64-bit)
3. **Decompression stub** unpacks the kernel (RHEL 9 uses zstd, RHEL 8 uses xz)
4. **`start_kernel()`** runs -- initializes memory management, scheduler, interrupts, VFS, console
5. **`rest_init()`** creates the first processes:

| PID | Process | Role |
|-----|---------|------|
| 0 | idle | Runs when CPU has nothing to do |
| **1** | **`/sbin/init` (systemd from initramfs)** | **The init system -- manages everything** |
| 2 | kthreadd | Manages kernel threads |

**At this point:** The kernel is running but has NOT mounted the real root filesystem yet. It can't -- it may need LVM, RAID, or storage drivers that are inside the initramfs.

---

## Stage 5: initramfs -- Mounting the Real Root

**initramfs** (Initial RAM File System) is a compressed cpio archive containing a **mini OS** in RAM -- with systemd, drivers, LVM tools, and udev. It exists to solve the chicken-and-egg problem: the kernel needs storage drivers to read the root filesystem, but those drivers live ON the root filesystem.

**What happens inside initramfs (systemd runs as PID 1 using `initrd.target`):**

```
1. sysinit.target
   └── Load storage/filesystem drivers, start udevd (hardware detection)

2. initrd-root-device.target
   └── Activate LVM volumes, assemble RAID, unlock LUKS encryption

3. initrd-root-fs.target
   └── Run filesystem check, mount root filesystem READ-ONLY on /sysroot

4. initrd.target
   └── Mount /boot and other early filesystems
   
   ════ rd.break pauses HERE ════

5. switch_root
   ├── Delete all initramfs files (free the RAM)
   ├── Make /sysroot the new /  (pivot_root)
   └── Re-execute systemd from the REAL OS filesystem
```

### /sysroot

Inside initramfs, the real root filesystem is mounted at **`/sysroot`**. After `switch_root`, what was `/sysroot` becomes `/` and the initramfs is gone.

### Root Password Reset (rd.break)

```bash
# 1. Boot → GRUB2 → press 'e' → append 'rd.break' to linux line → Ctrl+x
# 2. At the switch_root prompt:
mount -o remount,rw /sysroot
chroot /sysroot
passwd root
touch /.autorelabel          # SELinux isn't active yet -- new /etc/shadow has no labels
exit
exit
# 3. System boots, relabels (takes a few minutes), reboots, done.
```

---

## Stage 6: systemd on the Real OS

After `switch_root`, a **new systemd** runs from the real filesystem. It reads **`default.target`** and starts everything needed to reach that state.

**Target dependency chain:**

```
local-fs.target          ← Mount all filesystems from /etc/fstab, remount / read-write
    └── sysinit.target   ← Set hostname, load modules, start journald, udevd, apply sysctl
        └── basic.target ← Start D-Bus, logind, timers, sockets
            └── multi-user.target ← sshd, crond, chronyd, firewalld, NetworkManager, getty
                └── graphical.target ← gdm.service (GNOME login screen)
```

systemd starts units **in parallel** wherever there's no ordering dependency -- this is why modern Linux boots fast.

### Targets You Need to Know

| Target | What You Get | Old Runlevel |
|--------|-------------|-------------|
| `multi-user.target` | Text login (servers) | 3 |
| `graphical.target` | GUI login (desktops) | 5 |
| `rescue.target` | Minimal services, root FS read-write, root password required | 1 |
| `emergency.target` | Almost nothing, root FS **read-only**, root password required | S |

```bash
systemctl get-default                        # What target boots by default
sudo systemctl set-default multi-user.target # Change default
sudo systemctl isolate rescue.target         # Switch target at runtime
```

### How Services Get Started

When you `systemctl enable sshd`, it creates a symlink:

```
/etc/systemd/system/multi-user.target.wants/sshd.service → /usr/lib/systemd/system/sshd.service
```

When systemd reaches `multi-user.target`, it reads all symlinks in `.wants/` and starts those services.

---

## Stage 7: Login Screen

| Default Target | Login Type | Service |
|---------------|-----------|---------|
| `multi-user.target` | Text: `servera login: _` | `getty@tty1.service` |
| `graphical.target` | GUI: GNOME login screen | `gdm.service` |

**Text login flow:** `getty` → `/bin/login` → PAM checks `/etc/passwd` + `/etc/shadow` → shell

The system is now fully booted.

---

## Complete Flow Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│  POWER ON → PSU "Power Good" → CPU wakes up                    │
└──────────────────────────┬──────────────────────────────────────┘
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│  BIOS/UEFI: POST → Find bootable device → Load bootloader      │
│  BIOS: Read MBR (512 bytes) → boot.img                         │
│  UEFI: Read ESP → shimx64.efi → grubx64.efi                    │
└──────────────────────────┬──────────────────────────────────────┘
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│  boot.img → diskboot.img → core.img (filesystem drivers load)   │
│  core.img reads /boot partition → loads full GRUB2               │
└──────────────────────────┬──────────────────────────────────────┘
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│  GRUB2: Read grub.cfg → show menu → load vmlinuz + initramfs   │
│  Pass kernel command line → hand off to kernel                  │
└──────────────────────────┬──────────────────────────────────────┘
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│  KERNEL: Decompress → start_kernel() → init memory, scheduler  │
│  → rest_init() → create PID 1 (systemd from initramfs)         │
└──────────────────────────┬──────────────────────────────────────┘
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│  INITRAMFS (systemd, target: initrd.target):                    │
│  Load drivers → Activate LVM → fsck → Mount root on /sysroot   │
│  ─── rd.break pauses HERE ───                                   │
│  switch_root: delete initramfs, /sysroot becomes /, re-exec     │
└──────────────────────────┬──────────────────────────────────────┘
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│  SYSTEMD (real OS, target: default.target):                     │
│  local-fs → sysinit → basic → multi-user → graphical            │
│  Start services in parallel → LOGIN SCREEN                      │
└─────────────────────────────────────────────────────────────────┘
```

---

## Troubleshooting Quick Reference

| What You See | Where It Failed | Fix |
|-------------|----------------|-----|
| No display, beeps | POST/Hardware | Check RAM, PSU, cables |
| "No bootable device" | Firmware | Fix boot order, check disk |
| `grub>` prompt | GRUB2 config missing | `set root=(hd0,msdos1)` → `linux /vmlinuz...` → `initrd /initramfs...` → `boot` |
| Kernel panic (no root FS) | initramfs/kernel | Fix `root=` in GRUB2 or `dracut --force` |
| `dracut:/#` shell | initramfs can't mount root | Check LVM, device names |
| Hangs on "A start job..." | Bad `/etc/fstab` | Boot `emergency.target` → fix fstab → `systemctl daemon-reload` |
| Emergency shell | Critical failure | `mount -o remount,rw /` → `journalctl -xb -p err` → fix → reboot |
| Can't log in | Password/SELinux | `rd.break` password reset, or boot with `enforcing=0` |

### Key Troubleshooting Commands

```bash
# Boot analysis
systemd-analyze                    # Overall boot time
systemd-analyze blame              # Slowest services
systemd-analyze critical-chain     # Bottleneck path

# Logs
journalctl -b                      # Current boot log
journalctl -b -1 -p err            # Previous boot errors
dmesg                              # Kernel messages
cat /proc/cmdline                  # Kernel command line used

# Services
systemctl --failed                 # Failed units
systemctl list-jobs                # Stuck jobs
systemctl status svc               # Service status + logs

# GRUB2
grub2-mkconfig -o /boot/grub2/grub.cfg   # Regenerate config
grub2-install /dev/sda                     # Reinstall GRUB2 (BIOS)

# initramfs
dracut --force                     # Rebuild initramfs
lsinitrd                           # View initramfs contents
```

---

## All Key Files -- Where Everything Lives

| File | Purpose |
|------|---------|
| `/boot/vmlinuz-*` | Compressed kernel |
| `/boot/initramfs-*.img` | Initial RAM filesystem |
| `/boot/grub2/grub.cfg` | GRUB2 config (auto-generated, don't edit) |
| `/etc/default/grub` | GRUB2 settings (edit this) |
| `/etc/fstab` | Filesystem mount table |
| `/etc/dracut.conf.d/` | initramfs build config |
| `/etc/systemd/system/default.target` | Default boot target symlink |
| `/boot/efi/EFI/redhat/` | UEFI bootloader files |
| `/boot/loader/entries/*.conf` | BLS kernel entries (RHEL 8/9) |

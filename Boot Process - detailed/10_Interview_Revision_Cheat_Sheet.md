# Interview Revision Cheat Sheet: RHEL Boot Process

> One file. Everything you need. Read this the night before your interview.

## Index

1. [The Boot Process in One Sentence](#1-the-boot-process-in-one-sentence)
2. [The 7 Stages at a Glance](#2-the-7-stages-at-a-glance)
3. [Key Vocabulary](#3-key-vocabulary)
4. [BIOS vs UEFI](#4-bios-vs-uefi)
5. [POST (Power-On Self-Test)](#5-post-power-on-self-test)
6. [MBR -- The First 512 Bytes](#6-mbr----the-first-512-bytes)
7. [MBR Partition Table Entry (16 Bytes)](#7-mbr-partition-table-entry-16-bytes)
8. [GRUB2 Multi-Stage Boot](#8-grub2-multi-stage-boot)
9. [boot.img, diskboot.img, core.img](#9-bootimg-diskbootimg-coreimg)
10. [GRUB2 Configuration](#10-grub2-configuration)
11. [/etc/default/grub Parameters](#11-etcdefaultgrub-parameters)
12. [Kernel Command Line Parameters](#12-kernel-command-line-parameters)
13. [vmlinuz -- The Kernel](#13-vmlinuz----the-kernel)
14. [Kernel Initialization Sequence](#14-kernel-initialization-sequence)
15. [initramfs](#15-initramfs)
16. [initramfs Target Chain](#16-initramfs-target-chain)
17. [/sysroot and switch_root](#17-sysroot-and-switch_root)
18. [systemd as PID 1](#18-systemd-as-pid-1)
19. [systemd Targets](#19-systemd-targets)
20. [Target Dependency Chain](#20-target-dependency-chain)
21. [systemd Targets vs SysV Runlevels](#21-systemd-targets-vs-sysv-runlevels)
22. [systemd Unit Types](#22-systemd-unit-types)
23. [systemd Dependency Keywords](#23-systemd-dependency-keywords)
24. [The Login Prompt](#24-the-login-prompt)
25. [Rescue vs Emergency vs rd.break](#25-rescue-vs-emergency-vs-rdbreak)
26. [Root Password Reset Procedure](#26-root-password-reset-procedure)
27. [All Boot-Related Files -- Master List](#27-all-boot-related-files----master-list)
28. [BIOS vs UEFI File Locations](#28-bios-vs-uefi-file-locations)
29. [All Boot-Related Commands](#29-all-boot-related-commands)
30. [Boot Troubleshooting Quick Reference](#30-boot-troubleshooting-quick-reference)
31. [dracut Commands (initramfs Management)](#31-dracut-commands-initramfs-management)
32. [Boot Performance Analysis](#32-boot-performance-analysis)
33. [Service Management Commands](#33-service-management-commands)
34. [Persistent Journal Setup](#34-persistent-journal-setup)
35. [SELinux and Boot](#35-selinux-and-boot)
36. [RHEL Kernel Versions](#36-rhel-kernel-versions)
37. [Boot Flow -- One-Line Summary](#37-boot-flow----one-line-summary)
38. [Interview Power Answers -- One-Liners](#38-interview-power-answers----one-liners)
39. [Interview Tips](#39-interview-tips)

---

## 1. The Boot Process in One Sentence

> The firmware (BIOS/UEFI) initializes hardware and finds GRUB2, which loads the kernel and initramfs into memory; the kernel starts systemd inside initramfs to mount the real root filesystem, then pivots to it and starts a new systemd instance that brings up services until the login screen appears.

---

## 2. The 7 Stages at a Glance

| Stage | What Happens | Key Component |
|-------|-------------|---------------|
| 1 | Hardware power-on, POST | BIOS/UEFI firmware |
| 2 | Find bootable device, load MBR | MBR (512 bytes), boot.img |
| 3 | Load full bootloader, show menu | GRUB2, grub.cfg |
| 4 | Load and decompress kernel | vmlinuz |
| 5 | Mount real root filesystem | initramfs, /sysroot |
| 6 | Start services, reach target | systemd, targets |
| 7 | Display login prompt | getty / GDM |

---

## 3. Key Vocabulary

| Term | Definition |
|------|-----------|
| **POST** | Power-On Self-Test -- firmware checks CPU, RAM, video, and other hardware |
| **MBR** | Master Boot Record -- first 512 bytes of disk: 446B code + 64B partition table + 2B signature |
| **GRUB2** | GRand Unified Bootloader v2 -- default bootloader on RHEL 7/8/9 |
| **vmlinuz** | The compressed Linux kernel image (vm=Virtual Memory, linu=Linux, z=compressed) |
| **initramfs** | Initial RAM FileSystem -- temporary root FS in RAM with drivers needed to find the real root |
| **switch_root** | The operation that pivots from initramfs to the real root filesystem |
| **/sysroot** | Directory where the real root filesystem is mounted inside initramfs |
| **systemd** | Init system and service manager, PID 1 on RHEL 7/8/9 |
| **Target** | A systemd unit representing a system state (group of services) |
| **dracut** | The tool that builds initramfs on RHEL |
| **boot.img** | 446-byte GRUB2 Stage 1 embedded in the MBR |
| **diskboot.img** | First sector of core.img; loads the rest of core.img from MBR gap |
| **core.img** | GRUB2 Stage 1.5: diskboot.img + filesystem modules, lives in MBR gap |
| **ESP** | EFI System Partition -- FAT32 partition with EFI bootloaders (UEFI systems) |
| **BLS** | Boot Loader Specification -- kernel entries in `/boot/loader/entries/` (RHEL 8/9) |
| **shimx64.efi** | Secure Boot first-stage loader, signed by Microsoft, loads GRUB2 |
| **PID 1** | The first userspace process (systemd); if it dies, kernel panics |
| **getty** | "Get TTY" -- the program that displays the text login prompt |
| **GDM** | GNOME Display Manager -- the graphical login screen |

---

## 4. BIOS vs UEFI

| Feature | BIOS | UEFI |
|---------|------|------|
| CPU mode | 16-bit Real Mode | 32/64-bit |
| Partition table | MBR | GPT |
| Max disk size | 2 TB | 9.4 ZB |
| Max partitions | 4 primary | 128+ |
| Bootloader location | MBR (first 512 bytes) | EFI System Partition (FAT32) |
| Secure Boot | No | Yes |
| Boot speed | Slower | Faster |
| RHEL boot chain | MBR → boot.img → core.img → GRUB2 | UEFI → shimx64.efi → grubx64.efi |

```bash
# Check if booted in BIOS or UEFI
[ -d /sys/firmware/efi ] && echo "UEFI" || echo "BIOS"

# View UEFI boot entries
efibootmgr -v
```

---

## 5. POST (Power-On Self-Test)

Checks in order: CPU → BIOS ROM → RAM → Video → Keyboard → Storage → Peripherals

| Result | Meaning |
|--------|---------|
| 1 short beep | POST successful |
| No beep | PSU or motherboard failure |
| Continuous beep | RAM failure |
| 1 long + 2 short | Video card failure |

---

## 6. MBR -- The First 512 Bytes

```
┌────────────────────────────────┐
│ Bootstrap (boot.img)   446 B  │ → Loads diskboot.img from MBR gap
│ Partition Table         64 B  │ → 4 entries × 16 bytes each
│ Boot Signature           2 B  │ → 0x55AA = "I'm bootable"
└────────────────────────────────┘
Total = 512 bytes
```

**Why max 4 primary partitions:** Only 64 bytes for the partition table, each entry is 16 bytes. 64 ÷ 16 = 4.

**Why max 2 TB:** Sector count is 4 bytes (32 bits). 2^32 × 512 bytes = 2 TB.

---

## 7. MBR Partition Table Entry (16 Bytes)

| Offset | Size | Field |
|--------|------|-------|
| 0 | 1 B | Boot indicator (`0x80` = active) |
| 1-3 | 3 B | CHS start address |
| 4 | 1 B | Partition type (`0x83` = Linux, `0x82` = swap, `0x8e` = LVM) |
| 5-7 | 3 B | CHS end address |
| 8-11 | 4 B | LBA start address |
| 12-15 | 4 B | Number of sectors |

---

## 8. GRUB2 Multi-Stage Boot

```
Stage 1            Stage 1.5                    Stage 2
boot.img ────────► diskboot.img + core.img ───► /boot/grub2/
(446 bytes,        (~32 KB, in MBR gap,         (full GRUB2, on
 in MBR)            has FS drivers)              /boot partition)
```

---

## 9. boot.img, diskboot.img, core.img

| Image | Size | Location | Purpose |
|-------|------|----------|---------|
| `boot.img` | 446 B | MBR (sector 0) | Loads first sector of core.img from a hardcoded location |
| `diskboot.img` | 512 B | First sector of core.img (in MBR gap) | Loads the rest of core.img |
| `core.img` | ~32 KB | MBR gap (sectors 1-2047) | Contains FS drivers, loads GRUB2 from `/boot/grub2/` |

Source files: `/usr/lib/grub/i386-pc/boot.img`, `/usr/lib/grub/i386-pc/diskboot.img`

**MBR gap:** Sectors 1-2047, approximately 1 MB. Between MBR and first partition.

---

## 10. GRUB2 Configuration

```
/etc/default/grub       ← Your settings (timeout, kernel params)
        +
/etc/grub.d/            ← Scripts that build the config
        │
        ▼  grub2-mkconfig
        │
/boot/grub2/grub.cfg    ← The actual config GRUB2 reads (NEVER edit directly)
```

```bash
# Regenerate grub.cfg (BIOS)
sudo grub2-mkconfig -o /boot/grub2/grub.cfg

# Regenerate grub.cfg (UEFI)
sudo grub2-mkconfig -o /boot/efi/EFI/redhat/grub.cfg
```

### /etc/grub.d/ Scripts

| Script | Generates |
|--------|----------|
| `00_header` | Basic GRUB2 setup |
| `10_linux` | Linux kernel entries |
| `30_os-prober` | Other OS entries |
| `40_custom` | Your custom entries |

---

## 11. /etc/default/grub Parameters

| Parameter | Purpose | Example |
|-----------|---------|---------|
| `GRUB_TIMEOUT` | Menu timeout in seconds | `5` |
| `GRUB_DEFAULT` | Default entry | `saved` |
| `GRUB_CMDLINE_LINUX` | Kernel command line | `"crashkernel=auto ... rhgb quiet"` |
| `GRUB_DISABLE_RECOVERY` | Hide recovery entries | `"true"` |
| `GRUB_TERMINAL_OUTPUT` | Output mode | `"console"` |

---

## 12. Kernel Command Line Parameters

| Parameter | Meaning |
|-----------|---------|
| `root=/dev/mapper/rhel-root` | Root filesystem device |
| `ro` | Mount root read-only initially |
| `rd.lvm.lv=rhel/root` | Activate this LVM volume in initramfs |
| `rd.lvm.lv=rhel/swap` | Activate swap LVM volume |
| `crashkernel=auto` | Reserve memory for kdump |
| `rhgb` | Red Hat Graphical Boot splash |
| `quiet` | Suppress non-error kernel messages |
| `rd.break` | Break into initramfs before switch_root |
| `systemd.unit=rescue.target` | Boot to rescue mode |
| `systemd.unit=emergency.target` | Boot to emergency mode |
| `init=/bin/bash` | Skip systemd, go straight to bash |
| `enforcing=0` | Set SELinux to permissive |
| `console=ttyS0,115200` | Serial console output |

```bash
# View the kernel command line used to boot
cat /proc/cmdline
```

---

## 13. vmlinuz -- The Kernel

| Property | Detail |
|----------|--------|
| **Name** | vm (Virtual Memory) + linu (Linux) + z (compressed) |
| **Format** | bzImage (big zImage) |
| **Location** | `/boot/vmlinuz-<version>` |
| **Compressed size** | ~11 MB |
| **Decompressed size** | ~30-50 MB |
| **Compression (RHEL 9)** | zstd (fast decompression) |
| **Compression (RHEL 8)** | xz |

```bash
uname -r                        # Current kernel version
ls -lh /boot/vmlinuz-*         # All installed kernels
file /boot/vmlinuz-$(uname -r) # File type info
```

### vmlinuz Structure

```
Real-mode header (~15 KB)  →  Boot params, CPU mode switching
Decompression stub (~20 KB) →  Decompresses the payload
Compressed kernel (~10 MB)  →  The actual kernel code
```

---

## 14. Kernel Initialization Sequence

| Step | Function | Purpose |
|------|----------|---------|
| 1 | CPU mode switch | Real Mode → Protected → Long Mode (64-bit) |
| 2 | Decompress kernel | zstd/xz/gzip → full kernel in RAM |
| 3 | `start_kernel()` | Main init: memory, scheduler, VFS, interrupts |
| 4 | `rest_init()` | Creates PID 1 (systemd), PID 2 (kthreadd) |

**After `rest_init()`:**

| PID | Process | Role |
|-----|---------|------|
| 0 | idle | Runs when CPU has nothing to do |
| 1 | systemd (from initramfs) | Init system, manages everything |
| 2 | kthreadd | Manages kernel threads |

---

## 15. initramfs

| Property | Detail |
|----------|--------|
| **Full name** | Initial RAM File System |
| **Format** | Compressed cpio archive |
| **Location** | `/boot/initramfs-<version>.img` |
| **Size** | ~34 MB |
| **Contains** | systemd, drivers, LVM/RAID tools, dracut scripts, udev |
| **Built by** | `dracut` |
| **Purpose** | Break the chicken-and-egg: kernel needs drivers from root FS, but can't read root FS without drivers |

```bash
lsinitrd /boot/initramfs-$(uname -r).img    # View contents
lsinitrd /boot/initramfs-$(uname -r).img -m  # View modules
```

---

## 16. initramfs Target Chain

```
sysinit.target → initrd-root-device.target → initrd-root-fs.target → initrd.target → switch_root
     │                    │                        │                        │
  Load drivers     Activate LVM/RAID         Mount root on           Pivot to
  Start udev       Unlock LUKS              /sysroot (read-only)    real root
```

---

## 17. /sysroot and switch_root

| Concept | What It Means |
|---------|-------------|
| `/sysroot` | The mountpoint inside initramfs where the real root FS is mounted |
| `switch_root` | Deletes initramfs, makes `/sysroot` the new `/`, re-execs systemd |
| `rd.break` | Pauses BEFORE switch_root -- gives you a shell with `/sysroot` mounted read-only |
| `chroot /sysroot` | Used during `rd.break` to "enter" the real OS for repairs |

**switch_root sequence:**

```
1. Delete all initramfs files from tmpfs (free RAM)
2. pivot_root: make /sysroot the new /
3. Execute /sbin/init (systemd) from the real root filesystem
```

---

## 18. systemd as PID 1

| Property | Detail |
|----------|--------|
| PID | 1 (always) |
| Binary | `/usr/lib/systemd/systemd` |
| `/sbin/init` | Symlink to `../lib/systemd/systemd` |
| Runs twice | Once in initramfs, once on real OS |
| initramfs target | `initrd.target` |
| Real OS target | `default.target` (→ multi-user or graphical) |

```bash
ps -p 1 -o comm=                    # Shows: systemd
ls -la /sbin/init                    # Shows symlink to systemd
```

---

## 19. systemd Targets

| Target | Purpose | SysV Equivalent |
|--------|---------|----------------|
| `poweroff.target` | Shut down | Runlevel 0 |
| `rescue.target` | Single-user, root password required | Runlevel 1 |
| `multi-user.target` | Multi-user, text login, network | Runlevel 3 |
| `graphical.target` | Multi-user, graphical login | Runlevel 5 |
| `reboot.target` | Reboot | Runlevel 6 |
| `emergency.target` | Minimal, root FS read-only | Runlevel S |

```bash
systemctl get-default                              # View default target
sudo systemctl set-default multi-user.target       # Set default target
sudo systemctl set-default graphical.target        # Set default to GUI
sudo systemctl isolate rescue.target               # Switch target at runtime
```

---

## 20. Target Dependency Chain

```
graphical.target
  └── multi-user.target (Requires)
        └── basic.target (Requires)
              ├── sockets.target
              ├── timers.target
              ├── paths.target
              └── sysinit.target (Requires)
                    ├── local-fs.target (mount filesystems)
                    ├── swap.target (enable swap)
                    └── cryptsetup.target (unlock encryption)
```

```bash
systemctl list-dependencies graphical.target | grep target
```

---

## 21. systemd Targets vs SysV Runlevels

| SysV | systemd | Command |
|------|---------|---------|
| `init 0` | `systemctl poweroff` | Shut down |
| `init 1` | `systemctl isolate rescue.target` | Rescue mode |
| `init 3` | `systemctl isolate multi-user.target` | Text mode |
| `init 5` | `systemctl isolate graphical.target` | GUI mode |
| `init 6` | `systemctl reboot` | Reboot |
| `runlevel` | `systemctl get-default` | Current target |

---

## 22. systemd Unit Types

| Type | Extension | Purpose |
|------|-----------|---------|
| Service | `.service` | Daemons and one-shot processes |
| Target | `.target` | Group of units (system state) |
| Socket | `.socket` | Socket-based activation |
| Mount | `.mount` | Filesystem mount point |
| Timer | `.timer` | Scheduled execution |
| Device | `.device` | Hardware device |
| Swap | `.swap` | Swap partition |
| Path | `.path` | File/directory watcher |

---

## 23. systemd Dependency Keywords

| Keyword | Meaning |
|---------|---------|
| `Requires=` | Hard dependency -- this unit fails if required unit fails |
| `Wants=` | Soft dependency -- this unit starts even if wanted unit fails |
| `After=` | Start AFTER the named unit (ordering only) |
| `Before=` | Start BEFORE the named unit (ordering only) |
| `Conflicts=` | Stop the conflicting unit when starting this one |

`Requires`/`Wants` = WHAT starts. `After`/`Before` = WHEN it starts. Independent of each other.

---

## 24. The Login Prompt

| Type | Service | Target | Access |
|------|---------|--------|--------|
| Text login | `getty@tty1.service` | `multi-user.target` | Console / Ctrl+Alt+F1-F6 |
| Graphical login | `gdm.service` | `graphical.target` | Console (tty1 or tty2) |
| SSH login | `sshd.service` | `multi-user.target` | Network |

```
Login flow: getty → /bin/login → PAM → /etc/passwd + /etc/shadow → shell
```

---

## 25. Rescue vs Emergency vs rd.break

| Feature | rescue.target | emergency.target | rd.break |
|---------|--------------|-----------------|----------|
| Root FS | Read-write | **Read-only** | Read-only at `/sysroot` |
| Services | sysinit + basic | Almost none | None (initramfs) |
| systemd | Real OS | Real OS | initramfs |
| Root password | Required | Required | **Not required** |
| Use case | Service issues | fstab/FS issues | Password reset |
| How to enter | `systemd.unit=rescue.target` | `systemd.unit=emergency.target` | Append `rd.break` |

---

## 26. Root Password Reset Procedure

```bash
# 1. Boot → interrupt GRUB2 → press 'e' → add 'rd.break' → Ctrl+x
# 2. At switch_root prompt:
mount -o remount,rw /sysroot
chroot /sysroot
passwd root
touch /.autorelabel
exit
exit
# 3. Wait for SELinux relabel → automatic reboot → login with new password
```

**Why `touch /.autorelabel`:** `passwd` creates a new `/etc/shadow` without SELinux labels. Without relabeling, SELinux blocks login.

---

## 27. All Boot-Related Files -- Master List

| File/Directory | Purpose |
|---------------|---------|
| `/boot/vmlinuz-*` | Compressed kernel image |
| `/boot/initramfs-*.img` | Initial RAM filesystem |
| `/boot/config-*` | Kernel build configuration |
| `/boot/System.map-*` | Kernel symbol table |
| `/boot/grub2/grub.cfg` | GRUB2 config (auto-generated, DO NOT edit) |
| `/boot/grub2/grubenv` | GRUB2 environment (saved_entry) |
| `/boot/grub2/i386-pc/` | GRUB2 modules (BIOS) |
| `/boot/grub2/x86_64-efi/` | GRUB2 modules (UEFI) |
| `/boot/loader/entries/*.conf` | BLS kernel entries (RHEL 8/9) |
| `/boot/efi/EFI/redhat/` | EFI bootloaders (UEFI systems) |
| `/etc/default/grub` | GRUB2 settings (edit this) |
| `/etc/grub.d/` | Scripts that generate grub.cfg |
| `/etc/fstab` | Filesystem mount table |
| `/etc/dracut.conf` | dracut main config |
| `/etc/dracut.conf.d/` | dracut drop-in configs |
| `/etc/systemd/system/default.target` | Symlink to the default boot target |
| `/etc/systemd/system/*.wants/` | Service enable symlinks |
| `/usr/lib/systemd/system/` | Vendor-provided unit files |
| `/etc/systemd/journald.conf` | Journal (log) configuration |
| `/etc/selinux/config` | SELinux configuration |

---

## 28. BIOS vs UEFI File Locations

| Component | BIOS | UEFI |
|-----------|------|------|
| GRUB2 binary | In MBR + MBR gap | `/boot/efi/EFI/redhat/grubx64.efi` |
| Secure Boot shim | N/A | `/boot/efi/EFI/redhat/shimx64.efi` |
| grub.cfg | `/boot/grub2/grub.cfg` | `/boot/efi/EFI/redhat/grub.cfg` (minimal) |
| Install command | `grub2-install /dev/sda` | `grub2-install --target=x86_64-efi` |
| Modules | `/boot/grub2/i386-pc/` | `/boot/grub2/x86_64-efi/` |

---

## 29. All Boot-Related Commands

### GRUB2

| Command | Purpose |
|---------|---------|
| `grub2-install /dev/sda` | Install GRUB2 to MBR (BIOS) |
| `grub2-mkconfig -o /boot/grub2/grub.cfg` | Regenerate grub.cfg |
| `grub2-set-default 0` | Set default kernel by index |
| `grub2-set-default "title..."` | Set default kernel by title |
| `grub2-reboot 1` | Boot specific kernel once |
| `grub2-editenv list` | View GRUB2 environment |
| `grubby --info=ALL` | List all kernel entries |
| `grubby --default-kernel` | Show default kernel path |

### Kernel

| Command | Purpose |
|---------|---------|
| `uname -r` | Current kernel version |
| `uname -a` | Full kernel info |
| `cat /proc/cmdline` | Kernel command line used at boot |
| `dmesg` | Kernel ring buffer messages |
| `dmesg -T` | Kernel messages with timestamps |

### initramfs / dracut

| Command | Purpose |
|---------|---------|
| `lsinitrd` | List initramfs contents |
| `lsinitrd -m` | List included modules |
| `dracut --force` | Rebuild initramfs for current kernel |
| `dracut --force --no-hostonly` | Rebuild with ALL drivers |
| `dracut --regenerate-all --force` | Rebuild for all kernels |
| `dracut --add-drivers "driver"` | Include specific driver |

### systemd Boot

| Command | Purpose |
|---------|---------|
| `systemctl get-default` | View default boot target |
| `systemctl set-default multi-user.target` | Set default target |
| `systemctl isolate rescue.target` | Switch to rescue mode |
| `systemctl isolate emergency.target` | Switch to emergency mode |
| `systemctl list-dependencies graphical.target` | View target deps |
| `systemctl list-units --type=target` | List active targets |
| `systemctl daemon-reload` | Reload unit files after editing |
| `systemctl reboot` | Reboot |
| `systemctl poweroff` | Shut down |
| `systemctl reboot --firmware-setup` | Reboot into UEFI setup |

### Boot Analysis

| Command | Purpose |
|---------|---------|
| `systemd-analyze` | Overall boot timing |
| `systemd-analyze blame` | Slowest services |
| `systemd-analyze critical-chain` | Bottleneck dependency path |
| `systemd-analyze plot > boot.svg` | Visual boot timeline |

### Logs

| Command | Purpose |
|---------|---------|
| `journalctl -b` | Current boot logs |
| `journalctl -b -1` | Previous boot logs |
| `journalctl -b -1 -p err` | Previous boot errors only |
| `journalctl -k` | Kernel messages |
| `journalctl -u service.service` | Logs for specific service |
| `journalctl --list-boots` | List available boots |

---

## 30. Boot Troubleshooting Quick Reference

| Symptom | Cause | Fix |
|---------|-------|-----|
| No display, beeps | Hardware failure | Check RAM, video card, PSU |
| "No bootable device" | Wrong boot order or missing MBR | Fix BIOS boot order, reinstall GRUB2 |
| `grub>` prompt | Missing grub.cfg | Manual boot, then `grub2-mkconfig` |
| `grub rescue>` | Missing GRUB2 modules | `insmod normal`, then manual boot |
| Kernel panic (no root FS) | Wrong root= or missing initramfs drivers | Fix root= in GRUB2 or rebuild initramfs |
| `dracut:#` emergency shell | initramfs can't find/mount root | Check device names, LVM, RAID |
| `A start job is running...` (hang) | Bad /etc/fstab entry | Boot to emergency, fix fstab |
| Emergency shell (password prompt) | Critical FS or service failure | Fix fstab or failed services |
| Can't log in | Wrong password, SELinux | rd.break password reset, fix SELinux |
| Slow boot | Service waiting on timeout | `systemd-analyze blame`, disable slow services |

---

## 31. dracut Commands (initramfs Management)

```bash
dracut --force                                  # Rebuild for current kernel
dracut --force --verbose                        # Rebuild with verbose output
dracut --force --no-hostonly                     # Include ALL drivers
dracut --force --add lvm                        # Add LVM support
dracut --force --add-drivers "megaraid_sas"     # Add specific driver
dracut --regenerate-all --force                 # Rebuild for all kernels
lsinitrd /boot/initramfs-$(uname -r).img        # View contents
lsinitrd /boot/initramfs-$(uname -r).img -m     # View modules
```

---

## 32. Boot Performance Analysis

```bash
systemd-analyze                  # Kernel: 1.5s + initrd: 2.3s + userspace: 8.2s = 12.0s
systemd-analyze blame            # Sorted list of slowest units
systemd-analyze critical-chain   # Longest dependency path (bottleneck)
systemd-analyze plot > boot.svg  # SVG timeline of entire boot
```

---

## 33. Service Management Commands

| Command | Purpose |
|---------|---------|
| `systemctl start svc` | Start now |
| `systemctl stop svc` | Stop now |
| `systemctl restart svc` | Restart now |
| `systemctl reload svc` | Reload config (no restart) |
| `systemctl status svc` | Status + recent logs |
| `systemctl enable svc` | Start at boot |
| `systemctl disable svc` | Don't start at boot |
| `systemctl enable --now svc` | Enable + start |
| `systemctl is-active svc` | Running? |
| `systemctl is-enabled svc` | Enabled at boot? |
| `systemctl mask svc` | Block completely |
| `systemctl unmask svc` | Unblock |
| `systemctl --failed` | List failed units |
| `systemctl list-units --type=service` | All loaded services |
| `systemctl cat svc` | View unit file |
| `systemctl edit svc` | Create override |

---

## 34. Persistent Journal Setup

```bash
# Enable persistent logging (survives reboots)
sudo mkdir -p /var/log/journal
sudo vim /etc/systemd/journald.conf
# Set: Storage=persistent
sudo systemctl restart systemd-journald

# Now you can inspect previous boots:
journalctl --list-boots
journalctl -b -1 -p err
```

---

## 35. SELinux and Boot

| Kernel Parameter | Effect |
|-----------------|--------|
| `enforcing=0` | Permissive mode (logs but doesn't block) |
| `enforcing=1` | Enforcing mode (default) |
| `selinux=0` | Disable SELinux entirely (NOT recommended) |

```bash
getenforce                          # Current mode
setenforce 0                        # Temporarily permissive
setenforce 1                        # Temporarily enforcing
restorecon -Rv /path                # Fix SELinux labels
touch /.autorelabel && reboot       # Full filesystem relabel
```

---

## 36. RHEL Kernel Versions

| RHEL | Kernel | Compression |
|------|--------|-------------|
| RHEL 7 | 3.10.x | gzip |
| RHEL 8 | 4.18.x | xz |
| RHEL 9 | 5.14.x | zstd |

Version format: `5.14.0-284.el9.x86_64` = major.minor.patch-build.distro.arch

---

## 37. Boot Flow -- One-Line Summary

```
Power → POST → MBR (boot.img) → diskboot.img → core.img → GRUB2 →
vmlinuz + initramfs → Kernel → systemd (initramfs) → mount /sysroot →
switch_root → systemd (real OS) → targets → services → LOGIN
```

---

## 38. Interview Power Answers -- One-Liners

| Question | Power Answer |
|----------|-------------|
| Describe the RHEL boot process | Firmware runs POST and loads GRUB2 from MBR/ESP; GRUB2 loads the kernel and initramfs; the kernel starts systemd in initramfs to mount the real root on /sysroot; after switch_root, systemd on the real OS starts services until the login target is reached. |
| What is the MBR? | The first 512 bytes of a disk: 446 bytes of bootstrap code (boot.img), 64 bytes of partition table (4 entries), and a 2-byte boot signature (0x55AA). |
| What is boot.img? | A 446-byte program in the MBR that loads the first sector of core.img (diskboot.img) from the MBR gap. |
| What is diskboot.img? | The first 512 bytes of core.img; its job is to load the rest of core.img which contains filesystem drivers needed to find /boot. |
| What is core.img? | GRUB2's Stage 1.5 -- lives in the MBR gap, contains filesystem modules that let GRUB2 read /boot/grub2/ from a real filesystem. |
| What is GRUB2? | The bootloader on RHEL -- it displays a menu, loads the kernel and initramfs into RAM, passes kernel parameters, and hands control to the kernel. |
| What is vmlinuz? | The compressed Linux kernel image. "vm" = virtual memory, "z" = compressed. GRUB2 loads it into RAM and the decompression stub unpacks it. |
| What is initramfs? | A temporary RAM-based root filesystem containing drivers and tools needed to find and mount the real root. Solves the chicken-and-egg problem of needing drivers that are stored on the root filesystem. |
| What is /sysroot? | The mountpoint inside initramfs where the real root filesystem is mounted before switch_root pivots to it. |
| What is switch_root? | The operation that deletes the initramfs tmpfs, makes /sysroot the new /, and re-executes systemd from the real OS. |
| What is rd.break? | A kernel parameter that pauses boot just before switch_root, giving you an initramfs shell. Used for root password resets. |
| Why touch /.autorelabel? | Because rd.break operates before SELinux is active. Password changes create files without SELinux labels. /.autorelabel forces a full relabel on next boot. |
| What is systemd? | The init system on RHEL 7/8/9. Runs as PID 1, manages services, mounts, sockets, and targets. Replaced SysV init. |
| What is a systemd target? | A group of units representing a system state -- like milestones. multi-user.target = text mode, graphical.target = GUI mode. |
| How does systemd know which services to start? | Enabled services have symlinks in `/etc/systemd/system/<target>.wants/`. systemd resolves dependencies and starts units in parallel. |
| Rescue vs emergency mode? | Rescue: root FS read-write, basic services running. Emergency: root FS read-only, almost nothing running. Both require root password. |
| How to reset root password? | Boot → interrupt GRUB2 → add rd.break → remount /sysroot rw → chroot /sysroot → passwd root → touch /.autorelabel → exit twice. |
| What does GRUB2 load into memory? | Two things: vmlinuz (the kernel) and initramfs (the initial filesystem with drivers). |
| BIOS vs UEFI? | BIOS: 16-bit, MBR, 2TB max, no Secure Boot. UEFI: 32/64-bit, GPT, unlimited disk size, Secure Boot, faster. |
| What is dracut? | The tool that builds initramfs on RHEL. `dracut --force` rebuilds it. |
| What is getty? | The program that displays the text login prompt on virtual terminals. Managed by systemd as `getty@tty1.service`. |
| How to check boot time? | `systemd-analyze` for overall timing, `systemd-analyze blame` for per-service breakdown. |
| What is PID 1? Why is it special? | systemd. Special because: it's the first userspace process, it can't be killed (kernel panics if it dies), and it adopts all orphan processes. |
| What is the MBR gap? | ~1 MB of space between the MBR (sector 0) and the first partition (sector 2048). GRUB2's core.img lives here on BIOS systems. |
| Where is GRUB2 config stored? | `/boot/grub2/grub.cfg` (auto-generated). Settings are in `/etc/default/grub`. Regenerate with `grub2-mkconfig`. |

---

## 39. Interview Tips

1. **Walk through the stages sequentially** -- Firmware → MBR → GRUB2 → Kernel → initramfs → systemd → Login. This shows structured thinking.

2. **Know the "why" behind each stage** -- Why initramfs exists (chicken-and-egg), why MBR has stages (446 bytes isn't enough), why systemd runs twice (once for mounting, once for services).

3. **Know the troubleshooting procedures** -- Root password reset (rd.break), broken fstab (emergency mode), broken GRUB2 (manual boot from grub> prompt).

4. **Know the key files and commands** -- `/etc/default/grub`, `grub2-mkconfig`, `dracut --force`, `systemctl set-default`, `cat /proc/cmdline`.

5. **Use real examples** -- "On our production servers, we use multi-user.target because we don't need a GUI, and we've had to use rd.break to reset passwords after lockouts."

6. **It's okay to say "I'd check the man page"** -- For exact `dracut` flags or `grub.cfg` syntax, saying you'd verify is professional.

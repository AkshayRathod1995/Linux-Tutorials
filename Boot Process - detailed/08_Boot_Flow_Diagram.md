# Boot Flow Diagram -- Complete Visual Reference

> Print this page or keep it open during revision. It covers the entire RHEL boot process from power button to login screen.

---

## The Complete Boot Flow (BIOS/MBR System)

```
╔══════════════════════════════════════════════════════════════════════════════════╗
║                           POWER BUTTON PRESSED                                  ║
╚══════════════════════════════════════════════════════════════════════════════════╝
                                      │
                                      ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│  STAGE 1: FIRMWARE (BIOS/UEFI)                                                  │
│                                                                                  │
│  1. PSU sends "Power Good" signal to motherboard                                │
│  2. CPU wakes up, jumps to firmware code (0xFFFF0)                              │
│  3. POST (Power-On Self-Test):                                                  │
│     ├── Test CPU                                                                │
│     ├── Test RAM                                                                │
│     ├── Test Video                                                              │
│     ├── Test Keyboard                                                           │
│     └── Test Storage controllers                                                │
│  4. Search boot order for bootable device                                       │
│  5. Read first 512 bytes (MBR) of bootable disk                                │
│  6. Check boot signature (0x55AA at bytes 510-511)                              │
│  7. Load MBR into memory at 0x7C00 and jump to it                              │
│                                                                                  │
│  Key files: None (firmware is on a chip)                                        │
│  Config: BIOS/UEFI setup (F2/Del during boot)                                  │
└──────────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│  STAGE 2: MBR & GRUB2 STAGE 1 (boot.img)                                       │
│                                                                                  │
│  MBR (512 bytes):                                                               │
│  ┌──────────────────────────────────┐                                           │
│  │ Bootstrap code (boot.img) 446 B  │ → Tiny program, loads diskboot.img        │
│  │ Partition table             64 B  │ → 4 partition entries × 16 bytes          │
│  │ Boot signature (0x55AA)      2 B  │ → "This disk is bootable"                │
│  └──────────────────────────────────┘                                           │
│                                                                                  │
│  boot.img loads the first sector of core.img (which is diskboot.img)            │
│  from a hardcoded sector number in the MBR gap                                  │
│                                                                                  │
│  Key files: /usr/lib/grub/i386-pc/boot.img                                     │
│  Config: grub2-install /dev/sda                                                 │
└──────────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│  STAGE 2.5: MBR GAP (diskboot.img + core.img)                                  │
│                                                                                  │
│  diskboot.img loads the rest of core.img (~32 KB) from the MBR gap              │
│  (sectors 1-2047, between MBR and first partition)                              │
│                                                                                  │
│  core.img = diskboot.img + decompressor + filesystem modules + GRUB2 kernel     │
│                                                                                  │
│  Now GRUB2 can read filesystems (XFS, ext4) and find /boot/grub2/               │
│                                                                                  │
│  Key files: /usr/lib/grub/i386-pc/diskboot.img, core.img                        │
│  Config: Built automatically by grub2-install                                   │
└──────────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│  STAGE 3: GRUB2 (Full Bootloader)                                               │
│                                                                                  │
│  1. Read /boot/grub2/grub.cfg                                                   │
│  2. Display boot menu (kernel selection)                                        │
│     ┌───────────────────────────────────────────────┐                            │
│     │  Red Hat Enterprise Linux                     │                            │
│     │    ► Kernel 5.14.0-284.el9.x86_64            │                            │
│     │      Kernel 5.14.0-70.el9.x86_64             │                            │
│     │                                               │                            │
│     │  Press 'e' to edit, 'c' for command line     │                            │
│     └───────────────────────────────────────────────┘                            │
│  3. Load vmlinuz (compressed kernel) into RAM                                   │
│  4. Load initramfs-*.img into RAM                                               │
│  5. Pass kernel command line parameters                                         │
│  6. Transfer control to the kernel                                              │
│                                                                                  │
│  Key files: /boot/grub2/grub.cfg, /boot/vmlinuz-*, /boot/initramfs-*           │
│  Config: /etc/default/grub → grub2-mkconfig                                    │
└──────────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│  STAGE 4: KERNEL LOADING                                                        │
│                                                                                  │
│  1. Real-mode setup code runs (from vmlinuz header)                             │
│     ├── Parse boot parameters                                                   │
│     ├── Detect CPU features                                                     │
│     ├── Query memory map from BIOS                                              │
│     └── Switch CPU: Real Mode → Protected Mode → Long Mode (64-bit)            │
│  2. Decompression stub runs                                                     │
│     └── Decompress kernel payload (gzip/xz/zstd) → ~30-50 MB                   │
│  3. start_kernel() runs:                                                        │
│     ├── Initialize memory management (page tables, allocator)                   │
│     ├── Initialize scheduler, interrupts, timers                                │
│     ├── Initialize VFS (Virtual File System)                                    │
│     ├── Initialize console (for kernel messages → dmesg)                        │
│     └── rest_init():                                                            │
│         ├── Create PID 1 → /sbin/init (systemd from initramfs)                 │
│         ├── Create PID 2 → kthreadd (kernel thread daemon)                     │
│         └── Become PID 0 → idle process                                        │
│                                                                                  │
│  Key files: /boot/vmlinuz-*                                                     │
│  Config: Kernel command line (from GRUB2)                                       │
└──────────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│  STAGE 5: initramfs (Initial RAM Filesystem)                                    │
│                                                                                  │
│  Kernel unpacks initramfs (cpio archive) into tmpfs at /                        │
│  systemd starts as PID 1 (from initramfs)                                       │
│                                                                                  │
│  ┌─ sysinit.target ──────────────────────────────────────────┐                  │
│  │  Load kernel modules (storage drivers, FS drivers)        │                  │
│  │  Start udevd (detect hardware, create /dev nodes)         │                  │
│  │  Parse kernel command line (root=, rd.lvm.lv=)            │                  │
│  └───────────────────────────────────────────────────────────┘                  │
│                          │                                                       │
│  ┌─ initrd-root-device.target ───────────────────────────────┐                  │
│  │  Activate LVM: /dev/mapper/rhel-root                      │                  │
│  │  Assemble RAID arrays (if any)                            │                  │
│  │  Unlock LUKS encryption (if any)                          │                  │
│  │  Wait for root device to appear                           │                  │
│  └───────────────────────────────────────────────────────────┘                  │
│                          │                                                       │
│  ┌─ initrd-root-fs.target ───────────────────────────────────┐                  │
│  │  Run filesystem check (fsck / XFS journal replay)         │                  │
│  │  Mount root filesystem READ-ONLY on /sysroot              │                  │
│  └───────────────────────────────────────────────────────────┘                  │
│                          │                                                       │
│  ┌─ initrd.target ──────────────────────────────────────────┐                   │
│  │  Mount /boot and other early filesystems on /sysroot     │                   │
│  │  Everything ready for switch_root                        │                   │
│  └──────────────────────────────────────────────────────────┘                   │
│                          │                                                       │
│         ══════ rd.break pauses HERE (if set) ══════                             │
│                          │                                                       │
│  ┌─ switch_root ────────────────────────────────────────────┐                   │
│  │  1. Delete all initramfs files (free RAM)                │                   │
│  │  2. Make /sysroot the new / (pivot_root)                 │                   │
│  │  3. Re-execute systemd from real OS                      │                   │
│  └──────────────────────────────────────────────────────────┘                   │
│                                                                                  │
│  Key files: /boot/initramfs-*, /etc/fstab (on real root)                        │
│  Config: /etc/dracut.conf.d/, kernel command line                               │
└──────────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│  STAGE 6: systemd on Real OS                                                    │
│                                                                                  │
│  systemd (PID 1) re-executes from /usr/lib/systemd/systemd                      │
│  Reads default.target → resolves dependencies → starts units in parallel        │
│                                                                                  │
│  ┌─ local-fs.target ────────────────────────────────────────┐                   │
│  │  Mount all filesystems from /etc/fstab                   │                   │
│  │  Remount / as read-write                                 │                   │
│  │  Enable swap                                             │                   │
│  └──────────────────────────────────────────────────────────┘                   │
│                          │                                                       │
│  ┌─ sysinit.target ────────────────────────────────────────┐                    │
│  │  Set hostname, load kernel modules                       │                   │
│  │  Apply sysctl, start journald, start udevd               │                   │
│  └──────────────────────────────────────────────────────────┘                   │
│                          │                                                       │
│  ┌─ basic.target ──────────────────────────────────────────┐                    │
│  │  Start D-Bus, logind, timers, sockets, paths             │                   │
│  └──────────────────────────────────────────────────────────┘                   │
│                          │                                                       │
│  ┌─ multi-user.target ─────────────────────────────────────┐                    │
│  │  Start: sshd, crond, chronyd, firewalld, NetworkManager  │                   │
│  │  Start: rsyslog, tuned, auditd, and all enabled services │                   │
│  │  Start: getty@tty1 (text login prompt)                    │                   │
│  └──────────────────────────────────────────────────────────┘                   │
│                          │                                                       │
│  ┌─ graphical.target (if default) ─────────────────────────┐                    │
│  │  Start: gdm.service (GNOME Display Manager)              │                   │
│  │  → X server / Wayland compositor                         │                   │
│  │  → Graphical login screen                                │                   │
│  └──────────────────────────────────────────────────────────┘                   │
│                                                                                  │
│  Key files: /usr/lib/systemd/system/*.service, /etc/systemd/system/             │
│  Config: systemctl get-default, systemctl set-default                           │
└──────────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
╔══════════════════════════════════════════════════════════════════════════════════╗
║                         LOGIN SCREEN APPEARS                                    ║
║                                                                                  ║
║   multi-user.target → Text:  "servera login: _"                                ║
║   graphical.target  → GUI:   GDM / GNOME login screen                          ║
╚══════════════════════════════════════════════════════════════════════════════════╝
```

---

## The Complete Boot Flow (UEFI/GPT System)

The UEFI flow differs only in the early stages:

```
╔══════════════════════════════════════════════════════════════════════════╗
║  POWER BUTTON → PSU → Power Good → CPU wakes up                       ║
╚══════════════════════════════════════════════════════════════════════════╝
                                │
                                ▼
┌──────────────────────────────────────────────────────────────────────────┐
│  UEFI Firmware                                                          │
│  ├── POST (same as BIOS)                                               │
│  ├── Read NVRAM boot entries (efibootmgr)                              │
│  ├── Find EFI System Partition (ESP, FAT32)                            │
│  └── Load /boot/efi/EFI/redhat/shimx64.efi                            │
└──────────────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌──────────────────────────────────────────────────────────────────────────┐
│  Secure Boot Chain                                                      │
│  shimx64.efi (signed by Microsoft)                                     │
│      │                                                                  │
│      └── Verifies and loads grubx64.efi (signed by Red Hat)            │
│              │                                                          │
│              └── GRUB2 loads vmlinuz + initramfs (same as BIOS flow)   │
└──────────────────────────────────────────────────────────────────────────┘
                                │
                                ▼
                  (Same as BIOS flow from Stage 3 onward)
```

---

## Simplified One-Line Summary

```
Power → BIOS/UEFI → POST → MBR/ESP → boot.img → diskboot.img → GRUB2 →
vmlinuz + initramfs → Kernel → systemd(initramfs) → mount /sysroot →
switch_root → systemd(real OS) → targets → services → LOGIN
```

---

## Where Each Piece Lives on Disk

```
Disk: /dev/sda
├── Sector 0 (MBR)
│   └── boot.img (446 bytes) + partition table (64 bytes) + 0x55AA (2 bytes)
├── Sectors 1-2047 (MBR Gap)
│   └── core.img (diskboot.img + filesystem modules + GRUB2 kernel)
├── Partition 1: /boot (XFS, ~1 GB)
│   ├── vmlinuz-5.14.0-284.el9.x86_64          ← Compressed kernel
│   ├── initramfs-5.14.0-284.el9.x86_64.img    ← Initial RAM filesystem
│   ├── config-5.14.0-284.el9.x86_64           ← Kernel build config
│   ├── System.map-5.14.0-284.el9.x86_64       ← Kernel symbol table
│   ├── grub2/
│   │   ├── grub.cfg                            ← GRUB2 config (auto-generated)
│   │   ├── grubenv                             ← GRUB2 environment (saved_entry)
│   │   └── i386-pc/                            ← GRUB2 modules (BIOS)
│   └── loader/entries/                          ← BLS kernel entries (RHEL 8/9)
│       └── *.conf
├── Partition 2: LVM Physical Volume
│   └── Volume Group: rhel
│       ├── LV: rhel-root → / (XFS)
│       │   ├── /usr/lib/systemd/systemd        ← systemd binary
│       │   ├── /etc/systemd/system/            ← systemd config & symlinks
│       │   ├── /etc/fstab                      ← Filesystem mount table
│       │   ├── /etc/default/grub               ← GRUB2 settings
│       │   └── /etc/dracut.conf.d/             ← dracut config
│       └── LV: rhel-swap → swap
└── (UEFI only) EFI System Partition: /boot/efi (FAT32)
    └── EFI/redhat/
        ├── shimx64.efi                         ← Secure Boot shim
        ├── grubx64.efi                         ← GRUB2 EFI binary
        └── grub.cfg                            ← Minimal EFI GRUB config
```

---

## Troubleshooting Quick Reference: Where to Intervene

```
PROBLEM AT                      │  HOW TO INTERVENE
────────────────────────────────┼──────────────────────────────────────
BIOS/POST fails                 │  Check hardware, reseat RAM, check PSU
No bootable device              │  Check boot order in BIOS, disk connections
GRUB2 menu doesn't appear       │  Reinstall GRUB2: grub2-install /dev/sda
GRUB2 shows error               │  Press 'c' for command line, boot manually
Wrong kernel boots               │  Edit entry with 'e', change vmlinuz version
Kernel panic (no root FS)        │  Rebuild initramfs: dracut --force
initramfs can't find root        │  Fix root= parameter in GRUB2
rd.break                        │  Pause here to reset root password
systemd fails (emergency shell)  │  Fix /etc/fstab, run systemctl daemon-reload
Service won't start             │  journalctl -u service-name, systemctl status
No login prompt                 │  Check getty or gdm: systemctl status getty@tty1
```

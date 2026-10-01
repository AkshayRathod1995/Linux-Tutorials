# Stage 5: initramfs & Mounting the Real Root

## What Is initramfs?

**Initial RAM File System** -- a compressed cpio archive containing a mini OS (systemd, drivers, LVM tools, udev) that lives entirely in RAM.

It solves the **chicken-and-egg problem:** the kernel needs storage/filesystem drivers to read the root filesystem, but those drivers are stored ON the root filesystem. initramfs carries them in RAM.

```bash
ls -lh /boot/initramfs-$(uname -r).img    # ~34 MB
lsinitrd /boot/initramfs-$(uname -r).img   # View contents
```

---

## What Happens Inside initramfs

The kernel unpacks the cpio archive into a tmpfs at `/`. systemd starts as PID 1 using `initrd.target`:

```
1. sysinit.target
   └── Load storage/filesystem drivers, start udevd (hardware detection)

2. initrd-root-device.target
   └── Activate LVM, assemble RAID, unlock LUKS encryption

3. initrd-root-fs.target
   └── Filesystem check → mount root READ-ONLY on /sysroot

4. initrd.target
   └── Mount /boot and other early filesystems

   ════ rd.break pauses HERE ════

5. switch_root
   ├── Delete initramfs files (free RAM)
   ├── Make /sysroot the new /
   └── Re-execute systemd from real OS
```

---

## /sysroot

The real root filesystem is mounted at `/sysroot` inside initramfs. After `switch_root`, `/sysroot` becomes `/` and initramfs is deleted from memory.

```
BEFORE switch_root:               AFTER switch_root:
/ (initramfs tmpfs)                / (real XFS root)
├── /sbin/init                     ├── /bin/
├── /usr/lib/...                   ├── /etc/
└── /sysroot/  ← real OS here     ├── /usr/
    ├── bin/                       └── /var/
    ├── etc/                       (initramfs gone)
    └── usr/
```

---

## Root Password Reset (rd.break)

```bash
# 1. Reboot → GRUB2 → press 'e' → add 'rd.break' to linux line → Ctrl+x
# 2. At switch_root prompt:
mount -o remount,rw /sysroot
chroot /sysroot
passwd root
touch /.autorelabel     # SELinux isn't active yet -- force relabel
exit
exit
# 3. System boots, does SELinux relabel, reboots. Log in with new password.
```

**Why `/.autorelabel`:** `passwd` creates a new `/etc/shadow` without SELinux labels. Without relabeling, SELinux blocks login.

---

## Rebuilding initramfs (dracut)

```bash
sudo dracut --force                        # Rebuild for current kernel
sudo dracut --force --no-hostonly           # Include ALL drivers (works on any hardware)
sudo dracut --regenerate-all --force       # Rebuild for all installed kernels
sudo dracut --force --add-drivers "nvme"   # Add a specific driver
```

| Config | Purpose |
|--------|---------|
| `/etc/dracut.conf` | Main dracut config |
| `/etc/dracut.conf.d/*.conf` | Drop-in configs |

**Next:** systemd on the real OS takes over.

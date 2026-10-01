# Troubleshooting Boot Issues on RHEL

## Which Stage Failed?

| Symptom | Stage | Fix Area |
|---------|-------|----------|
| No display, beep codes | BIOS/POST | Hardware |
| "No bootable device" | MBR/BIOS | GRUB reinstall |
| `grub>` or `grub rescue>` prompt | GRUB2 | grub.cfg / grub2-install |
| Kernel panic | Kernel / initramfs | Rebuild initramfs, fix root= |
| Dropped to emergency shell | systemd / fstab | Fix fstab, filesystem |
| Services fail, no login | systemd services | Disable bad service |

---

## Problem 1: GRUB2 Missing / Corrupted

**Symptom:** `grub>` prompt, `grub rescue>`, or "No bootable device"

**At grub> prompt (manual boot):**
```bash
grub> set root=(hd0,msdos1)
grub> linux /vmlinuz-5.14.0-284.el9.x86_64 root=/dev/mapper/rhel-root ro
grub> initrd /initramfs-5.14.0-284.el9.x86_64.img
grub> boot
```

**After booting (permanent fix):**
```bash
sudo grub2-install /dev/sda
sudo grub2-mkconfig -o /boot/grub2/grub.cfg
```

---

## Problem 2: Kernel Panic -- Can't Mount Root

**Symptom:** `VFS: Unable to mount root fs` or `Kernel panic - not syncing`

**Causes:** Wrong `root=` parameter, missing LVM/RAID drivers in initramfs, corrupted initramfs

**Fix:**
```bash
# Boot into rescue mode, then:
sudo dracut --force --regenerate-all     # Rebuild initramfs
cat /proc/cmdline                         # Verify root= is correct
```

---

## Problem 3: Bad /etc/fstab Entry

**Symptom:** Drops to emergency shell with `Failed to mount` errors

**Fix:**
```bash
# In emergency shell:
mount -o remount,rw /                    # Make root writable
vi /etc/fstab                            # Fix or comment out bad entry
systemctl daemon-reload
exit                                     # Continue boot
```

**Prevention:** Always test fstab changes with `mount -a` before rebooting.

---

## Problem 4: Forgotten Root Password

```bash
# 1. Reboot → GRUB2 → press 'e'
# 2. Add 'rd.break' to end of linux line → Ctrl+x
# 3. At prompt:
mount -o remount,rw /sysroot
chroot /sysroot
passwd root
touch /.autorelabel
exit
exit
# 4. System reboots, relabels SELinux, reboots again. Done.
```

---

## Problem 5: Emergency / Rescue Shell

```bash
# Already in emergency shell:
journalctl -xb                           # Check what failed
systemctl list-units --failed             # List failed units
mount -o remount,rw /                     # Make root writable (emergency mode)

# Boot into rescue mode deliberately:
# GRUB2 → press 'e' → add: systemd.unit=rescue.target
```

| Mode | Root FS | Services | Use When |
|------|---------|----------|----------|
| `rescue.target` | Read-write | Minimal | Need to fix services/config |
| `emergency.target` | **Read-only** | Almost none | Filesystem repair |

---

## Problem 6: Service Blocking Boot

```bash
# In rescue mode:
systemctl list-units --failed
systemctl disable bad-service.service    # Don't start at boot
systemctl mask bad-service.service       # Prevent starting entirely
systemctl default                        # Continue to normal boot
```

---

## Problem 7: SELinux Blocking Boot

**Symptom:** Login fails after password change, services fail with AVC denials

```bash
# Temporary: boot with SELinux permissive
# GRUB2 → press 'e' → add: enforcing=0

# Fix: relabel filesystem
touch /.autorelabel
reboot

# Check for SELinux issues:
ausearch -m AVC -ts recent
restorecon -Rv /path/to/affected/files
```

---

## Problem 8: Filesystem Corruption

```bash
# Boot into emergency mode, then:
umount /dev/sda1                         # Unmount first
xfs_repair /dev/sda1                     # XFS filesystem
fsck.ext4 /dev/sda1                      # ext4 filesystem
```

**Never run xfs_repair/fsck on a mounted filesystem.**

---

## Problem 9: Slow Boot

```bash
systemd-analyze                          # Total boot time
systemd-analyze blame                    # Slowest units
systemd-analyze critical-chain           # What blocked what
systemd-analyze plot > boot.svg          # Visual timeline
```

---

## Quick Reference

| Task | Command |
|------|---------|
| Rebuild GRUB2 | `grub2-install /dev/sda && grub2-mkconfig -o /boot/grub2/grub.cfg` |
| Rebuild initramfs | `dracut --force` |
| Reset root password | `rd.break` → remount → chroot → passwd → autorelabel |
| Check boot logs | `journalctl -b` |
| Check failed services | `systemctl list-units --failed` |
| SELinux relabel | `touch /.autorelabel && reboot` |
| Boot to rescue | `systemd.unit=rescue.target` in GRUB2 |

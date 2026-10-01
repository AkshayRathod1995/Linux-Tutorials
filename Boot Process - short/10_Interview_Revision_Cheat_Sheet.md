# Interview Revision Cheat Sheet: RHEL Boot Process

> One file. Everything you need. Read this the night before.

---

## The Boot Process in One Sentence

**Power → BIOS/UEFI (POST) → MBR/ESP → boot.img → core.img → GRUB2 → vmlinuz → initramfs → switch_root → systemd → login**

---

## 7 Stages at a Glance

| # | Stage | What Happens |
|---|-------|-------------|
| 1 | **BIOS/UEFI** | POST → find bootable device |
| 2 | **MBR** | boot.img (446B) → diskboot.img → core.img (MBR gap) |
| 3 | **GRUB2** | Read grub.cfg → show menu → load vmlinuz + initramfs |
| 4 | **Kernel** | Decompress → start_kernel → PID 1 (systemd in initramfs) |
| 5 | **initramfs** | Load drivers → LVM → mount root on /sysroot → switch_root |
| 6 | **systemd** | Reach default.target → start services |
| 7 | **Login** | getty (text) or GDM (graphical) |

---

## Key Vocabulary

| Term | Meaning |
|------|---------|
| **POST** | Power-On Self-Test -- hardware check by firmware |
| **MBR** | Master Boot Record -- first 512 bytes of disk |
| **MBR gap** | Sectors 1-2047 (~1 MB) between MBR and first partition. Holds core.img |
| **boot.img** | 446-byte bootstrap in MBR, loads diskboot.img |
| **diskboot.img** | First sector of core.img, loads rest of core.img |
| **core.img** | Filesystem drivers (XFS/ext4), can read /boot |
| **GRUB2** | Bootloader. Loads kernel + initramfs, passes params |
| **vmlinuz** | Compressed kernel image (~11 MB → ~40 MB decompressed) |
| **initramfs** | RAM filesystem with mini OS. Solves chicken-and-egg problem |
| **/sysroot** | Where real root is mounted inside initramfs |
| **switch_root** | Makes /sysroot the new /, deletes initramfs, re-execs systemd |
| **dracut** | Tool to build/rebuild initramfs images |

---

## BIOS vs UEFI

| | BIOS | UEFI |
|---|---|---|
| CPU mode | 16-bit | 32/64-bit |
| Partition | MBR (2 TB, 4 partitions) | GPT (no practical limits) |
| Boot chain | MBR → boot.img → core.img | shimx64.efi → grubx64.efi |
| Secure Boot | No | Yes |
| Check | N/A | `[ -d /sys/firmware/efi ] && echo UEFI` |

---

## MBR Structure (512 Bytes)

```
Bootstrap (boot.img)   446 B
Partition Table         64 B   ← 4 entries × 16 B (why max 4 primaries)
Boot Signature           2 B   ← 0x55AA
```

**Why max 2 TB:** 4-byte sector count → 2^32 × 512 = 2 TB

---

## GRUB2 Quick Facts

- Config: `/etc/default/grub` → `grub2-mkconfig` → `/boot/grub2/grub.cfg`
- **Never edit grub.cfg directly**
- Key params: `root=`, `rd.lvm.lv=`, `rhgb quiet`, `rd.break`, `systemd.unit=`
- `cat /proc/cmdline` → see what was passed at boot
- Temp edit: GRUB menu → `e` → edit linux line → `Ctrl+x`

---

## Kernel (vmlinuz)

```
Real-mode header (~15 KB)     → Boot params, CPU detection
Decompression stub (~20 KB)   → Unpacks kernel
Compressed payload (~10 MB)   → The actual kernel
```

**start_kernel() → rest_init() → PID 0 (idle), PID 1 (systemd), PID 2 (kthreadd)**

---

## initramfs

- Compressed cpio archive in RAM
- Contains systemd, drivers, LVM tools, udev
- Target chain: `sysinit → initrd-root-device → initrd-root-fs → initrd.target → switch_root`
- `rd.break` pauses BEFORE switch_root (used for password reset)
- Rebuild: `dracut --force`

---

## systemd Targets

| Target | = Runlevel | Meaning |
|--------|-----------|---------|
| `poweroff.target` | 0 | Halt |
| `rescue.target` | 1 | Single user, all FS mounted |
| `multi-user.target` | 3 | Full system, no GUI |
| `graphical.target` | 5 | Full system + GUI |
| `reboot.target` | 6 | Reboot |
| `emergency.target` | -- | Root shell, root FS read-only |

```bash
systemctl get-default                          # Check default target
systemctl set-default multi-user.target        # Change default
systemctl isolate rescue.target                # Switch now
```

---

## Root Password Reset Procedure

```
1. Reboot → GRUB2 → press 'e'
2. Add 'rd.break' to linux line → Ctrl+x
3. mount -o remount,rw /sysroot
4. chroot /sysroot
5. passwd root
6. touch /.autorelabel     ← CRITICAL (SELinux)
7. exit → exit → reboots → relabels → reboots → done
```

---

## Essential Commands

| Task | Command |
|------|---------|
| Current kernel | `uname -r` |
| BIOS or UEFI? | `[ -d /sys/firmware/efi ] && echo UEFI \|\| echo BIOS` |
| Boot parameters | `cat /proc/cmdline` |
| Kernel messages | `dmesg -T` or `journalctl -k` |
| Boot logs | `journalctl -b` |
| Boot time | `systemd-analyze` |
| Slowest services | `systemd-analyze blame` |
| Failed services | `systemctl list-units --failed` |
| Rebuild initramfs | `dracut --force` |
| Rebuild GRUB2 | `grub2-install /dev/sda && grub2-mkconfig -o /boot/grub2/grub.cfg` |
| View initramfs | `lsinitrd /boot/initramfs-$(uname -r).img` |
| List kernels | `grubby --info=ALL` |
| Service control | `systemctl start/stop/enable/disable/status <service>` |

---

## Top Interview Questions (Quick Answers)

**Q: Explain the Linux boot process.**
> Power → BIOS POST → MBR (boot.img loads core.img from MBR gap) → GRUB2 (loads vmlinuz + initramfs) → kernel decompresses, runs start_kernel, spawns PID 1 → systemd in initramfs loads drivers, mounts root on /sysroot, switch_root → systemd on real OS reaches default.target → login.

**Q: Why do we need initramfs?**
> Chicken-and-egg: kernel needs storage drivers to read root filesystem, but drivers are ON the root filesystem. initramfs carries them in RAM.

**Q: How to reset root password?**
> rd.break at GRUB → remount /sysroot rw → chroot → passwd → touch /.autorelabel → exit × 2.

**Q: What is switch_root?**
> Deletes initramfs from RAM, makes /sysroot the new /, re-executes systemd from real OS.

**Q: BIOS vs UEFI?**
> BIOS: 16-bit, MBR, 2TB limit, no Secure Boot. UEFI: 64-bit, GPT, no limits, Secure Boot via shim.

**Q: What is the MBR gap?**
> ~1 MB space (sectors 1-2047) between MBR and first partition. Holds GRUB2's core.img.

**Q: What does systemd do during boot?**
> Runs twice. In initramfs: loads drivers, mounts root. On real OS: reaches default.target, starts all services.

**Q: Rescue vs Emergency mode?**
> Rescue: all filesystems mounted r/w, minimal services. Emergency: root FS read-only, almost nothing running.

**Q: How to troubleshoot slow boot?**
> `systemd-analyze blame` shows slowest units. `systemd-analyze critical-chain` shows blocking dependencies.

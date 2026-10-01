# Stage 4: Kernel Loading & vmlinuz

## What Is vmlinuz?

The compressed Linux kernel image. **v**irtual **m**emory **linu**x compresse**d**.

| Property | Value |
|----------|-------|
| Location | `/boot/vmlinuz-<version>` |
| Compressed size | ~11 MB |
| Decompressed size | ~30-50 MB |
| Compression (RHEL 9) | zstd (fast) |
| Compression (RHEL 8) | xz |
| Format | bzImage (big zImage) |

```bash
uname -r                         # Current kernel version
ls -lh /boot/vmlinuz-*           # All installed kernels
file /boot/vmlinuz-$(uname -r)   # File format info
```

---

## What vmlinuz Contains

```
┌─────────────────────────────────┐
│ Real-mode header (~15 KB)       │  Boot params, CPU mode switching code
├─────────────────────────────────┤
│ Decompression stub (~20 KB)     │  Unpacks the compressed kernel below
├─────────────────────────────────┤
│ Compressed kernel payload       │  The actual kernel (~10 MB → 40 MB)
└─────────────────────────────────┘
```

---

## What Happens After GRUB2 Hands Off

1. **Real-mode setup code** runs: parse boot params, detect CPU, query memory map from BIOS
2. **CPU mode switch:** Real Mode (16-bit) → Protected Mode (32-bit) → Long Mode (64-bit)
3. **Decompression stub** unpacks the kernel payload into RAM
4. **`start_kernel()`** runs:
   - Initialize memory management (page tables, allocator)
   - Initialize scheduler, interrupts, timers
   - Initialize VFS (Virtual File System), console
5. **`rest_init()`** creates the first userspace processes:

| PID | Process | Role |
|-----|---------|------|
| 0 | idle | Runs when CPU has nothing to do |
| **1** | **`/sbin/init` → systemd (from initramfs)** | **Init system, manages everything** |
| 2 | kthreadd | Manages kernel threads |

---

## Kernel Messages

```bash
dmesg                   # View kernel boot messages
dmesg -T                # With human-readable timestamps
dmesg | grep -i memory  # Filter for specific topics
journalctl -k           # Kernel messages via systemd journal
```

---

## RHEL Kernel Versions

| RHEL | Kernel | Format example |
|------|--------|---------------|
| 7 | 3.10.x | `3.10.0-1160.el7.x86_64` |
| 8 | 4.18.x | `4.18.0-477.el8.x86_64` |
| 9 | 5.14.x | `5.14.0-284.el9.x86_64` |

Version: `5.14.0-284.el9.x86_64` = major.minor.patch-build.distro.arch

**Next:** The kernel is running but has NOT mounted the real root filesystem. That's the job of initramfs.

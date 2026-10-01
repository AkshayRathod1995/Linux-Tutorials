# Boot Flow Diagram

> Print this or keep it open during revision.

---

## Complete RHEL Boot Flow

```
  ┌─────────────────────────────┐
  │      POWER BUTTON           │
  └──────────────┬──────────────┘
                 ▼
  ┌─────────────────────────────┐
  │  BIOS / UEFI                │
  │  PSU → CPU → POST           │
  │  Find bootable device        │
  └──────────────┬──────────────┘
                 ▼
       ┌─── BIOS? ───┐
       │              │
       ▼              ▼
  ┌─────────┐   ┌──────────┐
  │   MBR   │   │   UEFI   │
  │ 512 B   │   │   ESP     │
  └────┬────┘   └────┬─────┘
       │              │
       ▼              ▼
  ┌─────────┐   ┌──────────┐
  │boot.img │   │shimx64   │
  │  446 B  │   │  .efi    │
  └────┬────┘   └────┬─────┘
       │              │
       ▼              ▼
  ┌──────────┐  ┌──────────┐
  │diskboot  │  │grubx64   │
  │+core.img │  │  .efi    │
  │(MBR gap) │  │  (ESP)   │
  └────┬─────┘  └────┬─────┘
       │              │
       └──────┬───────┘
              ▼
  ┌─────────────────────────────┐
  │         GRUB2               │
  │  Read grub.cfg              │
  │  Show menu                  │
  │  Load vmlinuz + initramfs   │
  │  Pass kernel params         │
  └──────────────┬──────────────┘
                 ▼
  ┌─────────────────────────────┐
  │       KERNEL (vmlinuz)      │
  │  Decompress → start_kernel  │
  │  Init memory, scheduler     │
  │  PID 0 (idle)               │
  │  PID 1 (systemd/initramfs)  │
  │  PID 2 (kthreadd)           │
  └──────────────┬──────────────┘
                 ▼
  ┌─────────────────────────────┐
  │       INITRAMFS             │
  │  systemd Phase 1            │
  │  ┌───────────────────────┐  │
  │  │ sysinit.target        │  │
  │  │ initrd-root-device    │  │
  │  │ initrd-root-fs        │  │
  │  │   └─ mount / on       │  │
  │  │      /sysroot (RO)    │  │
  │  │ initrd.target         │  │
  │  │   └─ rd.break HERE    │  │
  │  │ switch_root           │  │
  │  │   └─ /sysroot → /     │  │
  │  └───────────────────────┘  │
  └──────────────┬──────────────┘
                 ▼
  ┌─────────────────────────────┐
  │   REAL OS (systemd Phase 2) │
  │  ┌───────────────────────┐  │
  │  │ sysinit.target        │  │
  │  │ basic.target          │  │
  │  │ multi-user.target     │  │
  │  │  └─ SSH, cron, etc.   │  │
  │  │ graphical.target      │  │
  │  │  └─ GDM (if desktop)  │  │
  │  └───────────────────────┘  │
  └──────────────┬──────────────┘
                 ▼
  ┌─────────────────────────────┐
  │     LOGIN SCREEN            │
  │  getty (text) or GDM (GUI)  │
  └─────────────────────────────┘
```

---

## One-Line Summary

```
Power → POST → MBR/ESP → boot.img/shim → GRUB2 → vmlinuz → initramfs → switch_root → systemd → login
```

---

## MBR Structure (512 Bytes)

```
┌──────────────────────────────────────┐
│  Bootstrap Code (boot.img)    446 B  │
│  Partition Table               64 B  │  ← 4 × 16 bytes
│  Boot Signature (0x55AA)        2 B  │
└──────────────────────────────────────┘
```

---

## systemd Runs Twice

```
                    switch_root
                        │
    IN INITRAMFS        │      ON REAL OS
    ────────────        │      ──────────
    sysinit.target      │      sysinit.target
    initrd-root-device  │      basic.target
    initrd-root-fs      │      multi-user.target
    initrd.target ──────┼────► graphical.target
                        │      (default.target)
```

---

## Key Intervention Points

| When | How | Use Case |
|------|-----|----------|
| GRUB2 menu | Press `e`, edit `linux` line | Add `rd.break`, change target |
| `rd.break` | Stops before switch_root | Root password reset |
| `rescue.target` | Add `systemd.unit=rescue.target` | Root shell, all FS mounted |
| `emergency.target` | Add `systemd.unit=emergency.target` | Root shell, root FS read-only |

# Stage 1: BIOS/UEFI & POST

## What Happens When You Press Power?

1. **PSU** stabilizes electricity, sends "Power Good" signal to motherboard
2. **CPU wakes up**, jumps to firmware code on a chip (BIOS at address `0xFFFF0`)
3. **POST (Power-On Self-Test)** runs -- checks CPU → RAM → Video → Keyboard → Storage
4. Firmware searches for a **bootable device** in the configured boot order

If POST fails on a critical component, you get **beep codes** (since the display may not work):

| Beep | Meaning |
|------|---------|
| 1 short | All OK |
| Continuous | RAM not detected |
| 1 long + 2 short | Video card failure |

---

## BIOS vs UEFI

| | BIOS | UEFI |
|---|---|---|
| **CPU mode** | 16-bit Real Mode | 32/64-bit |
| **Partition table** | MBR (max 2 TB, 4 partitions) | GPT (no practical limits) |
| **Bootloader location** | MBR (first 512 bytes of disk) | EFI System Partition (FAT32) |
| **Secure Boot** | No | Yes |
| **RHEL boot chain** | MBR → boot.img → core.img → GRUB2 | shimx64.efi → grubx64.efi → GRUB2 |

```bash
[ -d /sys/firmware/efi ] && echo "UEFI" || echo "BIOS"
```

---

## Finding the Bootable Device

**BIOS:** Reads first 512 bytes of each disk, checks last 2 bytes for `0x55AA` signature. Found → bootable → load and execute.

**UEFI:** Reads boot entries from NVRAM, finds EFI System Partition, loads `/boot/efi/EFI/redhat/shimx64.efi` (Secure Boot shim) → loads `grubx64.efi`.

```bash
efibootmgr -v    # View UEFI boot entries
```

**Next:** Firmware hands control to the bootloader.

# Stage 2: MBR, boot.img & diskboot.img

## The MBR -- First 512 Bytes of the Disk

```
┌──────────────────────────────────────┐
│  Bootstrap Code (boot.img)    446 B  │  ← Tiny program, loads next stage
│  Partition Table               64 B  │  ← 4 entries × 16 bytes
│  Boot Signature (0x55AA)        2 B  │  ← "This disk is bootable"
└──────────────────────────────────────┘
Total: 446 + 64 + 2 = 512 bytes
```

**Why max 4 primary partitions:** 64 bytes ÷ 16 bytes per entry = 4.

**Why max 2 TB:** Sector count field is 4 bytes (32-bit). 2^32 × 512 bytes = 2 TB.

---

## Each Partition Entry (16 Bytes)

| Offset | Size | Field |
|--------|------|-------|
| 0 | 1 B | Boot indicator (`0x80` = active, `0x00` = inactive) |
| 1-3 | 3 B | CHS start address (legacy) |
| 4 | 1 B | Partition type (`0x83` = Linux, `0x82` = swap, `0x8e` = LVM) |
| 5-7 | 3 B | CHS end address (legacy) |
| 8-11 | 4 B | LBA start address |
| 12-15 | 4 B | Number of sectors |

---

## The Problem: 446 Bytes Is Not Enough

GRUB2 needs hundreds of KB to show a menu and read filesystems. Solution: a **multi-stage chain**.

| Stage | Image | Where | What It Does |
|-------|-------|-------|-------------|
| **1** | `boot.img` | MBR (446 bytes) | Loads first sector of core.img |
| **1.5** | `diskboot.img` + `core.img` | **MBR gap** (sectors 1-2047, ~1 MB between MBR and first partition) | `diskboot.img` loads the rest of `core.img`. `core.img` contains filesystem drivers (XFS/ext4) |
| **2** | `/boot/grub2/` | `/boot` partition | Full GRUB2 with config, menu, modules |

**How they connect:**
- `boot.img` → loads `diskboot.img` (from hardcoded sector in MBR gap)
- `diskboot.img` → loads rest of `core.img`
- `core.img` → can now read XFS/ext4 → reads `/boot/grub2/grub.cfg` → full GRUB2

> **On UEFI:** No boot.img/diskboot.img/core.img. GRUB2 is a single `.efi` file on the ESP.

---

## Viewing the MBR

```bash
sudo fdisk -l /dev/sda                    # View partition table (dos = MBR, gpt = GPT)
sudo xxd -s 510 -l 2 /dev/sda             # Check boot signature (should show 55aa)
sudo dd if=/dev/sda of=/tmp/mbr.bak bs=512 count=1   # Backup MBR
```

## Installing GRUB2

```bash
# BIOS: writes boot.img to MBR, core.img to MBR gap
sudo grub2-install /dev/sda

# UEFI: copies grubx64.efi to ESP
sudo grub2-install --target=x86_64-efi --efi-directory=/boot/efi
```

**Next:** GRUB2 is loaded and takes over.

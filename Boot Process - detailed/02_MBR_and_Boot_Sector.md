# Stage 2: MBR, Boot Sector, boot.img & diskboot.img

## Index

1. [The Master Boot Record (MBR) -- The First 512 Bytes](#the-master-boot-record-mbr----the-first-512-bytes)
2. [MBR Structure -- Byte-by-Byte Breakdown](#mbr-structure----byte-by-byte-breakdown)
   - [Part 1: Bootstrap Code (446 Bytes)](#part-1-bootstrap-code-446-bytes)
   - [Part 2: Partition Table (64 Bytes)](#part-2-partition-table-64-bytes)
   - [Part 3: Boot Signature (2 Bytes)](#part-3-boot-signature-2-bytes)
3. [The Partition Table Entry -- 16 Bytes Dissected](#the-partition-table-entry----16-bytes-dissected)
4. [The Problem: 446 Bytes Is Not Enough](#the-problem-446-bytes-is-not-enough)
5. [GRUB2's Multi-Stage Boot Process](#grub2s-multi-stage-boot-process)
   - [Stage 1: boot.img (Inside the MBR)](#stage-1-bootimg-inside-the-mbr)
   - [Stage 1.5: diskboot.img + core.img (The MBR Gap)](#stage-15-diskbootimg--coreimg-the-mbr-gap)
   - [Stage 2: /boot/grub2/ (The Full GRUB2)](#stage-2-bootgrub2-the-full-grub2)
6. [The MBR Gap (Post-MBR Gap)](#the-mbr-gap-post-mbr-gap)
7. [GPT and UEFI: How Modern Systems Differ](#gpt-and-uefi-how-modern-systems-differ)
8. [Viewing the MBR on a Real RHEL System](#viewing-the-mbr-on-a-real-rhel-system)
9. [How GRUB2 Gets Installed on the Disk](#how-grub2-gets-installed-on-the-disk)
10. [What's Next?](#whats-next)

---

## The Master Boot Record (MBR) -- The First 512 Bytes

When the BIOS finishes POST, it reads exactly **512 bytes** from the very beginning of the bootable disk -- byte 0 through byte 511. This tiny piece of data is called the **Master Boot Record (MBR)**.

Think of the MBR as the **front door of a building**. It's small, but it serves two critical purposes:

1. **A tiny program** that knows how to find and load the next piece of the bootloader
2. **A map** of how the disk is divided into partitions

512 bytes is incredibly small -- less than half a kilobyte. That's about the size of a short tweet. You can't fit an operating system in there, or even a proper bootloader. But it's enough to start the chain reaction that eventually loads RHEL.

---

## MBR Structure -- Byte-by-Byte Breakdown

The 512 bytes of the MBR are divided into exactly three sections:

```
┌──────────────────────────────────────────────────────────┐
│                                                          │
│              BOOTSTRAP CODE (boot.img)                   │
│                                                          │
│                    446 bytes                              │
│                                                          │
│           (Tiny program -- finds and loads                │
│            the next stage of the bootloader)              │
│                                                          │
├──────────────────────────────────────────────────────────┤
│                                                          │
│                 PARTITION TABLE                           │
│                                                          │
│     Entry 1 (16 bytes)  │  Entry 2 (16 bytes)           │
│     Entry 3 (16 bytes)  │  Entry 4 (16 bytes)           │
│                                                          │
│                   64 bytes total                          │
│                                                          │
├──────────────────────────────────────────────────────────┤
│              BOOT SIGNATURE                              │
│              0x55  0xAA                                   │
│              2 bytes                                      │
└──────────────────────────────────────────────────────────┘

Total: 446 + 64 + 2 = 512 bytes
```

### Part 1: Bootstrap Code (446 Bytes)

| Field | Size | Description |
|-------|------|-------------|
| Bootstrap code | 446 bytes | Machine code (a tiny program) that the CPU executes |

This is where GRUB2's **`boot.img`** lives. When the BIOS reads these 446 bytes and jumps to the beginning of this code, it's executing `boot.img`. The code is written in assembly language and does exactly one thing: **find and load the next stage of GRUB2** from the sectors immediately following the MBR.

**Key point:** This code is so small that it cannot:
- Understand filesystems (ext4, XFS, etc.)
- Read configuration files
- Display a menu
- Do anything "smart"

All it does is load more code from a known location on disk.

### Part 2: Partition Table (64 Bytes)

| Field | Size | Description |
|-------|------|-------------|
| Partition entry 1 | 16 bytes | Describes the first partition |
| Partition entry 2 | 16 bytes | Describes the second partition |
| Partition entry 3 | 16 bytes | Describes the third partition |
| Partition entry 4 | 16 bytes | Describes the fourth partition |

This is why MBR disks can only have **4 primary partitions** -- there's only room for 4 entries of 16 bytes each. To work around this limit, one of the four entries can be an "extended partition" that contains "logical partitions" inside it.

### Part 3: Boot Signature (2 Bytes)

| Field | Size | Description |
|-------|------|-------------|
| Boot signature | 2 bytes | Must be `0x55AA` -- tells BIOS "this disk is bootable" |

The BIOS checks these last two bytes. If they are `0x55` followed by `0xAA`, the disk is considered bootable. If not, the BIOS skips this disk and tries the next one in the boot order.

> **Analogy:** The boot signature is like a "We're Open" sign on a shop door. If the sign isn't there, the customer (BIOS) walks past and tries the next shop (disk).

---

## The Partition Table Entry -- 16 Bytes Dissected

Each of the four partition entries in the MBR is exactly 16 bytes and contains:

| Byte Offset | Size | Field | Description |
|-------------|------|-------|-------------|
| 0 | 1 byte | **Boot indicator** | `0x80` = active/bootable, `0x00` = inactive |
| 1-3 | 3 bytes | **CHS address (start)** | Cylinder-Head-Sector address of first sector (legacy) |
| 4 | 1 byte | **Partition type** | Identifies the filesystem type |
| 5-7 | 3 bytes | **CHS address (end)** | Cylinder-Head-Sector address of last sector (legacy) |
| 8-11 | 4 bytes | **LBA address (start)** | Logical Block Address of the first sector |
| 12-15 | 4 bytes | **Number of sectors** | Total sectors in the partition |

### Common Partition Type IDs

| Type ID | Filesystem/Use |
|---------|---------------|
| `0x83` | Linux (ext2, ext3, ext4, XFS) |
| `0x82` | Linux swap |
| `0x8e` | Linux LVM |
| `0xfd` | Linux RAID autodetect |
| `0x05` | Extended partition (contains logical partitions) |
| `0x0c` | FAT32 (LBA) -- used for EFI System Partition |
| `0x07` | NTFS (Windows) |

### The 2 TB Limit Explained

The "Number of sectors" field is 4 bytes (32 bits). Each sector is 512 bytes.

```
Maximum addressable sectors = 2^32 = 4,294,967,296 sectors
Maximum disk size = 4,294,967,296 × 512 bytes = 2,199,023,255,552 bytes = 2 TB
```

This is why MBR cannot handle disks larger than 2 TB. GPT (used with UEFI) removes this limitation.

---

## The Problem: 446 Bytes Is Not Enough

Here's the fundamental challenge. To display a boot menu, read a configuration file, understand filesystems like XFS or ext4, and load a Linux kernel, GRUB2 needs **hundreds of kilobytes** of code. But the MBR only gives us **446 bytes**.

The solution? **A multi-stage boot process.** Each stage loads the next, slightly larger stage, like a chain:

```
446 bytes → ~32 KB → Full GRUB2 (hundreds of KB)
   │           │          │
   │           │          └── Stage 2: Reads config, shows menu, loads kernel
   │           └── Stage 1.5: Understands filesystems, finds /boot/grub2/
   └── Stage 1: Only knows how to load Stage 1.5 from raw disk sectors
```

---

## GRUB2's Multi-Stage Boot Process

### Stage 1: boot.img (Inside the MBR)

| Property | Value |
|----------|-------|
| **What is it?** | A 446-byte program installed in the MBR |
| **Where does it live?** | Bytes 0-445 of the disk (the MBR bootstrap area) |
| **Source file** | `/usr/lib/grub/i386-pc/boot.img` |
| **What can it do?** | Load exactly one sector (512 bytes) from a hardcoded disk location |
| **What can't it do?** | Understand filesystems, read config files, show menus |

**What `boot.img` does step by step:**

1. BIOS loads these 446 bytes into memory at address `0x7C00`
2. CPU starts executing the code
3. `boot.img` loads the **first sector of `core.img`** from a hardcoded location on disk (the sector number is written into `boot.img` when GRUB2 is installed)
4. That first sector of `core.img` is actually **`diskboot.img`**
5. Control passes to `diskboot.img`

### Stage 1.5: diskboot.img + core.img (The MBR Gap)

| Property | Value |
|----------|-------|
| **What is it?** | A larger program (~32 KB) containing filesystem drivers |
| **Where does it live?** | In the "MBR gap" -- sectors between the MBR and the first partition |
| **Source files** | `/usr/lib/grub/i386-pc/diskboot.img` + filesystem modules |
| **What can it do?** | Read filesystems (XFS, ext4), find `/boot/grub2/` |
| **What can't it do?** | Everything else GRUB2 does (menu, editing, etc.) |

**How `core.img` is built:**

When you install GRUB2, the `grub2-install` command assembles `core.img` by combining:

```
core.img = diskboot.img + lzma decompressor + filesystem modules + GRUB2 kernel
```

- **`diskboot.img`** -- the first sector of `core.img`. Its job is to load the rest of `core.img` from the subsequent sectors in the MBR gap
- **Filesystem modules** -- code that understands XFS, ext4, or whatever filesystem `/boot` uses. This is how GRUB2 can transition from "reading raw disk sectors" to "reading files from a filesystem"
- **GRUB2 kernel** -- the core logic of GRUB2 (not the Linux kernel! -- GRUB2 has its own "kernel")

**What `diskboot.img` does step by step:**

1. `boot.img` loads `diskboot.img` (the first 512 bytes of `core.img`) into memory
2. `diskboot.img` knows the locations of the remaining sectors of `core.img` (hardcoded during install)
3. `diskboot.img` loads the rest of `core.img` into memory
4. The filesystem modules inside `core.img` are now available
5. `core.img` can now read the `/boot` partition **as a filesystem** (not just raw sectors)
6. It reads `/boot/grub2/grub.cfg` and loads the full GRUB2 environment

### Stage 2: /boot/grub2/ (The Full GRUB2)

| Property | Value |
|----------|-------|
| **What is it?** | The complete GRUB2 bootloader with all its modules |
| **Where does it live?** | `/boot/grub2/` directory on the `/boot` partition |
| **Config file** | `/boot/grub2/grub.cfg` |
| **What can it do?** | Display menu, edit boot parameters, load kernel, load initramfs |

This is where the real GRUB2 lives. We'll cover this in detail in the next module.

---

## The MBR Gap (Post-MBR Gap)

The "MBR gap" is the space between the end of the MBR (byte 512) and the start of the first partition. This gap exists because the first partition traditionally starts at sector 2048 (1 MB into the disk) for alignment reasons.

```
Disk Layout:
┌─────────┬───────────────────────────────┬──────────────────┬──────────────────┐
│   MBR   │         MBR Gap               │   Partition 1    │  Partition 2 ... │
│ 512 B   │    ~1 MB (sectors 1-2047)     │   (/boot)        │   (/)            │
│         │                               │                  │                  │
│boot.img │  core.img lives here          │ /boot/grub2/     │  RHEL root       │
│         │  (diskboot.img + modules)     │ grub.cfg, etc.   │  filesystem      │
└─────────┴───────────────────────────────┴──────────────────┴──────────────────┘
 Sector 0   Sectors 1-2047                  Sector 2048+
```

**Size of the MBR gap:**

```
Sectors 1 through 2047 = 2047 sectors × 512 bytes/sector = ~1 MB
```

This ~1 MB is more than enough for `core.img` (which is typically 25-32 KB).

> **Historical note:** Older systems sometimes started the first partition at sector 63, leaving only ~31 KB for the MBR gap. This was barely enough for GRUB's core.img and could cause problems. Modern partitioning tools (like those used by RHEL's installer) always use sector 2048.

---

## GPT and UEFI: How Modern Systems Differ

On a UEFI system with GPT partitioning, the boot process is different:

| Aspect | BIOS + MBR | UEFI + GPT |
|--------|-----------|------------|
| **Bootloader location** | MBR (512 bytes) + MBR gap | EFI System Partition (ESP) |
| **GRUB2 format** | Raw binary (`boot.img` + `core.img`) | `.efi` executable file |
| **Bootloader file** | Embedded in MBR | `/boot/efi/EFI/redhat/grubx64.efi` |
| **No MBR gap needed** | `core.img` goes in the gap | Everything is on the ESP filesystem |
| **Partition table** | In the MBR (64 bytes) | In GPT headers (multiple copies) |
| **Protective MBR** | Not applicable | GPT disks include a "protective MBR" to prevent old tools from corrupting the disk |

**GPT Disk Layout:**

```
┌───────────────┬───────────┬──────────┬──────────────┬─────────────┬───────────┐
│ Protective    │ GPT       │ ESP      │ /boot        │ / (root)    │ GPT       │
│ MBR           │ Header    │ Partition│ Partition     │ Partition   │ Backup    │
│ (sector 0)    │ (sector 1)│ (FAT32)  │ (XFS/ext4)   │ (XFS)       │ Header    │
│               │           │ ~200 MB  │ ~1 GB        │             │ (last     │
│               │           │          │              │             │  sector)  │
└───────────────┴───────────┴──────────┴──────────────┴─────────────┴───────────┘
```

On UEFI systems:
- There is **no `boot.img`** in the MBR
- There is **no `diskboot.img`** or `core.img` in the MBR gap
- Instead, GRUB2 lives as a single `.efi` file on the ESP partition
- UEFI firmware can directly read FAT32 filesystems, so it loads `grubx64.efi` directly

---

## Viewing the MBR on a Real RHEL System

You can inspect the MBR of a disk using these commands:

```bash
# View the MBR partition table (safe, read-only)
sudo fdisk -l /dev/sda
```

```
Disk /dev/sda: 50 GiB, 53687091200 bytes, 104857600 sectors
Disklabel type: dos        ← "dos" means MBR, "gpt" means GPT
...
Device     Boot   Start       End  Sectors  Size Id Type
/dev/sda1  *       2048   2099199  2097152    1G 83 Linux    ← /boot (Boot flag set)
/dev/sda2       2099200 104857599 102758400   49G 8e Linux LVM ← LVM for / and swap
```

```bash
# Dump the raw MBR (first 512 bytes) to a file for inspection
sudo dd if=/dev/sda of=/tmp/mbr_backup.bin bs=512 count=1

# View the MBR in hexadecimal
sudo xxd /dev/sda | head -32
```

```bash
# Check the boot signature (last 2 bytes of the MBR)
sudo xxd -s 510 -l 2 /dev/sda
```

```
000001fe: 55aa
```

The `55aa` at offset `0x1FE` (byte 510-511) confirms this is a valid, bootable MBR.

```bash
# Check partition type (check if it's MBR or GPT)
sudo parted /dev/sda print
```

```bash
# On a UEFI system, check the EFI System Partition
ls -la /boot/efi/EFI/redhat/
```

```
grubx64.efi     ← GRUB2 EFI bootloader
shimx64.efi     ← Secure Boot shim loader
grub.cfg        ← Minimal config that points to /boot/grub2/grub.cfg
```

---

## How GRUB2 Gets Installed on the Disk

GRUB2 doesn't magically appear on the disk. It's installed by the `grub2-install` command (run automatically by the RHEL installer, or manually by an admin).

**On a BIOS/MBR system:**

```bash
# Install GRUB2 to the MBR of /dev/sda
sudo grub2-install /dev/sda
```

This command does three things:

1. Writes **`boot.img`** (446 bytes) into the MBR of `/dev/sda`
2. Builds **`core.img`** (from `diskboot.img` + filesystem modules) and writes it to the **MBR gap**
3. Hardcodes the sector numbers of `core.img` into `boot.img` so Stage 1 knows where to find Stage 1.5

**On a UEFI/GPT system:**

```bash
# Install GRUB2 to the ESP
sudo grub2-install --target=x86_64-efi --efi-directory=/boot/efi
```

This copies `grubx64.efi` to `/boot/efi/EFI/redhat/` and registers a boot entry in the UEFI NVRAM.

> **Critical warning:** Never run `grub2-install` on the wrong disk. If you install GRUB2 to the wrong MBR, you can make another operating system unbootable. Always double-check the target device.

---

## What's Next?

At this point, the boot process has:

1. Firmware (BIOS/UEFI) completed POST and found the bootable disk
2. BIOS loaded the MBR → `boot.img` → `diskboot.img` → `core.img`
3. OR UEFI loaded `shimx64.efi` → `grubx64.efi`
4. GRUB2's filesystem modules are loaded and it can now read `/boot/grub2/`

In the next module, we'll explore what GRUB2 does once it's fully loaded -- reading its configuration, displaying the boot menu, and loading the kernel and initramfs.

> **Remember:** The entire point of `boot.img`, `diskboot.img`, and `core.img` is to bridge the gap between "firmware can only load 512 bytes" and "GRUB2 needs hundreds of KB to do its job." Each stage loads the next, progressively larger stage.

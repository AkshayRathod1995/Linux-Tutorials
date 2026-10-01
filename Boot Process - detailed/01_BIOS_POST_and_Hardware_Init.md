# Stage 1: BIOS/UEFI, POST & Hardware Initialization

## Index

1. [What Happens When You Press the Power Button?](#what-happens-when-you-press-the-power-button)
2. [The Power Supply Unit (PSU)](#the-power-supply-unit-psu)
3. [System Firmware: BIOS vs UEFI](#system-firmware-bios-vs-uefi)
   - [BIOS (Basic Input/Output System)](#bios-basic-inputoutput-system)
   - [UEFI (Unified Extensible Firmware Interface)](#uefi-unified-extensible-firmware-interface)
   - [BIOS vs UEFI Comparison](#bios-vs-uefi-comparison)
4. [POST: Power-On Self-Test](#post-power-on-self-test)
   - [What POST Checks](#what-post-checks)
   - [POST Beep Codes](#post-beep-codes)
5. [Finding a Bootable Device](#finding-a-bootable-device)
   - [BIOS Boot Device Selection](#bios-boot-device-selection)
   - [UEFI Boot Device Selection](#uefi-boot-device-selection)
6. [How to Access BIOS/UEFI Settings](#how-to-access-biosuefi-settings)
7. [What's Next?](#whats-next)

---

## What Happens When You Press the Power Button?

Think of starting a computer like waking up in the morning. Before you can start your day (run applications), your body needs to:

1. **Wake up** (power reaches the CPU)
2. **Check that everything works** -- can you see? can you hear? can you move? (POST)
3. **Find your to-do list** -- what should I do first? (find the bootable device)
4. **Start your routine** (hand control to the bootloader)

The computer follows this exact pattern. The moment you press the power button, a precise sequence of events begins that eventually leads to your login screen.

---

## The Power Supply Unit (PSU)

Before any software runs, the hardware needs electricity. Here's what happens in the first fraction of a second:

1. You press the **power button**
2. The **PSU (Power Supply Unit)** starts converting AC power from the wall to the DC voltages the motherboard needs (3.3V, 5V, 12V)
3. The PSU sends a **"Power Good" signal** to the motherboard, telling it: "The electricity is stable, you can start safely"
4. The motherboard receives this signal and wakes up the **CPU**

> **Why does this matter?** If the PSU doesn't send the Power Good signal (because of a power surge or a faulty PSU), the computer won't even begin to boot. This is the very first potential failure point.

---

## System Firmware: BIOS vs UEFI

Once the CPU wakes up, it needs instructions. But the hard disk hasn't been read yet -- the operating system isn't loaded. So where do the first instructions come from?

They come from a small chip on the motherboard called the **firmware**. This firmware contains a tiny program that the CPU runs immediately after powering on. There are two types of firmware:

### BIOS (Basic Input/Output System)

BIOS is the **older** firmware standard, used since the 1980s. Think of it as the "original recipe."

- Stored on a **ROM/EEPROM chip** on the motherboard
- The CPU is hardcoded to jump to memory address **0xFFFF0** at startup -- this is where BIOS lives
- Operates in **16-bit Real Mode** (a legacy processor mode with limited memory access)
- Uses the **MBR (Master Boot Record)** partitioning scheme
- Can only access the **first 2 TB** of a disk (because MBR uses 32-bit addresses)
- Maximum **4 primary partitions** per disk
- Simple text-based setup screen (blue/gray background)
- No built-in network stack, no mouse support

### UEFI (Unified Extensible Firmware Interface)

UEFI is the **modern** replacement for BIOS, designed to overcome all of BIOS's limitations.

- Stored on a **flash memory chip** on the motherboard (or on a special partition called ESP)
- Operates in **32-bit or 64-bit mode** from the start (no legacy Real Mode limitation)
- Uses the **GPT (GUID Partition Table)** partitioning scheme
- Can access disks **larger than 2 TB** (theoretically up to 9.4 ZB)
- Supports **128+ partitions** per disk
- Has a graphical setup interface with mouse support
- Includes a **built-in network stack** for network booting
- Supports **Secure Boot** -- verifies that the bootloader hasn't been tampered with
- Reads from an **EFI System Partition (ESP)**, typically mounted at `/boot/efi`

> **Red Hat Enterprise Linux 8/9** supports both BIOS and UEFI. Most modern servers and desktops use UEFI.

### BIOS vs UEFI Comparison

| Feature | BIOS | UEFI |
|---------|------|------|
| **Age** | 1975 (IBM PC era) | 2005+ (Intel initiative) |
| **Processor mode** | 16-bit Real Mode | 32-bit or 64-bit |
| **Partition table** | MBR | GPT (can also read MBR) |
| **Max disk size** | 2 TB | 9.4 ZB (practically unlimited) |
| **Max partitions** | 4 primary (or 3 primary + 1 extended) | 128+ |
| **Bootloader location** | First 512 bytes of disk (MBR) | EFI System Partition (ESP) |
| **Secure Boot** | No | Yes |
| **Network boot** | Requires separate PXE ROM | Built-in network stack |
| **Interface** | Text-only, keyboard-only | Graphical, mouse support |
| **Boot speed** | Slower (sequential initialization) | Faster (parallel initialization) |
| **Bootloader format** | Raw binary in MBR | `.efi` executable files |
| **RHEL bootloader path** | MBR → GRUB2 | `/boot/efi/EFI/redhat/shimx64.efi` → GRUB2 |

---

## POST: Power-On Self-Test

The very first thing the firmware does is run **POST** -- a diagnostic check to make sure the essential hardware is working. Think of it as the computer's "morning health check."

### What POST Checks

POST runs these checks **in order**:

| Step | What It Checks | What Happens if It Fails |
|------|---------------|--------------------------|
| 1 | **CPU** -- Is the processor functional? | System doesn't start at all |
| 2 | **BIOS/UEFI ROM** -- Is the firmware intact? | Checksum failure, no boot |
| 3 | **RAM** -- Is memory present and accessible? | Beep codes, no display |
| 4 | **Basic hardware** -- Timer, DMA controller, interrupt controller | Beep codes |
| 5 | **Video/Display** -- Is a display adapter present? | Beep codes (you won't see anything) |
| 6 | **Keyboard** -- Is a keyboard connected? | Warning, may continue |
| 7 | **Storage controllers** -- Can it see hard disks? | Warning, may not boot |
| 8 | **Other peripherals** -- USB, serial, parallel ports | Warnings in POST screen |

> **Important for troubleshooting:** If POST fails on a critical component (CPU, RAM, video), the computer will produce **beep codes** -- a series of short and long beeps that tell a technician what went wrong. This is because if the display doesn't work, the only way to communicate is through sound.

### POST Beep Codes

Different BIOS vendors use different beep patterns. Here are common ones:

| Beep Pattern | Meaning |
|-------------|---------|
| 1 short beep | POST successful, everything OK |
| No beep | Power supply failure, motherboard failure, or speaker disconnected |
| Continuous beep | RAM not detected or not seated properly |
| 1 long, 2 short | Video card failure or not detected |
| 1 long, 3 short | Video card memory failure |
| 3 long beeps | Keyboard error |
| Repeating short beeps | Power problem |

**On servers (like Dell, HP, Lenovo):** Server-class machines typically use a front panel LCD or LED error codes instead of beeps. Check the vendor's documentation for specific codes.

---

## Finding a Bootable Device

After POST succeeds, the firmware's next job is to find a **bootable device** -- a disk (or network interface, or USB drive) that contains an operating system or bootloader.

### BIOS Boot Device Selection

In a BIOS system, the firmware searches for a bootable device in a specific order called the **boot order** (or boot priority). This is configured in the BIOS setup screen.

A typical boot order might be:

```
1. CD/DVD Drive
2. USB Drive
3. Hard Disk 0 (first hard disk)
4. Network (PXE Boot)
```

For each device in the list, the BIOS:

1. Reads the **first 512 bytes** of the device (the **MBR**)
2. Checks if the last two bytes are `0x55AA` (the **boot signature** or "magic number")
3. If the signature is found → this device is bootable → load and execute the MBR code
4. If not → move to the next device in the list

> If no bootable device is found, you see the dreaded **"No bootable device found"** or **"Operating system not found"** message.

### UEFI Boot Device Selection

UEFI takes a completely different approach:

1. It looks for an **EFI System Partition (ESP)** -- a small FAT32 partition (usually 200-600 MB)
2. Inside the ESP, it looks for `.efi` bootloader files at well-known paths
3. On RHEL, the path is: `/boot/efi/EFI/redhat/shimx64.efi`
4. UEFI stores its boot entries in **NVRAM** (non-volatile RAM on the motherboard) -- you can see and edit these with the `efibootmgr` command

```bash
# View UEFI boot entries (run this on a UEFI-booted RHEL system)
efibootmgr -v
```

```
BootCurrent: 0003
BootOrder: 0003,0001,0002
Boot0001* Network Boot   PciRoot(0x0)/Pci(0x1,0x0)/...
Boot0002* USB Boot       PciRoot(0x0)/Pci(0x14,0x0)/...
Boot0003* Red Hat        HD(1,GPT,...)/File(\EFI\redhat\shimx64.efi)
```

| Part | Meaning |
|------|---------|
| `BootCurrent: 0003` | The system booted from entry 0003 this time |
| `BootOrder: 0003,0001,0002` | Try Red Hat first, then Network, then USB |
| `Boot0003* Red Hat` | The asterisk `*` means this entry is active |
| `shimx64.efi` | The UEFI first-stage bootloader (includes Secure Boot support) |

**What is `shimx64.efi`?**

On RHEL with Secure Boot enabled, the firmware doesn't load GRUB2 directly. Instead, it loads `shim` first. `shim` is a small bootloader signed by Microsoft's UEFI signing key (which all hardware vendors trust). `shim` then verifies and loads GRUB2. This chain of trust is:

```
UEFI Firmware → shimx64.efi → grubx64.efi → kernel (vmlinuz)
```

---

## How to Access BIOS/UEFI Settings

| Hardware | Key to Press During Boot |
|----------|------------------------|
| Most PCs | **F2** or **Del** |
| HP | **F10** |
| Dell | **F2** or **F12** (boot menu) |
| Lenovo/ThinkPad | **F1** or **Enter** then **F1** |
| Virtual Machines (KVM/libvirt) | **Esc** or configured in VM settings |
| VMware | **F2** |

The key must be pressed **immediately** after power-on, before the OS starts loading. On fast-booting UEFI systems, you may need to press the key repeatedly.

**On RHEL, you can also access UEFI settings from the OS:**

```bash
# Reboot directly into UEFI firmware setup (if supported)
systemctl reboot --firmware-setup
```

---

## What's Next?

At this point, the firmware has:

1. Verified all hardware is working (POST)
2. Found a bootable device (hard disk with RHEL installed)
3. Read the first 512 bytes (BIOS) or loaded the EFI bootloader (UEFI)

In the next module, we'll dive into those **first 512 bytes** -- the **Master Boot Record** -- and understand how `boot.img` and `diskboot.img` bridge the gap between the firmware and GRUB2.

> **Remember:** The firmware's only job is to initialize hardware and hand off control to the bootloader. It doesn't know what Linux is, what RHEL is, or what a kernel is. It just finds the first piece of bootloader code and says "your turn."

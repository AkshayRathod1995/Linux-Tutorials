# Stage 3: GRUB2 -- The Grand Unified Bootloader

## Index

1. [What Is GRUB2?](#what-is-grub2)
2. [What GRUB2 Does](#what-grub2-does)
3. [GRUB2 Configuration Files](#grub2-configuration-files)
   - [/boot/grub2/grub.cfg -- The Main Config](#bootgrub2grubcfg----the-main-config)
   - [/etc/default/grub -- Your Settings](#etcdefaultgrub----your-settings)
   - [/etc/grub.d/ -- Menu Entry Scripts](#etcgrubd----menu-entry-scripts)
4. [Understanding /etc/default/grub](#understanding-etcdefaultgrub)
5. [Regenerating grub.cfg](#regenerating-grubcfg)
6. [The GRUB2 Boot Menu](#the-grub2-boot-menu)
7. [What Happens When You Select a Kernel](#what-happens-when-you-select-a-kernel)
8. [The Kernel Command Line](#the-kernel-command-line)
9. [Editing Boot Parameters at Boot Time](#editing-boot-parameters-at-boot-time)
10. [Managing Kernels and Default Boot Entry](#managing-kernels-and-default-boot-entry)
11. [GRUB2 Command Line (Emergency Shell)](#grub2-command-line-emergency-shell)
12. [BIOS vs UEFI GRUB2 File Locations](#bios-vs-uefi-grub2-file-locations)
13. [What's Next?](#whats-next)

---

## What Is GRUB2?

**GRUB2** stands for **GRand Unified Bootloader, version 2**. It's the default bootloader on RHEL 7, 8, and 9 (and nearly all modern Linux distributions).

Think of GRUB2 as a **receptionist at a hotel**. When a guest (the CPU) arrives, the receptionist:

1. **Shows the menu** -- "Which room (kernel) would you like?"
2. **Takes special requests** -- "Any preferences?" (kernel command line parameters)
3. **Escorts the guest** -- Loads the kernel and initramfs into memory
4. **Hands over the keys** -- Passes control to the kernel

GRUB2 is the last piece of software that runs before the Linux kernel takes over. Once GRUB2 hands off to the kernel, GRUB2 is gone from memory entirely.

---

## What GRUB2 Does

| Task | Description |
|------|-------------|
| **Displays the boot menu** | Shows a list of available kernels to choose from |
| **Loads the kernel** | Reads `vmlinuz` from `/boot` and places it in memory |
| **Loads the initramfs** | Reads `initramfs-*.img` from `/boot` and places it in memory |
| **Passes kernel parameters** | Sends options like `root=`, `rd.lvm.lv=`, `rhgb`, `quiet` to the kernel |
| **Provides a command line** | If the config is broken, you can type commands manually |
| **Supports multiple OSes** | Can chainload Windows or boot other Linux installations |
| **Understands filesystems** | Can read XFS, ext4, FAT32, and more -- unlike `boot.img` which can't |

---

## GRUB2 Configuration Files

GRUB2's configuration is split across multiple files. Here's how they work together:

```
┌─────────────────────┐     ┌────────────────────┐
│  /etc/default/grub  │     │  /etc/grub.d/       │
│                     │     │                    │
│  Your settings:     │     │  Scripts that       │
│  - Timeout          │     │  generate menu      │
│  - Default kernel   │     │  entries:           │
│  - Kernel params    │     │  00_header          │
│                     │     │  10_linux           │
│                     │     │  30_os-prober       │
│                     │     │  40_custom          │
└──────────┬──────────┘     └─────────┬──────────┘
           │                          │
           └──────────┬───────────────┘
                      │
                      ▼
              grub2-mkconfig
                      │
                      ▼
        ┌─────────────────────────┐
        │  /boot/grub2/grub.cfg   │
        │                         │
        │  The ACTUAL config      │
        │  that GRUB2 reads       │
        │  at boot time           │
        │                         │
        │  DO NOT EDIT THIS       │
        │  FILE DIRECTLY!         │
        └─────────────────────────┘
```

### /boot/grub2/grub.cfg -- The Main Config

This is the file GRUB2 actually reads at boot time. It's **auto-generated** -- you should **never edit it directly**.

```bash
# View the file (read-only!)
cat /boot/grub2/grub.cfg
```

### /etc/default/grub -- Your Settings

This is where you configure GRUB2's behavior. Changes here take effect after you regenerate `grub.cfg`.

### /etc/grub.d/ -- Menu Entry Scripts

This directory contains numbered scripts that generate parts of `grub.cfg`:

| Script | Purpose |
|--------|---------|
| `00_header` | Sets up the basic GRUB2 environment (timeout, default entry) |
| `01_users` | Configures GRUB2 password protection (if enabled) |
| `10_linux` | Generates boot entries for all installed Linux kernels |
| `30_os-prober` | Detects and adds entries for other operating systems (Windows, etc.) |
| `40_custom` | Your own custom menu entries (this is the right place to add them) |
| `41_custom` | Sources additional custom config from another file |

**The scripts run in numerical order** -- that's why they have number prefixes. Lower numbers run first and appear first in the generated `grub.cfg`.

---

## Understanding /etc/default/grub

```bash
cat /etc/default/grub
```

A typical RHEL `/etc/default/grub` file looks like:

```
GRUB_TIMEOUT=5
GRUB_DISTRIBUTOR="$(sed 's, release .*$,,g' /etc/system-release)"
GRUB_DEFAULT=saved
GRUB_DISABLE_SUBMENU=true
GRUB_TERMINAL_OUTPUT="console"
GRUB_CMDLINE_LINUX="crashkernel=auto resume=/dev/mapper/rhel-swap rd.lvm.lv=rhel/root rd.lvm.lv=rhel/swap rhgb quiet"
GRUB_DISABLE_RECOVERY="true"
```

Here's what each line means:

| Parameter | Value | What It Does |
|-----------|-------|-------------|
| `GRUB_TIMEOUT` | `5` | Shows the boot menu for 5 seconds before auto-booting the default. Set to `0` to skip the menu entirely. Set to `-1` to wait forever. |
| `GRUB_DISTRIBUTOR` | `"Red Hat Enterprise Linux"` | The label shown in the boot menu (e.g., "Red Hat Enterprise Linux (5.14.0-70.el9.x86_64) 9.0") |
| `GRUB_DEFAULT` | `saved` | Which kernel to boot by default. `saved` means "whatever was last booted" (set with `grub2-set-default`). Can also be a number (`0` = first entry) or a menu entry title. |
| `GRUB_DISABLE_SUBMENU` | `true` | Shows all kernels in a flat list (instead of nesting older kernels in a submenu) |
| `GRUB_TERMINAL_OUTPUT` | `"console"` | Where GRUB2 displays output. `console` = text mode. `gfxterm` = graphical mode. |
| `GRUB_CMDLINE_LINUX` | `"crashkernel=auto ..."` | Kernel command-line parameters added to **every** kernel entry. This is how you pass options to the kernel. |
| `GRUB_DISABLE_RECOVERY` | `"true"` | Don't generate recovery/rescue mode entries in the menu |

### Common Kernel Command Line Parameters

These go inside `GRUB_CMDLINE_LINUX`:

| Parameter | Meaning |
|-----------|---------|
| `root=/dev/mapper/rhel-root` | Which partition/device is the root filesystem |
| `rd.lvm.lv=rhel/root` | Tells initramfs to activate this LVM logical volume |
| `rd.lvm.lv=rhel/swap` | Tells initramfs to activate this swap LV |
| `resume=/dev/mapper/rhel-swap` | Resume from hibernation using this swap device |
| `crashkernel=auto` | Reserve memory for kdump (crash dump) |
| `rhgb` | **R**ed **H**at **G**raphical **B**oot -- shows a graphical boot splash |
| `quiet` | Suppress most kernel boot messages (show only errors) |
| `console=ttyS0,115200` | Send output to serial console (common on servers) |
| `rd.break` | Break into a shell before initramfs hands off to the real system |
| `systemd.unit=rescue.target` | Boot into rescue mode |
| `systemd.unit=emergency.target` | Boot into emergency mode |
| `init=/bin/bash` | Skip systemd entirely and drop to a bash shell (dangerous but useful) |
| `enforcing=0` | Boot with SELinux in permissive mode |
| `selinux=0` | Disable SELinux entirely (not recommended) |

---

## Regenerating grub.cfg

After editing `/etc/default/grub`, you **must** regenerate `grub.cfg`:

**On BIOS systems:**

```bash
sudo grub2-mkconfig -o /boot/grub2/grub.cfg
```

**On UEFI systems:**

```bash
sudo grub2-mkconfig -o /boot/efi/EFI/redhat/grub.cfg
```

**On RHEL 9 (simplified):**

```bash
# RHEL 9 uses BLS (Boot Loader Specification) -- grub.cfg is simpler
# Kernel entries are in /boot/loader/entries/*.conf
sudo grub2-mkconfig -o /boot/grub2/grub.cfg
```

> **Critical:** If you edit `/etc/default/grub` but forget to run `grub2-mkconfig`, your changes will **not** take effect on the next boot. The running config is always `/boot/grub2/grub.cfg`.

---

## The GRUB2 Boot Menu

When the system boots, GRUB2 shows a menu like this:

```
  Red Hat Enterprise Linux

     Red Hat Enterprise Linux (5.14.0-284.el9.x86_64) 9.2
     Red Hat Enterprise Linux (5.14.0-70.el9.x86_64) 9.0

     Use the ↑ and ↓ keys to change the selection.
     Press 'e' to edit the selected item, or 'c' for a command prompt.
     The selected entry will be started automatically in 5s.
```

| Action | Key |
|--------|-----|
| Move up/down the menu | Arrow keys ↑ ↓ |
| Boot the selected entry | **Enter** |
| Edit the selected entry (temporarily) | **e** |
| Enter GRUB2 command line | **c** |
| Stop the countdown timer | Any key (except Enter) |

---

## What Happens When You Select a Kernel

When you select a kernel entry and press Enter (or the timeout expires), GRUB2 executes commands similar to these (from `grub.cfg`):

```
menuentry 'Red Hat Enterprise Linux (5.14.0-284.el9.x86_64) 9.2' ... {
    load_video
    set gfxpayload=keep
    insmod gzio                              ← Load gzip decompression module
    insmod part_msdos                        ← Load MBR partition support
    insmod xfs                               ← Load XFS filesystem driver
    set root='hd0,msdos1'                    ← The /boot partition is on first disk, first partition
    linux /vmlinuz-5.14.0-284.el9.x86_64     ← Load the kernel into memory
        root=/dev/mapper/rhel-root           ← Tell kernel where root fs is
        rd.lvm.lv=rhel/root                  ← Activate LVM volume
        rd.lvm.lv=rhel/swap                  ← Activate swap
        crashkernel=auto                     ← Reserve crash dump memory
        rhgb quiet                           ← Graphical boot, suppress messages
    initrd /initramfs-5.14.0-284.el9.x86_64.img  ← Load initramfs into memory
}
```

**Step by step, GRUB2:**

1. **Loads filesystem modules** (`insmod xfs`) so it can read the `/boot` partition
2. **Sets the root** to the partition containing `/boot`
3. **Loads `vmlinuz`** (the compressed kernel) from `/boot` into RAM
4. **Loads `initramfs`** (the initial RAM filesystem) from `/boot` into RAM
5. **Passes the kernel command line** (everything after the `linux` line)
6. **Transfers control to the kernel** -- GRUB2's job is done

> **Key insight:** GRUB2 loads two things into memory -- the **kernel** (`vmlinuz`) and the **initramfs** (`initramfs-*.img`). Both must be loaded before control is handed to the kernel. The kernel needs the initramfs to find and mount the real root filesystem.

---

## The Kernel Command Line

The kernel command line is the set of parameters passed from GRUB2 to the kernel. After the system boots, you can see what was passed:

```bash
# View the kernel command line that was used to boot
cat /proc/cmdline
```

```
BOOT_IMAGE=(hd0,msdos1)/vmlinuz-5.14.0-284.el9.x86_64 root=/dev/mapper/rhel-root ro crashkernel=auto resume=/dev/mapper/rhel-swap rd.lvm.lv=rhel/root rd.lvm.lv=rhel/swap rhgb quiet
```

| Part | Meaning |
|------|---------|
| `BOOT_IMAGE=(hd0,msdos1)/vmlinuz-...` | Which kernel was loaded, from which partition |
| `root=/dev/mapper/rhel-root` | The root filesystem device |
| `ro` | Mount root filesystem read-only initially (remounted read-write later by systemd) |
| `crashkernel=auto` | Memory reserved for kernel crash dump |
| `resume=/dev/mapper/rhel-swap` | Device to resume from hibernation |
| `rd.lvm.lv=rhel/root` | LVM logical volume to activate in initramfs |
| `rhgb` | Red Hat Graphical Boot splash |
| `quiet` | Suppress non-critical boot messages |

---

## Editing Boot Parameters at Boot Time

You can temporarily modify kernel parameters at boot time without changing any files on disk. This is essential for troubleshooting.

**Steps:**

1. Reboot the system
2. When the GRUB2 menu appears, press any key to stop the countdown
3. Use arrow keys to select the kernel entry
4. Press **`e`** to edit
5. Find the line starting with `linux` (or `linux16` on older RHEL)
6. Make your changes:
   - To boot into rescue mode: add `systemd.unit=rescue.target` at the end
   - To boot into emergency mode: add `systemd.unit=emergency.target`
   - To break before systemd: add `rd.break`
   - To see all boot messages: remove `rhgb quiet`
7. Press **Ctrl+x** to boot with the modified parameters

> **These changes are temporary** -- they only affect this one boot. On the next reboot, the original `grub.cfg` parameters are used. To make permanent changes, edit `/etc/default/grub` and run `grub2-mkconfig`.

---

## Managing Kernels and Default Boot Entry

```bash
# List all installed kernels
sudo grubby --info=ALL
```

```bash
# See which kernel is the default
sudo grub2-editenv list
```

```
saved_entry=4b86f0e4c90a4fd9a26d49a1a1060a2b-5.14.0-284.el9.x86_64
```

```bash
# Set a specific kernel as the default (by index, 0 = first)
sudo grub2-set-default 0

# Set a specific kernel by title
sudo grub2-set-default "Red Hat Enterprise Linux (5.14.0-284.el9.x86_64) 9.2"

# Boot a specific kernel ONCE (next boot only, then revert to default)
sudo grub2-reboot 1
```

```bash
# On RHEL 8/9 with BLS (Boot Loader Specification), kernels are in:
ls /boot/loader/entries/
```

```
4b86f0e4c90a4fd9a26d49a1a1060a2b-5.14.0-284.el9.x86_64.conf
4b86f0e4c90a4fd9a26d49a1a1060a2b-5.14.0-70.el9.x86_64.conf
```

```bash
# View a BLS entry
cat /boot/loader/entries/4b86f0e4c90a4fd9a26d49a1a1060a2b-5.14.0-284.el9.x86_64.conf
```

```
title Red Hat Enterprise Linux (5.14.0-284.el9.x86_64) 9.2
version 5.14.0-284.el9.x86_64
linux /vmlinuz-5.14.0-284.el9.x86_64
initrd /initramfs-5.14.0-284.el9.x86_64.img
options root=/dev/mapper/rhel-root ro crashkernel=auto resume=/dev/mapper/rhel-swap rd.lvm.lv=rhel/root rd.lvm.lv=rhel/swap rhgb quiet
grub_users $grub_variable
grub_arg --unrestricted
grub_class rhel
```

---

## GRUB2 Command Line (Emergency Shell)

If GRUB2 can't find its config file (corrupted disk, deleted `/boot`), it drops you to a command line:

```
grub>
```

Or if GRUB2 can't even find its modules:

```
grub rescue>
```

### Useful GRUB2 Command Line Commands

| Command | What It Does |
|---------|-------------|
| `ls` | List all disks and partitions GRUB2 can see |
| `ls (hd0,msdos1)/` | List files on a partition |
| `set root=(hd0,msdos1)` | Set the root partition |
| `linux /vmlinuz-... root=/dev/mapper/rhel-root` | Load the kernel |
| `initrd /initramfs-....img` | Load the initramfs |
| `boot` | Boot with the loaded kernel and initramfs |
| `set` | Show all GRUB2 environment variables |
| `set pager=1` | Enable paging for long output |
| `cat (hd0,msdos1)/grub2/grub.cfg` | View the config file |

### Manual Boot from GRUB2 Command Line

If `grub.cfg` is missing or corrupted, you can boot manually:

```
grub> set root=(hd0,msdos1)
grub> linux /vmlinuz-5.14.0-284.el9.x86_64 root=/dev/mapper/rhel-root ro
grub> initrd /initramfs-5.14.0-284.el9.x86_64.img
grub> boot
```

Once booted, fix the problem by regenerating `grub.cfg`:

```bash
sudo grub2-mkconfig -o /boot/grub2/grub.cfg
```

---

## BIOS vs UEFI GRUB2 File Locations

| File/Directory | BIOS System | UEFI System |
|----------------|-------------|-------------|
| **GRUB2 modules** | `/boot/grub2/i386-pc/` | `/boot/grub2/x86_64-efi/` |
| **Main config** | `/boot/grub2/grub.cfg` | `/boot/efi/EFI/redhat/grub.cfg` (minimal) + `/boot/grub2/grub.cfg` |
| **GRUB2 environment** | `/boot/grub2/grubenv` | `/boot/grub2/grubenv` |
| **EFI bootloader** | N/A | `/boot/efi/EFI/redhat/grubx64.efi` |
| **Secure Boot shim** | N/A | `/boot/efi/EFI/redhat/shimx64.efi` |
| **BLS entries (RHEL 8/9)** | `/boot/loader/entries/*.conf` | `/boot/loader/entries/*.conf` |
| **Install command** | `grub2-install /dev/sda` | `grub2-install --target=x86_64-efi` |
| **Config regeneration** | `grub2-mkconfig -o /boot/grub2/grub.cfg` | `grub2-mkconfig -o /boot/efi/EFI/redhat/grub.cfg` |

```bash
# Check if you're booted in BIOS or UEFI mode
[ -d /sys/firmware/efi ] && echo "UEFI" || echo "BIOS"
```

---

## What's Next?

GRUB2 has loaded two files into memory:

1. **`vmlinuz`** -- the compressed Linux kernel
2. **`initramfs-*.img`** -- the initial RAM filesystem

It has passed the kernel command line and handed control to the kernel. GRUB2 is now completely out of the picture.

In the next module, we'll explore what `vmlinuz` actually is, how the kernel gets decompressed and loaded into memory, and how RAM is prepared before the kernel begins executing.

> **Remember:** GRUB2's job is to be a "middle man" -- it bridges the gap between the firmware (which knows nothing about Linux) and the kernel (which needs to be loaded into the right place in memory with the right parameters).

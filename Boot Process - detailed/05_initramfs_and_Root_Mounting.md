# Stage 5: initramfs, Mounting the Real Root & Pivot

## Index

1. [What Is initramfs?](#what-is-initramfs)
2. [Why Does Linux Need initramfs?](#why-does-linux-need-initramfs)
3. [What's Inside initramfs?](#whats-inside-initramfs)
4. [How initramfs Gets Into Memory](#how-initramfs-gets-into-memory)
5. [initramfs as a Complete Mini Operating System](#initramfs-as-a-complete-mini-operating-system)
6. [Step by Step: What Happens Inside initramfs](#step-by-step-what-happens-inside-initramfs)
   - [Phase 1: Kernel Unpacks initramfs](#phase-1-kernel-unpacks-initramfs)
   - [Phase 2: systemd Starts Inside initramfs](#phase-2-systemd-starts-inside-initramfs)
   - [Phase 3: Finding and Mounting the Real Root (/sysroot)](#phase-3-finding-and-mounting-the-real-root-sysroot)
   - [Phase 4: Filesystem Checks](#phase-4-filesystem-checks)
   - [Phase 5: Pivot Root (switch_root)](#phase-5-pivot-root-switch_root)
7. [Understanding /sysroot](#understanding-sysroot)
8. [The rd.break Trick -- Pausing Before Pivot](#the-rdbreak-trick----pausing-before-pivot)
9. [Inspecting initramfs on a Running System](#inspecting-initramfs-on-a-running-system)
10. [Building and Rebuilding initramfs with dracut](#building-and-rebuilding-initramfs-with-dracut)
11. [Common initramfs Problems](#common-initramfs-problems)
12. [What's Next?](#whats-next)

---

## What Is initramfs?

**initramfs** stands for **Initial RAM File System**. It's a temporary, small filesystem that lives entirely in RAM. The kernel uses it as a stepping stone to get to the real root filesystem on disk.

Think of it like this:

> Imagine you arrive at a new city (the computer powers on). You don't know where your hotel (root filesystem) is yet. You need a taxi (initramfs) that has a GPS (drivers), knows the city (filesystem modules), and can take you to the right address (mount the root filesystem). Once you arrive at the hotel, you don't need the taxi anymore.

The initramfs file lives in `/boot/`:

```bash
ls -lh /boot/initramfs-$(uname -r).img
```

```
-rw-------. 1 root root 34M Sep 15 10:30 /boot/initramfs-5.14.0-284.el9.x86_64.img
```

---

## Why Does Linux Need initramfs?

The kernel needs to mount the root filesystem to start the real operating system. But there's a chicken-and-egg problem:

```
THE PROBLEM:

  To mount the root filesystem, the kernel needs drivers for:
  ├── Storage controller (SCSI, NVMe, virtio, etc.)
  ├── Filesystem type (XFS, ext4, Btrfs)
  ├── LVM (if root is on a logical volume)
  ├── LUKS/dm-crypt (if root is encrypted)
  ├── Software RAID (mdadm)
  └── Network (if root is on NFS or iSCSI)

  But these drivers are stored AS FILES on the root filesystem.
  The kernel can't read files without mounting the filesystem first.

  Root FS needs drivers → Drivers are on Root FS → ???
```

**The solution is initramfs:**

```
THE SOLUTION:

  GRUB2 loads initramfs into RAM alongside the kernel.
  initramfs CONTAINS all the drivers and tools needed to mount the root FS.

  1. Kernel boots with initramfs in RAM
  2. Kernel unpacks initramfs into a temporary root filesystem (tmpfs)
  3. initramfs loads the necessary drivers
  4. initramfs finds and mounts the REAL root filesystem
  5. System pivots from initramfs to the real root
  6. initramfs is discarded from memory
```

---

## What's Inside initramfs?

The initramfs is a **compressed cpio archive** (not a disk image). On RHEL 8/9, it contains a complete, minimal Linux system:

```bash
# List the contents of initramfs (without extracting)
lsinitrd /boot/initramfs-$(uname -r).img | head -50

# Count the files inside
lsinitrd /boot/initramfs-$(uname -r).img | wc -l
```

Here's what you'll find inside:

| Directory/File | Purpose |
|---------------|---------|
| `/usr/lib/systemd/systemd` | systemd binary -- the init system inside initramfs |
| `/sbin/init` → systemd | PID 1 inside initramfs |
| `/usr/lib/modules/` | Kernel modules (drivers) for storage, filesystem, network, etc. |
| `/usr/lib/dracut/` | Dracut scripts that orchestrate the boot process |
| `/etc/` | Minimal configuration files |
| `/usr/bin/`, `/usr/sbin/` | Essential utilities (mount, modprobe, lvm, fsck, etc.) |
| `/usr/lib/systemd/system/` | systemd unit files for the initramfs environment |
| `/etc/cmdline.d/` | Default kernel command line options |
| `/usr/lib/udev/` | udev rules for device detection |

### Key Kernel Modules Included

| Module Type | Examples | Why It's Needed |
|-------------|---------|-----------------|
| **Storage drivers** | `ahci.ko`, `nvme.ko`, `virtio_blk.ko`, `megaraid_sas.ko` | To access the hard disk/SSD |
| **Filesystem drivers** | `xfs.ko`, `ext4.ko` | To read the root filesystem |
| **LVM** | `dm-mod.ko`, `dm-log.ko`, `dm-mirror.ko` | If root is on an LVM logical volume |
| **LUKS/encryption** | `dm-crypt.ko`, crypto modules | If root is encrypted |
| **RAID** | `md-mod.ko`, `raid1.ko`, `raid456.ko` | If root is on software RAID |
| **Network** | `e1000e.ko`, `bnx2.ko`, NFS modules | If root is on NFS/iSCSI |

---

## How initramfs Gets Into Memory

```
1. GRUB2 reads /boot/initramfs-5.14.0-284.el9.x86_64.img from disk
2. GRUB2 places the compressed image into a high memory region
3. GRUB2 tells the kernel where in memory the initramfs is (via boot parameters)
4. Kernel unpacks the cpio archive into a tmpfs filesystem mounted at /
```

The kernel's boot parameter structure includes:

```
initrd_start = 0x37F00000    ← Start address of initramfs in RAM
initrd_size  = 35651584      ← Size in bytes (~34 MB)
```

---

## initramfs as a Complete Mini Operating System

On RHEL 8/9, the initramfs is not just a collection of drivers -- it's a **complete, bootable mini OS** with its own:

- **systemd** (as PID 1)
- **udev** (device manager)
- **Basic shell** (/bin/sh)
- **Essential utilities** (mount, lvm, fsck, modprobe, ip, etc.)
- **Dracut** scripts (the boot orchestration framework)

This is a significant change from older Linux systems where initramfs used simple shell scripts. RHEL 8/9 uses systemd inside initramfs, which means the boot process inside initramfs follows the same target/unit model as the real system.

---

## Step by Step: What Happens Inside initramfs

### Phase 1: Kernel Unpacks initramfs

1. The kernel sees the initramfs in memory (GRUB2 told it where)
2. It creates a **tmpfs** filesystem in RAM
3. It unpacks the **cpio archive** from the initramfs image into this tmpfs
4. The tmpfs becomes the initial root filesystem (`/`)
5. The kernel executes `/sbin/init` from the initramfs -- which is **systemd**

```
Memory Before:                    Memory After:
┌───────────────────┐             ┌───────────────────┐
│ Kernel (running)  │             │ Kernel (running)  │
├───────────────────┤             ├───────────────────┤
│ Compressed        │   unpack    │ tmpfs (/)          │
│ initramfs blob    │  ────────►  │ ├── /sbin/init     │
│ (34 MB)           │             │ ├── /usr/lib/...   │
│                   │             │ ├── /etc/...       │
│                   │             │ └── /sysroot/ (empty, mountpoint) │
└───────────────────┘             └───────────────────┘
```

### Phase 2: systemd Starts Inside initramfs

systemd inside the initramfs uses a special target: **`initrd.target`**

The boot process inside initramfs follows these targets:

```
systemd (PID 1 in initramfs)
    │
    ├── sysinit.target
    │       ├── Load kernel modules (modprobe)
    │       ├── Start udev (detect hardware, create /dev nodes)
    │       ├── Set up /dev, /proc, /sys
    │       └── Process kernel command line parameters
    │
    ├── initrd-root-device.target
    │       ├── Activate LVM volumes (if needed)
    │       ├── Assemble RAID arrays (if needed)
    │       ├── Unlock LUKS encryption (if needed)
    │       └── Wait for root device to appear
    │
    ├── initrd-root-fs.target
    │       ├── Run filesystem check (fsck) on root device
    │       └── Mount root filesystem on /sysroot
    │
    ├── initrd-fs.target
    │       └── Mount other early filesystems (like /boot on /sysroot/boot)
    │
    └── initrd.target
            └── Everything is ready → trigger switch-root
```

### Phase 3: Finding and Mounting the Real Root (/sysroot)

The kernel command line tells initramfs where the real root filesystem is:

```
root=/dev/mapper/rhel-root
```

initramfs must:

1. **Load the storage driver** -- so it can talk to the hard disk
2. **Activate LVM** -- because `rhel-root` is an LVM logical volume

   ```
   LVM activation:
   
   Physical Disk (/dev/sda)
       └── Physical Volume (/dev/sda2)
           └── Volume Group (rhel)
               ├── Logical Volume (rhel-root) → /dev/mapper/rhel-root
               └── Logical Volume (rhel-swap) → /dev/mapper/rhel-swap
   ```

3. **Mount the root filesystem** -- mount `/dev/mapper/rhel-root` on `/sysroot`

   ```bash
   # Inside initramfs, this effectively happens:
   mount -t xfs /dev/mapper/rhel-root /sysroot
   ```

After this step, the real RHEL installation is accessible at `/sysroot`:

```
initramfs root (/)          Real RHEL root (/sysroot)
├── /sbin/init (systemd)    ├── /sysroot/bin/
├── /usr/lib/...            ├── /sysroot/etc/
├── /etc/...                ├── /sysroot/home/
├── /dev/...                ├── /sysroot/usr/
├── /proc/...               ├── /sysroot/var/
├── /sys/...                ├── /sysroot/boot/
└── /sysroot/ ←─────────────┘   (mounted here)
```

### Phase 4: Filesystem Checks

Before mounting the root filesystem read-write, initramfs runs **`fsck`** (filesystem check) to ensure the filesystem is consistent:

```bash
# For ext4 filesystems:
fsck.ext4 -a /dev/mapper/rhel-root

# For XFS filesystems:
# XFS performs its own log replay at mount time -- no separate fsck needed
# But xfs_repair can be used for offline repair
```

| Filesystem | Check Tool | When It Runs |
|-----------|-----------|-------------|
| ext4 | `fsck.ext4` (or `e2fsck`) | Before mounting, if needed (based on mount count or time) |
| XFS | Internal journal replay | Automatically at mount time |
| XFS (manual repair) | `xfs_repair` | Must be run on unmounted filesystem |

> **On RHEL with XFS (default):** XFS doesn't use a traditional `fsck`. When XFS is mounted, it replays its write-ahead journal to recover from any unclean shutdown. This is much faster than a full filesystem check.

**The root filesystem is initially mounted read-only:**

```bash
mount -o ro /dev/mapper/rhel-root /sysroot
```

It will be remounted read-write later by systemd on the real system. Mounting read-only first ensures the filesystem check can run safely.

### Phase 5: Pivot Root (switch_root)

Once the real root filesystem is mounted at `/sysroot`, the system needs to **switch** from the initramfs root to the real root. This is called **pivot_root** or **switch_root**.

```
BEFORE switch_root:
┌─────────────────────┐
│  / (initramfs tmpfs) │  ← Current root
│  ├── /sbin/init      │
│  ├── /usr/lib/...    │
│  └── /sysroot/       │  ← Real OS mounted here
│      ├── bin/        │
│      ├── etc/        │
│      ├── usr/        │
│      └── var/        │
└─────────────────────┘

              ↓  switch_root  ↓

AFTER switch_root:
┌─────────────────────┐
│  / (real XFS root)   │  ← New root (was /sysroot)
│  ├── /bin/           │
│  ├── /etc/           │
│  ├── /usr/           │
│  ├── /var/           │
│  └── /boot/          │
└─────────────────────┘
   (initramfs tmpfs has been deleted from memory)
```

**What `switch_root` does:**

1. Deletes **all files** from the initramfs tmpfs (to free the RAM)
2. Makes `/sysroot` the new `/` (root)
3. Executes **`/sbin/init`** (systemd) from the **real root filesystem**
4. The new systemd instance re-executes itself and takes over as PID 1

> **The difference between `pivot_root` and `switch_root`:** `pivot_root` is the kernel system call. `switch_root` is a higher-level command (from `util-linux`) used by initramfs. It calls `pivot_root` and then cleans up the old root. On RHEL, systemd's `systemctl switch-root` is used, which internally handles the whole transition.

---

## Understanding /sysroot

`/sysroot` is a critical concept during the boot process:

| State | What `/sysroot` Is |
|-------|-------------------|
| **Before initramfs mounts root** | An empty directory in the initramfs tmpfs |
| **After initramfs mounts root** | The real RHEL root filesystem, mounted here |
| **After switch_root** | No longer exists -- what was `/sysroot` is now `/` |
| **When using `rd.break`** | The real root, mounted **read-only** at `/sysroot` |

**Why `rd.break` matters:**

When you append `rd.break` to the kernel command line, the initramfs **pauses just before** executing `switch_root`. This gives you a shell where:

- `/` is the initramfs tmpfs
- `/sysroot` is the real RHEL root (mounted **read-only**)
- You can remount `/sysroot` read-write and make changes

This is how you **reset a lost root password** on RHEL.

---

## The rd.break Trick -- Pausing Before Pivot

This is one of the most important troubleshooting techniques for RHEL admins:

**Steps:**

1. Reboot → interrupt GRUB2 → press `e` on the kernel entry
2. Find the `linux` line and append `rd.break`
3. Press `Ctrl+x` to boot

You'll get a shell prompt:

```
switch_root:/#
```

At this point:

```
/ (initramfs)                    /sysroot (real OS, read-only)
├── Everything from initramfs    ├── The real RHEL installation
└── You are HERE                 └── Mounted READ-ONLY
```

**To reset the root password:**

```bash
# Step 1: Remount /sysroot as read-write
mount -o remount,rw /sysroot

# Step 2: Enter the real root filesystem
chroot /sysroot

# Step 3: Change the password
passwd root

# Step 4: Trigger SELinux relabel on next boot
touch /.autorelabel

# Step 5: Exit chroot and exit the shell
exit
exit
```

> **Why `touch /.autorelabel`?** The `passwd` command creates a new `/etc/shadow` file. Since SELinux is not yet active in the initramfs environment, this new file has no SELinux context. Without relabeling, SELinux would block authentication because the context is wrong. The `/.autorelabel` file tells RHEL to perform a full SELinux relabel on the next boot.

---

## Inspecting initramfs on a Running System

```bash
# View the contents of initramfs (list files)
lsinitrd /boot/initramfs-$(uname -r).img

# View only the included kernel modules
lsinitrd /boot/initramfs-$(uname -r).img -m

# Extract a specific file from initramfs
lsinitrd /boot/initramfs-$(uname -r).img /etc/fstab

# View the dracut modules included
lsinitrd /boot/initramfs-$(uname -r).img | grep -E "^drw"

# Check the size of initramfs
ls -lh /boot/initramfs-$(uname -r).img
```

---

## Building and Rebuilding initramfs with dracut

**dracut** is the tool that builds the initramfs on RHEL. It's like a "compiler" for the initramfs.

```bash
# Rebuild initramfs for the current kernel
sudo dracut --force /boot/initramfs-$(uname -r).img $(uname -r)

# Rebuild with verbose output (see what's being included)
sudo dracut --force --verbose /boot/initramfs-$(uname -r).img $(uname -r)

# Rebuild and include a specific driver
sudo dracut --force --add-drivers "megaraid_sas" /boot/initramfs-$(uname -r).img $(uname -r)

# Rebuild for ALL installed kernels
sudo dracut --regenerate-all --force

# Build a "rescue" initramfs with all drivers (larger but works on any hardware)
sudo dracut --force --no-hostonly /boot/initramfs-$(uname -r)-rescue.img $(uname -r)
```

### dracut Configuration

| File | Purpose |
|------|---------|
| `/etc/dracut.conf` | Main dracut configuration |
| `/etc/dracut.conf.d/*.conf` | Drop-in configuration files |
| `/usr/lib/dracut/modules.d/` | Dracut module scripts |

```bash
# Example /etc/dracut.conf.d/custom.conf
# Add extra drivers to initramfs
add_drivers+=" nvme megaraid_sas "

# Add extra dracut modules
add_dracutmodules+=" lvm dm "

# Force inclusion of all drivers (not just host-specific)
hostonly="no"
```

> **`hostonly` mode (default):** dracut only includes drivers for the hardware detected on THIS machine. This makes the initramfs smaller and faster. The downside is that if you move the disk to different hardware, it won't boot because the drivers won't be in initramfs.
>
> **`no-hostonly` mode:** dracut includes drivers for ALL supported hardware. The initramfs is much larger but will boot on any compatible hardware. Use this for rescue images or when building images that might run on different hardware.

---

## Common initramfs Problems

| Problem | Symptom | Fix |
|---------|---------|-----|
| Missing storage driver | `Kernel panic - not syncing: VFS: Unable to mount root fs` | Rebuild initramfs with the correct driver: `dracut --add-drivers "driver_name" --force` |
| Corrupted initramfs | Kernel panic during boot, garbled messages | Rebuild: `dracut --force` |
| Missing LVM modules | `root`: waiting for device /dev/mapper/... | Rebuild with LVM: `dracut --force --add lvm` |
| Wrong root= parameter | `mount: /sysroot: can't find...` | Fix in GRUB2 kernel command line or `/etc/default/grub` |
| initramfs too old | Boot fails after kernel upgrade | `dracut --force /boot/initramfs-$(uname -r).img $(uname -r)` |
| SELinux blocks after rd.break | Can't log in after password reset | Create `/.autorelabel` before exiting `rd.break` |

---

## What's Next?

At this point:

1. The kernel has started, loaded drivers from initramfs
2. systemd (inside initramfs) has activated LVM, run filesystem checks, and mounted the real root on `/sysroot`
3. The system has pivoted from initramfs to the real root filesystem
4. A **new instance of systemd** is now running from the real filesystem

In the next module, we'll explore how this new systemd instance brings the rest of the system to life -- starting services, reaching targets, and eventually presenting you with a login screen.

> **Remember:** The initramfs exists because the kernel can't "see" the real root filesystem without drivers, and those drivers are stored on the root filesystem. initramfs breaks this deadlock by carrying the essential drivers in a RAM-based filesystem that the kernel can access immediately.

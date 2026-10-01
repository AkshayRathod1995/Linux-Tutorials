# Stage 4: Kernel Loading & vmlinuz

## Index

1. [What Is vmlinuz?](#what-is-vmlinuz)
2. [Why Is the Kernel Compressed?](#why-is-the-kernel-compressed)
3. [Anatomy of vmlinuz](#anatomy-of-vmlinuz)
4. [How GRUB2 Loads the Kernel into Memory](#how-grub2-loads-the-kernel-into-memory)
5. [RAM Preparation Before Kernel Execution](#ram-preparation-before-kernel-execution)
6. [Kernel Decompression](#kernel-decompression)
7. [What the Kernel Does After Decompression](#what-the-kernel-does-after-decompression)
   - [Phase 1: Architecture-Specific Initialization](#phase-1-architecture-specific-initialization)
   - [Phase 2: start_kernel() -- The Main Initialization](#phase-2-start_kernel----the-main-initialization)
8. [Kernel Messages (dmesg)](#kernel-messages-dmesg)
9. [Kernel Files on a RHEL System](#kernel-files-on-a-rhel-system)
10. [Kernel Versioning in RHEL](#kernel-versioning-in-rhel)
11. [What's Next?](#whats-next)

---

## What Is vmlinuz?

`vmlinuz` is the **Linux kernel** -- the compressed, bootable kernel image file that lives in `/boot/`.

The name breaks down like this:

| Part | Meaning |
|------|---------|
| `vm` | **Virtual Memory** -- the kernel supports virtual memory management |
| `linu` | **Linux** -- it's the Linux kernel |
| `z` | **Compressed** -- the kernel is compressed (using gzip, bzip2, xz, or zstd) |

```bash
# View your kernel files
ls -lh /boot/vmlinuz-*
```

```
-rwxr-xr-x. 1 root root 11M Sep 15 10:30 /boot/vmlinuz-5.14.0-284.el9.x86_64
```

The file is about **11 MB** compressed. When decompressed, the kernel is typically **30-50 MB**.

> **Historical note:** Before virtual memory existed, the kernel was called `vmlinux` (uncompressed). The `z` was added when compression was introduced. The original uncompressed kernel is still built during compilation as `vmlinux`, and then compressed into `vmlinuz`.

---

## Why Is the Kernel Compressed?

Compressing the kernel solves several problems:

| Problem | How Compression Helps |
|---------|----------------------|
| **Disk space** | An 11 MB compressed kernel vs 40+ MB uncompressed saves significant space on `/boot` |
| **Load speed** | Reading 11 MB from disk is faster than reading 40 MB. Decompression in RAM is fast |
| **Multiple kernels** | `/boot` is typically 1 GB. Compression allows storing several kernel versions |
| **BIOS limitations** | Older BIOS could only read from the first 1024 cylinders -- smaller files helped stay within limits |

```bash
# See the compression format used by your kernel
file /boot/vmlinuz-$(uname -r)
```

```
/boot/vmlinuz-5.14.0-284.el9.x86_64: Linux kernel x86 boot executable bzImage, version 5.14.0-284.el9.x86_64 (...), RO-rootFS, swap_dev 0XA, Normal VGA
```

The `bzImage` stands for **"big zImage"** -- a compressed kernel image format that can be larger than 512 KB (unlike the original `zImage` format which had a 512 KB limit).

---

## Anatomy of vmlinuz

The `vmlinuz` file is not just a compressed blob. It has a specific structure:

```
┌──────────────────────────────────────────────────────────┐
│                                                          │
│  REAL-MODE BOOT HEADER (setup.bin)                       │
│  ~15 KB                                                  │
│                                                          │
│  Contains:                                               │
│  - Boot protocol version                                 │
│  - Kernel command line pointer                           │
│  - initramfs location in memory                          │
│  - Video mode settings                                   │
│  - Memory map from BIOS/UEFI                             │
│  - CPU mode switching code (Real Mode → Protected Mode)  │
│                                                          │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  DECOMPRESSION STUB                                      │
│  ~20 KB                                                  │
│                                                          │
│  A small program that:                                   │
│  1. Sets up a temporary environment                      │
│  2. Decompresses the kernel payload below                │
│  3. Jumps to the decompressed kernel                     │
│                                                          │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  COMPRESSED KERNEL PAYLOAD                               │
│  ~10 MB                                                  │
│                                                          │
│  The actual Linux kernel, compressed with                │
│  gzip, bzip2, xz, lzma, lz4, or zstd                    │
│                                                          │
│  When decompressed → 30-50 MB of kernel code             │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

---

## How GRUB2 Loads the Kernel into Memory

When GRUB2 processes the `linux` command from `grub.cfg`, here's exactly what happens:

**Step 1: GRUB2 reads `vmlinuz` from `/boot`**

GRUB2 uses its filesystem driver (XFS, ext4, etc.) to read the `vmlinuz` file from the `/boot` partition.

**Step 2: GRUB2 places `vmlinuz` at a specific memory address**

```
Memory Layout After GRUB2 Loads vmlinuz:

Address          Content
─────────────────────────────────
0x00000000       Interrupt Vector Table (IVT)
0x00000400       BIOS Data Area
...
0x00007C00       Where BIOS originally loaded the MBR (no longer needed)
...
0x00010000       Real-mode kernel setup code (from vmlinuz header)
     │           - Boot parameters
     │           - Kernel command line
     │           - Memory map
...
0x00100000       Protected-mode kernel (the compressed payload)
(1 MB mark)      - This is where the decompressor and compressed
                   kernel image are loaded
...
```

**Step 3: GRUB2 loads `initramfs` into memory**

When GRUB2 processes the `initrd` command, it loads the initramfs image into a high memory region and records its location in the kernel's boot parameters.

**Step 4: GRUB2 hands off to the kernel**

GRUB2 jumps to the entry point specified in the kernel's boot header. The CPU begins executing the kernel code.

---

## RAM Preparation Before Kernel Execution

Before the kernel can run, memory must be prepared. This happens in stages:

### What GRUB2 Does (Before Kernel Starts)

| Step | What Happens |
|------|-------------|
| 1 | Queries BIOS/UEFI for the **memory map** (how much RAM is installed, which regions are usable vs reserved for hardware) |
| 2 | Places this memory map in the kernel's boot parameters structure |
| 3 | Sets up a minimal **GDT** (Global Descriptor Table) for protected mode |
| 4 | Loads the kernel's real-mode code at a low memory address |
| 5 | Loads the kernel's protected-mode code (compressed) at the 1 MB mark |
| 6 | Loads the initramfs at a high memory address |
| 7 | Records the initramfs location and size in the boot parameters |

### What the Kernel's Setup Code Does (Before Decompression)

| Step | What Happens |
|------|-------------|
| 1 | **Validates boot parameters** -- checks the boot protocol version |
| 2 | **Detects CPU capabilities** -- checks for required CPU features |
| 3 | **Queries additional hardware info** -- keyboard, video mode, APM/ACPI |
| 4 | **Switches CPU from Real Mode → Protected Mode** (BIOS) or stays in the appropriate mode (UEFI) |
| 5 | **Enables the A20 gate** -- allows accessing memory above 1 MB (a historical quirk of x86) |
| 6 | **Jumps to the decompression stub** |

> **Why Real Mode → Protected Mode?** On BIOS systems, the CPU starts in 16-bit Real Mode (a legacy mode from the 8086 era). In Real Mode, the CPU can only access 1 MB of RAM. The kernel needs much more, so it switches to 32-bit Protected Mode (or 64-bit Long Mode), which can access the full RAM.

---

## Kernel Decompression

Once the CPU is in the correct mode, the decompression stub runs:

```
Before Decompression:
┌─────────────────────────────┐  ┌─────────────────────────────┐
│     Decompression Stub      │  │  Compressed Kernel Payload   │
│         (~20 KB)            │  │        (~10 MB)              │
└─────────────────────────────┘  └─────────────────────────────┘

                    ↓ Decompressor runs ↓

After Decompression:
┌──────────────────────────────────────────────────────────────┐
│                                                              │
│              Decompressed Linux Kernel                        │
│                    (~30-50 MB)                                │
│                                                              │
│  Now the FULL kernel code is in memory and ready to execute  │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

The decompression process:

1. The stub allocates a memory region for the decompressed kernel
2. It decompresses the payload using the appropriate algorithm (gzip/xz/zstd)
3. It relocates the decompressed kernel to its final address
4. It clears the BSS segment (uninitialized data) to zero
5. It jumps to `startup_64` (on 64-bit systems) -- the kernel's true entry point

**Compression algorithms used in RHEL kernels:**

| Algorithm | Speed | Ratio | RHEL Version |
|-----------|-------|-------|-------------|
| gzip | Medium | Medium | RHEL 7 |
| xz | Slow (decompress: medium) | Best | RHEL 8 |
| zstd | Fast | Good | RHEL 9 |

RHEL 9 switched to **zstd** because it offers nearly the same compression ratio as xz but decompresses much faster, reducing boot time.

---

## What the Kernel Does After Decompression

### Phase 1: Architecture-Specific Initialization

These are the very first things the decompressed kernel does:

| Step | What Happens | Why It Matters |
|------|-------------|----------------|
| 1 | **Set up the page tables** | Enables virtual memory -- every process will get its own virtual address space |
| 2 | **Initialize memory management** | Kernel now manages all physical RAM |
| 3 | **Set up the IDT** (Interrupt Descriptor Table) | Allows the kernel to handle hardware interrupts and CPU exceptions |
| 4 | **Enable paging** | Switches from physical to virtual memory addresses |
| 5 | **Initialize the CPU** | Set up per-CPU data structures, enable SSE/AVX if available |

### Phase 2: start_kernel() -- The Main Initialization

The `start_kernel()` function in the kernel source code is where the "real" kernel initialization happens. It runs a long sequence of initialization functions:

| Step | Function (simplified) | What It Does |
|------|----------------------|-------------|
| 1 | `setup_arch()` | Parse boot parameters, detect CPU, set up memory zones |
| 2 | `setup_per_cpu_areas()` | Allocate per-CPU data structures |
| 3 | `build_all_zonelists()` | Create the memory zone lists for the page allocator |
| 4 | `page_alloc_init()` | Initialize the page allocator (the core memory manager) |
| 5 | `trap_init()` | Set up CPU exception handlers |
| 6 | `mm_init()` | Initialize the memory management subsystem |
| 7 | `sched_init()` | Initialize the process scheduler |
| 8 | `init_IRQ()` | Initialize interrupt handling |
| 9 | `time_init()` | Set up the system clock and timer |
| 10 | `console_init()` | Initialize the console for kernel messages |
| 11 | `vfs_caches_init()` | Set up the Virtual File System (VFS) |
| 12 | `signals_init()` | Initialize the signal handling system |
| 13 | **`rest_init()`** | **Create PID 1 (init/systemd) and PID 2 (kthreadd)** |

The final step, `rest_init()`, is critical:

1. It creates the **init process** (PID 1) by running `/sbin/init` from the initramfs
2. On RHEL, `/sbin/init` is a symlink to **systemd**
3. It creates **kthreadd** (PID 2) -- the kernel thread daemon that manages all kernel threads
4. The initialization thread becomes the **idle process** (PID 0) -- the process that runs when the CPU has nothing else to do

```
After start_kernel() completes:

PID 0 → idle process (the scheduler runs this when CPU is idle)
PID 1 → /sbin/init (systemd) from initramfs
PID 2 → kthreadd (manages kernel threads)
```

---

## Kernel Messages (dmesg)

During all of this initialization, the kernel prints messages about what it's doing. These messages are stored in the **kernel ring buffer** and can be viewed with `dmesg`:

```bash
# View all kernel boot messages
dmesg

# View only the first 50 lines (earliest boot messages)
dmesg | head -50

# Filter for specific topics
dmesg | grep -i "memory"    # Memory detection
dmesg | grep -i "cpu"       # CPU detection
dmesg | grep -i "scsi\|sda" # Disk detection
dmesg | grep -i "error"     # Any errors during boot

# View with timestamps
dmesg -T

# View with human-readable timestamps and priority levels
dmesg -T --level=err,warn

# Using journalctl (systemd-based, persistent)
journalctl -k               # Kernel messages from current boot
journalctl -k -b -1         # Kernel messages from previous boot
```

**Sample early boot messages:**

```
[    0.000000] Linux version 5.14.0-284.el9.x86_64 (mockbuild@x86-64-02.build.eng.rdu2.redhat.com)
               (gcc (GCC) 11.3.1 20221121 (Red Hat 11.3.1-4)) #1 SMP ...
[    0.000000] Command line: BOOT_IMAGE=(hd0,msdos1)/vmlinuz-5.14.0-284.el9.x86_64
               root=/dev/mapper/rhel-root ro crashkernel=auto ...
[    0.000000] BIOS-provided physical RAM map:
[    0.000000]  BIOS-e820: [mem 0x0000000000000000-0x000000000009fbff] usable
[    0.000000]  BIOS-e820: [mem 0x0000000000100000-0x00000000bffdffff] usable
[    0.000000] Memory: 8052012K/8388080K available (14340K kernel code, ...)
[    0.000004] CPU: Intel(R) Xeon(R) CPU E5-2690 v4 @ 2.60GHz
[    0.100000] PCI: Using configuration type 1
[    0.500000] SCSI subsystem initialized
[    1.200000] sd 0:0:0:0: [sda] 104857600 512-byte logical blocks
```

---

## Kernel Files on a RHEL System

```bash
# All kernel-related files in /boot
ls -lh /boot/
```

| File | Purpose |
|------|---------|
| `vmlinuz-5.14.0-284.el9.x86_64` | The compressed kernel image |
| `initramfs-5.14.0-284.el9.x86_64.img` | The initial RAM filesystem (covered in next module) |
| `config-5.14.0-284.el9.x86_64` | The kernel build configuration (what features are compiled in) |
| `System.map-5.14.0-284.el9.x86_64` | Symbol table mapping kernel function names to memory addresses (used for debugging) |
| `symvers-5.14.0-284.el9.x86_64.gz` | Module symbol version information |

```bash
# Currently running kernel version
uname -r
```

```
5.14.0-284.el9.x86_64
```

```bash
# Detailed kernel info
uname -a
```

```
Linux servera.lab.example.com 5.14.0-284.el9.x86_64 #1 SMP PREEMPT_DYNAMIC ... x86_64 GNU/Linux
```

---

## Kernel Versioning in RHEL

Understanding RHEL kernel version numbers:

```
5.14.0-284.el9.x86_64
│  │  │  │    │    │
│  │  │  │    │    └── Architecture (64-bit x86)
│  │  │  │    └── Distribution (Enterprise Linux 9)
│  │  │  └── RHEL build number (284th build of this kernel series)
│  │  └── Patch level
│  └── Minor version
└── Major version
```

| RHEL Version | Kernel Series |
|-------------|--------------|
| RHEL 7 | 3.10.x |
| RHEL 8 | 4.18.x |
| RHEL 9 | 5.14.x |

> Red Hat backports security fixes and features from newer upstream kernels into their stable kernel series. So RHEL 9's kernel `5.14.0-284` is not the same as upstream `5.14.0` -- it has years of backported fixes and features.

---

## What's Next?

At this point:

1. The kernel has been decompressed and is running
2. It has initialized all core subsystems (memory, scheduler, interrupts, VFS)
3. It has created PID 1 (systemd from initramfs) and PID 2 (kthreadd)
4. But it **has not mounted the real root filesystem yet**

The kernel knows the initramfs is in memory (GRUB2 told it where). In the next module, we'll explore what's inside the initramfs, how the kernel uses it to find and mount the real root filesystem, and how the system transitions from the temporary initramfs environment to the real RHEL installation.

> **Remember:** The kernel cannot mount the real root filesystem directly at this point because it might need drivers that aren't compiled into the kernel (like LVM, RAID, or specific storage controller drivers). These drivers are inside the initramfs. The initramfs is the kernel's "toolbox" that it uses to prepare the real system.

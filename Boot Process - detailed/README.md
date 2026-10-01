# The Red Hat Enterprise Linux Boot Process

A complete guide to understanding how a RHEL system goes from a powered-off state to a login prompt. Covers every stage in detail -- firmware (BIOS/UEFI), MBR, GRUB2, kernel loading, initramfs, systemd, and service initialization -- with real-world troubleshooting, flow diagrams, and interview preparation.

Whether you are preparing for an RHCSA exam, debugging a system that won't boot, or simply want to understand what happens when you press the power button, this guide takes you from fundamentals through advanced boot recovery.

---

## Modules

| # | Module | Description |
|---|--------|-------------|
| 01 | [BIOS, POST & Hardware Initialization](01_BIOS_POST_and_Hardware_Init.md) | What happens the instant you press the power button -- firmware, POST, and finding a bootable device. |
| 02 | [MBR & the Boot Sector](02_MBR_and_Boot_Sector.md) | The first 512 bytes of the disk, how they are structured, and how `boot.img` and `diskboot.img` bridge the gap to GRUB2. |
| 03 | [GRUB2 -- The Bootloader](03_GRUB2_Bootloader.md) | Role of GRUB2, its configuration, the boot menu, and how it loads the kernel and initramfs into memory. |
| 04 | [Kernel Loading & vmlinuz](04_Kernel_Loading_vmlinuz.md) | What `vmlinuz` is, how the kernel is decompressed and loaded, and how RAM is prepared before the kernel takes over. |
| 05 | [initramfs & Mounting the Real Root](05_initramfs_and_Root_Mounting.md) | The temporary root filesystem, how the real OS is mounted on `/sysroot`, filesystem checks, and the pivot to the real root. |
| 06 | [systemd -- From initramfs to the Real OS](06_systemd_Boot_and_Targets.md) | How systemd starts inside initramfs, transitions to the real OS, targets, and dependency management. |
| 07 | [Services, Targets & the Login Screen](07_Services_and_Login_Screen.md) | How services are initialized, the difference between `multi-user.target` and `graphical.target`, and how you get the login prompt. |
| 08 | [Boot Flow Diagram](08_Boot_Flow_Diagram.md) | A complete visual flow diagram of the entire boot process for quick reference and revision. |
| 09 | [Troubleshooting Boot Issues](09_Troubleshooting_Boot_Issues.md) | Diagnosing and fixing boot failures -- broken GRUB, kernel panics, initramfs problems, bad `/etc/fstab`, resetting root password, and more. |
| 10 | [Interview Revision Cheat Sheet](10_Interview_Revision_Cheat_Sheet.md) | Condensed single-page reference of the entire boot process, all commands, config files, and interview-ready answers. |

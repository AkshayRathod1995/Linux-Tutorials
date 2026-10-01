# The Red Hat Enterprise Linux Boot Process

A concise guide to the RHEL boot process -- from power button to login screen. Covers firmware, MBR, GRUB2, kernel, initramfs, systemd, and services with troubleshooting and interview prep.

---

## Modules

| # | Module | Description |
|---|--------|-------------|
| 00 | [Complete Boot Process (Single Page)](00_Complete_Boot_Process.md) | **Start here.** The entire boot process on one page. |
| 01 | [BIOS, POST & Hardware Init](01_BIOS_POST_and_Hardware_Init.md) | Power button → POST → finding a bootable device. |
| 02 | [MBR & Boot Sector](02_MBR_and_Boot_Sector.md) | First 512 bytes, boot.img, diskboot.img, core.img. |
| 03 | [GRUB2 Bootloader](03_GRUB2_Bootloader.md) | Config, boot menu, loading kernel + initramfs. |
| 04 | [Kernel Loading (vmlinuz)](04_Kernel_Loading_vmlinuz.md) | Decompression, start_kernel, PID 1/2. |
| 05 | [initramfs & Root Mounting](05_initramfs_and_Root_Mounting.md) | Drivers, /sysroot, switch_root, rd.break, dracut. |
| 06 | [systemd & Targets](06_systemd_Boot_and_Targets.md) | Two phases, targets, rescue/emergency modes. |
| 07 | [Services & Login Screen](07_Services_and_Login_Screen.md) | Service management, getty, GDM, boot complete. |
| 08 | [Boot Flow Diagram](08_Boot_Flow_Diagram.md) | Visual flow of the entire process. |
| 09 | [Troubleshooting](09_Troubleshooting_Boot_Issues.md) | Diagnosing and fixing boot failures. |
| 10 | [Interview Revision Cheat Sheet](10_Interview_Revision_Cheat_Sheet.md) | Everything in one page for interview prep. |

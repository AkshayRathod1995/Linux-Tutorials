# Stage 3: GRUB2 -- The Bootloader

## What GRUB2 Does

1. Reads `/boot/grub2/grub.cfg`
2. Displays boot menu with available kernels
3. **Loads `vmlinuz`** (compressed kernel) into RAM
4. **Loads `initramfs`** image into RAM
5. Passes kernel command line parameters
6. Hands control to the kernel -- GRUB2 is done

---

## Configuration

```
/etc/default/grub          ← Your settings (edit this)
       +
/etc/grub.d/               ← Scripts that generate menu entries
       │
       ▼  grub2-mkconfig
       │
/boot/grub2/grub.cfg       ← What GRUB2 actually reads (NEVER edit directly)
```

```bash
# After editing /etc/default/grub, regenerate:
sudo grub2-mkconfig -o /boot/grub2/grub.cfg        # BIOS
sudo grub2-mkconfig -o /boot/efi/EFI/redhat/grub.cfg  # UEFI
```

### Key /etc/default/grub Settings

| Parameter | Meaning | Example |
|-----------|---------|---------|
| `GRUB_TIMEOUT` | Seconds before auto-boot | `5` |
| `GRUB_DEFAULT` | Default kernel | `saved` |
| `GRUB_CMDLINE_LINUX` | Kernel parameters for every boot | `"crashkernel=auto root=/dev/mapper/rhel-root rd.lvm.lv=rhel/root rhgb quiet"` |

---

## Important Kernel Command Line Parameters

| Parameter | Meaning |
|-----------|---------|
| `root=/dev/mapper/rhel-root` | Root filesystem device |
| `rd.lvm.lv=rhel/root` | Activate LVM volume in initramfs |
| `rhgb quiet` | Graphical boot, suppress messages |
| `rd.break` | **Pause before switch_root (password reset)** |
| `systemd.unit=rescue.target` | Boot into rescue mode |
| `systemd.unit=emergency.target` | Boot into emergency mode |
| `enforcing=0` | SELinux permissive mode |

```bash
cat /proc/cmdline    # See what was passed at boot
```

---

## Editing Boot Parameters Temporarily

1. Reboot → interrupt GRUB2 menu (press any key except Enter)
2. Select kernel entry → press **`e`**
3. Find line starting with `linux` → make changes
4. Press **`Ctrl+x`** to boot

Changes only affect **this one boot** -- original config is untouched.

---

## GRUB2 Emergency (grub> prompt)

If GRUB2 can't find its config, you can boot manually:

```
grub> set root=(hd0,msdos1)
grub> linux /vmlinuz-5.14.0-284.el9.x86_64 root=/dev/mapper/rhel-root ro
grub> initrd /initramfs-5.14.0-284.el9.x86_64.img
grub> boot
```

Then fix permanently: `grub2-install /dev/sda` + `grub2-mkconfig -o /boot/grub2/grub.cfg`

---

## Managing Default Kernel

```bash
grubby --info=ALL              # List all kernels
grub2-editenv list             # See current default
sudo grub2-set-default 0      # Set default by index
```

**Next:** GRUB2 has loaded vmlinuz and initramfs into memory.

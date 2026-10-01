# Stage 7: Services & the Login Screen

## How Services Start

After `switch_root`, systemd on the real OS works toward `default.target`. It reads unit files and starts services **in parallel** based on dependency ordering.

```
Unit file locations (priority order):
/etc/systemd/system/         ← Admin overrides (highest priority)
/run/systemd/system/         ← Runtime-generated
/usr/lib/systemd/system/     ← Package-installed defaults
```

---

## Service Unit File (Quick Look)

```ini
# /usr/lib/systemd/system/sshd.service
[Unit]
Description=OpenSSH server daemon
After=network.target           # Start after networking
Wants=sshd-keygen.target       # Also start key generation

[Service]
Type=notify                    # Service tells systemd when it's ready
ExecStart=/usr/sbin/sshd -D
Restart=on-failure

[Install]
WantedBy=multi-user.target     # Enable → symlink into multi-user.target.wants/
```

---

## Enabling & Managing Services

```bash
systemctl enable sshd          # Start at boot (creates symlink)
systemctl disable sshd         # Don't start at boot (removes symlink)
systemctl start sshd           # Start now
systemctl stop sshd            # Stop now
systemctl restart sshd         # Restart now
systemctl status sshd          # Check status
systemctl is-enabled sshd      # Check if enabled
systemctl mask sshd            # Prevent starting entirely (even manually)
systemctl unmask sshd          # Undo mask
```

**What `enable` actually does:**

```bash
# Creates a symlink:
/etc/systemd/system/multi-user.target.wants/sshd.service → /usr/lib/systemd/system/sshd.service
```

---

## Text Login (multi-user.target)

systemd starts `getty@tty1.service` → runs `/sbin/agetty` → displays:

```
Red Hat Enterprise Linux 9.2 (Plow)
Kernel 5.14.0-284.el9.x86_64 on an x86_64

server1 login: _
```

**Login flow:** agetty → reads username → calls `/bin/login` → PAM authenticates → starts shell

---

## Graphical Login (graphical.target)

systemd starts `gdm.service` (GNOME Display Manager) → launches Xorg/Wayland → shows graphical login.

```
graphical.target
├── multi-user.target (all its services)
└── gdm.service → GDM login screen
```

---

## Boot Complete -- What's Running?

```bash
systemctl list-units --type=service --state=running   # Running services
pstree -p 1 | head -20                                # Process tree from PID 1
systemd-analyze                                        # Boot time breakdown
systemd-analyze blame                                  # Slowest services
```

**The boot is complete.** From power button to login:

```
Power → BIOS/UEFI → MBR/ESP → GRUB2 → vmlinuz → initramfs → switch_root → systemd → login
```

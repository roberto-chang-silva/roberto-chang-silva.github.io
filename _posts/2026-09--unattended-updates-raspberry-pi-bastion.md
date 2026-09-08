---
title: "Setting up a Raspberry Pi bastion host for long-term unattended operation"
date: 2026-09-08
permalink: /posts/unattended-updates-raspberry-pi-bastion/
tags:
  - raspberry-pi
  - linux
  - security
  - automation
  - homelab
  - tailscale
  - ssh
categories:
  - homelab-systems
excerpt: "Configuring unattended upgrades, watchdog, fail2ban, SSH hardening, and Tailscale auto-update on a headless Pi left alone for long-term."
---

I'm preparing a Raspberry Pi to leave it at my relative's home as a bastion host while I'm away, far away with no one to physically touch it. The default Raspberry Pi OS install does nothing in the way of automatic updates, no watchdog, no brute-force protection, nothing, and this is expected because the Raspberry Pi was not designed for that. This is everything I configured to make it as self-sufficient as possible.

# Automatic security updates

Install unattended-upgrades if it isn't already:

```bash
sudo apt install unattended-upgrades
sudo dpkg-reconfigure -plow unattended-upgrades
```

The second command prompts you to enable automatic updates. Say yes.

Before configuring origins, check what's actually on your system:

```bash
apt-cache policy
```

On Trixie you'll see `Debian`, `Debian-Security`, and `Raspberry Pi Foundation`. Don't copy origins from older guides, the `Raspbian` label that appears in a lot of tutorials does not exist in Trixie.

Open the config:

```bash
sudo nano /etc/apt/apt.conf.d/50unattended-upgrades
```

Replace the `Origins-Pattern` block with:

```yaml
Unattended-Upgrade::Origins-Pattern {
        "origin=Debian,codename=${distro_codename},label=Debian";
        "origin=Debian,codename=${distro_codename},label=Debian-Security";
        "origin=Debian,codename=${distro_codename}-security,label=Debian-Security";
        "origin=Debian,codename=${distro_codename}-updates";
};
```

Then uncomment and set these (they're all in the file already, just commented out):

```ini
Unattended-Upgrade::AutoFixInterruptedDpkg "true";
Unattended-Upgrade::MinimalSteps "true";
Unattended-Upgrade::Remove-Unused-Kernel-Packages "true";
Unattended-Upgrade::Remove-New-Unused-Dependencies "true";
Unattended-Upgrade::Remove-Unused-Dependencies "true";
Unattended-Upgrade::Automatic-Reboot "true";
Unattended-Upgrade::Automatic-Reboot-WithUsers "false";
Unattended-Upgrade::Automatic-Reboot-Time "03:00";
Unattended-Upgrade::SyslogEnable "true";
Unattended-Upgrade::SyslogFacility "daemon";
```

`MinimalSteps` breaks the upgrade into small chunks so a power loss mid-upgrade is less likely to leave dpkg broken. `Automatic-Reboot-WithUsers "false"` means a stuck open session never blocks a security patch from applying.

## Testing it

```bash
sudo unattended-upgrade --dry-run --debug
```

> Packages that will be upgraded: ...
> No packages found that can be upgraded unattended and no pending auto-removals

No errors means the config parsed correctly. If origins are wrong they don't throw errors, they just match nothing, so cross-reference the output against `apt list --upgradable`.

# Scheduled reboot

A weekly reboot clears memory leaks and keeps things fresh. The unattended-upgrades auto-reboot is set to 3am, so schedule the weekly one an hour later to avoid any overlap:

```bash
sudo crontab -e
```

Add at the bottom:

```bash
0 4 * * 0 /sbin/reboot
```

Verify it saved:

```bash
sudo crontab -l
```

> 0 4 * * 0 /sbin/reboot

# Hardware watchdog

The Pi 4 and 5 have a hardware watchdog built in. If the system hangs and stops petting the watchdog, it reboots automatically. Without this, a hung kernel just sits there indefinitely until the scheduled weekly reboot.

```bash
sudo apt install watchdog
sudo systemctl enable watchdog --now
```

Edit `/etc/watchdog.conf`:

```ini
watchdog-device = /dev/watchdog
watchdog-timeout = 15
max-load-1 = 24
```

`max-load-1` triggers a reboot if the 1-minute load average exceeds 24, which catches a runaway process choking the system.

## Testing it

```bash
sudo systemctl status watchdog
```

> Active: active (running) since ...

# SSH hardening

Edit `/etc/ssh/sshd_config`:

```bash
sudo nano /etc/ssh/sshd_config
```

Set these:

```ini
PermitRootLogin no
PasswordAuthentication no
MaxAuthTries 3
```

Then restart SSH:

```bash
sudo systemctl restart ssh
```

Make absolutely sure your key-based auth is working in another session before closing your current one. Locking yourself out of a device remotely away is not a recoverable situation.

## Testing it

```bash
ssh -o PasswordAuthentication=no user@raspberry-pi
```

> Welcome to Raspberry Pi OS...

If it connects without prompting for a password, key auth is working and it's safe to disable password auth.

# fail2ban

Bans IPs after repeated failed SSH attempts. The defaults are fine for most setups (5 failures, 10 minute ban).

```bash
sudo apt install fail2ban
sudo systemctl enable fail2ban --now
```

## Testing it

```bash
sudo fail2ban-client status sshd
```

> Status for the jail: sshd
> |- Filter: ...
> `- Actions: ...

# Tailscale auto-update

Without this, Tailscale stays on whatever version it was installed at. For a device left alone for years that's a problem.

```bash
sudo tailscale set --auto-update
```

Verify:

```bash
tailscale version
```

> 1.x.x
> ...
>   auto-update: enabled

# Keep services alive after reboot

Connect runs as a user-level service, not root. Without user-lingering enabled, it only starts after you log in, which means after a reboot the Pi is unreachable via Connect until someone logs in locally. Since that someone is 10,000km away, enable lingering:

```bash
loginctl enable-linger user
```

This keeps your user session alive at boot without requiring an interactive login, so Connect and any other user-level services come up automatically.

## Testing it

Reboot the Pi and try to connect via Raspberry Pi Connect before logging in locally:

```bash
sudo reboot
```

If Connect is available in the dashboard within a minute or two of the Pi coming back online, lingering is working.

# What this does and does not replace

Unattended-upgrades handles security patches automatically. It does not replace a deliberate `sudo apt upgrade` for everything else. For a bastion host that's intentional: you don't want `rpi-connect` or `tailscale` randomly upgrading mid-session and dropping your only connection to the device. Do full upgrades manually, wrapped in `tmux` so the session survives any disconnection.

Notes:

- Logs for unattended-upgrades: `journalctl -t unattended-upgrades` or `/var/log/unattended-upgrades/`
- Logs for fail2ban: `journalctl -u fail2ban`
- Logs for watchdog: `journalctl -u watchdog`
- If you have an SMTP relay, uncomment `Unattended-Upgrade::Mail` and `Unattended-Upgrade::MailReport "on-change"` to get notified on failures
- The SSD removes the biggest long-term reliability risk on a Pi. If you're still on SD card, consider moving before leaving the device unattended

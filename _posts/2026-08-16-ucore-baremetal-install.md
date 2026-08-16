---
title: 'uCore Bare-Metal install (Butane, Podman, Tailscale)'
date: 2026-08-16
permalink: /posts/ucore-baremetal-install/
tags:
  - homelab
  - podman
  - tailscale
---

ucore-baremetal-install
uCore is Fedora CoreOS with sane homelab defaults (Tailscale, nvidia/zfs variants) baked in by Universal Blue. I use it for containers in my homelab servers as it is really easy to set, manage and use. /usr is read-only, only /etc and /var persist, everything gets declared upfront in Butane, transpiled to Ignition, consumed once on first boot. No installer wizard.

# How to do it?

## Get the ISO

Grab the live ISO for your arch/stream from [fedoraproject.org/coreos/download](https://fedoraproject.org/coreos/download/). Used x86_64 stable here.

Ventoy doesn't boot these ISOs reliably (drops to bare `grub>`). Use Fedora Media Writer instead, or hit Ctrl+r on the Ventoy menu to force GRUB2 mode (Though I tried this and couldn't boot it using Ventoy).

## Pick a flavor/tag

Check the [ucore README](https://github.com/ublue-os/ucore) flavor matrix. Went with `ghcr.io/ublue-os/ucore:stable` here.

## Generate password hash

```bash
podman run -ti --rm quay.io/coreos/mkpasswd --method=yescrypt
```

## Write the Butane config

Based on the [ucore-autorebase example](https://github.com/ublue-os/ucore/blob/main/examples/ucore-autorebase.butane):

```yaml
variant: fcos
version: 1.4.0

passwd:
  users:
    - name: user
      ssh_authorized_keys:
        - KEY1
      password_hash: password1
      home_dir: /home/user
      groups:
        - wheel
      shell: /bin/bash

    - name: ai
      ssh_authorized_keys:
        - KEY1
      password_hash: password2
      home_dir: /home/ai
      shell: /bin/bash

storage:
  directories:
    - path: /etc/ucore-autorebase
      mode: 0754
  files:
    - path: /etc/hostname
      mode: 0644
      overwrite: true
      contents:
        inline: my-pc

systemd:
  units:
    - name: ucore-unsigned-autorebase.service
      enabled: true
      contents: |
        [Unit]
        Description=uCore autorebase to unsigned OCI and reboot
        ConditionPathExists=!/etc/ucore-autorebase/unverified
        ConditionPathExists=!/etc/ucore-autorebase/signed
        After=network-online.target
        Wants=network-online.target
        [Service]
        Type=oneshot
        StandardOutput=journal+console
        ExecStart=/usr/bin/rpm-ostree rebase --bypass-driver ostree-unverified-registry:ghcr.io/ublue-os/ucore:stable
        ExecStart=/usr/bin/touch /etc/ucore-autorebase/unverified
        ExecStart=/usr/bin/systemctl disable ucore-unsigned-autorebase.service
        ExecStart=/usr/bin/systemctl reboot
        [Install]
        WantedBy=multi-user.target
    - name: ucore-signed-autorebase.service
      enabled: true
      contents: |
        [Unit]
        Description=uCore autorebase to signed OCI and reboot
        ConditionPathExists=/etc/ucore-autorebase/unverified
        ConditionPathExists=!/etc/ucore-autorebase/signed
        After=network-online.target
        Wants=network-online.target
        [Service]
        Type=oneshot
        StandardOutput=journal+console
        ExecStart=/usr/bin/rpm-ostree rebase --bypass-driver ostree-image-signed:docker://ghcr.io/ublue-os/ucore:stable
        ExecStart=/usr/bin/touch /etc/ucore-autorebase/signed
        ExecStart=/usr/bin/systemctl disable ucore-signed-autorebase.service
        ExecStart=/usr/bin/systemctl reboot
        [Install]
        WantedBy=multi-user.target
```

Please, never commit real hashes/keys, keep the actual `.bu`/`.ign` out of version control.

## Transpile to Ignition

```bash
podman pull quay.io/coreos/butane:release

podman run --interactive --rm quay.io/coreos/butane:release \
       --pretty --strict < ucore-autorebase.bu > ucore-autorebase.ign
```

## Stage the Ignition file

Second USB, HTTP, whatever `coreos-installer` can reach.

## Firmware

Disable Secure Boot, UEFI mode on.

## Boot live ISO, install

Check the disk were you will install it first! my case was `nvme0n1` make sure you don't have important information in that disk as it will be wiped. Follow the 3 BBB's rule before doing this: Backup, backup and backup!

```bash
sudo coreos-installer install /dev/nvme0n1 \
  --ignition-file /path/to/ucore-autorebase.ign
```

## Reboot, let it rebase

```bash
sudo systemctl reboot
```

Don't power-cycle manually, it reboots itself twice (unsigned rebase, then signed).

## Enable services post-rebase

Nothing's on by default:
```bash
sudo systemctl enable --now cockpit.service
```

## SecureBoot (optional, after confirming ucore is running)

```bash
sudo mokutil --import /etc/pki/akmods/certs/akmods-ublue.der
```
Set a password, reboot into MOK import, register the key, flip SecureBoot on in BIOS.

## Checks

| Check                   | Command                                                                                                                        |
| :---------------------- | :----------------------------------------------------------------------------------------------------------------------------- |
| Current deployment      | `rpm-ostree status`                                                                                                            |
| Hostname applied        | `hostnamectl status --static`                                                                                                  |
| Rebase service disabled | `systemctl is-enabled ucore-signed-autorebase.service` otherwise `sudo systemctl enable --now ucore-signed-autorebase.service` |
| Cockpit listening       | `sudo ss -tulnp \| grep 9090`                                                                                                  |
| Tailscale up            | `tailscale status`                                                                                                             |
| SSH access              | `ssh my-pc@<tailscale-ip>`                                                                                                     |
| /usr read-only          | `mount \| grep ' /usr '` → should show `ro`                                                                                    |

## Service commands

| Action               | Command                                         |
| :------------------- | :---------------------------------------------- |
| Deployed OS/image    | `rpm-ostree status`                             |
| Rollback             | `sudo rpm-ostree rollback`                      |
| Layer a package      | `sudo rpm-ostree install <package>`             |
| Enable+start service | `sudo systemctl enable --now <service>.service` |
| Stop / restart       | `sudo systemctl stop/restart <service>.service` |
| Live logs            | `journalctl -u <service>.service -f`            |
| Boot logs            | `journalctl -b`                                 |
| Firewalld zones      | `sudo firewall-cmd --get-active-zones`          |
| Reload firewalld     | `sudo firewall-cmd --reload`                    |

## Tailscale

```bash
sudo systemctl enable --now tailscaled
sudo tailscale up
```
Once `tailscale0` is up, lock SSH/Cockpit/admin stuff to that interface only via firewalld.

## Podman rootless + Quadlet

```ini
# ~/.config/containers/systemd/myapp.container
[Container]
Image=docker.io/library/myapp:latest
PublishPort=8080:8080

[Service]
Restart=always

[Install]
WantedBy=default.target
```

```bash
systemctl --user daemon-reload
systemctl --user enable --now myapp.service
```

Enable linger so containers survive logout:
```bash
loginctl enable-linger $USER
```

## Firewalld

Don't forget always to check your rules!

## Access map

Cockpit has no key/2FA login, PAM password only, keep it Tailscale-only unless there's a reverse proxy with real auth in front.

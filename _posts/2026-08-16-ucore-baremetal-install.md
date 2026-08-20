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
```bash
# Let it get the image and start the container 
Trying to pull quay.io/coreos/mkpasswd:latest...
Getting image source signatures
Copying blob a07a6b06265a done   | 
Copying blob 87ee49847e03 done   | 
Copying config cf86c76b05 done   | 
Writing manifest to image destination
Password: # Type your super secure password and hit Enter
```
```bash
dfasdFAESFASDfasdFQ@#W$!@F@#Fasdf2#F@#FASDF@34234 # Your hash (example)!
```

## Write the Butane config

Based on the [ucore-autorebase example](https://github.com/ublue-os/ucore/blob/main/examples/ucore-autorebase.butane):

In your working directory

```bash
nano ucore-autorebase.bu
```
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

storage:
  directories:
    - path: /etc/ucore-autorebase
      mode: 0754
  files:
    - path: /etc/hostname
      mode: 0644
      overwrite: true
      contents:
        inline: my-new-hostname

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

**Save the file!**

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

On your bare metal device go to your bios and disable Secure Boot, UEFI mode on.

## Boot live ISO, install

Check the disk were you will install it first! my case was `nvme0n1` make sure you don't have important information in that disk as it will be wiped. Follow the 3 BBB's rule before doing this: Backup, backup and backup!

```bash
# Check your current devices
lsblk
```
```bash
nvme0n1     259:0    0 119.2G  0 disk # I used this disk to install the image.
```
```bash
sudo coreos-installer install /dev/nvme0n1 \
  --ignition-file /path/to/ucore-autorebase.ign
```

## Reboot, let it rebase

Once the installation is finished simply run:

```bash
sudo systemctl reboot
```

Don't power-cycle manually, it reboots itself twice (unsigned rebase, then signed).

## SecureBoot (optional, after confirming ucore is running)

```bash
sudo mokutil --import /etc/pki/akmods/certs/akmods-ublue.der
```
Set a password, reboot into MOK import, register the key, flip SecureBoot on in BIOS.

## Enable services post-rebase

Nothing's on by default:

### TuneD

```bash
sudo systemctl enable --now tuned
tuned-adm active
```

See available profiles

```bash
tuned-adm list
```
```bash
Available profiles:
- accelerator-performance     - Throughput performance based tuning with disabled higher latency STOP states
- atomic-guest                - Optimize virtual guests based on the Atomic variant
- atomic-host                 - Optimize bare metal systems running the Atomic variant
- aws                         - Optimize for aws ec2 instances
- balanced                    - General non-specialized tuned profile
- balanced-battery            - Balanced profile biased towards power savings changes for battery
- desktop                     - Optimize for the desktop use-case
- hpc-compute                 - Optimize for HPC compute workloads
- intel-sst                   - Configure for Intel Speed Select Base Frequency
- latency-performance         - Optimize for deterministic performance at the cost of increased power consumption
- network-latency             - Optimize for deterministic performance at the cost of increased power consumption, focused on low latency network performance
- network-throughput          - Optimize for streaming network throughput, generally only necessary on older CPUs or 40G+ networks
- optimize-serial-console     - Optimize for serial console use.
- powersave                   - Optimize for low power consumption
- throughput-performance      - Broadly applicable tuning that provides excellent performance across a variety of common server workloads
- virtual-guest               - Optimize for running inside a virtual guest
- virtual-host                - Optimize for running KVM guests
Current active profile: balanced
```

Choose a prefered profile

```bash
sudo tuned-adm profile latency-performance
```

### Cockpit:
```bash
sudo systemctl enable --now cockpit.service
sudo systemctl start cockpit.service
```

### Tailscale:

```bash
sudo systemctl enable --now tailscaled.service
sudo tailscale up
```

When setting `--accept-routes` if you see: *Subnet routes and exit nodes may not work correctly.* see and apply [https://tailscale.com/s/ip-forwarding](https://tailscale.com/s/ip-forwarding)

When setting `--advertise-exit-nodes` if you see: *UDP GRO forwarding is suboptimally configured on some physical interfaces, UDP forwarding throughput capability will increase with a configuration change.* see and apply [https://tailscale.com/s/ethtool-config-udp-gro](https://tailscale.com/s/ethtool-config-udp-gro)

Once `tailscale0` is up, lock SSH/Cockpit/admin stuff to that interface only via firewalld.

After that one could use `tailscale serve` to get a signed url with the tailnet domain.

### Podman rootless + Quadlet:

```bash
sudo systemctl enable --now podman.socket
systemctl --user start podman.socket
```

In the uCore instructions it is said to run `systemctl --user enable podman-restart.service` to enable podman containers as services, but that documentation is describing the older, more manual podman-restart.service approach because it's a general-purpose fallback that works regardless of how you started your containers, but Quadlets are genuinely the better answer for almost everyone on uCore/CoreOS today, and it's worth understanding why the docs still lead with the older method.

Instead just create your services manually or use `podlet`

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

### Homebrew package manager

I normally use distroboxes for cli apps but in case it is needed install homebrew for linux here [https://brew.sh/](https://brew.sh/). To me, this is easier to run zellij, btop and others though it was recommended using distroboxes.

## Firewalld

Don't forget always to check your rules!

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
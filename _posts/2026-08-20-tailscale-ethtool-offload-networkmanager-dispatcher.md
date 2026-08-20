---
title: "Getting Tailscale's ethtool offload tweak to persist on Fedora uCore"
date: 2026-08-20
permalink: /posts/tailscale-ethtool-offload-networkmanager-dispatcher/
tags: [fedora, ucore, networkmanager, tailscale, selinux, homelab]
---

Tailscale's docs for subnet routers and exit nodes tell you to run an `ethtool` command to enable UDP GRO forwarding, then persist it with a `networkd-dispatcher` script. Problem: on Fedora uCore (and Kinoite, Silverblue, anything ostree-based really), there's no `networkd-dispatcher` at all. These systems run NetworkManager, not systemd-networkd, so the official instructions just don't apply. Here's what actually works.

# Why the official instructions don't apply here

`networkd-dispatcher` is a hook mechanism for `systemd-networkd`. uCore doesn't use `systemd-networkd`, it uses NetworkManager, same as regular Fedora Workstation. The good news is NetworkManager has had its own equivalent dispatcher mechanism for years, it's already enabled by default, and you don't need to layer any extra package to use it.

# Setting up the NetworkManager dispatcher script

NetworkManager runs scripts from `/etc/NetworkManager/dispatcher.d/` in alphabetical order whenever an interface changes state (up, down, connectivity change, etc). Since `/etc` is writable and survives `rpm-ostree` deployments, the script lives there permanently, same as your `/etc/NetworkManager/system-connections/` files.

Create the script:

```bash
sudo mkdir -p /etc/NetworkManager/dispatcher.d
sudo tee /etc/NetworkManager/dispatcher.d/50-tailscale <<'EOF'
#!/bin/sh

if [ "$2" = "up" ] || [ "$2" = "connectivity-change" ]; then
    NETDEV=$(ip -o route get 8.8.8.8 | cut -f 5 -d " ")
    ethtool -K "$NETDEV" rx-udp-gro-forwarding on rx-gro-list off
fi
EOF
```

Then fix ownership and permissions. NetworkManager silently refuses to run dispatcher scripts that are group or world writable, or not owned by root, so this isn't optional:

```bash
sudo chown root:root /etc/NetworkManager/dispatcher.d/50-tailscale
sudo chmod 0755 /etc/NetworkManager/dispatcher.d/50-tailscale
sudo restorecon /etc/NetworkManager/dispatcher.d/50-tailscale
```

That last `restorecon` call matters more than it looks. SELinux expects a specific context on dispatcher scripts, and if it's wrong the script just won't run, with no error telling you why.

# Testing it

Run it manually first before trusting the automatic trigger:

```bash
sudo /etc/NetworkManager/dispatcher.d/50-tailscale
echo $?
```

> 0

A `0` means it ran clean. If `ethtool` isn't installed (uCore's base image is minimal, it's not guaranteed to be there), you'll get a command not found error instead. Check with `which ethtool`; if it's missing, layer it with `rpm-ostree install ethtool` and reboot, or run it from a toolbox container if you'd rather not layer packages on the host.

Next, confirm the dispatcher service itself is alive:

```bash
systemctl status NetworkManager-dispatcher
```

Don't panic if this shows `inactive (dead)`. It's D-Bus activated, meaning it spawns on demand when NetworkManager fires an event, does its thing, then exits. That's normal, not broken. If it's masked or disabled outright though, turn it on with `sudo systemctl enable --now NetworkManager-dispatcher.service`.

To confirm it actually fires and applies the setting, bounce the relevant interface and check the flags:

```bash
nmcli connection down enp1s0 && nmcli connection up enp1s0
ethtool -k "$(ip -o route get 8.8.8.8 | cut -f 5 -d ' ')" | grep -E "rx-udp-gro-forwarding|rx-gro-list"
```

> rx-udp-gro-forwarding: on
> rx-gro-list: off

If you don't see that, tail the dispatcher's log while you reconnect the interface, it'll tell you if the script is even being invoked:

```bash
journalctl -u NetworkManager-dispatcher -f
```

# Notes

- If you've got a dual-NIC box (internet-facing NIC plus a second one for LAN or a shared connection), double check `$NETDEV` resolves to the actual uplink interface, not the LAN-facing one. The `ip -o route get 8.8.8.8` trick handles this correctly on its own since it follows the real default route, but worth a sanity check the first time.
- Tailscale's upstream doc assumes a single-NIC box. On multi-NIC setups just make sure the dispatcher script is targeting the interface that's actually acting as your exit-node or subnet-router uplink.
- The permission and SELinux context steps aren't optional extras, skip either one and the script fails silently with no obvious error in the logs, which is a fun way to lose twenty minutes.
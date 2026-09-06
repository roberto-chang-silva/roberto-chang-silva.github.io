---
title: "Tailscale for people who just want their devices to talk to each other"
date: 2026-09-06
permalink: /posts/tailscale-basics/
tags:
  - tailscale
  - networking
  - ssh
  - self-hosting
  - wireguard
categories:
  - homelab-systems
excerpt: "Tailscale builds a private network across all your devices with no port forwarding, no dynamic DNS, and no pain. Here is how to set it up."
---

I have written about Tailscale before, mostly assuming you already know what it is and why it is useful. A few people asked me to back up and explain it from scratch, so here we go. This is the post I wish existed when I first set it up: no fluff, just the mental model and the steps.

# What problem does this solve?

Your devices live on different networks. Your laptop is on home Wi-Fi, your phone is on mobile data, your server is somewhere in a datacenter or a closet. Normally they cannot talk to each other without doing something annoying: opening firewall ports, setting up dynamic DNS, configuring a VPN from scratch, or just giving up and using a cloud relay for everything.

Tailscale sidesteps all of that. It builds a private virtual network (called a tailnet) where every device you add gets a stable IP in the `100.x.x.x` range. That address never changes and always works, regardless of where the device physically is. Your phone on LTE can SSH into your home server the same way it would if they were on the same LAN, because as far as Tailscale is concerned, they are.

Under the hood it runs on WireGuard, which is a modern, fast, and relatively simple VPN protocol. The part Tailscale adds on top is the coordination layer: key exchange, NAT traversal, device discovery. You just log in and it handles the rest. Traffic goes peer-to-peer between your devices when possible. Tailscale's servers are the matchmaker, not the middleman.

# Installing on two devices

The free tier covers up to 50 tagged devices (up to the date of this post), which is more than enough for personal use.

First, create an account at [tailscale.com](https://tailscale.com). You can sign in with Google, GitHub, or a few other identity providers. Yeah, I am not happy only a few services are allowed to sign in, mostly due to privacy concerns. There is no separate "Tailscale password"; your identity provider is your auth, and that's understandable, they don't want to have your access keys.

**On a Linux server**, the quickest path is the install script:

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```
Unless it is baked in your distro, say immutable distros that include Taislcale.

Then bring the interface up:

```bash
sudo tailscale up
```

It will print a URL. Open it in a browser, log in, and the device is authorized.

**On a second device** (another Linux box, a Mac, or Windows), the process is the same. On desktop operating systems there is also a GUI app you can download from the site or your system's package manager. Log in with the same account and both machines appear in your admin console at `login.tailscale.com/admin/machines`, each with their assigned `100.x.x.x` address.

# Confirming it works

From one device, ping the other using its Tailscale IP. You can find the address in the admin console or by running:

```bash
tailscale ip -4
```

> `100.73.42.11`

Then from the other machine:

```bash
ping 100.73.42.11
```

> ```
> PING 100.73.42.11 (100.73.42.11) 56(84) bytes of data.
> 64 bytes from 100.73.42.11: icmp_seq=1 ttl=64 time=4.21 ms
> ```

If you get a response, the tailnet is working.

# SSH from your phone

This is where it gets genuinely useful. Install the Tailscale app on your phone (available for both Android and iOS) and log in with the same account. Your phone joins the tailnet and gets its own `100.x.x.x` address.

Then install an SSH client. On Android, Termux or Termius works fine for free. On iOS, Termius or Blink are common choices (I had to search these as I don't use iOS devices).

Connect to your server using its Tailscale IP on port 22 as you normally would. No port forwarding on your router, no exposed SSH port on the public internet, no special config. It works because both devices are on the same private network now.

One step further: Tailscale has its own SSH feature that handles authentication through your Tailscale identity, so you do not need to manage SSH keys at all. Enable it on the server with:

```bash
sudo tailscale up --ssh
```

Then in the admin console, under the machine's settings, you can configure which users or devices are allowed to SSH into it. After that, connecting from any authorized device just works, no key setup required.

# A few settings worth knowing

**MagicDNS** lets you use hostnames instead of IP addresses. Instead of `ssh user@100.73.42.11` you can write `ssh user@myserver`. Enable it in the admin console under DNS settings. Tailscale assigns names based on the machine names you set.
**Key expiry** is on by default: devices are periodically asked to re-authenticate. For a server you manage and trust, you can disable expiry per-device in the admin console. For personal devices it is reasonable to leave it on.
**Exit nodes** are optional. Any device in your tailnet can be configured to route all outbound traffic from other devices through itself. This turns that device into a traditional VPN exit point. Useful if you want to route your phone's traffic through your home connection when you are away.
**ACLs** (access control lists) let you define which devices can reach which other devices and on which ports. The default policy allows everything between your own devices, which is fine for personal use. If you share your tailnet with others or want to lock things down, the admin console has a policy editor.

# Notes

- If `sudo tailscale up` gives you a "failed to connect to local Tailscale daemon" error, the service probably did not start automatically. Run `sudo systemctl enable --now tailscaled` and try again.
- Tailscale SSH and regular key-based SSH are not mutually exclusive. You can run both; Tailscale SSH just adds another way in.
- The `100.x.x.x` address space is CGNAT range (RFC 6598). It will not conflict with most home or office networks, but if your router happens to use that range for something, you will have a bad time. Worth checking.
- Everything here applies to the free personal plan. Teams and enterprise tiers add SSO, more granular ACLs, and other things you probably do not need yet.

---
title: "Exposing an SSH Server on a Non-Tailscale Machine via Tailscale Serve"
date: 2026-08-20
permalink: /posts/tailscale-service-ssh-proxy/
tags: [tailscale, networking, ssh, proxy]
---

I ran into a problem recently. I needed to SSH into a server sitting on my office's local network that cannot install Tailscale as it is a shared device in a small office, using my pc from outside. Fortunately, my workstation at the office right next to that server *does* have Tailscale installed. Instead of messing with complex router port forwarding or VPN gateways, I used Tailscale Services and a TCP forwarder to bridge these devices.

# Setting Up the Proxy

To route traffic from my laptop through the workstation and into the non-Tailscale server, I needed to configure both the Tailscale control panel and the workstation itself.

**Step 1:** I want to my Tailscale Admin Console and create a new Service. I named  it something descriptive like `svc:my-office-local-server` and assign the incoming port as `tcp:2222`.

**Step 2:** On my remote workstation (same physical network of the shared server), I ran the Tailscale serve command to forward incoming Tailnet traffic over the local physical network to the target machine.

```bash
sudo tailscale serve --service=svc:my-office-local-server --tcp=2222 tcp://192.168.1.50:22
```

Possibly you'll need to approve the connection in your Tailnet console.

# Testing It

Once the service is active and bound, I can attempt to connect from my remote pc or any other device on my Tailnet (if my ACLs allow it).

```bash
ssh username@my-office-local-server -p 22
```

> Last login: Thu Aug 20 16:15:02 2026 from 100.64.0.5
> username@office-server:~$

And as simple as that, there's no need for me to remember IPs or setting ssh-config files for remote connections on each personal machine.
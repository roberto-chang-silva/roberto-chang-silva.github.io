---
title: "Serving a Localhost-Only Port Through Tailscale via an SSH Tunnel"
date: 2026-08-31
permalink: /posts/tailscale-serve-ssh-tunnel-localhost-port/
tags:
  - tailscale
  - ssh
  - systemd
  - homelab
  - networking
categories:
  - workflows-code
excerpt: "How to expose a service bound to localhost on a remote server by chaining a persistent SSH tunnel into Tailscale Serve, plus the systemd unit to keep it alive."
---

Following up on the SSH-over-Tailscale-Serve setup from last time: I ran into a case where the service I wanted to reach wasn't bound to the server's LAN interface at all, just to `localhost` on the box itself. That's why my normal SSH command included `-L 8080:localhost:8080`, I was already tunneling into loopback because nothing outside that machine could see the port directly. Which meant `tailscale serve --tcp=8080 tcp://192.168.1.50:8080` wasn't going to work; the workstation had no way to reach a port that only exists on the server's own loopback.

# The problem

Two ways a service on a remote box can be listening:

- Bound to `0.0.0.0` or the LAN IP: reachable from anywhere on that network, including my Tailscale workstation.

- Bound to `127.0.0.1`: only reachable from that machine itself, hence the SSH `-L` forward in the first place.

Mine was the second case. So instead of pointing Tailscale Serve at the server directly, I needed the workstation to run its own tunnel into that loopback port, then serve *that* local copy.

# How to do it

**Step 1:** On the workstation (not my laptop), open a tunnel to the server's loopback port.

```bash
ssh -N -L 8080:localhost:8080 user@192.168.1.50 -p 22
```

**Step 2:** Still on the workstation, point Tailscale Serve at the tunnel's local end instead of the server's IP.

```bash
sudo tailscale serve --service=svc:remote-webapp --tcp=8080 tcp://localhost:8080
```

**Step 3:** Confirm the port shows up as forwarded.

```bash
tailscale serve status
```

Now the chain is: tailnet device to `remote-webapp.your-tailnet.ts.net:8080`, which hits the workstation's tailnet interface, which hits its own `localhost:8080`, which is the live end of an SSH tunnel into the server's loopback service.

# Making the tunnel survive reboots

Running that `ssh -L` command by hand is fine until the connection drops or the workstation reboots, then the whole chain is broken silently. A systemd user unit fixes that.

```ini
# ~/.config/systemd/user/remote-tunnel.service
[Unit]
Description=SSH tunnel to remote server (8080 -> localhost:8080)
After=network-online.target
Wants=network-online.target
StartLimitIntervalSec=300
StartLimitBurst=4

[Service]
Type=simple
ExecStart=/usr/bin/ssh -N -i /home/user/.ssh/some_key -o IdentitiesOnly=yes -o BatchMode=yes -o ConnectTimeout=10 -o ExitOnForwardFailure=yes -o ServerAliveInterval=30 -o ServerAliveCountMax=3 -o StrictHostKeyChecking=accept-new -L 8080:localhost:8080 -p 22 user@192.168.1.50
Restart=always
RestartSec=15

[Install]
WantedBy=default.target
```

A couple of flags are doing more than they look like. `ExitOnForwardFailure=yes` matters because without it, SSH can come up "successfully" even when the port bind failed, sitting there alive but useless, and `Restart=always` never triggers because systemd thinks it's fine. `ServerAliveInterval` and `ServerAliveCountMax` exist for the same reason: a dead connection (router blip, server reboot) can otherwise sit as a zombie process instead of exiting and letting systemd restart it.

Enable it as a user unit, since everything else on this workstation already runs rootless:

```bash
mkdir -p ~/.config/systemd/user
systemctl --user daemon-reload
systemctl --user enable --now remote-tunnel.service
```

# Testing it

Check the tunnel process is actually up:

```bash
systemctl --user status remote-tunnel.service
```

> Active: active (running) since ...
> Main PID: 48213 (ssh)

Then confirm the tailnet path end to end from another device on the tailnet:

```bash
curl remote-webapp.your-tailnet.ts.net:8080
```
If that returns the expected response, the whole chain (tailnet to workstation to loopback tunnel to server) is working.
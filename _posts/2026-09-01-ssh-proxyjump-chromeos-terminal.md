---
title: "Setting Up SSH ProxyJump in ChromeOS Terminal"
date: 2026-09-01
permalink: /posts/ssh-proxyjump-chromeos-terminal/
tags:
  - ssh
  - chromeos
  - networking
categories:
  - homelab-systems
excerpt: How to set up multi-hop SSH tunneling and port forwarding inside the sandboxed ChromeOS Terminal app without ProxyJump pipe errors.
---

I recently needed to connect to an internal target host through a bastion jump box from this Chromebook I bought. Naturally, I tried my usual SSH configuration with `ProxyJump`, but the sandboxed ChromeOS Secure Shell terminal immediately threw a fatal pipe communication error. After ruling out heavy alternatives like running a full Crostini Linux VM or Termux just for basic remote access, I put together a clean workaround using native OpenSSH directives in my local SSH configuration file.

# The problem with ProxyJump on ChromeOS

When you try running standard ProxyJump flags like `ssh -J` or setting the `ProxyJump` directive in `~/.ssh/config` inside ChromeOS Terminal, the connection drops instantly. The terminal client relies on an hterm WebAssembly sandbox that restricts standard operating system inter-process communication. Because `ProxyJump` attempts to spawn a local background process and create inter-process pipes, the sandbox blocks the call and returns an error.

```bash
ssh -J bastion-user@my-bastion remote-user@10.0.1.50
```

> Could not create pipes to communicate with the proxy: Function not implemented

Attempting to swap `ProxyJump` for `ProxyCommand ssh -W %h:%p` fails with the exact same message because it still relies on local process piping.

# How to do it?

The workaround is to instruct the jump host to allocate a pseudo-terminal and immediately execute a secondary SSH session to the target host. This handles the secondary hop on the remote server side, completely bypassing the local ChromeOS pipe limitation.

Edit your `~/.ssh/config` file and add entries for both your jump host and your target destination:

```ssh-config
# The Bastion / Jump Host
Host my-bastion
    HostName my-bastion
    User bastion-user
    ForwardAgent yes

# The Target Machine (internal private host)
Host remote-server
    HostName my-bastion
    User bastion-user
    ForwardAgent yes
    RequestTTY force
    RemoteCommand ssh remote-user@remote-server
```

If you also need to forward a local port from the target machine (for example, accessing a web application running on port 8080 of the internal host), chain `LocalForward` alongside `RemoteCommand`:

```ssh-config
Host remote-server
    HostName my-bastion
    User bastion-user
    ForwardAgent yes
    RequestTTY force
    LocalForward 8080 localhost:8080
    RemoteCommand ssh -L 8080:localhost:8080 remote-user@remote-server
```

# Testing it

With the configuration saved, trigger the connection using the target profile alias:

```bash
ssh remote-server
```

> Authenticating to my-bastion...
> remote-user@remote-server:~$

If you configured `remote-server-forward`, open a new browser tab on your Chromebook and navigate to `http://localhost:8080` to verify the forwarded service.

# Notes

* Setting `ForwardAgent yes` is required if your target machine relies on the private SSH key stored on your Chromebook.
* Using `RequestTTY force` replaces the interactive `-t` command line flag, which is mandatory when executing a nested shell command via `RemoteCommand`.
* If you prefer not to edit your configuration file, the single-command equivalent is `ssh -A -t bastion-user@my-bastion "ssh remote-user@remote-server"`.
* Installing full Crostini Linux or Termux also fixes this issue natively, but adds roughly 3 GB to 5 GB of disk overhead and extra battery draw.
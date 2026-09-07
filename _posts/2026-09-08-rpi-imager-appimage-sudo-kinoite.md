---
title: "Running the Raspberry Pi Imager AppImage as root on Fedora Kinoite"
date: 2026-09-08
permalink: /posts/rpi-imager-appimage-sudo-kinoite/
tags:
  - appimage
  - fedora-kinoite
  - raspberry-pi
  - wayland
  - pkexec
  - tailscale
categories:
  - homelab-systems
excerpt: "Getting Raspberry Pi Imager AppImage to run with sudo on Fedora Kinoite via pkexec, fixing Wayland environment passthrough so writting images and clickable links actually work."
---

I wanted to set up a Raspberry Pi at my relative's place to use as a remote bastion. The idea was simple: install Tailscale on the Pi, advertise it as an exit node, and route my traffic through it occasionally to watch Netflix without paying for a VPN subscription I'd barely use. Flashing the SSD should have been the boring part.

For whatever piece of software excecute in my machine, I prefer to run in this order of preference AppImage > Flatpak > Brew > Distrobox container > Podman container > None else (Yes, I don't like to modify my base image using `rpm-ostree`). I manage AppImages with Gearlever (the Flatpak), which handles the `.desktop` file generation nicely. But the auto-generated entry runs the imager as my regular user, and Raspberry Pi Imager needs root to write to block devices. So I had to edit the `.desktop` manually.

# Getting pkexec to work

The imager AppImage bundles its own Polkit policy, so `pkexec` works without any extra configuration. I confirmed this in the terminal first:

```bash
pkexec /home/user/AppImages/raspberry_pi_imager.appimage
```

It prompted for a password and opened. Good enough. I then edited the `.desktop` file at `~/.local/share/applications/` to update the `Exec` line:

```ini
Exec=pkexec /home/user/AppImages/raspberry_pi_imager.appimage %u
```

Then refreshed the database:

```bash
update-desktop-database ~/.local/share/applications/
```

The app launched with root privileges from the application menu. First problem solved.

# Why clickable links inside the app broke

Clicking any link inside the imager (like the connect.raspberrypi.com ones) did nothing. Running it from terminal showed the actual problem:

```
Sandbox: CanCreateUserNamespace() clone() failure: EPERM
Error: no DISPLAY environment variable specified
Opening URL: "https://connect.raspberrypi.com/imager/"
Started runuser xdg-open
```

When `pkexec` elevates to root, it drops your session environment. On Wayland there is no `DISPLAY` to pass, and `xdg-open` has no idea where to send the browser. The fix is to forward the relevant Wayland session variables explicitly:

```ini
Exec=pkexec env WAYLAND_DISPLAY=$WAYLAND_DISPLAY XDG_RUNTIME_DIR=/run/user/1000 DBUS_SESSION_BUS_ADDRESS=$DBUS_SESSION_BUS_ADDRESS /home/user/AppImages/raspberry_pi_imager.appimage %u
```

Replace `1000` with your actual UID if it differs (`id -u` to check). KDE on Kinoite does expand `$VARIABLES` in `Exec` lines at runtime, so no wrapper script is needed here.

# Testing it

Launch the imager from the app menu, then click any link inside it.

```bash
# From terminal to verify the full env is passed:
pkexec env WAYLAND_DISPLAY=$WAYLAND_DISPLAY XDG_RUNTIME_DIR=/run/user/1000 DBUS_SESSION_BUS_ADDRESS=$DBUS_SESSION_BUS_ADDRESS /home/user/AppImages/raspberry_pi_imager.appimage
```

> The app opens, prompts for your password via Polkit, and clicking links inside launches your browser correctly.

# The final .desktop

```ini
[Desktop Entry]
Type=Application
Version=1.5
Name=Raspberry Pi Imager
Name[zh_CN]=树莓派启动盘制作工具
Comment=Tool for writing images to SD cards for Raspberry Pi
Comment[zh_CN]=将镜像写入SD卡的树莓派工具
Icon=/home/user/AppImages/.icons/raspberry_pi_imager
TryExec=/home/user/AppImages/raspberry_pi_imager.appimage
Exec=pkexec env WAYLAND_DISPLAY=$WAYLAND_DISPLAY XDG_RUNTIME_DIR=/run/user/1000 DBUS_SESSION_BUS_ADDRESS=$DBUS_SESSION_BUS_ADDRESS /home/user/AppImages/raspberry_pi_imager.appimage %u
MimeType=x-scheme-handler/rpi-imager;application/vnd.raspberrypi.imager-manifest+json;
Categories=Utility;
StartupNotify=false
X-AppImage-Name=Raspberry Pi Imager
Path=/home/user/AppImages
```

Notes:

- Gearlever is useful for initial AppImage registration but its generated `.desktop` won't have the `pkexec` line. Edit manually after importing.
- Once the Pi is flashed and on the network, Tailscale exit node setup on Raspberry Pi OS is a separate thing entirely, but this was the annoying prerequisite.

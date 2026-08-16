---
title: 'Installing Microsoft fonts on a linux desktop distro (no root)'
date: 2026-08-16
permalink: /posts/linux-microsoft-fonts/
tags:
  - linux
---

Time ago I left behind Windows and replaced it with a Linux distro. Of course, they don't come with the Windows fonts. Easiest fix: drop the fonts into your user directory.

# How to do it?

## What you need

- A Windows machine with your own personal genuine license (to grab the fonts from)
- The font files copied over (`.ttf`, `.otf`, `.TTF`)

## Grab fonts from Windows

System fonts live in `C:\Windows\Fonts`. Copy them out via File Explorer to a USB stick or shared folder.

1. Open File Explorer, go to `C:\Windows\Fonts`
2. Search for the font name (e.g. `calibri`)
3. Select all matching files (bold, italic, etc.)
4. Copy to USB or shared folder

Fonts may have redistribution license restrictions, only use it on your own machines using your genuine Windows installation.

## Copy fonts into your Linux machine

Create the fonts directory

```bash
mkdir -p ~/.local/share/fonts/microsoft
```

Copy the fonts over

```bash
cp /path/to/source/*.ttf ~/.local/share/fonts/microsoft/
cp /path/to/source/*.TTF ~/.local/share/fonts/microsoft/
cp /path/to/source/*.otf ~/.local/share/fonts/microsoft/
```

Refresh font cache

```bash
fc-cache -fv
```

Verify

```bash
fc-list | grep -i calibri
fc-list | grep -i aptos
fc-list | grep -i arial
```

## Notes

Fonts in `~/.local/share/fonts` are picked up automatically by all Flatpak apps (LibreOffice, GIMP, etc.), no reboot needed. Just close and reopen any apps that were already running.

For system-wide fonts in a Fedora immutable distro (all users), use `rpm-ostree install` with a font RPM and reboot instead. Not recommended though unless it is strictly necessary to update your base image.

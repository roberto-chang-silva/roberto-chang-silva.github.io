---
title: 'Firefox/librewolf profile manager at start'
date: 2026-08-16
permalink: /posts/libreWolf-firefox-profile-selection-on-start/
tags:
  - Firefox
  - Librewolf
  - Flatpak
---

Firefox/librewolf has currently two ways to create profiles (per profile or objective, it's on you). I prefer the old way (small classic GUI to select a profile) because I can set it to open it on start prior opening a default profile. I like it since i can decide which profile should open a link I found on VSCode or a script or any other source or even to reject opening a link in case I just misclicked.

# How to do it?

## Base command (Flatpak)

```bash
flatpak run --branch=stable --arch=x86_64 --command=librewolf --file-forwarding io.gitlab.librewolf-community @@u %u @@
```

## With ProfileManager (profile selector on launch)

```bash
run --branch=stable --arch=x86_64 --command=librewolf --file-forwarding io.gitlab.librewolf-community --ProfileManager @@u %u @@
```

## With specific profile (no selector)

```bash
run --branch=stable --arch=x86_64 --command=librewolf --file-forwarding io.gitlab.librewolf-community -P ProfileName
```
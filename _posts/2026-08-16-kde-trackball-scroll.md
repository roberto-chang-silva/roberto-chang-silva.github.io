---
title: 'M575 Trackball: Scroll with the trackball instead of the wheel in KDE'
date: 2026-08-16
permalink: /posts/kde-trackball-scroll/
tags:
  - trackball
  - kde
---

Trackballs over mice for me (except for gaming), thumb trackballs specifically, I like those best. The one thing I need is scroll-via-ball. Had this working on Windows via AutoHotkey with a smooth scrolling [script](https://github.com/eynsai/Smooth-Trackball-Scrolling), but that doesn't carry over to Linux. Luckily KDE DE (Desktop Environment) has a core implementation for this. Here's how I set it up.

# How to do it?

## Hardware identification

Identify your device, in my case is the logitech 575

```bash
# Check Logitech ERGO M575
cat /proc/bus/input/devices | grep -A 10 -B 2 -i "M575"
```

Output:
```
Vendor=046d Product=4096  # unique VID:PID
Name="Logitech ERGO M575"
Handlers=event9 mouse1
```

## qdbus6 (Plasma Wayland)

```bash
# Find qdbus6
find /usr -name qdbus* 2>/dev/null
# → /usr/lib/qt6/bin/qdbus6 (Plasma 6)
```

Enable scroll with a custom button, went with the right button, tried middle click first and hated it, right felt more intuitive.

```bash
# Your M575 (event26 in my case)
# BTN_RIGHT

qdbus-qt6 org.kde.KWin /org/kde/KWin/InputDevice/event26 org.kde.KWin.InputDevice.scrollButton 273
```

Make it persistent at the user level:
```bash
kwriteconfig6 --file kcminputrc --group "Mouse" --key "ScrollButton" 273
```

## Extra notes

#TODO in my [fork](https://github.com/roberto-chang-silva/Smooth-Trackball-Scrolling) of AutoHotkey, added an update so another click (besides right button) raises/lowers volume, haven't gotten to porting that to Linux yet.
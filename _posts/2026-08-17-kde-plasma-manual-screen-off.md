---
title: 'KDE Plasma: Manual Screen Off Button'
date: 2026-08-17
permalink: /posts/kde-plasma-manual-screen-off/
tags:
  - fedora
  - kde
---

The other day (could be months, week, days ago) I was running some commands in my laptop and was unable to turn off the screen on demand so I can leave it unattended without burning the pixels of my screen. So this is a quick guide to create a dedicated menu item that instantly powers off the monitor (putting it into standby) while keeping background tasks running.

# The Screen Off Command

KDE uses `kscreen-doctor` to control display power states safely on both Wayland and X11:

```bash
kscreen-doctor --dpms off
```

This command tells the display server to turn off the monitor's signal without suspending the system, locking the session, or interrupting any running processes.

# Create a Custom Application Launcher

1. Open the `KDE Menu Editor`
2. Create a new item.
3. Set the **Name** field to something like `Turn off screen`.
4. Set the **Command** field to:
```bash
kscreen-doctor --dpms off
```
5. Assign an icon (a monitor/display icon) for quick visual recognition. Merely optional.
6. Save the entry.
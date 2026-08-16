---
title: 'Disable Wifi powersave mode in my laptop (old, seems to be fixed)'
date: 2026-08-17
permalink: /posts/disable-wifi-powersave-mode/
tags:
  - fedora
  - thinkpad
  - wifi
---

Wi-Fi **power-save** in my Thinkpad (fedora OS) makes the card enter low-power mode when idle, causing **high latency**, **jitter**, and **packet loss** on sensitive connections like Tailscale, gaming, or VoIP. This doesn't happen on Ethernet because it's wired and stable. This happens becasue as I normally work with my laptop remotely to a workstation so to save some energy and get 8hrs battery i ran it in powersave mode. Also the Wifi signal is interrupted after suspending the laptop in this mode and can't connect it without a reboot.

## Solution, create a rule
```bash
sudo tee /etc/NetworkManager/conf.d/00-wifi-powersave.conf << EOF > /dev/null
[connection]
wifi.powersave=2
EOF

sudo systemctl restart NetworkManager.service
```

## What does each line do?
| Line | Function |
|-------|---------|
| `sudo tee ... << EOF` | Writes root file with heredoc (avoids `Permission denied`) |
| `wifi.powersave=2` | **Disables** power-save globally (2=off, 3=on) |
| `restart NetworkManager` | Applies changes immediately |

## Verify it works
```bash
# 1. File created
cat /etc/NetworkManager/conf.d/00-wifi-powersave.conf

# 2. Power management OFF
iwconfig wlp2s0 | grep "Power Management"

# 3. Test Tailscale
tailscale ping <remote-IP>
```

```
# for Fedora
iw dev wlp2s0 get power_save
```

## `wifi.powersave` values
```
0 = Default (automatic)
1 = Don't touch
2 = OFF (this is what we want) ✅
3 = ON (avoid)
```

**Note**: Related to this [gist.github](https://gist.github.com/jcberthon/ea8cfe278998968ba7c5a95344bc8b55)

---
title: 'Reorder firefox/librewolf profiles in the profile manaer window'
date: 2026-08-16
permalink: /posts/reorder-firefox-librewolf-profiles/
tags:
  - Firefox
  - Librewolf
  - Flatpak
---

For daily usage, I prefer to use profiles to separate logins and cookies; one profile for work accounts, another for personal use, and a separate one for gaming.

The catch is that in Firefox and LibreWolf, once you create a profile using `about:profiles`, there is no built-in GUI way to change the order in which they appear.

As I mentioned in my previous post ([[libreWolf-firefox-profile-selection-on-start]]), I configure my browser to always open the profile selector by default. This way, whenever I click an external link, I can choose which profile handles it. However, the profile order varies across my devices despite having the identical profile names. Since there is no graphical way to fix this, here is how you can manually reorder them:

# How to proceed?

- First find the `profiles.ini` file
```bash
user@hostname:~$ find ~ -name "profiles.ini" 2>/dev/null

>>> /home/user/.local/share/Trash/files/io.gitlab.librewolf-community/.librewolf/profiles.ini
>>> /home/user/.var/app/org.mozilla.firefox/config/mozilla/firefox/profiles.ini
```

(The output might be different depending whether your browser installation is as Flatpak, AppImage or repo package.)

- Edit your `profiles.ini`
``` bash
user@hostname:~$ nano /home/user/.var/app/io.gitlab.librewolf-community/.librewolf/profiles.ini
```

(Output might change device by device)

```bash 
[InstallFSDFS]
Default=fasdf.default-default
Locked=1

[Profile0]
Name=Primary
IsRelative=1
Path=afsdfsd.default-default

[Profile1]
Name=Work
IsRelative=1
Path=fqlkwe.Work
StoreID=asdf
ShowSelector=0

[General]
StartWithLastProfile=0
Version=2

[Profile2]
Name=Secondary
IsRelative=1
Path=fsdfsd.Secondary
```

- Replace the number in `[Profile#]` with the position you might want that profile be.
```bash 
[InstallFSDFS]
Default=fasdf.default-default
Locked=1

[Profile1] # Now this profile goes second
Name=Primary
IsRelative=1
Path=afsdfsd.default-default

[Profile0] # Now this profile goes first
Name=Work
IsRelative=1
Path=fqlkwe.Work
StoreID=asdf
ShowSelector=0

[General]
StartWithLastProfile=0
Version=2

[Profile2]
Name=Secondary
IsRelative=1
Path=fsdfsd.Secondary
```

Thats it!
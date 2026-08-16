---
title: 'VSCodium Flatpak: Trash + KWallet fixes'
date: 2026-08-16
permalink: /posts/vscodium-flatpak-trash-kwallet/
tags:
  - VSCode
  - kde
---

Flatpak sandboxing breaks two things in VSCodium/VSCode on a Fedora Kinoite-based distro (at least in my case): trashbin doesn't work, and it won't talk to KWallet to store github credentials, so it either nags for passwords or stores tokens in plain text.

# How to do it?

## Fix trash

Electron tries to delete files directly instead of going through the portal. Force it to use gvfs:

```bash
flatpak override --user --filesystem=xdg-data/Trash com.vscodium.codium
flatpak override --user --env=ELECTRON_TRASH=gvfs-trash com.vscodium.codium
```

(Or same thing via Flatseal: Filesystem > Other files > `xdg-data/Trash`, then Environment > `ELECTRON_TRASH=gvfs-trash`.)

## Fix KWallet integration

In Flatseal, VSCodium > Session Bus > Talks, add:
- `org.kde.kwalletd5`
- `org.kde.KWallet`
- `org.kde.kwalletd6` (Plasma 6+)

Close and reopen VSCodium!

## Verify

```bash
flatpak override --user --show com.vscodium.codium
```

- Trash: delete a file in VSCodium, should just work, no permanent-delete prompt.
- KWallet: sign into GitHub, KWallet should intercept. If it keeps asking for a password, check `pam_kwallet` is installed and your KWallet password matches your login password.
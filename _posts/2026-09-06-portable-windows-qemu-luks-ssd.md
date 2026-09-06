---
title: "Portable Windows 10 VM on an Encrypted External SSD with QEMU"
date: 2026-09-04
permalink: /posts/portable-windows-qemu-luks-ssd/
tags:
- qemu
- windows
- luks
- btrfs
- virtualization
- linux
categories:
- homelab-systems
excerpt: "Setting up a portable encrypted Windows QEMU VM on an external SSD with LUKS2, btrfs, and a launch script that auto-detects KVM."
---

In the country I currently live, my bank has a lot of unattended frictions to access their services on Linux machines. To access the banking platforms it is required around 4 mini pieces of software for verification, signatures, etc, etc. Not in a "it's a bit janky" way, in a hard "this portal requires Windows" way. I didn't want to keep a Windows partition (dual boot) around just for that, so I decided to put a Windows 10 VM on an external SSD I can carry around and run on any Linux machine with QEMU installed. This is the full setup from wiping the drive to the daily-use launch script.

# Preparing the SSD

I wanted LUKS2 encryption on the drive in case I lose it, and btrfs inside for the `nodatacow` attribute that matters later for the qcow2 file.

From the command line (the GUI partition manager kept silently skipping the LUKS step):

```bash
sudo wipefs -a /dev/sdX
sudo parted /dev/sdX --script mklabel gpt mkpart primary 0% 100%
sudo cryptsetup luksFormat --type luks2 --label "MySSD" /dev/sdX1
sudo cryptsetup open /dev/sdX1 MySSD
sudo mkfs.btrfs -L "MySSD" /dev/mapper/MySSD
```

A few things worth knowing here. LUKS2 is the right choice over LUKS1 unless you need GRUB to unlock `/boot`, which you don't for a data drive. The Argon2id key derivation in LUKS2 is significantly more resistant to brute force. The label on the LUKS header and the btrfs label are independent. The btrfs label is what shows up as the mount point name in `/run/media/`.

On desktops with udisks2 and a graphical session (basically any distro with GNOME or KDE), plugging in the drive will prompt for the LUKS passphrase without asking for sudo. On a minimal system you will need to open it manually with `cryptsetup open`. The sudo prompt in the GUI on Fedora specifically is a polkit policy thing, fixable with a rule in `/etc/polkit-1/rules.d/` if you want, but on other distros it just works.

# Creating the VM directory

Once the drive is mounted at `/run/media/user/MySSD`:

```bash
mkdir -p /run/media/user/MySSD/qemu-win
chattr +C /run/media/user/MySSD/qemu-win/
```

The `chattr +C` disables copy-on-write on that directory at the btrfs level. This needs to happen before creating the qcow2 file inside it. The attribute does not apply retroactively to existing files, so order matters.

Verify it took:

```bash
lsattr -d /run/media/user/MySSD/qemu-win/
```

> `---------------C------ /run/media/user/MySSD/qemu-win/`

Then fix ownership so you can write to it without sudo:

```bash
sudo chown -R $USER:$USER /run/media/user/MySSD/
```

# Creating the disk image

```bash
qemu-img create -f qcow2 \
  -o preallocation=off,lazy_refcounts=on \
  /run/media/user/MySSD/qemu-win/windows.qcow2 \
  40G
```

Confirm it looks right:

```bash
qemu-img info /run/media/user/MySSD/qemu-win/windows.qcow2
```

> ```
> image: /run/media/user/MySSD/qemu-win/windows.qcow2
> file format: qcow2
> virtual size: 40 GiB (42949672960 bytes)
> disk size: 196 KiB
> ```

40 GiB virtual, under 200 KiB on disk. It grows as Windows writes to it.

# Installing Windows 10

I used the Windows 10 22H2 evaluation ISO from Microsoft (90 days, free, legal). Drop it in the same folder as the qcow2.

The installation command:

```bash
qemu-system-x86_64 \
  -machine type=q35,accel=kvm \
  -cpu qemu64 \
  -m 4096 \
  -drive file=/run/media/user/MySSD/qemu-win/windows.qcow2,format=qcow2,if=ide,cache=writeback \
  -drive file=/run/media/user/MySSD/qemu-win/Win10_22H2_English_x64v1.iso,media=cdrom,if=ide \
  -boot order=d \
  -vga std \
  -netdev user,id=net0 \
  -device rtl8139,netdev=net0 \
  -usb \
  -device usb-tablet \
  -rtc base=localtime \
  -display sdl
```

A few notes on the parameters:

| Parameter | Reason |
|---|---|
| `q35` | Faster with Windows 10 than i440fx, which has a known ~1m45s boot delay with IDE cdrom |
| `qemu64` | Generic virtual CPU compatible with any x86_64 host, Intel or AMD |
| `if=ide` | Native Windows drivers, no extras needed |
| `rtl8139` | Network card Windows recognizes without installing anything |
| `accel=kvm` | Hardware acceleration; see the launch script below for handling hosts without it |

During installation, when it asks for a product key, click "I don't have a product key". Select the Windows 10 you have license with.

Once installed, will quote a reddit user "let it update until Windows it's happy"

# Post-install cleanup

Before taking the template snapshot, inside Windows:

1. Open Run (`Win+R`), type `%temp%`, delete everything inside.
2. Open Run again, type `temp`, delete everything inside.
3. Shut down from the Start menu (do not close the QEMU window directly, that is equivalent to pulling the power cable).

Then from the host, take a snapshot of the clean state:

```bash
qemu-img snapshot -c "clean" /run/media/user/MySSD/qemu-win/windows.qcow2
```

And make a template copy:

```bash
cp /run/media/user/MySSD/qemu-win/windows.qcow2 \
   /run/media/user/MySSD/qemu-win/windows-template.qcow2
```

If the working copy gets corrupted or bloated, restore from the template with `cp` or roll back to the snapshot:

```bash
qemu-img snapshot -a "clean" /run/media/user/MySSD/qemu-win/windows.qcow2
```

# The daily launch script

Saved as `deploy-vm.sh` in the same folder as the qcow2, so the whole directory stays self-contained on the SSD:

```bash
#!/bin/bash
DISK="$(dirname "$0")/windows.qcow2"

if [ -e /dev/kvm ]; then
    ACCEL="kvm"
else
    ACCEL="tcg"
fi

qemu-system-x86_64 \
  -machine type=q35,accel=$ACCEL \
  -cpu qemu64 \
  -m 2048 \
  -drive file="$DISK",format=qcow2,if=ide,cache=writeback \
  -vga std \
  -netdev user,id=net0 \
  -device rtl8139,netdev=net0 \
  -usb \
  -device usb-tablet \
  -rtc base=localtime \
  -display sdl
```

```bash
chmod +x /run/media/user/MySSD/qemu-win/deploy-vm.sh
```

The script checks for `/dev/kvm` and falls back to TCG automatically. TCG is pure software emulation, noticeably slower, but the virtual hardware seen by Windows is identical either way since we are using `qemu64` rather than `-cpu host`. Moving the SSD to a different machine and running the script just works.

# What lives on the SSD

```
MySSD/
└── qemu-win/
    ├── windows.qcow2           # working image
    ├── windows-template.qcow2  # clean backup, never boot this directly
    ├── Win10_22H2_*.iso        # keep it in case you need to reinstall
    └── deploy-vm.sh             # run this
```

# Notes

- After the creation of your template, in a new copy you can install your required apps and use the VM.
- The evaluation license is 90 days. After it expires you reinstall from the ISO. The template snapshot does not help here since it is stored inside the qcow2 and the license timer is baked into the Windows install itself, or just get your license.
- To compact the image after heavy use: run `sdelete64.exe -z C:` inside Windows first (from Sysinternals), then `qemu-img convert -O qcow2 -c windows.qcow2 windows-compact.qcow2` from the host.
- If you plug the SSD into a machine that does not have `qemu-system-x86_64` installed, none of this works. That is the only real dependency.
```

---
title: 'GitHub multi-account in VSCode'
date: 2026-08-16
permalink: /posts/vscode-multi-profile-github-accounts/
tags:
  - linux
  - VScode
  - GitHub
---

I like to use separated GitHub accounts for work (research) and personal stuff (homelab). For dev I normally use VSCode, however it doesn't support multiple github accounts easily (theres a workaround out there in stackoverflow but it was difficult to me to decide which account was syncing my repos). Thankfully, VSCode is built using Electron and with some small changes I can get multi-profile VSCode with separated GitHub accounts. This is a simple guide to set up separate VS Code profiles with different GitHub accounts on Linux (like work vs. personal accounts) using the `--user-data-dir` flag and KDE Plasma application launchers.

# How to do it?
By using VS Code's built-in `--user-data-dir` flag, you can maintain completely separate environments for different tasks (e.g., **primary**, **work**, **personal**) while sharing a single underlying VS Code application installation.

**What gets separated:**
- GitHub account authentication & sessions
- Installed extensions and themes
- Editor settings and workspace history

## Create the destination directories

This is just a linux convention to keep files organized, though you could use a different path.

```bash
mkdir -p "$HOME/.local/share/vscode-profiles/"{work,personal}/data
```

## Create Your Profile Launchers

In KDE Plasma, you can create custom desktop entries or application menu items. 

1. Open your KDE Application Menu, right-click, and select Edit Application (or open KMenuEdit).
2. Create a new item (or duplicate the default VS Code entry) for each profile (e.g., VS Code - Work, VS Code - Personal).
3. In the Command / Arguments field, make sure to use your absolute home path (do not use $HOME inside .desktop files) and the %F argument:

```bash
%F --user-data-dir 'home/user/.local/share/vscode-profiles/work/data'
# or/and
%F --user-data-dir 'home/user/.local/share/vscode-profiles/personal/data'
```

(Remember to replace user with your actual Linux login username, and adjust work to match your profile name).

4. Save the entry. Your new isolated profile will now appear in your application launcher and can be pinned to your taskbar.

## Sign In to Your GitHub Accounts

Because each profile uses a completely clean data directory, you'll need to authenticate each GitHub account separately.

1. Launch your **Work** profile using its KDE launcher.
2. Click the Accounts icon (the person silhouette/gear) in the bottom-left corner.
3. Select **Sign in to Sync Settings...**, or click **GitHub**, and complete the browser authentication.
4. Close that window, launch your **Personal** profile, and repeat the same steps with your personal GitHub account.

### Caveat: the browser will try to redirect to the wrong VS Code profile

Since this setup isn't VS Code's default configuration, the OS doesn't know which profile initiated the sign-in. When you complete authentication in the browser, it will try to hand control back to your **default** VS Code installation (or whichever profile was opened first) instead of the profile you're actually trying to sign into.

To work around this and complete sign-in on the correct profile, follow this sequence:

1. Start the sign-in flow. The browser will attempt to redirect back into VS Code via a link, close that prompt and click **Cancel** on the floating notification in the bottom-right corner.
2. VS Code will then offer a second authentication method, click **Yes**. This will also try to redirect via a link, again, close it and click **Cancel**.
3. VS Code will now fall back to a third method: a manual verification code. Click **Yes** and follow the on-screen steps.
4. Copy the 8-character code shown, paste it into the browser when prompted, and confirm.

This forces VS Code to authenticate using the device code flow instead of the redirect, so the sign-in completes on the profile you actually launched, not the default one that happened to intercept the callback.

**Note:** or instead on doing this auth using ssh-keys but I will not cover this.
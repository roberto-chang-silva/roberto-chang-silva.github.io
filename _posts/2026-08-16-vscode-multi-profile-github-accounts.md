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
By using VS Code’s built-in `--user-data-dir` flag, you can maintain completely separate environments for different tasks (e.g., **primary**, **work**, **personal**) while sharing a single underlying VS Code application installation.

**What gets separated:**
- GitHub account authentication & sessions
- Installed extensions and themes
- Editor settings and workspace history

## Create Your Profile Launchers

In KDE Plasma, you can create custom desktop entries or application menu items. 

1. Open your KDE Application Menu, right-click, and select Edit Application (or open KMenuEdit).
2. Create a new item (or duplicate the default VS Code entry) for each profile (e.g., VS Code - Work, VS Code - Personal).
3. In the Command / Arguments field, make sure to use your absolute home path (do not use $HOME inside .desktop files) and the %F argument:

code --user-data-dir '/home/user/.local/share/vscode-profiles/work/data' %F

(Remember to replace YOUR_USERNAME with your actual Linux login username, and adjust work to match your profile name).

4. Save the entry. Your new isolated profile will now appear in your application launcher and can be pinned to your taskbar.

## Sign In to Your GitHub Accounts

Because each profile uses a completely clean data directory, you will need to authenticate your accounts individually:

1. Launch your Work profile using your new KDE launcher.
2. Click the Accounts icon (person silhouette/gear) in the bottom-left corner.
3. Select Sign in to Sync Settings... or click GitHub and complete the browser authentication.
4. Close that window, open your Personal profile, and repeat the process with your personal GitHub account.
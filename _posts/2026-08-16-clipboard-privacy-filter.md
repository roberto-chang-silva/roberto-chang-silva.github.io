---
title: 'Filtering clipboard data before pasting into LLMs'
date: 2026-08-16
permalink: /posts/clipboard-privacy-filter/
tags:
  - privacy
---

This is a way I found to "filter" your clipboard right before pasting it into an LLM chatbot (ChatGPT, Mistral, Gemini, etc.). Specifically for my case, I use KDE as a desktop environment, which happens to have an option to invoke actions on a clipboard element. I thought, why not create a filter to remove names, numbers, hostnames, IPs, usernames, emails, and other personal details, so the AI can still help me with my inquiries.

# How to do it?

## Creating the `sed` rules file

Create the file that will contain the replacement regular expressions:

```bash
touch ~/.privacy_action.sed
nano ~/.privacy_action.sed
```

Paste the following content (`sed` basic syntax, control characters escaped with `\`):

```bash
# --- EMAILS ---
s/[a-zA-Z0-9._%+-]\+@[a-zA-Z0-9.-]\+\.[a-zA-Z]\{2,\}/EMAIL_REDACTED/gI

# --- ORCID (dash-numeric, must be locked in before phone regex runs) ---
s/\b[0-9]\{4\}-[0-9]\{4\}-[0-9]\{4\}-[0-9]\{3\}[0-9X]\b/ORCID_REDACTED/g

# --- TAILSCALE IPs (dotted-numeric, must be locked in before phone regex runs) ---
s/\b100\.\(6[4-9]\|[7-9][0-9]\|1[01][0-9]\|12[0-7]\)\.[0-9]\{1,3\}\.[0-9]\{1,3\}\b/TAILSCALE_IP_REDACTED/g

# --- GENERIC IPv4 ---
s/\b\([0-9]\{1,3\}\.\)\{3\}[0-9]\{1,3\}\b/IP_REDACTED/g

# --- MAC ADDRESSES ---
s/\b\([0-9a-f]\{2\}[:-]\)\{5\}[0-9a-f]\{2\}\b/MAC_REDACTED/gI

# --- IPv6 ---
s/\b\([0-9a-f]\{1,4\}:\)\{3,7\}[0-9a-f]\{1,4\}\b/IPV6_REDACTED/gI
s/\b\([0-9a-f]\{1,4\}:\)\{1,7\}:\([0-9a-f]\{1,4\}:\)*[0-9a-f]\{0,4\}\b/IPV6_REDACTED/gI
s/::\([0-9a-f]\{1,4\}:\)\{1,7\}[0-9a-f]\{1,4\}\b/IPV6_REDACTED/gI
s/\(^\| \)::1\b/\1IPV6_REDACTED/g

# --- TAILSCALE DOMAINS ---
s/\b[a-zA-Z0-9_-]\+\.\(ts\|tailscale\)\.net\b/TAILSCALE_REDACTED/gI

# --- SOCIAL MEDIA / GITHUB ---
s/@[a-zA-Z0-9_]\{3,\}/@USER_REDACTED/g
s|github\.com/[a-zA-Z0-9_-]\+|github.com/USER_REDACTED|gI

# --- USERS AND HOSTNAMES ---
s/\b\(my-username\|root\|admin\)\b/USER_REDACTED/gI
s|/home/[a-zA-Z0-9_-]\+|/home/USER_REDACTED|g
s|/var/home/[a-zA-Z0-9_-]\+|/var/home/USER_REDACTED|g
s/\b\(fedora-bazzite\|mi-laptop\|router-openwrt\)\b/HOST_REDACTED/gI
s/\b[a-zA-Z0-9_-]\+@[a-zA-Z0-9_-]\+\b/USER@HOST_REDACTED/g

# --- PERSONAL NAMES ---
s/\b\(Stephen\|Rykard\|Cole\|Kleene\)\([ -]\(Stephen\|Rykard\|Cole\|Kleene\)\)\+\b/FULL_NAME/gI
s/\b\(Stephen\|Rykard\|Cole\|Kleene\)\b/NAME_REDACTED/gI
```

Here you I just need to update accordingly with my personal information.

## Configuration in KDE

1. Right-click the clipboard icon in the system tray and select **Configure Clipboard...** 
2. Go to the **Actions Configuration** tab.
3. Add a new action with the following parameters:

| Parameter                              | Value to enter                                                 |
| :------------------------------------- | :------------------------------------------------------------- |
| **Regular expression (Match Pattern)** | `.{4,}` *(Applies to any text of 4 characters or more)*        |
| **Description**                        | `Automatic Privacy Filter`                                     |
| **Command**                            | `echo -n "%s" \| sed -f ~/.privacy_action.sed`                 |
| **Output**                             | `Replace clipboard` or `Ignore` (according to your preference) |

# Testing and Verification

To verify that the script works, you can test the output with the clipboard icon in the system tray, copy your text, clic and select `Invoke action` :

**Clipboard content:**
```
Hi, I'm Stephen Rykard Cole Kleene, my ORCID is 0000-0000-0000-0000.
Email me at stephen.kleene@example.com.

My GitHub is github.com/stephenkleene and on Twitter I'm @kleene_dev.

SSH into my box: root@fedora-bazzite, home dir is /home/my-username.
Router is at 192.168.1.1, my Tailscale node is 100.200.300.400 
and reachable at mi-pc.tailscale.net.

MAC address of the NIC: 00:1a:2b:3c:4d:5e
IPv6 local: fe80::a1b2:3c4d:5e6f, loopback is ::1
```

**Expected output:**
> Hi, I'm FULL_NAME, my ORCID is ORCID_REDACTED.
> Email me at EMAIL_REDACTED.
>
> My GitHub is github.com/USER_REDACTED and on Twitter I'm @USER_REDACTED.
> 
> SSH into my box: USER@HOST_REDACTED, home dir is /home/USER_REDACTED.
> Router is at IP_REDACTED, my Tailscale node is TAILSCALE_IP_REDACTED 
> and reachable at TAILSCALE_REDACTED.
> 
> MAC address of the NIC: MAC_REDACTED
> IPv6 local: IPV6_REDACTED, loopback is IPV6_REDACTED
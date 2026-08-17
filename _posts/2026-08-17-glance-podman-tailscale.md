---
title: "A small Glance setup with Podman, feeds, and Tailscale. A personal feed!"
date: 2026-08-17
permalink: /posts/glance-podman-tailscale/
tags:
  - podman
  - glance
  - rss
  - youtube
  - tailscale
  - self-hosting
---

I wanted a lightweight dashboard for research, news, and YouTube feeds without having to look to all the bloat is out there in the internet. Glance fits nicely (at least for me): it is just a self-hosted dashboard, so the setup is mostly a container, a config directory, and accepting that RSS is still the least exciting but most reliable web API. I deploy this in a spare raspberry pi I had with all my devices using Tailscale personal VPN.

# Running Glance with Podman

Glance listens on port `8080` (as of today, at least) in its container. Binding that port only to loopback keeps it off the LAN; Tailscale Serve will provide the HTTPS endpoint for devices in the tailnet. Glance’s upstream image uses `/app/config` for its configuration files. [github](https://github.com/glanceapp/glance)

**Step 1:** Create a working directory and configuration directory.

```bash
mkdir -p ~/containers/glance/config
cd ~/containers/glance
```

**Step 2:** Create `compose.yml`.

```yaml
services:
  glance:
    container_name: glance
    image: docker.io/glanceapp/glance:latest
    restart: unless-stopped
    ports:
      - "127.0.0.1:8080:8080"
    volumes:
      - ./config:/app/config:Z
```

The `:Z` volume suffix relabels the bind-mounted directory for SELinux, which avoids the usual Fedora-family “permission denied” surprise when the container tries to read the config.

**Step 3:** Start it with Podman Compose.

```bash
podman compose up -d
```

> Container `glance` starts and port `127.0.0.1:8080` is published on the host.

# Adding feeds

Create `config/glance.yml` with one page and a couple of feed widgets. Glance’s feed widget accepts feed URLs, optional titles, and item limits. [raw.githubusercontent](https://raw.githubusercontent.com/glanceapp/glance/main/docs/configuration.md)

This is only an example because i will not share my own personal setup 

```yaml
pages:
  - name: Home
    columns:
      - size: small
        widgets:
          - type: rss
            title: Research and climate
            limit: 8
            feeds:
              - url: https://www.nature.com/nclimate.rss
                title: Nature Climate Change
              - url: https://agupubs.onlinelibrary.wiley.com/feed/19422466/most-recent
                title: AGU Earth and Space Science

      - size: full
        widgets:
          - type: rss
            title: Infrastructure
            limit: 8
            feeds:
              - url: https://www.redhat.com/en/rss/blog/channel/red-hat-enterprise-linux
                title: Red Hat Enterprise Linux
              - url: https://www.cncf.io/feed/
                title: CNCF

      - size: small
        widgets:
          - type: videos
            title: YouTube
            channels:
              - UCg6gPGh8HU2U01vaFCAsvmQ
              - UCR-DXc1voovS8nhAvccRZhg
```

The `videos` widget uses YouTube channel IDs rather than a normal channel URL. A practical way to find one is to open the channel’s page source or use YouTube’s channel URL after it resolves to a `/channel/UC...` address. Glance also supports a YouTube search URL pattern where a widget needs search-based discovery rather than a fixed channel. [raw.githubusercontent](https://raw.githubusercontent.com/glanceapp/glance/main/docs/configuration.md)

Restart the container after editing the config.

```bash
podman compose restart glance
```

# Testing it

First, verify that Glance is answering locally.

```bash
curl -I http://127.0.0.1:8080
```

> `HTTP/1.1 200 OK`

Then check the container’s parsed configuration and feed errors, if any.

# Serving it through Tailscale

Tailscale Serve can reverse-proxy a local HTTP service to an HTTPS URL that is reachable only inside the tailnet. It supports a local port or loopback URL as the proxy target. [tailscale](https://tailscale.com/docs/reference/examples/serve)

**Step 1:** Confirm that Tailscale is connected on the host.

```bash
tailscale status
```

> The current machine appears as a connected node in the tailnet.

**Step 2:** Point Tailscale Serve at Glance’s loopback-only port.

```bash
sudo tailscale serve http://127.0.0.1:8080
```

> Available within your tailnet:
>
> `https://<machine-name>.<tailnet-name>.ts.net`

**Step 3:** Inspect the active Serve configuration.

```bash
tailscale serve status
```

> The status should show HTTPS traffic being proxied to `http://127.0.0.1:8080`.

Open the displayed `https://<machine-name>.<tailnet-name>.ts.net` address from another Tailscale device. The dashboard should load over HTTPS without exposing port `8080` to the local network or public internet. [tailscale](https://tailscale.com/docs/features/tailscale-serve)

# Management commands

| Task | Command |
|---|---|
| Start Glance | `podman compose up -d` |
| Stop Glance | `podman compose down` |
| Restart after config edits | `podman compose restart glance` |
| Follow logs | `podman logs -f glance` |
| Check container state | `podman ps --filter name=glance` |
| Check Tailscale proxy | `tailscale serve status` |
| Disable Tailscale Serve | `sudo tailscale serve off` |

# Notes

- Keep the host binding as `127.0.0.1:8080:8080`; changing it to `8080:8080` makes Glance reachable directly on every network interface.
- If SELinux blocks the bind mount on Fedora, ensure the volume ends in `:Z`; this was the missing piece when a container could start but could not read its mounted config.
- `tailscale serve` is tailnet-only. Do not use `tailscale funnel` unless the goal is deliberate public exposure; Funnel is the internet-facing counterpart. [tailscale](https://tailscale.com/blog/reintroducing-serve-funnel)
- Feed availability is outside the container’s control. If one widget is empty, test its RSS URL directly before assuming Glance is broken.

That is enough dashboard infrastructure for one afternoon.
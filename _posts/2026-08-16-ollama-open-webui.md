---
title: 'Local AI: Ollama + Open WebUI (old)'
date: 2026-08-16
permalink: /posts/ollama-open-webui/
tags:
  - immutable distro
  - podman
  - container
---

Local, private AI setup, Ollama as the inference engine, Open WebUI as the frontend. Runs as Podman rootless via Quadlets/systemd, works well on atomic desktops like Fedora/Bluefin/Aurora/uCore/etc!. Though this is old, use llama.cpp instead!

# How to do it?

## Backend: Ollama (Quadlet)

`~/.config/containers/systemd/ollama.container`

```ini
[Unit]
Description=Ollama AI Engine
After=network-online.target tailscaled.service

[Container]
ContainerName=ollama
Image=docker.io/ollama/ollama:latest
# NVIDIA GPU support (needs nvidia-container-toolkit)
PodmanArgs=--gpus all
SecurityLabelDisable=true
PublishPort=11434:11434
Environment=OLLAMA_HOST=0.0.0.0
Volume=ollama_data:/root/.ollama:z

[Service]
Restart=always
RestartSec=15s

[Install]
WantedBy=default.target
```

## Frontend: Open WebUI (Quadlet)

`~/.config/containers/systemd/open-webui.container`

```ini
[Unit]
Description=Open WebUI Interface
After=ollama.service

[Container]
ContainerName=open-webui
Image=ghcr.io/open-webui/open-webui:main
AddHost=host.containers.internal:host-gateway
Environment=OLLAMA_BASE_URL=http://host.containers.internal:11434
PublishPort=127.0.0.1:3051:8080
Volume=open-webui_data:/app/backend/data:z

[Service]
Restart=always
RestartSec=15s

[Install]
WantedBy=default.target
```

After creating/editing `.container` files:
```bash
systemctl --user daemon-reload
systemctl --user start [name].service
```

## Verify

```bash
curl http://localhost:11434/api/tags
```

Pull a model:
```bash
podman exec -it ollama ollama run llama3
```

## Troubleshooting

| Symptom | Cause | Fix |
| :--- | :--- | :--- |
| Connection refused on WebUI | Ollama only listening on 127.0.0.1 | Set `Environment=OLLAMA_HOST=0.0.0.0` |
| GPU not detected | Missing NVIDIA hook/drivers | Install `nvidia-container-toolkit`, use `PodmanArgs=--gpus all` |
| Permission denied on volumes | SELinux on Fedora Atomic | Add `:z` suffix to volume declarations |
| Host unreachable | WebUI can't resolve host | `AddHost=host.containers.internal:host-gateway` |

Don't disable SELinux if containers fail writing to volumes, just make sure mounts have `:z` so Podman relabels permissions automatically.

## Service commands

| Action | Command |
| :--- | :--- |
| Start everything | `systemctl --user start ollama open-webui` |
| Stop everything | `systemctl --user stop ollama open-webui` |
| Logs (Ollama) | `journalctl --user -u ollama -f` |
| Logs (WebUI) | `journalctl --user -u open-webui -f` |
| Status | `systemctl --user status open-webui` |

## Access

Local:
- WebUI: `http://localhost:3051`
- Ollama API: `http://localhost:11434`

Via Tailscale, two options:
1. Direct, change to `PublishPort=3051:8080` (opens the port to the whole LAN, careful).
2. Tailscale Serve, safer, exposes only 3051:
```bash
tailscale serve http:3051 http://127.0.0.1:3051
```

## Extra notes

#TODO Also podman networks could be use to let container communicate each other.
# Architecture

This page describes how the homelab stack fits together — the hardware, software layers, networking topology, and how services relate to each other.

## Stack Overview

```
┌─────────────────────────────────────────────────────┐
│                   Bare Metal Host                   │
│                  (Proxmox VE)                       │
│                                                     │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────┐  │
│  │  NAS VM      │  │  Services VM │  │Compute VM│  │
│  │ (OMV / ZFS)  │  │  (Debian)    │  │          │  │
│  │              │  │              │  │Heavy jobs│  │
│  │  ZFS pool    │  │  Docker      │  │FAH (idle)│  │
│  │  NFS / SMB   │  │  ┌────────┐  │  │          │  │
│  └──────────────┘  │  │Nextcld │  │  └──────────┘  │
│                    │  │Gitea   │  │                 │
│                    │  │Jellyfin│  │                 │
│                    │  │Authentk│  │                 │
│                    │  │Grafana │  │                 │
│                    │  └────────┘  │                 │
│                    └──────────────┘                 │
└─────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────┐
│              Mini PCs (×N, bare metal)            │
│                                                  │
│  Joined via Tailscale — used as compute nodes    │
│  Parallel jobs via GNU Parallel over SSH         │
│  Folding@Home (nice -n 19) when idle             │
└──────────────────────────────────────────────────┘
```

## Layers

### 1. Hypervisor — Proxmox VE

Proxmox runs on the bare metal host and manages all virtual machines. It provides:

- VM isolation — each workload runs in its own VM
- Snapshots and backups at the VM level
- Web UI for managing resources, storage, and networking
- Support for both KVM VMs and LXC containers

See: [Proxmox setup](../linux-setup/proxmox.md)

### 2. Storage — ZFS / OpenMediaVault

A dedicated storage VM (or the host itself) runs ZFS for reliable local storage:

- ZFS provides checksumming, compression, and snapshots
- OpenMediaVault provides a web UI for managing shares (NFS, SMB)
- Other VMs mount NAS shares for persistent data (media, backups, etc.)

See: [ZFS](../linux-setup/zfs.md), [OpenMediaVault](../linux-setup/openmediavault.md)

### 3. Services VM — Debian + Docker

A general-purpose Debian VM runs all self-hosted services via Docker Compose:

- Each service is a Docker Compose stack with its own directory and `.env` file
- Volumes are either local (bind mounts) or mounted from the NAS
- A shared Docker network allows services to communicate internally

See: [Services setup](../services/setup.md)

### 4. Networking

```
Internet
    │
    ├── Cloudflare (DNS + proxy)
    │       │
    │       └── Cloudflare Tunnel ──► Services VM (public services)
    │
    └── Tailscale (mesh VPN)
            │
            └── Direct access to all VMs (private/admin access)
```

- **Cloudflare** manages the domain's DNS and acts as a CDN/proxy for public-facing services
- **Cloudflare Tunnel** exposes specific services to the internet without opening firewall ports
- **Tailscale** provides secure remote access to everything (including Proxmox, NAS, admin UIs) without exposing them publicly

See: [Domain Registration](../networking/domain-registration.md), [Cloudflare](../networking/cloudflare.md), [Cloudflare Tunnel](../networking/cloudflare-tunnel.md), [Tailscale](../networking/tailscale.md)

### 5. Authentication — Authentik

Authentik acts as a centralised identity provider (IdP) and SSO layer:

- Sits in front of services exposed via Cloudflare Tunnel
- Services delegate login to Authentik via OAuth2/OIDC or a forward auth proxy
- Single place to manage users, MFA, and access policies

See: [Authentik](../services/authentik.md)

### 6. Monitoring — Prometheus + Grafana

A Prometheus + Grafana stack collects and visualises metrics from all VMs and services:

- Node Exporter runs on each VM to expose host metrics
- cAdvisor exposes Docker container metrics
- Prometheus scrapes all exporters on a schedule
- Grafana queries Prometheus and displays dashboards

See: [Monitoring overview](../monitoring/monitoring.md), [Prometheus](../monitoring/prometheus.md), [Grafana](../monitoring/grafana.md)

### 7. Compute — Proxmox Server + Mini PCs

The homelab also serves as a small compute cluster:

- A dedicated compute VM on the Proxmox server handles resource-intensive jobs without contending with the services VM
- Mini PCs are joined to the Tailscale network and used as additional nodes for parallelisable workloads (via GNU Parallel over SSH)
- When no jobs are running, idle CPU cycles across all nodes are donated to Folding@Home — running at low process priority so it yields immediately when real work starts
- NFS shares from the NAS give all nodes a common filesystem for job inputs and outputs

See: [Compute overview](../compute/overview.md), [Benchmarking](../compute/benchmarking.md), [Folding@Home](../services/folding-at-home.md)

## How It All Connects

1. A public request hits the domain → Cloudflare proxies it → Cloudflare Tunnel delivers it to the services VM → Authentik validates the session → the target service handles it.
2. Admin/private access goes through Tailscale directly to any VM or mini PC — no public exposure.
3. All services store persistent data in Docker volumes or NAS mounts backed by ZFS.
4. Prometheus scrapes metrics from all layers; Grafana visualises them.
5. Compute jobs run on the Proxmox compute VM or mini PCs; job outputs land on NFS shares. When nodes are idle, Folding@Home runs in the background at low priority.

# homelab

A self-hosted homelab running on Proxmox, documented as both a personal reference and a guide others can follow.

Checkout this repo hosted on Gitea here: [gitea.example.com/trentzz/homelab](https://gitea.example.com/trentzz/homelab)

---

## Architecture

The homelab is built in layers:

```
Proxmox (bare metal hypervisor)
  └── VMs
        ├── NAS VM: ZFS + OpenMediaVault (storage)
        └── Services VM: Debian + Docker
              ├── Nextcloud, Gitea, Jellyfin, Folding@Home
              └── Authentik (SSO), Prometheus + Grafana (monitoring)

Mini PCs (×N, bare metal or lightweight VMs)
  └── Compute nodes for parallel/heavy workloads
        └── Folding@Home on idle compute

Networking
  ├── Cloudflare DNS + Cloudflare Tunnel  (public services)
  └── Tailscale mesh VPN                  (private/admin access)
```

- **Proxmox** runs everything in isolated VMs
- **ZFS / OpenMediaVault** provides reliable networked storage
- **Docker Compose** manages all services on a Debian VM
- **Cloudflare Tunnel** exposes public services without opening firewall ports
- **Authentik** provides SSO and MFA in front of exposed services
- **Tailscale** gives private access to everything else (Proxmox, admin UIs, SSH)
- **Prometheus + Grafana** monitor the whole stack
- **Mini PCs** extend the cluster for compute-heavy workloads; idle compute is donated via Folding@Home

For a deeper look at how the pieces connect, see [Architecture](docs/overview/architecture.md).

---

## Recommended Setup Order

If you're following this as a guide, here's the sequence that makes sense:

1. [Proxmox](docs/linux-setup/proxmox.md) — hypervisor on bare metal
2. [ZFS Storage](docs/linux-setup/zfs.md) + [OpenMediaVault](docs/linux-setup/openmediavault.md) — storage layer
3. Base VM — [Debian](docs/linux-setup/debian.md), [Fedora](docs/linux-setup/fedora.md), or [NixOS](docs/linux-setup/nix.md)
4. [Services setup](docs/services/setup.md) — Docker and Compose conventions
5. [Domain Registration](docs/networking/domain-registration.md) + [Cloudflare](docs/networking/cloudflare.md) — DNS
6. [Cloudflare Tunnel](docs/networking/cloudflare-tunnel.md) — public exposure without open ports
7. [Authentik](docs/services/authentik.md) — auth layer before deploying user-facing services
8. Core services — [Nextcloud](docs/services/nextcloud.md), [Gitea](docs/services/gitea.md), [Jellyfin](docs/services/jellyfin.md)
9. [Tailscale](docs/networking/tailscale.md) — private remote access
10. [Monitoring](docs/monitoring/monitoring.md) — Prometheus + Grafana
11. [Security hardening](docs/security/security.md) + [Backups](docs/security/backups.md)
12. [Compute setup](docs/compute/overview.md) — using the cluster for workloads, benchmarking nodes, Folding@Home on idle compute

See [Getting Started](docs/overview/getting-started.md) for more detail on each step.

---

## Table of Contents

### [Overview](docs/overview/)

- [Architecture](docs/overview/architecture.md)
- [Getting Started](docs/overview/getting-started.md)

### [Linux Setup](docs/linux-setup/)

- [Proxmox](docs/linux-setup/proxmox.md)
- [ZFS Storage](docs/linux-setup/zfs.md)
- [Openmediavault](docs/linux-setup/openmediavault.md)
- [Debian](docs/linux-setup/debian.md)
- [Fedora](docs/linux-setup/fedora.md)
- [Nix and NixOS](docs/linux-setup/nix.md)

### [Networking](docs/networking/)

- [Domain Registration](docs/networking/domain-registration.md)
- [Cloudflare](docs/networking/cloudflare.md)
- [Cloudflare Tunnel](docs/networking/cloudflare-tunnel.md)
- [Split Tunneling with a VPN](docs/networking/split-tunneling.md)
- [Tailscale](docs/networking/tailscale.md)

### [Services](docs/services/)

- [Setup](docs/services/setup.md)
- [Nextcloud](docs/services/nextcloud.md)
- [Gitea](docs/services/gitea.md)
- [Jellyfin](docs/services/jellyfin.md)
- [Authentik](docs/services/authentik.md)
- [Ansible](docs/services/ansible.md)
- [Folding@Home](docs/services/folding-at-home.md)

### [Monitoring](docs/monitoring/)

- [Overview](docs/monitoring/monitoring.md)
- [Beszel](docs/monitoring/beszel.md)
- [Prometheus](docs/monitoring/prometheus.md)
- [Grafana](docs/monitoring/grafana.md)

### [Compute](docs/compute/)

- [Overview](docs/compute/overview.md)
- [Benchmarking](docs/compute/benchmarking.md)

### [Security](docs/security/)

- [Security Hardening](docs/security/security.md)
- [Backups](docs/security/backups.md)

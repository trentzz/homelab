# Getting Started

This page describes the recommended order to set up the homelab from scratch. Each step links to the relevant guide. Feel free to skip sections that don't apply to your setup.

## Recommended Setup Order

### 1. Install Proxmox

Set up the hypervisor on bare metal first — everything else runs on top of it.

→ [Proxmox](../linux-setup/proxmox.md)

### 2. Set up Storage

Configure ZFS on the host or a dedicated VM, then install OpenMediaVault for a web-managed NAS.

→ [ZFS Storage](../linux-setup/zfs.md)
→ [OpenMediaVault](../linux-setup/openmediavault.md)

### 3. Provision a Base VM

Create the VM that will run your services. Debian is a stable choice; Fedora and NixOS are covered if you prefer those.

→ [Debian](../linux-setup/debian.md)
→ [Fedora](../linux-setup/fedora.md)
→ [NixOS](../linux-setup/nix.md)

### 4. Install Docker and configure the services environment

Set up Docker and Docker Compose on the services VM, establish the directory layout and naming conventions for stacks.

→ [Services setup](../services/setup.md)

### 5. Register a domain and configure Cloudflare

Buy a domain and point its nameservers to Cloudflare. This unlocks DNS management, proxying, and tunnels.

→ [Domain Registration](../networking/domain-registration.md)
→ [Cloudflare](../networking/cloudflare.md)

### 6. Set up Cloudflare Tunnel

Expose services to the internet without opening firewall ports. Requires a domain on Cloudflare.

→ [Cloudflare Tunnel](../networking/cloudflare-tunnel.md)

### 7. Deploy Authentik

Set up the authentication layer before deploying user-facing services, so you can protect them from day one.

→ [Authentik](../services/authentik.md)

### 8. Deploy core services

With the networking and auth layers in place, deploy the services you actually want:

→ [Nextcloud](../services/nextcloud.md)
→ [Gitea](../services/gitea.md)
→ [Jellyfin](../services/jellyfin.md)
→ [Folding@Home](../services/folding-at-home.md)

### 9. Set up remote access with Tailscale

Add Tailscale to each VM for secure private access — admin UIs, SSH, and anything you don't want public.

→ [Tailscale](../networking/tailscale.md)
→ [Split Tunneling](../networking/split-tunneling.md)

### 10. Set up monitoring

Deploy Prometheus and Grafana to keep an eye on the health of the whole stack.

→ [Monitoring Overview](../monitoring/monitoring.md)
→ [Prometheus](../monitoring/prometheus.md)
→ [Grafana](../monitoring/grafana.md)

### 11. Automation with Ansible

Once the setup is working, use Ansible to codify it so you can reproduce or update it reliably.

→ [Ansible](../services/ansible.md)

### 12. Security and backups

Review the security hardening checklist and set up a backup strategy.

→ [Security](../security/security.md)
→ [Backups](../security/backups.md)

### 13. Compute and benchmarking

Configure the Proxmox server and mini PCs as a compute cluster. Benchmark each node so you know what you're working with. Set up Folding@Home to donate idle compute.

→ [Compute overview](../compute/overview.md)
→ [Benchmarking](../compute/benchmarking.md)
→ [Folding@Home](../services/folding-at-home.md)

## Architecture Reference

For a diagram of how all the pieces fit together, see the [Architecture page](architecture.md).

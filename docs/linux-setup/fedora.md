# Fedora

Fedora is a community-driven Linux distribution and the upstream source for Red Hat Enterprise Linux (RHEL). Fedora Server is purpose-built for server workloads and ships with more recent packages than Debian stable, including newer kernels and drivers.

## Advantages for Homelabs

- **Up-to-date packages** — newer kernels, drivers, and software than Debian stable. Better hardware support for recent systems.
- **Cockpit** — a web-based management console included by default. Manages services, storage, networking, and logs from a browser.
- **SELinux** — mandatory access control enabled by default, providing a stronger security baseline.
- **Podman** — pre-installed Docker-compatible container runtime with rootless container support and systemd integration.

## Installing Fedora Server

Download the Fedora Server ISO from [fedoraproject.org](https://fedoraproject.org/server/download) and boot it in a Proxmox VM. The Anaconda installer handles disk layout and user creation.

## Initial Setup

### Static IP

Fedora uses NetworkManager. Set a static IP with `nmcli`:

```bash
nmcli con show  # find interface name

sudo nmcli con mod ens18 ipv4.addresses 10.10.1.20/24
sudo nmcli con mod ens18 ipv4.gateway 10.10.1.1
sudo nmcli con mod ens18 ipv4.dns "10.10.1.1"
sudo nmcli con mod ens18 ipv4.method manual
sudo nmcli con up ens18
```

### SSH

```bash
sudo systemctl enable --now sshd
```

### Firewall

Fedora uses `firewalld`:

```bash
sudo firewall-cmd --state

sudo firewall-cmd --permanent --add-service=ssh
sudo firewall-cmd --permanent --add-port=8080/tcp
sudo firewall-cmd --reload

sudo firewall-cmd --list-all
```

## Package Management

Fedora uses `dnf`:

```bash
sudo dnf upgrade                # update all packages
sudo dnf install htop           # install a package
sudo dnf search nginx           # search packages
sudo dnf remove nginx           # remove a package
sudo dnf autoremove             # clean up unused dependencies
```

## Cockpit

Cockpit provides browser-based server management at `https://your-server-ip:9090`.

If not installed:

```bash
sudo dnf install cockpit
sudo systemctl enable --now cockpit.socket
sudo firewall-cmd --permanent --add-service=cockpit
sudo firewall-cmd --reload
```

Cockpit capabilities:
- CPU, memory, disk, and network monitoring
- systemd service management
- Log viewing and search
- Storage and network interface management
- User account management
- System updates

> [!NOTE] Cockpit is available on Debian/Ubuntu but has the best integration on Fedora Server.

## Podman

Fedora ships with Podman, a Docker-compatible container runtime. The CLI is identical to Docker:

```bash
podman pull nginx
podman run -d -p 8080:80 nginx
podman ps
```

Key differences from Docker:

- **Rootless** — containers run without root privileges by default
- **No daemon** — each container is a standalone process
- **Systemd integration** — generate service files directly from containers:

```bash
podman generate systemd --name my-container --files
sudo mv container-my-container.service /etc/systemd/system/
sudo systemctl enable --now container-my-container
```

- **Compose support** — use `podman-compose` for existing compose files:

```bash
sudo dnf install podman-compose
podman-compose up -d
```

To use Docker instead:

```bash
sudo dnf install dnf-plugins-core
sudo dnf config-manager --add-repo https://download.docker.com/linux/fedora/docker-ce.repo
sudo dnf install docker-ce docker-ce-cli containerd.io docker-compose-plugin
sudo systemctl enable --now docker
```

## Automatic Updates

Install `dnf-automatic` for unattended updates:

```bash
sudo dnf install dnf-automatic
```

Edit `/etc/dnf/automatic.conf`:

```ini
[commands]
apply_updates = yes
upgrade_type = security
```

Enable:

```bash
sudo systemctl enable --now dnf-automatic.timer
```

## Fedora CoreOS

[Fedora CoreOS](https://fedoraproject.org/coreos/) is a minimal, immutable OS designed exclusively for container workloads. It auto-updates and is configured via Ignition files at boot. Suitable for servers that run only Docker or Podman containers with no manual package management.

## Fedora vs Debian

| | Fedora | Debian |
|---|---|---|
| Package freshness | Cutting-edge | Conservative/stable |
| Support cycle | ~13 months per release | ~5 years (LTS) |
| Default firewall | firewalld | ufw |
| Containers | Podman (rootless) | Docker |
| Management UI | Cockpit (built-in) | Manual |
| SELinux | Enabled by default | Not included |
| Tutorial coverage | Moderate | Extensive |

Fedora suits homelab setups where recent hardware support, Cockpit, or Podman are priorities. Debian is the safer choice when long-term stability and broad community documentation matter more.

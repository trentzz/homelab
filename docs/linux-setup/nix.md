# Nix and NixOS

NixOS is a Linux distribution where the entire system configuration is declared in a single file (`configuration.nix`). Every change produces a new system generation that can be rolled back. The Nix package manager can also be installed standalone on other distros.

## Advantages for Homelabs

- **Declarative configuration** — packages, services, users, firewall rules, and networking are all defined in `configuration.nix`. The file is the complete record of what's on the system.
- **Atomic upgrades and rollbacks** — every `nixos-rebuild switch` creates a new generation. Previous generations remain bootable from the GRUB menu.
- **Reproducibility** — applying the same `configuration.nix` to a fresh install produces an identical system.
- **Large package repository** — nixpkgs contains over 100,000 packages.

## Installing NixOS

Download the NixOS ISO from [nixos.org](https://nixos.org/download/) and boot it in a VM. For a Proxmox server VM, the minimal ISO is sufficient:

```bash
# Partition and mount disk
sudo fdisk /dev/sda
sudo mkfs.ext4 /dev/sda1
sudo mount /dev/sda1 /mnt

# Generate initial config
sudo nixos-generate-config --root /mnt

# Edit config
sudo nano /mnt/etc/nixos/configuration.nix

# Install
sudo nixos-install
```

## Basic configuration.nix

```nix
{ config, pkgs, ... }:

{
  boot.loader.grub.enable = true;
  boot.loader.grub.device = "/dev/sda";

  networking.hostName = "homeserver";
  networking.networkmanager.enable = true;
  networking.firewall.allowedTCPPorts = [ 22 80 443 ];

  time.timeZone = "America/New_York";

  users.users.admin = {
    isNormalUser = true;
    extraGroups = [ "wheel" "docker" "networkmanager" ];
    openssh.authorizedKeys.keys = [
      "ssh-ed25519 AAAA... your-key-here"
    ];
  };

  environment.systemPackages = with pkgs; [
    vim git htop curl wget docker-compose
  ];

  services.openssh.enable = true;
  services.openssh.settings.PermitRootLogin = "no";

  virtualisation.docker.enable = true;

  nix.gc = {
    automatic = true;
    dates = "weekly";
    options = "--delete-older-than 30d";
  };

  system.stateVersion = "24.05";
}
```

Apply changes:

```bash
sudo nixos-rebuild switch
```

Adding a service is as simple as adding a line to the config. For example, to enable Nginx:

```nix
services.nginx.enable = true;
services.nginx.virtualHosts."homeserver.local" = {
  root = "/var/www/html";
};
```

## Rollbacks

Every rebuild creates a new generation. To roll back:

```bash
# List generations
sudo nix-env --list-generations --profile /nix/var/nix/profiles/system

# Roll back to previous generation
sudo nixos-rebuild switch --rollback
```

Previous generations are also selectable from the GRUB boot menu.

## Flakes

Flakes are the modern approach to Nix project management. They pin dependencies for reproducible builds and provide a standardized structure:

```nix
{
  description = "Homelab NixOS configuration";

  inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs/nixos-24.05";
  };

  outputs = { self, nixpkgs }: {
    nixosConfigurations.homeserver = nixpkgs.lib.nixosSystem {
      system = "x86_64-linux";
      modules = [ ./configuration.nix ];
    };
  };
}
```

Build with:

```bash
sudo nixos-rebuild switch --flake .#homeserver
```

## Using Nix Without NixOS

The Nix package manager can be installed on any Linux distro or macOS:

```bash
sh <(curl -L https://nixos.org/nix/install) --daemon
```

Install packages in isolation:

```bash
nix-env -iA nixpkgs.htop
nix-env -iA nixpkgs.neovim
```

Use `nix-shell` for temporary environments:

```bash
nix-shell -p python3 nodejs
```

## Tradeoffs

- **Learning curve** — the Nix language is functional and differs significantly from standard shell scripting or config management. Standard Linux workflows don't translate directly.
- **Filesystem layout** — config files are not in standard locations. Many guides and tutorials assume a traditional distro.
- **Documentation** — official docs are improving but can be incomplete; some configuration options require reading nixpkgs source or community forums.
- **Disk usage** — the Nix store grows over time. Periodic garbage collection is required.

## NixOS vs Other Distros

NixOS suits homelab setups where reproducibility and rollbacks are priorities, particularly when managing multiple similar machines. Debian or Fedora are more practical choices when broad tutorial coverage, familiarity, or minimal setup overhead matter more.

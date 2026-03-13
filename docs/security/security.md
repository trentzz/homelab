# Security Hardening

Baseline security practices for a self-hosted homelab. These aren't exhaustive, but cover the most impactful steps.

## SSH Hardening

### Disable password authentication

Edit `/etc/ssh/sshd_config`:

```
PasswordAuthentication no
ChallengeResponseAuthentication no
PermitRootLogin no
```

Restart SSH:

```bash
sudo systemctl restart sshd
```

Make sure your SSH key is in `~/.ssh/authorized_keys` before doing this, or you'll lock yourself out.

### Use ed25519 keys

Generate a key pair on your local machine if you don't have one:

```bash
ssh-keygen -t ed25519 -C "your-comment"
```

Copy the public key to the server:

```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@server
```

### Change the default SSH port (optional)

Not security through obscurity in isolation, but reduces noise in logs from automated scanners:

```
Port 2222
```

Update your firewall rules and SSH config (`~/.ssh/config`) accordingly.

## Fail2ban

Fail2ban bans IPs that repeatedly fail authentication.

```bash
sudo apt install fail2ban
```

Create a local override at `/etc/fail2ban/jail.local`:

```ini
[DEFAULT]
bantime = 1h
findtime = 10m
maxretry = 5

[sshd]
enabled = true
```

```bash
sudo systemctl enable --now fail2ban
```

## Firewall

### UFW (Debian/Ubuntu)

```bash
sudo apt install ufw
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow ssh
sudo ufw enable
```

Only open ports you actually need. For services behind Cloudflare Tunnel, you don't need to open HTTP/HTTPS on the public interface at all.

### Firewalld (Fedora/RHEL)

```bash
sudo firewall-cmd --permanent --set-default-zone=drop
sudo firewall-cmd --permanent --add-service=ssh
sudo firewall-cmd --reload
```

## Keeping Services Updated

### Debian/Ubuntu — unattended-upgrades

```bash
sudo apt install unattended-upgrades
sudo dpkg-reconfigure --priority=low unattended-upgrades
```

This installs security updates automatically. Review `/etc/apt/apt.conf.d/50unattended-upgrades` to configure email notifications and automatic reboots.

### Fedora — dnf-automatic

```bash
sudo dnf install dnf-automatic
```

Edit `/etc/dnf/automatic.conf` and set:

```ini
upgrade_type = security
apply_updates = yes
```

```bash
sudo systemctl enable --now dnf-automatic.timer
```

### Docker images

Docker containers don't update automatically. Use Watchtower to pull and restart updated images:

```yaml
services:
  watchtower:
    image: containrrr/watchtower
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    command: --schedule "0 0 4 * * *" --cleanup
```

Or update manually on a schedule:

```bash
docker compose pull && docker compose up -d
```

## Secrets Management

### Never commit secrets to git

Use `.env` files for secrets and add them to `.gitignore`:

```
# .gitignore
.env
*.env
```

Check for accidentally committed secrets with [git-secrets](https://github.com/awslabs/git-secrets) or [truffleHog](https://github.com/trufflesecurity/trufflehog).

### Ansible Vault

For secrets in Ansible playbooks, use Ansible Vault instead of plaintext variables:

```bash
# Encrypt a file
ansible-vault encrypt vars/secrets.yml

# Edit an encrypted file
ansible-vault edit vars/secrets.yml

# Run a playbook with vault
ansible-playbook site.yml --ask-vault-pass
```

Store the vault password in a password manager, not in the repo.

## Authentik as the Auth Layer

Any service exposed via Cloudflare Tunnel should be behind Authentik. This means:

- Even if a service has no built-in auth (e.g. a plain web UI), Authentik's forward auth proxy blocks unauthenticated access
- MFA is enforced at the Authentik level and applies to all protected services
- User access can be revoked in one place

See [Authentik setup](../services/authentik.md) for configuration details.

## Cloudflare Tunnel Security

Cloudflare Tunnel is the primary exposure point for public services. A few things to keep in mind:

- Only tunnel services that need to be public — keep admin UIs behind Tailscale instead
- Use Cloudflare Access policies in front of sensitive services for an extra layer
- Cloudflare's WAF (even on the free plan) blocks a lot of common attacks automatically

## Proxmox

- Change the default `root` password immediately after install
- Add a non-root user for day-to-day access
- Proxmox's web UI should not be exposed publicly — access it only over Tailscale
- Enable two-factor authentication in the Proxmox UI under `Datacenter → Permissions → Two Factor`

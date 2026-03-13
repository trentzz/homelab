# Tailscale

Tailscale is a mesh VPN built on WireGuard. It handles key management, NAT traversal, and device coordination automatically, giving every device on your tailnet a stable IP in the `100.x.x.x` range. No port forwarding or manual VPN configuration required.

## Installing Tailscale

On Debian/Ubuntu:

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

Or add the repo manually:

```bash
sudo apt install curl apt-transport-https
curl -fsSL https://pkgs.tailscale.com/stable/ubuntu/jammy.noarmor.gpg | sudo tee /usr/share/keyrings/tailscale-archive-keyring.gpg >/dev/null
curl -fsSL https://pkgs.tailscale.com/stable/ubuntu/jammy.tailscale-keyring.list | sudo tee /etc/apt/sources.list.d/tailscale.list
sudo apt update
sudo apt install tailscale
```

Bring it up:

```bash
sudo tailscale up
```

A login URL will be printed. Open it in a browser, authenticate, and the device is added to your tailnet. Repeat on every device you want connected.

## Subnet Routing

Subnet routing lets one server act as a gateway to your entire LAN, so you don't need Tailscale installed on every device. Enable IP forwarding first:

```bash
echo 'net.ipv4.ip_forward = 1' | sudo tee -a /etc/sysctl.d/99-tailscale.conf
echo 'net.ipv6.conf.all.forwarding = 1' | sudo tee -a /etc/sysctl.d/99-tailscale.conf
sudo sysctl -p /etc/sysctl.d/99-tailscale.conf
```

Advertise your LAN subnet:

```bash
sudo tailscale up --advertise-routes=10.10.1.0/24
```

Then approve the advertised routes in the Tailscale admin console. All tailnet devices can now reach your LAN — NFS shares, router admin, local services, etc.

## MagicDNS

MagicDNS resolves device hostnames across your tailnet without managing DNS manually. Enable it in the admin console under **DNS**. Devices become reachable by name:

```bash
ssh user@homeserver
```

A custom tailnet name (e.g. `homeserver.tail1234.ts.net`) is also assigned automatically.

## Tailscale SSH

Tailscale SSH handles authentication via your identity provider, removing the need to manage SSH keys.

Enable on a server:

```bash
sudo tailscale up --ssh
```

Define access in your ACL policy:

```json
{
  "ssh": [
    {
      "action": "accept",
      "src": ["autogroup:members"],
      "dst": ["autogroup:self"],
      "users": ["autogroup:nonroot"]
    }
  ]
}
```

## Access Control Lists (ACLs)

ACLs control which devices can communicate with which. Managed in the admin console under **Access Controls**.

```json
{
  "acls": [
    {
      "action": "accept",
      "src": ["group:admin"],
      "dst": ["*:*"]
    },
    {
      "action": "accept",
      "src": ["group:mobile"],
      "dst": ["homeserver:443", "homeserver:8096"]
    }
  ]
}
```

## Exit Nodes

An exit node routes all internet traffic from a client device through the homelab server.

On the server:

```bash
sudo tailscale up --advertise-exit-node
```

Approve it in the admin console, then on the client:

```bash
sudo tailscale up --exit-node=homeserver
```

## Headscale

[Headscale](https://github.com/juanfont/headscale) is a self-hosted, open-source implementation of the Tailscale control server. The official Tailscale clients work with it unchanged. Use it if you want full control over the coordination infrastructure.

> [!NOTE] The free Tailscale plan supports up to 100 devices and 3 users. Headscale is only necessary if self-hosted control is a requirement.

# Beszel

[Beszel](https://beszel.dev) is a lightweight self-hosted monitoring platform that sits between Uptime Kuma and a full Prometheus + Grafana stack. It gives you per-server dashboards, Docker container stats, historical graphs, and alerting — with a fraction of the setup complexity.

**GitHub**: https://github.com/henrygd/beszel (~20k stars, MIT licence)

## When to Use Beszel vs. Prometheus + Grafana

| | Beszel | Prometheus + Grafana |
|---|---|---|
| Setup complexity | Low — two services, minimal config | High — exporters, scrape configs, dashboards |
| Resource overhead | ~25–35 MB per agent | Higher (depends on scrape interval and retention) |
| Docker container stats | Built-in | Requires cAdvisor + configuration |
| Custom queries | No | Yes (PromQL) |
| Best for | Homelab, small clusters | Large/complex environments |

If you want visibility into CPU, memory, disk, network, and containers across all your nodes without a lot of configuration, Beszel is the right call. If you need custom alerting logic, long-term metrics correlation, or integration with many exporters, Prometheus + Grafana is more appropriate. They're not mutually exclusive — both can run side-by-side.

## Architecture

Beszel uses a hub-and-agent model:

- **Hub** — central web UI and database (built on PocketBase). Receives metrics, stores history, shows dashboards, and sends alerts. Runs on port `8090`.
- **Agent** — lightweight process on each monitored system. Collects metrics and reports them to the hub. Runs on port `45876`.

Two connection modes:

- **SSH (hub → agent)** — hub initiates an outbound SSH connection to the agent. The agent's SSH server only accepts the hub's ED25519 key. Use this when the hub can reach agents directly (e.g. same LAN or Tailscale).
- **WebSocket (agent → hub)** — agent initiates an HTTPS WebSocket connection to the hub. Use this when agents can reach the hub but not vice versa (e.g. agents behind NAT).

## What It Monitors

- CPU usage and load average
- Memory and swap
- Disk usage and I/O per partition
- Network bandwidth per interface
- Docker / Podman container CPU, memory, and network (per container, with history)
- System temperatures (via lm-sensors)
- GPU usage and power draw (Nvidia, AMD, Intel)
- S.M.A.R.T. disk health
- Battery / UPS status

## Installation

### Hub (Docker Compose)

```yaml
services:
  beszel:
    image: henrygd/beszel
    container_name: beszel
    restart: unless-stopped
    ports:
      - "8090:8090"
    volumes:
      - ./beszel-data:/beszel/data
    environment:
      - APP_URL=https://beszel.example.com   # set to your actual URL
```

```bash
docker compose up -d
```

Access the UI at `http://your-server:8090`. On first visit, create an admin account.

### Agent (Docker Compose)

The hub generates a pre-filled `docker-compose.yml` for each agent when you add a new system. The general form:

```yaml
services:
  beszel-agent:
    image: henrygd/beszel-agent
    container_name: beszel-agent
    restart: unless-stopped
    network_mode: host    # required for network interface stats
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro   # for container monitoring
    environment:
      - KEY=<paste-public-key-from-hub>
      # For WebSocket mode (agent initiates connection):
      # - TOKEN=<token-from-hub-settings>
      # - HUB_URL=https://beszel.example.com
```

> Host network mode is required for the agent to see network interface stats. This is normal and documented.

### Agent (binary, for systems without Docker)

```bash
curl -sL https://get.beszel.dev | bash
```

Or download a binary from the [releases page](https://github.com/henrygd/beszel/releases) and run it directly. Useful for mini PCs or VMs where you don't want to run Docker just for the agent.

## Adding Systems

1. In the hub UI, click **Add System**
2. Enter the hostname/IP and port (`45876` default)
3. The hub shows the public key to paste into the agent's `KEY` environment variable
4. Start the agent — it should appear as connected within a few seconds

## Alerting

Alerts are configured per system in the hub UI. Available alert types:

- CPU, memory, disk above a threshold
- Network bandwidth threshold
- Temperature threshold
- System down / not reporting

Notifications are sent via [Shoutrrr](https://containrrr.dev/shoutrrr/) URL schemas. Supported channels include: Discord, Slack, Telegram, Ntfy, Gotify, Pushover, email (SMTP), and webhooks, among others.

Example Ntfy URL: `ntfy://ntfy.example.com/beszel-alerts`
Example Discord URL: `discord://token@channel-id`

## Reverse Proxy

Beszel works behind a reverse proxy. Set `APP_URL` to the external URL and proxy to port `8090`.

Nginx example:

```nginx
location / {
    proxy_pass http://localhost:8090;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host $host;
}
```

The WebSocket upgrade headers are required if you're using WebSocket mode for agents.

## Integration with Tailscale

Beszel works well with Tailscale for multi-node setups:

- Run the hub on the Proxmox server or services VM
- Install the agent (binary or Docker) on each mini PC
- Use each machine's Tailscale hostname as the address in the hub — no port forwarding needed, no public exposure

This covers the whole cluster (Proxmox VMs + mini PCs) from a single hub with no additional networking setup.

## Alongside Prometheus + Grafana

Beszel and Prometheus + Grafana serve different purposes well and can run together:

- Beszel for quick at-a-glance node and container health across the cluster
- Prometheus + Grafana for custom dashboards, long-term retention, and PromQL-based alerting

If you're starting fresh and want monitoring up quickly, Beszel alone covers the most common needs. Add Prometheus + Grafana later if you need more.

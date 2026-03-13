# Monitoring

Monitoring gives you visibility into the health and resource usage of your servers and services.

## What to Monitor

- **Compute** — CPU usage per core, load averages, process count
- **Memory** — RAM usage, swap usage
- **Storage** — disk usage per mount, read/write throughput
- **Network** — ingress/egress per interface, packet loss
- **Containers** — per-container CPU and memory usage
- **Services** — uptime and response status

## Options

### Beszel (recommended starting point)

[Beszel](beszel.md) is a lightweight hub-and-agent monitoring tool. Install the hub once, deploy a small agent on each node, and get per-server dashboards with CPU, memory, disk, network, Docker container stats, temperatures, and GPU — all with built-in alerting and history. Much faster to get running than Prometheus + Grafana.

Good choice if you want visibility across the whole cluster without a lot of configuration.

### Prometheus + Grafana (full stack)

```
  Node Exporter        ← host metrics (CPU, RAM, disk, network)
  cAdvisor             ← Docker container metrics
       │
       │  scraped every 15s
       ▼
   Prometheus          ← stores time-series metrics
       │
       ▼
    Grafana            ← dashboards and alerts
```

- **[Prometheus](prometheus.md)** — set up first; Grafana depends on it as a data source
- **[Grafana](grafana.md)** — connects to Prometheus, import pre-built dashboards, configure alerts

More setup required, but gives you full PromQL query flexibility, custom dashboards, and better support for large or complex environments. The two can run alongside each other — Beszel for at-a-glance health, Prometheus + Grafana for deeper analysis.

## Pre-built Dashboards

| ID | Name | Description |
|----|------|-------------|
| 1860 | Node Exporter Full | Per-core CPU, memory, disk I/O, network |
| 193 | Docker / cAdvisor | Per-container resource usage |

## Alerting

Grafana supports alerting with notifications via email, Slack, Discord, Telegram, and others. Common alert conditions:

- Disk usage above 85%
- Server offline
- RAM sustained above 90%
- Container restart loop

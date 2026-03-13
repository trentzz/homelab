# Compute Overview

The homelab includes a compute cluster made up of a Proxmox server and several mini PCs, all accessible over Tailscale.

## Hardware

| Node | Role | Notes |
|------|------|-------|
| Proxmox server | Primary hypervisor and high-resource workloads | Runs VMs; can dedicate a VM or run jobs directly on the host |
| Mini PCs (×N) | Additional CPU/memory nodes | Bare metal or lightweight VMs; good for parallel jobs |

Mini PCs are joined to the Tailscale network and can be reached by hostname from any other node.

## Resource Allocation Strategy

The Proxmox server runs both persistent services (see [Services](../services/setup.md)) and on-demand compute. The key challenge is keeping services healthy while allowing compute jobs to use headroom.

**General approach:**

- Services VM: fixed CPU/memory allocation in Proxmox, never touched by compute jobs
- Compute VM (or bare metal): separate allocation that can be adjusted as needed
- Mini PCs: used as burst nodes for parallelisable workloads

**Proxmox CPU allocation tips:**

- Use CPU pinning to give compute VMs dedicated cores (avoids NUMA-related slowdowns for sensitive workloads)
- Leave at least 2 cores for the Proxmox host itself
- For memory-intensive jobs, configure memory ballooning off and set a fixed allocation to avoid swapping

## Running Compute Jobs

### Interactive jobs (screen/tmux)

For one-off or exploratory jobs, SSH in and use `tmux` or `screen` so the job survives disconnection:

```bash
tmux new -s myjob
# run your workload
# detach: Ctrl+b d
# reattach: tmux attach -t myjob
```

### Parallel jobs across nodes

GNU Parallel can distribute work across multiple machines over SSH:

```bash
# Run a command on a list of inputs across nodes
parallel --sshloginfile nodes.txt --transferfile input.txt \
  "myprogram {}" ::: input1 input2 input3
```

`nodes.txt` format:
```
user@node1
user@node2
user@minipc1
```

For jobs that need a shared filesystem, mount a NFS share from the NAS on all nodes.

### Containerised workloads

Docker is useful for reproducibility and dependency isolation. On any node:

```bash
docker run --rm -v /mnt/data:/data myimage myprogram --input /data/input --output /data/output
```

Resource limits prevent a runaway container from starving other processes:

```yaml
deploy:
  resources:
    limits:
      cpus: "8"
      memory: 16G
```

### Priority and niceness

When running jobs alongside services, lower the priority of compute processes so the OS scheduler favours service processes under contention:

```bash
nice -n 10 myprogram          # lower CPU priority
ionice -c 3 myprogram         # idle IO priority
nice -n 10 ionice -c 3 myprogram  # both
```

Or set niceness in a Docker Compose service:

```yaml
services:
  compute:
    image: myimage
    cpu_shares: 512   # default is 1024; lower = lower priority
```

## Idle Compute → Folding@Home

When no jobs are running, idle CPU (and GPU) cycles are donated to [Folding@Home](../services/folding-at-home.md).

The key is that FAH runs at low process priority so it yields immediately when a real job starts. It runs on:

- The Proxmox server (compute VM or host, depending on config)
- Mini PCs when idle

**Recommended setup:**

1. Run FAH via Docker with a CPU limit set to the number of cores you're comfortable donating
2. Set `POWER=full` — FAH will use what it's given, but the CPU limit caps it
3. Set the container's `cpu_shares` low so the OS always prefers your jobs over FAH
4. On mini PCs, run FAH natively with `nice -n 19` so it only runs when nothing else wants the CPU

```yaml
services:
  foldingathome:
    image: ghcr.io/foldingathome/fah-client:latest
    restart: unless-stopped
    cpu_shares: 256       # very low priority vs other containers
    ports:
      - "7396:7396"
    volumes:
      - ./fah-config:/config
    environment:
      - USER=your_username
      - TEAM=0
      - POWER=full        # let the cpu_shares limit handle priority
    deploy:
      resources:
        limits:
          cpus: "6"       # cap at N cores
          memory: 8G
```

See [Folding@Home](../services/folding-at-home.md) for full setup instructions.

## Storage for Compute

For jobs that produce large outputs, mount a NFS share from the NAS rather than writing to the local disk of a VM:

- NAS has larger capacity and is backed by ZFS (checksummed, snapshotted)
- Output is immediately accessible from other nodes
- Simplifies cleanup — just delete the share content

See [Services setup](../services/setup.md) for how NFS mounts are configured.

## Monitoring Compute Workloads

Use standard Linux tools to observe resource usage during jobs:

```bash
htop            # CPU and memory per process
iotop           # disk IO per process
nethogs         # network IO per process
nvtop           # GPU usage (if applicable)
```

Prometheus + Grafana (see [Monitoring](../monitoring/monitoring.md)) give a historical view of CPU, memory, and disk across all nodes — useful for spotting resource bottlenecks after the fact.

For GPU nodes, add the `nvidia_gpu_exporter` to Prometheus to track GPU utilisation over time.

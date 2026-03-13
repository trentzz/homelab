# Folding@Home

Folding@Home (FAH) is a distributed computing project that simulates protein folding to assist research into diseases such as Alzheimer's, cancer, and Parkinson's. Idle compute on homelab servers can be contributed as work units, which are processed and returned to the project.

## Installing on Linux

### Direct Install

On Debian/Ubuntu:

```bash
wget https://download.foldingathome.org/releases/public/fah-client/debian-10-64bit/release/fah-client_7.6.21_amd64.deb
sudo dpkg -i fah-client_7.6.21_amd64.deb
```

> [!NOTE] Check the Folding@Home website for the latest version — the URL above may be outdated.

Start the service:

```bash
sudo systemctl enable --now fah-client
```

### Docker

```yaml
services:
  foldingathome:
    image: ghcr.io/foldingathome/fah-client:latest
    container_name: foldingathome
    restart: unless-stopped
    ports:
      - "7396:7396"
    volumes:
      - ./fah-config:/config
    environment:
      - USER=your_username
      - TEAM=0
      - POWER=light
    deploy:
      resources:
        limits:
          cpus: "4"
          memory: 4G
```

```bash
docker compose up -d
```

## Web Interface

Access the web interface at `http://your-server-ip:7396` to view active work units, adjust power settings, and monitor progress. Overall contribution stats are available at `https://stats.foldingathome.org`.

## Resource Usage

The **power level** controls CPU usage:

- **Light** — minimal resources, folds only when idle
- **Medium** — moderate usage, allows other tasks to run normally
- **Full** — uses all available CPU cores

On a server running other services, `light` is recommended to avoid resource contention.

FAH runs at a low process priority (nice level) by default. Verify with:

```bash
ps -eo pid,ni,comm | grep fah
```

CPU limits can also be enforced via the `deploy.resources.limits` section in the Docker Compose config shown above.

### GPU Folding

GPU work units contribute significantly more than CPU ones. To enable GPU folding with an Nvidia GPU in Docker:

```yaml
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
```

## Teams

Teams pool contributions from multiple users. Enter a team number in the config or via the web interface. Team `0` means no team. Teams can be found on the Folding@Home website or created at `https://stats.foldingathome.org`.

## Thermals and Power

Monitor CPU temperatures with `lm-sensors`:

```bash
sudo apt install lm-sensors
sensors
```

To run FAH only at certain times (e.g. off-peak hours):

```bash
sudo systemctl stop fah-client   # stop
sudo systemctl start fah-client  # start
```

Use a systemd timer or cron job to automate this on a schedule.

## BOINC

[BOINC](https://boinc.berkeley.edu/) is an alternative distributed computing platform hosting multiple research projects including climate modeling, gravitational wave detection, and mathematics. Projects can be selected individually. BOINC and FAH can run simultaneously if resource limits are configured appropriately.

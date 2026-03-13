# Backups

A backup strategy you don't test is not a backup strategy. This page covers what to back up, how to do it, and how to verify it works.

## What to Back Up

| Data | Location | Priority |
|------|----------|----------|
| Docker volumes (app data) | `/var/lib/docker/volumes/` or bind mount paths | High |
| Docker Compose files and `.env` | e.g. `/opt/stacks/` | High |
| Databases (Postgres, MySQL, SQLite) | Dump separately — don't copy raw data files | High |
| NAS data (media, documents) | ZFS pool | Medium–High |
| Proxmox VM configs | `/etc/pve/` | Medium |
| System configs | `/etc/` key files | Low–Medium |

### Database dumps

Don't back up raw database files while the database is running — they may be inconsistent. Dump first:

**PostgreSQL:**
```bash
docker exec postgres pg_dumpall -U postgres > backup.sql
```

**MySQL/MariaDB:**
```bash
docker exec mysql mysqldump -u root -p"$MYSQL_ROOT_PASSWORD" --all-databases > backup.sql
```

**SQLite:**
```bash
sqlite3 /path/to/db.sqlite ".backup '/path/to/backup.sqlite'"
```

## Tool: restic

[restic](https://restic.net) is a fast, encrypted, deduplicated backup tool. It supports local, SFTP, S3, Backblaze B2, and other backends.

### Install

```bash
# Debian/Ubuntu
sudo apt install restic

# Fedora
sudo dnf install restic
```

### Initialise a repository

```bash
# Local
restic init --repo /mnt/backup/restic-repo

# Backblaze B2
restic -r b2:bucket-name:path init

# S3-compatible
restic -r s3:https://s3.example.com/bucket init
```

### Back up

```bash
restic -r /mnt/backup/restic-repo backup /opt/stacks /var/lib/docker/volumes
```

### Exclude files

```bash
restic -r /mnt/backup/restic-repo backup /opt/stacks \
  --exclude "*.log" \
  --exclude "node_modules"
```

### List snapshots

```bash
restic -r /mnt/backup/restic-repo snapshots
```

### Restore

```bash
# Restore latest snapshot to a directory
restic -r /mnt/backup/restic-repo restore latest --target /tmp/restore

# Restore a specific file
restic -r /mnt/backup/restic-repo restore latest --target /tmp/restore \
  --include /opt/stacks/nextcloud
```

### Prune old snapshots

```bash
restic -r /mnt/backup/restic-repo forget \
  --keep-daily 7 \
  --keep-weekly 4 \
  --keep-monthly 6 \
  --prune
```

## The 3-2-1 Rule

A widely-used baseline:

- **3** copies of the data
- **2** different storage media/types
- **1** offsite (cloud, remote server, or physically separate location)

For a homelab this might look like:

1. Live data on ZFS (local, checksummed)
2. restic backup to an external drive or second pool (local copy)
3. restic backup to Backblaze B2 or similar (offsite copy)

## Scheduling

### systemd timer (recommended on Debian/Fedora)

Create `/etc/systemd/system/restic-backup.service`:

```ini
[Unit]
Description=restic backup

[Service]
Type=oneshot
ExecStart=/usr/bin/restic -r /mnt/backup/restic-repo backup /opt/stacks
Environment=RESTIC_PASSWORD=your-password-here
```

Create `/etc/systemd/system/restic-backup.timer`:

```ini
[Unit]
Description=Run restic backup daily

[Timer]
OnCalendar=daily
Persistent=true

[Install]
WantedBy=timers.target
```

```bash
sudo systemctl enable --now restic-backup.timer
```

### cron

```cron
0 3 * * * restic -r /mnt/backup/restic-repo backup /opt/stacks >> /var/log/restic.log 2>&1
```

### Environment variables for passwords

Store the repository password and backend credentials in environment variables or a file rather than inline:

```bash
# /etc/restic-env (chmod 600)
export RESTIC_REPOSITORY=/mnt/backup/restic-repo
export RESTIC_PASSWORD=your-repo-password
export B2_ACCOUNT_ID=your-b2-id
export B2_ACCOUNT_KEY=your-b2-key
```

Source it in your backup script:

```bash
source /etc/restic-env
restic backup /opt/stacks
```

## Testing Restores

A backup you haven't tested is unverified. Do this regularly:

1. Pick a snapshot and restore it to a temporary directory
2. Check the files look correct
3. For databases, restore the dump to a test container and run a basic query
4. Document the last time you tested a restore

```bash
# Verify repository integrity
restic -r /mnt/backup/restic-repo check

# Test restore
restic -r /mnt/backup/restic-repo restore latest --target /tmp/test-restore
ls /tmp/test-restore
```

## Proxmox-level Backups

Proxmox has built-in VM backup (vzdump) which creates full or incremental backups of VMs and containers:

- Go to `Datacenter → Backup` to schedule VM backups
- Backups can be stored on a local disk, NFS share, or PBS (Proxmox Backup Server)
- VM backups complement application-level backups — they're useful for full disaster recovery

For important VMs, consider both: VM-level backups via Proxmox and application-level backups via restic.

# Hard Drive Storage (ZFS)

Now that we have proxmox setup, we can configure the hard drive setup for our NAS. I'm using 4x 12TB hard drives in a ZFS RAID5 (aka RAIDZ1) array, but any storage setup is fine. I would highly recommend using RAID though.

## Setting Up a ZFS RAID-Z1 (RAID5) Array on Proxmox VE

**1. Identify Your Disks**

Before proceeding, determine the device names of your 4x 12TB drives:

```bash
lsblk
```

You'll see a list of drives. If your drives are `/dev/sdb`, `/dev/sdc`, `/dev/sdd`, and `/dev/sde`, these will be used in the array.

**2. Wipe Existing Partitions (Optional but Recommended)**

If the drives were used previously, clean them:

```bash
for disk in /dev/sdb /dev/sdc /dev/sdd /dev/sde; do
    wipefs -a "$disk"
    dd if=/dev/zero of="$disk" bs=1M count=100
done
```

**3. Create the ZFS RAID-Z1 Pool**

Run the following command to create a RAID-Z1 pool named `tank`:

```bash
zpool create -f tank raidz1 /dev/sdb /dev/sdc /dev/sdd /dev/sde
```

**4. Verify the Pool**

Check the status of your new ZFS pool:

```bash
zpool status
```

**5. Enable Compression (Recommended)**

ZFS has built-in compression that improves performance and saves space:

```bash
zfs set compression=lz4 tank
```

**6. Enable Deduplication (Optional)**

Deduplication saves space but requires significant RAM:

```bash
zfs set dedup=on tank
```

**7. Enable Automatic Scrubbing**

To maintain data integrity, schedule a scrub:

```bash
echo "0 3 * * 0 root zpool scrub tank" >> /etc/crontab
```

This runs a scrub every Sunday at 3 AM.

**8. Mount the ZFS Pool**

By default, ZFS mounts pools under `/tank`. You can verify this with:

```bash
zfs list
```

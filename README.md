# homelab

Homelab Adventure! This is intended as a blog or guide, as well as a way to document this project!

If you would like to follow this as a guide, it's structured more or less in the order of what you should do first, but feel free to skip around to the sections you're most interested in.

Checkout this repo hosted on Gitea here: [gitea.trentz.me/trentzz/homelab](https://gitea.trentz.me/trentzz/homelab)

WIP at the moment. Stay tuned!

- [homelab](#homelab)
  - [Server hardware, VMs and NAS](#server-hardware-vms-and-nas)
    - [Proxmox](#proxmox)
    - [Hard drive storage](#hard-drive-storage)
    - [Openmediavault](#openmediavault)
    - [Debian](#debian)
  - [Initial networking setup](#initial-networking-setup)
    - [Domain registration](#domain-registration)
    - [Cloudflare](#cloudflare)
    - [Cloudflare tunnel](#cloudflare-tunnel)
  - [Services](#services)
    - [Nextcloud](#nextcloud)
    - [Gitea](#gitea)

## Server hardware, VMs and NAS

### Proxmox

#### What is proxmox? (from forum.proxmox)

Proxmox is a complete open source server virtualization management solution. Proxmox offers a web interface accessible after installation on your server which makes management easy, usually only needing a few clicks.

#### More resources

- [Proxmox Forum Tutorial](https://forum.proxmox.com/threads/proxmox-beginner-tutorial-how-to-set-up-your-first-virtual-machine-on-a-secondary-hard-disk.59559/)
- [XDA Developeres Proxmox Guide](https://www.xda-developers.com/proxmox-guide/)

### Hard drive storage

Now that we have proxmox setup, we can configure the hard drive setup for our NAS. I'm using 4x 12TB hard drives in a ZFS RAID5 array, but any storage setup is fine. I would highly recommend using RAID though.

#### Setting Up a ZFS RAID-Z1 (RAID5) Array on Proxmox VE

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

### Openmediavault

For most of the services we're going to setup, we want to have a dedicated or shared "data" folder that they use. For example, where all the nextcloud data goes, where your gitea remote repositories live, etc. There are a few ways to enable that, but here we're going to first make a NAS VM and use "shared folders" and nfs connections to our other services. This way, our storage is more or less managed in one central location.

### Debian

With our storage solution setup, we can make the VM we're going to use for the majority of the container based services we're running. For this I'm going to use debian since that's what I'm most familiar with, but feel free to use anything.

## Initial networking setup

### Domain registration

### Cloudflare

### Cloudflare tunnel

## Services

### Nextcloud

### Gitea

#### Container setup

#### Networking setup

#### Ssh setup

A bit annoying to setup. Since we're using cloudflared tunnel, we will use a new tunnel record for gitea ssh pointing to the port 22.

```bash
ssh giteassh.trentz.me
```

Then in our ssh config, we add the line:

```text
Host gitea.trentz.me
  HostName giteassh.trentz.me
  ProxyCommand /home/linuxbrew/.linuxbrew/bin/cloudflared access ssh --hostname %h
```

This tells our ssh that whenever we're trying to connect to `gitea.trentz.me` over ssh, we actually want to redirect to `giteassh.trentz.me` and we use a custom proxy command that routes the traffic through cloudflared. Here we just have to specify the executable path for cloudflared.

I installed it on ubuntu using homebrew, if you install it through other means the path will be different.

For windows it looks more like this:

```text
Host gitea.trentz.me
    HostName giteassh.trentz.me
    ProxyCommand & 'C:\Program Files (x86)\cloudflared\cloudflared.exe' access ssh --hostname %h
```

Assuming cloudflared is installed with winget, if you installed it through something else, the path could be different.

To find where the cloudflared executable lives, you can use:

```cmd
gcm cloudflared
```

And you should get a result like this:

```text
CommandType     Name                                               Version    Source
-----------     ----                                               -------    ------
Application     cloudflared.exe                                    0.0.0.0    C:\Program Files (x86)\cloudflared\cloudflared.exe
```

If you only want the path, you can use:

```text
(gcm cloudflared).path
```

#### Connecting multiple remotes to a git repo

With this self hosted gitea, you may want to start pushing to both something like github as well as your own gitea instance. To do this, you can use:

```bash
git remote add <remote-name> <remote-url>
```

For example:

```bash
git remote add gitea git@gitea.trentz.me:trentzz/homelab.git
```

Then when you want to push:

```bash
git push -u gitea main
```

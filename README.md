# homelab

Homelab Adventure! This is intended as a blog or guide, as well as a way to document this project!

If you would like to follow this as a guide, it's structured more or less in the order of what you should do first, but feel free to skip around to the sections you're most interested in.

Checkout this repo hosted on Gitea here: [gitea.example.com/trentzz/homelab](https://gitea.example.com/trentzz/homelab)

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
    - [Setting up your services VM](#setting-up-your-services-vm)
    - [Setting up a new service](#setting-up-a-new-service)
    - [Nextcloud](#nextcloud)
    - [Gitea](#gitea)
    - [Jellyfin](#jellyfin)
    - [Authentik](#authentik)
  - [Extras](#extras)
    - [Split tunneling with a VPN](#split-tunneling-with-a-vpn)

## Server hardware, VMs and NAS

### Proxmox

#### What is proxmox? (from forum.proxmox)

Proxmox is a complete open source server virtualization management solution. Proxmox offers a web interface accessible after installation on your server which makes management easy, usually only needing a few clicks.

#### More resources

- [Proxmox Forum Tutorial](https://forum.proxmox.com/threads/proxmox-beginner-tutorial-how-to-set-up-your-first-virtual-machine-on-a-secondary-hard-disk.59559/)
- [XDA Developeres Proxmox Guide](https://www.xda-developers.com/proxmox-guide/)

### Hard drive storage

Now that we have proxmox setup, we can configure the hard drive setup for our NAS. I'm using 4x 12TB hard drives in a ZFS RAID5 (aka RAIDZ1) array, but any storage setup is fine. I would highly recommend using RAID though.

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

#### Shared folders

A shared folder is simply a folder entity that can be used in other parts of OMV, e.g. through samba share or nfs shares. You can specify both the name and path of these shared files, and you can make use of the path to organise the shared folders in the underlying folder structure.

For example, if you want to have all your service (e.g. nextcloud, jellyfin) folders under the same parent `/SERVICES/JELLYFIN` folder, you can specify that in the `Relative path` field.

To make a new shared folder, navigate to `Storage > Shared Folders` and click the add button.

| Field         | Info                                                                                                                                                                                                                                                 |
| ------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Name          | This is the name that identifies the shared folder throughout OMV. This is also the name that you use to connect to the folder using nfs. E.g. if you name the folder `EXAMPLE-SHARE` then you would connect to it using `10.10.1.XX:/EXAMPLE-SHARE` |
| File system   | Which logical drive this folder should be.                                                                                                                                                                                                           |
| Relative path | The path of the folder. You can use this to organise the structure of your shared folders.                                                                                                                                                           |
| Permissions   | Who can read/write to this shared folder.                                                                                                                                                                                                            |

#### Shared folders with nfs

Navigate to `Services > NFS > Shares`. Click the add button, then specify the client IP address, and whether the client should have `read` or `read/write` access.

Make sure to save and update after creating this share.

#### Shared folders with multiple clients using nfs

To share a shared folder with multiple clients, simply make multiple entries in `Services > NFS > Shares`. These should then appear in your `/etc/exports` on the same line, e.g.

```bash
$ cat /etc/exports

/export/EXAMPLE-EXPORT 10.10.1.XX(xx,xx,xx,xx) 10.10.1.YY(yy, yy, yy, yy)
```

I would recommend checking this the first time you do this to make sure it got added correctly.

### Debian

With our storage solution setup, we can make the VM we're going to use for the majority of the container based services we're running. For this I'm going to use debian since that's what I'm most familiar with, but feel free to use anything.

#### Setting a static internal IP address

To organise your VMs a little on the networking side, I would recommend setting a static IP for your internal network. This also enables you to share nfs folders easily without having to worry about the client IP address changing after reboot.

You can do this using the GUI through: `Settings > Network > Gear icon > IPv4 > IPv4 Method Manual` and specifying your:

- Address, e.g. `10.10.1.XX`
- Netmask, e.g. `255.255.255.0`
- Gateway, e.g. `10.10.1.1`

Then click `Apply` and restart your connection. You can check that this worked by checking the output of:

```bash
ip a
```

#### SSH access

SSH access can be done simply for internal IP addresses by installing SSH server and opening the port 22 using `ufw`.

Check that the server has ssh:

```bash
sudo systemctl status ssh
```

Then install if missing:

```bash
sudo apt install openssh-server  # For Debian/Ubuntu
sudo systemctl enable --now ssh
```

Check if `ufw` is enabled on the server:

```bash
sudo ufw status
```

Allow SSH:

```bash
sudo ufw allow ssh
```

Open the port `22` explicitly:

```bash
sudo ufw allow 22/tcp
```

Alternatively, allow SSH from a specific IP range:

```bash
sudo ufw allow from 10.10.1.0/24 to any port 22 proto tcp
```

Change the subnet to match the range of IP addresses you want to be able to access your server.

Check everything so far:

```bash
sudo ufw reload
sudo ufw status verbose
```

Check the server's IP address:

```bash
ip a
```

And connect to the server:

```bash
ssh username@10.10.1.XX
```

If you are having issues, you can also use verbose mode to help debug:

```bash
ssh -vvv username@10.10.1.XX
```

#### Connecting via VSCode on Linux

> [!NOTE] This section assumes you are also using cloudflare tunneling for your reverse proxy.

TODO

#### Connecting vis VSCode on Windows

> [!NOTE] This section assumes you are also using cloudflare tunneling for your reverse proxy.

## Initial networking setup

### Domain registration

### Cloudflare

### Cloudflare tunnel

## Services

### Setting up your services VM

For the majority of services, we are going to use a single VM that runs all of the docker containers/services/scripts. This way we can have more dynamic resource paritioning across all of our services.

I would recommend using something stable like `debian` or `fedora`, whatever you're most familiar with. Then make sure you provision enough resources, this will depend on how many services and what kind of resources you are planning on running. For my VM I gave it 6 cores and 42GB of memory. YMMV.

### Setting up a new service

For each service, we are going to make a new shared folder in OMV and share it with this VM over nfs. This way each of the services' data are separated and the setup is more flexible.

For this suppose we're making a shared folder under the parent `SERVICES` folder in OMV: `/SERVICES/EXAMPLE`.

1. Make a new folder in OMV (and make it read/write for  most of the cases)
2. Share it the VM using NFS.
3. Check the connection works on the services VM using `sudo mount -t nfs 10.10.1.XX:/SERVICES/EXAMPLE /mnt/example`
4. Update your `/etc/fstab` to add that entry so that it auto mounts on boot.

### Nextcloud

#### Useful `occ` commands

`occ` commands can be executed through the AIO docker container using `docker exec`. E.g. for filescan:

```bash
docker exec -u www-data nextcloud-aio-nextcloud php occ files:scan --all
```

Where `www-data` is the user and `nextcloud-aio-nextcloud` is the container name. You can find other occ commands in the official nextcloud documentation.

### Gitea

#### Container setup

#### Networking setup

#### Ssh setup

A bit annoying to setup. Since we're using cloudflared tunnel, we will use a new tunnel record for gitea ssh pointing to the port 22.

```bash
ssh giteassh.example.com
```

Then in our ssh config, we add the line:

```text
Host gitea.example.com
  HostName giteassh.example.com
  ProxyCommand /home/linuxbrew/.linuxbrew/bin/cloudflared access ssh --hostname %h
```

This tells our ssh that whenever we're trying to connect to `gitea.example.com` over ssh, we actually want to redirect to `giteassh.example.com` and we use a custom proxy command that routes the traffic through cloudflared. Here we just have to specify the executable path for cloudflared.

I installed it on ubuntu using homebrew, if you install it through other means the path will be different.

For windows it looks more like this:

```text
Host gitea.example.com
    HostName giteassh.example.com
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
git remote add gitea git@gitea.example.com:trentzz/homelab.git
```

Then when you want to push:

```bash
git push -u gitea main
```

### Jellyfin

### Authentik

## Extras

This section contains any tidbits and extras I think are useful to document.

### Split tunneling with a VPN

Suppose we want to use a VPN to access something specific, e.g. using a mesh VPN like netbird to access another computer, or Mullvad/NordVPN or a host of others to spoof location. We run into an issue if we also want to access internal IP addresses, like nfs shares.

Enter split tunneling. With this we can specify that we want to bypass the VPN when accessing a specific range of IP addresses (e.g. your home network).

Search up how to do this with your VPN client, most should have easy instructions on how to do this.

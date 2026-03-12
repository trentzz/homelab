# Debian

With our storage solution setup, we can make the VM we're going to use for the majority of the container based services we're running. For this I'm going to use debian since that's what I'm most familiar with, but feel free to use anything.

## Setting a static internal IP address

To organise your VMs a little on the networking side, I would recommend setting a static IP for your internal network. This also enables you to share nfs folders easily without having to worry about the client IP address changing after reboot.

You can do this using the GUI through: `Settings > Network > Gear icon > IPv4 > IPv4 Method Manual` and specifying your:

- Address, e.g. `10.10.1.XX`
- Netmask, e.g. `255.255.255.0`
- Gateway, e.g. `10.10.1.1`

Then click `Apply` and restart your connection. You can check that this worked by checking the output of:

```bash
ip a
```

## SSH access

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

## Connecting via VSCode on Linux

> [!NOTE] This section assumes you are also using cloudflare tunneling for your reverse proxy.

TODO

## Connecting vis VSCode on Windows

> [!NOTE] This section assumes you are also using cloudflare tunneling for your reverse proxy.

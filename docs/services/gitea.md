# Gitea

## Container setup

## Networking setup

## Ssh setup

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

## Connecting multiple remotes to a git repo

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

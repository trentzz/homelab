# Dev Machine Setup

This covers setting up a Debian mini PC as a persistent, always-on dev node: disabling sleep, installing common dev tools, and configuring Git and Gitea.

---

## Prevent Sleep and Suspend

A mini PC used as a server should never sleep or suspend. Debian respects systemd power targets, so mask them all:

```bash
sudo systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target
```

Then update `/etc/systemd/logind.conf` to ignore lid and idle events:

```ini
HandleLidSwitch=ignore
HandleLidSwitchExternalPower=ignore
HandleSuspendKey=ignore
HandleHibernateKey=ignore
IdleAction=ignore
IdleActionSec=0
```

Apply the changes:

```bash
sudo systemctl restart systemd-logind
```

Confirm nothing is scheduled to sleep:

```bash
systemctl status sleep.target suspend.target hibernate.target hybrid-sleep.target
```

All four should show `masked`.

If the machine has a display connected and you want to prevent the screen from blanking (useful for dashboards), disable DPMS:

```bash
xset s off
xset -dpms
xset s noblank
```

To make this persistent across reboots, add those three commands to your `.xprofile` or the display manager's startup script.

Finally, if you want to ensure the machine always boots to a non-graphical multi-user target (skipping a desktop environment entirely):

```bash
sudo systemctl set-default multi-user.target
```

---

## Common Dev Tools

The following script installs the base build toolchain and common utilities. Run it on a fresh Debian install.

```bash
#!/usr/bin/env bash
set -euo pipefail

# Core build tools
sudo apt update
sudo apt install -y \
    build-essential \
    cmake \
    gcc \
    g++ \
    make \
    ninja-build \
    pkg-config \
    libssl-dev \
    libffi-dev \
    zlib1g-dev \
    libbz2-dev \
    libreadline-dev \
    libsqlite3-dev \
    liblzma-dev \
    libncurses-dev

# General utilities
sudo apt install -y \
    git \
    curl \
    wget \
    tmux \
    htop \
    unzip \
    jq \
    ripgrep \
    fd-find \
    bat \
    tree

echo "Base tools installed."
```

Save this as `install-base.sh`, make it executable with `chmod +x install-base.sh`, then run it.

---

## Homebrew

Homebrew on Linux (Linuxbrew) is useful for tools not in the Debian repos or when you want a newer version than what `apt` provides.

Install:

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

After the installer finishes, it prints a "Next steps" block with the exact `eval` line for your shell. Add it to your shell profile (`~/.bashrc` or `~/.zshrc`):

```bash
eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)"
```

Then reload your shell:

```bash
source ~/.bashrc   # or source ~/.zshrc
```

Verify:

```bash
brew --version
brew doctor
```

---

## Rust

Install via `rustup`, which manages toolchain versions and targets:

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

Follow the prompts and choose the default install. Then reload your shell or source the env file:

```bash
source "$HOME/.cargo/env"
```

Verify:

```bash
rustc --version
cargo --version
```

To update Rust later:

```bash
rustup update
```

---

## Python

Use `pyenv` to manage Python versions cleanly, without touching the system Python:

```bash
# Install pyenv
curl https://pyenv.run | bash
```

Add to your shell profile (`~/.bashrc` or `~/.zshrc`):

```bash
export PYENV_ROOT="$HOME/.pyenv"
export PATH="$PYENV_ROOT/bin:$PATH"
eval "$(pyenv init -)"
```

Reload your shell, then install a Python version:

```bash
pyenv install 3.12
pyenv global 3.12
```

Verify:

```bash
python --version
pip --version
```

For virtual environments, the built-in `venv` module is sufficient:

```bash
python -m venv .venv
source .venv/bin/activate
```

Alternatively, [uv](https://github.com/astral-sh/uv) is a much faster drop-in for `pip` and virtual environment management:

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

---

## Gitea CLI (tea)

`tea` is the official Gitea command-line client. Install it via Homebrew:

```bash
brew install tea
```

### Sign in to a custom Gitea instance

```bash
tea login add
```

This starts an interactive prompt. Provide:

- **URL**: your Gitea instance, e.g. `https://gitea.example.com`
- **Name**: a local alias for this login, e.g. `mygitea`
- **Token**: a personal access token from Gitea (`Settings > Applications > Generate Token`)

Alternatively, pass everything directly:

```bash
tea login add \
  --url https://gitea.example.com \
  --name mygitea \
  --token YOUR_API_TOKEN
```

Logins are stored in `~/.config/tea/config.yml`. To list configured logins:

```bash
tea login list
```

To set a login as the default:

```bash
tea login default mygitea
```

### Verify

```bash
tea whoami
tea repo list
```

---

## Git Configuration

### Identity

Set your name and email globally. These appear in every commit you make:

```bash
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
```

### Default branch name

```bash
git config --global init.defaultBranch main
```

### Credential helper

To avoid re-entering credentials on every push, use the `store` helper (saves credentials to `~/.git-credentials` in plaintext) or `cache` (keeps them in memory for a session):

```bash
# Persistent (plaintext file — fine on a private server, not on a shared machine)
git config --global credential.helper store

# Or in-memory for 15 minutes
git config --global credential.helper "cache --timeout=900"
```

### Persist credentials for a specific Gitea instance

If you use HTTPS for Git operations against your Gitea instance, you can scope the credential helper to that host and store a personal access token as the password:

```bash
git config --global credential.https://gitea.example.com.helper store
```

Then on the first `git push` or `git clone` to that host, Git will prompt for a username and password. Use your Gitea username and a personal access token as the password. The token is then saved to `~/.git-credentials` and reused automatically.

### URL rewrite (optional)

If you want to type a short alias like `gitea:user/repo` instead of the full URL, add a URL rewrite:

```bash
git config --global url."https://gitea.example.com/".insteadOf "gitea:"
```

Then you can clone with:

```bash
git clone gitea:username/myrepo
```

### Review your global config

```bash
git config --global --list
```

Or open the file directly:

```bash
cat ~/.gitconfig
```

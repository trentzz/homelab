# Ansible

Ansible is an agentless automation tool for configuration management and provisioning. It connects to target machines over SSH and executes tasks defined in YAML playbooks. No software needs to be installed on managed nodes.

## Ansible vs Puppet vs Chef

The three main configuration management tools for homelabs and production infrastructure:

| | Ansible | Puppet | Chef |
|---|---|---|---|
| **Agent required** | No — uses SSH | Yes — agent on each node | Yes — agent on each node |
| **Language** | YAML | Puppet DSL (Ruby-based) | Ruby |
| **Model** | Procedural (tasks run top to bottom) | Declarative (describe desired state) |  Declarative (describe desired state) |
| **Learning curve** | Low | Medium | High |
| **Execution** | Push — control node pushes changes | Pull — agents check in on a schedule | Pull — agents check in on a schedule |
| **Setup overhead** | Minimal | Moderate (requires Puppet server) | High (requires Chef server) |

**Ansible** is the most practical choice for homelabs. No agents, minimal setup, and YAML playbooks are readable without deep prior knowledge. The push model means you run changes when you want them.

**Puppet** uses a declarative model where you describe the desired state of a system and the agent continuously enforces it. Better suited to large fleets where drift prevention matters. Requires a Puppet server and an agent installed on every managed node.

**Chef** is similar to Puppet in architecture but uses Ruby for configuration, which increases the learning curve significantly. Rarely the right choice for a homelab.

For most homelab use cases, Ansible is sufficient. Puppet becomes worth considering if you're managing a large number of machines and want automatic drift correction rather than manually triggered playbook runs.

## Installing Ansible

Ansible is installed only on the **control node** (your workstation or a management VM):

```bash
sudo apt install pipx
pipx install ansible-core
pipx inject ansible-core argcomplete
```

Or with pip:

```bash
pip install ansible --user
```

Verify:

```bash
ansible --version
```

## Inventory

The inventory file defines the machines Ansible manages. Create `inventory.yml`:

```yaml
all:
  children:
    servers:
      hosts:
        debian-main:
          ansible_host: 10.10.1.10
        media-server:
          ansible_host: 10.10.1.11
        docker-host:
          ansible_host: 10.10.1.12
    proxmox:
      hosts:
        pve-node:
          ansible_host: 10.10.1.2
          ansible_user: root

  vars:
    ansible_user: your_username
    ansible_become: true
```

Test connectivity:

```bash
ansible all -i inventory.yml -m ping
```

## Playbooks

Playbooks are YAML files describing tasks to run on hosts:

```yaml
# playbooks/common.yml
---
- name: Common setup for all servers
  hosts: servers
  become: true

  tasks:
    - name: Update apt cache and upgrade packages
      apt:
        update_cache: true
        upgrade: dist
        cache_valid_time: 3600

    - name: Install common packages
      apt:
        name:
          - curl
          - wget
          - htop
          - vim
          - ufw
          - unattended-upgrades
        state: present

    - name: Ensure SSH is running
      service:
        name: ssh
        state: started
        enabled: true
```

Run a playbook:

```bash
ansible-playbook -i inventory.yml playbooks/common.yml
```

## Roles

Roles organize tasks into reusable units. A typical homelab structure:

```
homelab-ansible/
├── inventory.yml
├── site.yml
├── playbooks/
│   └── common.yml
└── roles/
    ├── docker/
    │   └── tasks/
    │       └── main.yml
    ├── monitoring/
    │   └── tasks/
    │       └── main.yml
    └── firewall/
        └── tasks/
            └── main.yml
```

### Example: Docker Role

```yaml
# roles/docker/tasks/main.yml
---
- name: Install Docker prerequisites
  apt:
    name:
      - ca-certificates
      - curl
      - gnupg
    state: present

- name: Add Docker GPG key
  apt_key:
    url: https://download.docker.com/linux/debian/gpg
    state: present

- name: Add Docker repository
  apt_repository:
    repo: "deb https://download.docker.com/linux/debian {{ ansible_distribution_release }} stable"
    state: present

- name: Install Docker
  apt:
    name:
      - docker-ce
      - docker-ce-cli
      - containerd.io
      - docker-compose-plugin
    state: present
    update_cache: true

- name: Add user to docker group
  user:
    name: "{{ ansible_user }}"
    groups: docker
    append: true

- name: Ensure Docker is running
  service:
    name: docker
    state: started
    enabled: true
```

Reference roles in your master playbook:

```yaml
# site.yml
---
- name: Apply common configuration
  hosts: servers
  roles:
    - common

- name: Set up Docker hosts
  hosts: docker-host
  roles:
    - docker
```

## Ansible Vault

Vault encrypts secrets so they can be safely committed to git:

```bash
# Create an encrypted vars file
ansible-vault create vars/secrets.yml

# Edit it later
ansible-vault edit vars/secrets.yml

# Run a playbook that uses encrypted vars
ansible-playbook -i inventory.yml site.yml --ask-vault-pass
```

Example encrypted file contents:

```yaml
cloudflare_api_token: "your-token-here"
db_password: "super-secret"
```

## Repo Structure Tips

- Keep Ansible in a separate git repo from docs and notes
- Use `group_vars/` and `host_vars/` for per-group and per-host variables
- Use tags to run subsets of tasks: `ansible-playbook site.yml --tags docker`
- Use `--check` for dry runs before applying changes: `ansible-playbook site.yml --check`

## Semaphore UI

[Semaphore](https://semaphoreui.com/) is an open-source web UI for Ansible. It manages inventories, runs playbooks, schedules tasks, and stores run history.

```yaml
services:
  semaphore:
    image: semaphoreui/semaphore:latest
    container_name: semaphore
    restart: unless-stopped
    ports:
      - "3000:3000"
    environment:
      SEMAPHORE_DB_DIALECT: bolt
      SEMAPHORE_ADMIN: admin
      SEMAPHORE_ADMIN_PASSWORD: changeme
      SEMAPHORE_ADMIN_EMAIL: admin@example.com
    volumes:
      - ./semaphore-data:/tmp/semaphore
```

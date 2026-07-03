<h1 align="center">🛠️ Server Bootstrap — Reusable Ansible Template for Debian</h1>

<p align="center"><em>A reusable Ansible template for provisioning new Debian servers: copy the scaffold, opt in the components a host needs — SSH, firewall, Docker, monitoring, shell — and validate the result with built-in checks.</em></p>

<p align="center">
  <img src="https://img.shields.io/badge/status-active-blue?style=for-the-badge" alt="Status" />
  <img src="https://img.shields.io/badge/Ansible-core%202.16-EE0000?style=for-the-badge&logo=ansible" alt="Ansible" />
  <img src="https://img.shields.io/badge/Debian-12%2B-A81D33?style=for-the-badge&logo=debian" alt="Debian" />
  <img src="https://img.shields.io/badge/Docker-CE-2496ED?style=for-the-badge&logo=docker" alt="Docker" />
  <img src="https://img.shields.io/badge/Node.js-24.13.1-339933?style=for-the-badge&logo=nodedotjs" alt="Node.js" />
  <img src="https://img.shields.io/badge/license-MIT-lightgrey?style=for-the-badge" alt="License" />
</p>

---

![All provisioned components — security, runtime, platform services, and system/monitoring, grouped by category with built-in validation](public/components.svg)

## Table of Contents

1. [Overview](#1-overview)
   - [1.1 The problem](#11-the-problem)
   - [1.2 What it does](#12-what-it-does)
   - [1.3 Highlights](#13-highlights)
2. [Validation & Tested Environment](#2-validation--tested-environment)
3. [Architecture](#3-architecture)
   - [3.1 Components](#31-components)
   - [3.2 The two role sets](#32-the-two-role-sets)
4. [Tech stack](#4-tech-stack)
5. [Getting Started (Use this template)](#5-getting-started-use-this-template)
   - [5.1 Prerequisites](#51-prerequisites)
   - [5.2 Installation](#52-installation)
   - [5.3 Configuration](#53-configuration)
   - [5.4 Use this template](#54-use-this-template)
6. [Project structure](#6-project-structure)
7. [Author](#7-author)

## 1. Overview

### 1.1 The problem

Provisioning a new server usually starts from nothing: every host is bootstrapped by hand, differently, with SSH hardening, firewall rules, Docker, monitoring agents, and shell setup redone from memory each time. This project exists so that provisioning a new server never starts from a blank page — it is a reusable Ansible template you copy and grow into your own flow, instead of a one-off script you rebuild per host.

### 1.2 What it does

- Provides a copy-and-extend template (`roles/debiancustom2`) for provisioning any new Debian 12+ host from a single control node over SSH.
- Ships a library of ready-made components you opt into per host: SSH hardening, UFW, fail2ban, chrony, Docker, Node.js, Cockpit, Coolify, and Tailscale.
- Includes monitoring agents: Prometheus Node Exporter and the New Relic infrastructure agent.
- Configures the shell: Zsh plus Oh My Posh, with a custom prompt theme supplied from the control node.
- Ships a matching `val_*` validation task for every component that asserts the result and prints a structured PASSED/FAILED report.

**Primary use case:** bootstrapping a new server — start from this template for any fresh Debian-based VPS, cloud instance, or lab host: pick the components it needs (SSH access, firewall rules, fail2ban, monitoring, shell and Git configuration, Docker runtime), run the playbook, and reuse the same template for the next host.

### 1.3 Highlights

- **Reusable by design** — `roles/debiancustom` is the untouched reference set; `roles/debiancustom2` is the copy you customize into your own server-bootstrap flow, host after host.
- **Idempotent, component-based tasks** — each capability lives in its own task file gated by an `*_enabled` flag, so a new server only gets the components it needs.
- **Validation built in** — every component has a paired `val_*` task that checks service state, effective configuration, and runtime reachability, then asserts.
- **Reproducibility over one-off scripts** — the same declarative template provisions many hosts without configuration drift, and makes rebuilds and redeployments consistent.
- **Safe defaults** — SSH on a non-standard port, root login and password auth disabled, source-restricted firewall rules, and a check that Docker is not exposed on TCP 2375.
- **Secrets kept out of the repo** — tracked config uses placeholders (`CHANGE_ME`) and RFC 5737 example networks.

## 2. Validation & Tested Environment

> By design, this repository ships as a role-based scaffold rather than a turnkey playbook: `playbooks/dev01.yml` lists a placeholder role name (`infra_base`) marking the extension point where your own role belongs — it is not a functional role in this project. Each included role's `tasks/main.yml` starts with every component commented out, so you opt in one at a time (see [5. Getting Started](#5-getting-started-use-this-template)).

There is no separate unit-test suite. Every component ships with a paired `val_*` task instead: it gathers effective state (service status, config, runtime reachability), records PASSED/FAILED per check, asserts on failure, and prints a structured summary — keeping correctness checks next to the code that applies the change. Example shape of the SSH validation output (`roles/debiancustom/tasks/val_ssh.yml`):

```text
==================================
VALIDATION SUMMARY
==================
service_status      : active
port_listening      : LISTEN
endpoint_accessible : OK
network_rule        : NOT_MANAGED
-----------------------------
# RESULT: CONFIGURATION VALID
```

| Component | Validation task |
|-----------|-----------------|
| base_packages | `val_base.yml` |
| ssh | `val_ssh.yml` |
| ufwrules | `val_ufwrules.yml` |
| fail2ban | `val_fail2ban.yml` |
| chrony | `test.yml` |
| cockpit | `val_cockpit.yml` |
| docker | `val_docker.yml` |
| nodejs | `val_nodejs.yml` |
| node_exporter | `val_node_exporter.yml` |
| newrelic | `val_newrelic.yml` |
| zsh | `val_zsh.yml` |
| coolify | `val_coolify.yml` |
| tailscale | `val_tailscale.yml` |
| git | `val_git.yml` |
| ipv6 | `val_ipv6.yml` |
| deactivate_updates | `val_deupdates.yml` |

Each validation asserts on failure, so a failed check stops the play — representative pattern:

```yaml
- name: Assert SSH validation
  ansible.builtin.assert:
    that:
      - ssh_failed | length == 0
    fail_msg: "SSH validation failed: {{ ssh_failed | join(', ') }}"
    success_msg: "SSH validation passed"
```

**Tested environment** — developed and validated in a local lab simulating realistic Debian provisioning flows before targeting cloud-style VPS bootstrap: Linux Mint 22.3 (control node), KVM / QEMU / libvirt, `virt-install` and cloud-init, qcow2 backing images, UEFI / OVMF, libvirt NAT networking. Tailscale and the New Relic infrastructure agent were additionally exercised end to end beyond the general lab flow above.

See [3. Architecture](#3-architecture) for the D2 diagram of the full control-node-to-target flow.

## 3. Architecture

An operator runs `ansible-playbook` on a control node. The playbook selects a role and its enabled components, then connects over SSH (non-standard port `22000`) to the target Debian host, applies each component's tasks, runs the matching validation, and pulls binaries from upstream sources (Docker apt repo, GitHub releases) as needed.

![Architecture — control node to target host](public/architecture.svg)

### 3.1 Components

Each component is an independent, flag-gated task file with a `val_*` counterpart:

- **base_packages** — baseline apt packages
- **ssh** — hardened `sshd` drop-in config (non-standard port, no root login, key-only auth)
- **ufwrules** — source-restricted UFW rules for SSH/HTTP/HTTPS
- **fail2ban** — SSH brute-force protection
- **chrony** — time synchronization
- **cockpit** — web management console
- **docker** — Docker CE repo, install, group setup, exposure check, hello-world test
- **nodejs** — Node.js runtime with npm/pnpm selection
- **node_exporter** — Prometheus Node Exporter
- **newrelic** — New Relic infrastructure agent
- **zsh** — Zsh + Oh My Posh with a custom prompt theme
- **coolify** — self-hosted deployment platform
- **tailscale** — mesh VPN access
- **git** — Git and GitHub SSH configuration
- **ipv6 / deactivate_updates** — kernel/network and unattended-update toggles

### 3.2 The two role sets

- **`roles/debiancustom`** — the reference role that holds the full set of component and validation tasks.
- **`roles/debiancustom2`** — an untouched copy of `debiancustom`, provided as the recommended starting point for building your own custom flow. Rather than editing the reference role, enable a single component in `debiancustom2`, run it to confirm it works on your target, then add components one at a time until you have your personalized flow.

> Note: the two role directories are currently byte-for-byte identical — `debiancustom2` is the copy you customize, so their contents will diverge as you build your own flow.

## 4. Tech stack

| Layer | Technology | Why this over alternatives |
|-------|-----------|----------------------------|
| Automation engine | Ansible core 2.16 | Agentless over SSH; declarative and idempotent — no bootstrap agent required on targets |
| Target OS | Debian 12+ | Stable server baseline; tasks assert Debian 12+ explicitly |
| Container runtime | Docker CE (+ compose plugin) | Installed from the official Docker apt repo for current, supported builds vs. distro `docker.io` |
| Runtime | Node.js 24.13.1 (npm / pnpm) | Pinned version with a manager selection flag for reproducible app tooling |
| Firewall | UFW | Simple, source-restricted rule model layered on top of iptables |
| Intrusion prevention | fail2ban | Bans repeated SSH auth failures using the systemd journal backend |
| Time sync | chrony | Modern NTP client with fast convergence and clear tracking output |
| Metrics | Prometheus Node Exporter 1.10.1 | De facto host-metrics exporter for Prometheus scraping |
| APM / infra monitoring | New Relic infrastructure agent | Turnkey host monitoring via the official install script |
| Deployment platform | Coolify | Self-hosted PaaS for app deployment on the provisioned host |
| Networking | Tailscale | Zero-config mesh VPN for private host access |
| Shell | Zsh + Oh My Posh | Scriptable shell with a portable, themeable prompt |

## 5. Getting Started (Use this template)

### 5.1 Prerequisites

| Requirement | Notes |
|-------------|-------|
| Ansible core 2.16+ | On the control node (validated with `ansible [core 2.16.3]`) |
| SSH access to the target | Key-based; the target must be reachable on the configured `ansible_port` |
| Target OS | Debian 12+ (tasks assert this) |
| `sudo`/become on the target | Playbook runs with `become: true` |

### 5.2 Installation

Clone the repository: `git clone https://github.com/dev-mikel/debian-ansible-init.git && cd debian-ansible-init`.

`ansible.cfg` already sets the roles path and points the inventory at `inventories/development`:

| Setting | Value | Why |
|---------|-------|-----|
| `roles_path` | `./roles` | Roles resolve relative to the repo, no global install needed |
| `inventory` | `./inventories/development` | Default inventory for the included example host |
| `stdout_callback` | `yaml` | Readable, structured task output |
| `bin_ansible_callbacks` | `True` | Enables the `yaml` callback for `ansible-playbook` |

### 5.3 Configuration

Edit the inventory host in `inventories/development/hosts.yml` and the variables in `inventories/development/group_vars/all/main.yml`. Replace every `CHANGE_ME` before running against a real host. Key variables:

| Variable | Description |
|----------|-------------|
| `ansible_host` / `ansible_user` / `ansible_port` | Target host address, login user, and SSH port (`22000`) |
| `permit_root_login` / `password_auth` | SSH hardening toggles (`no` by default) |
| `allowed_ssh_networks` | Source networks permitted to reach SSH |
| `ssh_port` | Hardened SSH listening port (`22000`) |
| `tailscale_authkey` | Tailscale auth key — **set your own**, do not commit |
| `docker_user` / `docker_version` / `docker_install_latest` | Docker group user and version pinning behavior |
| `nodejs_version` / `nodejs_install_npm` / `nodejs_install_pnpm` | Node.js version and package-manager selection |
| `node_exporter_version` / `node_exporter_port` | Node Exporter release and bind port |
| `newrelic_api_key` / `newrelic_account_id` | New Relic credentials — **set your own**, do not commit |
| `cockpit_bind_ip` / `cockpit_port` | Cockpit console bind address and port |
| `coolify_port` | Coolify published port |

Secrets (SSH keys, `tailscale_authkey`, New Relic credentials) should be kept out of version control. Store them with Ansible Vault: `ansible-vault create inventories/development/group_vars/all/vault.yml`, reference the vaulted vars, then run with `--ask-vault-pass`. The repo already excludes `*.vault*`, `.env*`, `*.key`, and `*.pem` via `.gitignore`.

### 5.4 Use this template

`playbooks/dev01.yml` ships with a placeholder role name (`infra_base`) marking the extension point where your own role belongs — it is not a functional role in this repository. Recommended workflow, one component at a time in `debiancustom2`:

1. Point the playbook at your working role:

   ```yaml
   # playbooks/dev01.yml
   roles:
     - debiancustom2   # or debiancustom, or your own custom role
   ```

2. In `roles/debiancustom2/tasks/main.yml`, uncomment a single component and its validation:

   ```yaml
   - import_tasks: ssh.yml
   - import_tasks: val_ssh.yml
   ```

3. Enable that component and adjust its variables in `inventories/development/group_vars/all/main.yml` (for example `ssh_enabled: true`).
4. Check syntax first (`ansible-playbook playbooks/dev01.yml --syntax-check`), then run: `ansible-playbook playbooks/dev01.yml --limit example-host --ask-vault-pass`.
5. Confirm the validation summary reports `CONFIGURATION VALID`, then repeat, adding one component at a time, until the flow matches what the host needs.

Keep `debiancustom` as the untouched reference set; run a subset ad hoc with tags or by limiting which imports are active.

> A full component workflow typically executes in roughly 5–10 minutes, depending on VPS performance, network and package-mirror speed, external download times, and host resources — an unmeasured estimate, not a benchmarked result. The larger value is reproducibility: the same declarative definition can be reused across many hosts without rebuilding configuration by hand, which reduces drift and makes rebuilds and redeployments consistent.

## 6. Project structure

```text
.
├── ansible.cfg                     # Roles path, inventory, YAML stdout callback
├── group_vars/all.yml              # Shared defaults (admin user, metrics)
├── inventories/development/
│   ├── hosts.yml                   # Target host definition
│   └── group_vars/all/main.yml     # Component variables (CHANGE_ME placeholders)
├── playbooks/
│   ├── dev01.yml                   # Entry playbook
│   └── assets/                     # Shell assets (zshrc, prompt theme reference)
└── roles/
    ├── debiancustom/               # Reference role: component + val_ tasks
    │   ├── tasks/                   # ssh.yml, docker.yml, val_ssh.yml, ...
    │   ├── handlers/main.yml
    │   ├── templates/zshrc.j2
    │   └── defaults/ meta/
    └── debiancustom2/              # Copy to customize into your own flow
```

> The Oh My Posh prompt theme is referenced from the control node by the **zsh** component (`shell_zsh_theme_src` → `shell_zsh_theme_path` in `inventories/development/group_vars/all/main.yml`). Supply your own `theme.omp.json`; it is intentionally not shipped as a finished theme in this repository.

## 7. Author

**Miguel Ladines** · [@dev-mikel](https://github.com/dev-mikel) · [LinkedIn](https://www.linkedin.com/in/ladinesmiguel)  
Electronics Engineer · AI Developer | Automation & Systems Integration

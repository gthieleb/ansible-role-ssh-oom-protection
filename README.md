# ansible-role-ssh-oom-protection

An Ansible role that protects SSH from Out-of-Memory (OOM) kills on Linux servers.

When a server runs low on memory, the Linux OOM killer may terminate the SSH daemon — locking you out of the machine. This role ensures SSH stays alive even under extreme memory pressure by using systemd cgroups, memory reservation, and the userspace OOM killer (`systemd-oomd`).

## What it does

- **Memory reservation:** Reserves RAM exclusively for the SSH daemon via `MemoryMin` in a systemd service override. This memory cannot be reclaimed by other processes.
- **OOM score adjustment:** Sets `OOMScoreAdjust=-1000`, making SSH the absolute last process to be killed by the OOM killer.
- **systemd-oomd configuration:** Deploys and enables `systemd-oomd`, the userspace OOM killer, configured to act when swap usage is high or memory pressure is sustained.
- **SSH keep-alive hardening:** Configures `ClientAliveInterval`, `ClientAliveCountMax`, and `TCPKeepAlive` to prevent SSH sessions from being dropped during memory pressure.
- **Built-in verification:** After deployment, the role automatically verifies all settings are active (cgroup memory.min, OOM score, oomd status, SSH config).

## Requirements

- Ansible >= 2.14
- Target system with systemd and cgroups v2
- Root/sudo access on the target

## Supported Platforms

| OS | Version |
|---|---|
| Ubuntu | 24.04 (Noble) |
| Debian | 12 (Bookworm) |
| RHEL / AlmaLinux / Rocky | 9 |

## Installation

### From Ansible Galaxy

```bash
ansible-galaxy install gthieleb.ssh_oom_protection
```

### From GitHub

```bash
ansible-galaxy install git+https://github.com/gthieleb/ansible-role-ssh-oom-protection.git
```

## Usage

### Minimal

```yaml
- hosts: all
  become: true
  roles:
    - gthieleb.ssh_oom_protection
```

### With custom settings

```yaml
- hosts: all
  become: true
  vars:
    ssh_oom_protection_memory_min: "512M"
    ssh_oom_protection_oom_score_adjust: -1000
    ssh_oom_protection_client_alive_interval: 30
    ssh_oom_protection_oomd_swap_limit: "80%"
  roles:
    - gthieleb.ssh_oom_protection
```

## Variables

| Variable | Default | Description |
|---|---|---|
| `ssh_oom_protection_memory_min` | `256M` | RAM reserved exclusively for the SSH daemon |
| `ssh_oom_protection_oom_score_adjust` | `-1000` | OOM score adjustment (-1000 = never kill) |
| `ssh_oom_protection_client_alive_interval` | `60` | SSH client alive interval in seconds |
| `ssh_oom_protection_client_alive_count_max` | `3` | SSH client alive count max |
| `ssh_oom_protection_tcp_keep_alive` | `true` | Enable TCP keep-alive |
| `ssh_oom_protection_oomd_swap_limit` | `90%` | Swap usage threshold for systemd-oomd |
| `ssh_oom_protection_oomd_pressure_limit` | `60%` | Memory pressure threshold for systemd-oomd |
| `ssh_oom_protection_oomd_pressure_duration` | `30s` | Duration of sustained pressure before action |
| `ssh_oom_protection_install_oomd` | `true` | Whether to install systemd-oomd package |

## How it works

1. Creates a systemd service override for the SSH daemon that sets `MemoryMin` and `OOMScoreAdjust`
2. Deploys SSH keep-alive settings via `sshd_config.d` drop-in
3. Configures and enables `systemd-oomd` for proactive OOM management
4. Verifies all settings are correctly applied by reading cgroup values and checking service status

The role automatically detects the correct SSH service name (`ssh` on Debian/Ubuntu, `sshd` on RHEL/Fedora).

## License

MIT

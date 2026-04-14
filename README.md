# pi-controlplane

Ansible playbook for configuring a Raspberry Pi 3 running Ubuntu 64-bit.

## Requirements

- Ansible 2.14+
- SSH access to the target host as the `ubuntu` user
- The target host resolvable via mDNS as `raspberry-1.local`

## Usage

```bash
ansible-playbook main.yaml
```

You will be prompted for the sudo password on each run.

## Structure

```
.
├── main.yaml          # Top-level playbook
├── inventory.yaml     # Host and variable definitions
├── ansible.cfg        # Project-level Ansible configuration
└── roles/
    ├── nut/                    # NUT (Network UPS Tools) — APC UPS over USB
    ├── wakelet/                # Wakelet — HomeKit bridge for Wake-on-LAN
    └── vm-builder-apiserver/  # VM Builder API server — homelab VM control plane
```

## Roles

### nut

Configures [NUT](https://networkupstools.org/) to monitor an APC Smart-UPS C 1500 connected via USB.

| What | Detail |
|------|--------|
| Driver | `usbhid-ups` |
| USB device | `051d:0003` (APC) |
| Mode | `netserver` — listens on `127.0.0.1:3493` |
| Packages | `nut-client`, `nut-server` 2.8.1 |
| `nut-monitor` | disabled and masked (upsmon not needed) |

A udev rule is written to `/etc/udev/rules.d/90-nut-ups.rules` granting the `nut` group access to the USB device.

### wakelet

Installs and configures [wakelet](https://github.com/tlhakhan/wakelet), a HomeKit bridge that exposes local network hosts as HomeKit accessories with Wake-on-LAN and shutdown support.

| What | Detail |
|------|--------|
| Install path | `/opt/wakelet` |
| Config | `/etc/wakelet/hosts.yaml` |
| SSH keys | `/etc/wakelet/private/` (auto-generated on first start) |
| State | `/var/lib/wakelet` |
| Port | 51826 (HomeKit Accessory Protocol) |

#### Host variables

| Variable | Default | Description |
|----------|---------|-------------|
| `wakelet_version` | `main` | Git tag, branch, or commit to deploy |
| `wakelet_hosts` | `[]` | List of hosts to expose as HomeKit accessories |

Each entry in `wakelet_hosts` takes:

```yaml
- name: hostname.local   # required
  mac: aa:bb:cc:dd:ee:ff # required — used for Wake-on-LAN
  holdup_timer: 60       # optional — seconds to wait after power-on before checking reachability
```

### vm-builder-apiserver

Installs and configures [vm-builder-apiserver](https://github.com/tlhakhan/vm-builder-apiserver), a REST API control plane for managing VMs across hypervisor agents via mTLS.

| What | Detail |
|------|--------|
| Binary | `/usr/local/bin/vm-builder-apiserver` |
| Listen port | `:8080` |
| Private keys / certs | `/etc/vm-builder-apiserver/private/` |
| Agent registry | `/var/lib/vm-builder-apiserver/registry.json` |
| Transport | mTLS to agents on port 8443 |

#### Host variables

| Variable | Default | Description |
|----------|---------|-------------|
| `vm_builder_apiserver_version` | `v1.2.0` | GitHub release tag to download |
| `vm_builder_apiserver_listen` | `:8080` | Address and port to listen on |
| `vm_builder_apiserver_agent_mtls` | `true` | Enable mTLS for agent connections |
| `vm_builder_apiserver_agent_insecure_skip_verify` | `true` | Skip TLS certificate verification for agents |
| `vm_builder_apiserver_health_interval` | `10s` | Agent health check interval |
| `vm_builder_apiserver_hosts` | `[]` | List of hypervisor agents to register |

Each entry in `vm_builder_apiserver_hosts` takes:

```yaml
- name: hypervisor-1                    # required
  url: https://hypervisor-1.local:8443  # required
```

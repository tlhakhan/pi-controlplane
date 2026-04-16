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
    ├── vm-builder-dashboard/   # VM Builder dashboard and control plane
    └── wakelet/                # Wakelet — HomeKit bridge for Wake-on-LAN
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

### vm-builder-dashboard

Installs and configures [vm-builder-dashboard](https://github.com/tlhakhan/vm-builder-dashboard), the FastAPI dashboard and VM control plane that manages `vm-builder-agent` hosts.

The role follows the upstream install flow:

```bash
git clone https://github.com/tlhakhan/vm-builder-dashboard
python3 -m venv venv
pip install -r requirements.txt
uvicorn main:app --host 127.0.0.1 --port 8081
```

| What | Detail |
|------|--------|
| Install path | `/opt/vm-builder-dashboard` |
| Virtualenv | `/opt/vm-builder-dashboard/venv` |
| Environment file | `/etc/vm-builder-dashboard/vm-builder-dashboard.env` |
| Secret key file | `/etc/vm-builder-dashboard/secret_key` |
| State | `/var/lib/vm-builder-dashboard` |
| Service user | `root` |
| Default listen address | `127.0.0.1:8081` |

If `vm_builder_dashboard_secret_key` is not set, the role generates a random key on the target host the first time it runs, stores it at `/etc/vm-builder-dashboard/secret_key`, and reuses it on later runs.

#### Host variables

| Variable | Default | Description |
|----------|---------|-------------|
| `vm_builder_dashboard_version` | `main` | Git tag, branch, or commit to deploy |
| `vm_builder_dashboard_secret_key` | auto-generated | Session signing key; if unset, persisted on the host |
| `vm_builder_dashboard_listen_host` | `127.0.0.1` | Interface for uvicorn |
| `vm_builder_dashboard_listen_port` | `8081` | TCP port for uvicorn |
| `vm_builder_dashboard_db_path` | `/var/lib/vm-builder-dashboard/db.sqlite3` | SQLite database path |
| `vm_builder_dashboard_agent_pki_dir` | `/var/lib/vm-builder-dashboard/pki` | Generated CA and client certificate directory |
| `vm_builder_dashboard_agent_health_interval` | `10` | Seconds between agent health checks |
| `vm_builder_dashboard_agent_timeout_seconds` | `30` | Timeout for proxied agent calls |
| `vm_builder_dashboard_agent_health_timeout_seconds` | `5` | Timeout for background health checks |
| `vm_builder_dashboard_agent_tls_insecure_skip_verify` | `false` | Skip TLS verification when calling HTTPS agents |

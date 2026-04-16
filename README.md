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

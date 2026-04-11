# ansible-role-amneziawg

Installs and configures [AmneziaWG](https://github.com/amnezia-vpn/amneziawg-linux-kernel-module) — a WireGuard fork with traffic obfuscation.

## Requirements

- Debian (bullseye, bookworm) or Ubuntu (focal, jammy, noble)
- Ansible collections: `ansible.posix`, `ansible.utils`

```bash
ansible-galaxy collection install ansible.posix ansible.utils
```

## Role Variables

```yaml
## Interface name
amneziawg_interface: "awg0"

## Server IP/CIDR
amneziawg_address: "10.0.0.1/24"

## UDP listen port
amneziawg_port: 51820

## Private key (auto-generated if empty)
amneziawg_private_key: ""

## DNS for clients
amneziawg_dns: "{{ amneziawg_address | ansible.utils.ipaddr('address') }}"

## Force config rewrite even if peers exist
amneziawg_force_config: false

## Obfuscation parameters (must match between server and client, except Jc/Jmin/Jmax)
amneziawg_jc: 4        # junk packets count (1–128)
amneziawg_jmin: 8      # min junk packet size
amneziawg_jmax: 80     # max junk packet size
amneziawg_s1: 30       # init junk size (S1+56 ≠ S2)
amneziawg_s2: 40       # response junk size
amneziawg_h1: 1234567891
amneziawg_h2: 1234567892
amneziawg_h3: 1234567893
amneziawg_h4: 1234567894
```

## Usage

### Standalone

```yaml
- hosts: vpn_servers
  roles:
    - role: ansible-role-amneziawg
      vars:
        amneziawg_interface: awg0
        amneziawg_address: "10.10.0.1/24"
        amneziawg_port: 51820
        amneziawg_jc: 6
        amneziawg_s1: 42
        amneziawg_s2: 100
        amneziawg_h1: 1985301761
        amneziawg_h2: 1985301762
        amneziawg_h3: 1985301763
        amneziawg_h4: 1985301764
```

### With WGDashboard

```yaml
- hosts: vpn_servers
  roles:
    - role: ansible-role-amneziawg
      vars:
        amneziawg_interface: awg0
        amneziawg_address: "10.10.0.1/24"

    - role: ansible-role-wgdashboard
      vars:
        wgdashboard_wg_interface: awg0
        wgdashboard_wg_backend: amneziawg
        wgdashboard_peer_global_dns: "10.10.0.1"
```

## Update mechanism

- Set `amneziawg_update: true` to upgrade packages to the latest version from PPA
- Default: `false` — packages are installed only if missing (`state: present`)
- When `true`: `state: latest` — upgrades `amneziawg-dkms` + `amneziawg-tools` via apt, DKMS rebuilds the kernel module

```bash
# Update via Makefile
make update-amneziawg HOST=<hostname>
```

## Installation details

- **Ubuntu**: adds PPA `ppa:amnezia/ppa`, installs `amneziawg` package
- **Debian**: adds Launchpad PPA key + focal repository, installs `amneziawg` package
- Private key is generated via `awg genkey` and saved to `/etc/wireguard/<interface>_privatekey` (mode 0600)
- Config is written to `/etc/wireguard/<interface>.conf` (mode 0600)
- Systemd unit: `awg-quick@<interface>`

## Obfuscation parameter constraints

| Param | Constraint | Recommended |
|-------|-----------|-------------|
| Jc | 1 ≤ Jc ≤ 128 | 4–12 |
| Jmin | < Jmax, < 1280 | 8 |
| Jmax | > Jmin, ≤ 1280 | 80 |
| S1 | ≤ 1132, S1+56 ≠ S2 | 15–150 |
| S2 | ≤ 1188 | 15–150 |
| H1–H4 | unique, 5–2147483647 | random values |

> Note: Jc, Jmin, Jmax may differ between server and client. All other parameters must match.

## NAT / Firewall

NAT and iptables rules are **not** configured by this role. Use `ansible-role-firewall` with:

```yaml
firewall_wireguard_enabled: true
firewall_wireguard_interface: "awg0"
firewall_wireguard_network: "{{ amneziawg_address }}"
```

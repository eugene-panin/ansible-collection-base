# wireguard

Brings up a WireGuard interface: installs the packages, generates the private
key **once**, renders the interface config with its peers, and manages
`wg-quick@<interface>`.

The private key is generated on the host by `wg genkey` guarded with
`creates:` — it never passes through Ansible's output, and a second run does
not mint a new one (which would silently invalidate every peer).

## Variables

| Variable | Default | Purpose |
|---|---|---|
| `wireguard_interface` | `wg0` | Interface name; also names the config, key and systemd unit |
| `wireguard_address` | `10.10.0.1/24` | This host's mesh address, with prefix |
| `wireguard_listen_port` | `51820` | UDP listen port |
| `wireguard_peers` | `[]` | Peers permitted to connect (see below) |
| `wireguard_manage_firewall` | `true` | Manage ufw rules for this interface; requires ufw installed |
| `wireguard_allow_mesh_traffic` | `true` | Accept everything arriving through the tunnel |
| `wireguard_private_key_path` | `/etc/wireguard/<interface>.key` | Where the key lives |

Peer entries take `public_key` and `allowed_ips` (both required), plus
optional `name`, `endpoint` and `persistent_keepalive`.

## Example

```yaml
- name: Bring up the mesh
  hosts: vps
  become: true
  roles:
    - role: eugene_panin.base.wireguard
      vars:
        wireguard_address: 10.10.0.1/24
        wireguard_peers:
          - name: laptop
            public_key: Pa1Lg3oieQOdJaAEko1C80S34kTViChY3O7pkQQ6LlU=
            allowed_ips: 10.10.0.2/32
```

## Notes

- Rotating the key is deliberately out of scope: delete
  `wireguard_private_key_path` by hand and re-run if you really mean it.
- `wireguard_manage_firewall: true` needs `ufw` present. Set it to `false` on
  hosts where the firewall is owned by something else.
- `wireguard_allow_mesh_traffic` exists because a default-deny host will
  complete the WireGuard handshake and then silently drop everything you
  built the tunnel for. Traffic arriving on the interface has already been
  authenticated by WireGuard's keys, so accepting it wholesale is reasonable;
  set the variable to `false` if you would rather open specific ports.
- Peers reaching *each other* (rather than just this host) additionally needs
  IP forwarding and NAT, which this role does not configure.
- `wireguard_address` is rendered verbatim, so a dual-stack value such as
  `10.77.0.1/24, fd00::1/64` works.

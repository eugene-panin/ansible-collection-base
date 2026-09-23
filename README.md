# eugene_panin.base

Host foundation layer — what a machine needs before any workload stack sits
on top of it. Deliberately usable on its own: nothing here depends on
Consul, Nomad, Vault or anything from another collection.

## Roles

| Role | Purpose |
|---|---|
| [`wireguard`](roles/wireguard/README.md) | WireGuard interface: generated-once key, declarative peers, `wg-quick@` unit |

## Requirements

- ansible-core >= 2.15
- `community.general` >= 8.0.0 (pulled in automatically)
- Debian or Ubuntu on the managed host. Other distributions get
  `wireguard-tools` from `vars/default.yml` and are untested.
- `ufw` on the managed host, if you leave `wireguard_manage_firewall` on.
  Turn it off and the roles touch no firewall state at all.

## Install

```yaml
# requirements.yml
collections:
  - name: eugene_panin.base
    source: https://github.com/eugene-panin/ansible-collection-base.git
    type: git
    version: main
```

```bash
ansible-galaxy collection install -r requirements.yml
```

## Use

Everything is configured through role variables — nothing about a particular
network, host or environment is baked in. Full list per role in its README
and in `meta/argument_specs.yml`.

```yaml
- name: Bring up the mesh
  hosts: all
  become: true
  roles:
    - role: eugene_panin.base.wireguard
      vars:
        wireguard_address: 10.77.0.1/24
        wireguard_listen_port: 51820
        wireguard_peers:
          - name: laptop
            public_key: EXAMPLEKeyReplaceMeWithYourOwnPeerPubKey0000=
            allowed_ips: 10.77.0.2/32
```

## Boundary

Roles here never read variables belonging to another collection. Anything
that has to cross a collection boundary is declared in the consuming
inventory and passed in explicitly — that is what keeps this collection
installable on a host that has nothing else on it.

## Development

```bash
yamllint .
ansible-lint --profile production
cd roles/<role> && molecule test
```

`ansible-lint` runs the production profile; if it fails, the code is fixed
rather than the rule being skipped.

## License

MIT — see [LICENSE](LICENSE).

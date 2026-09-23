# eugene_panin.base

Host foundation layer — what a machine needs before any workload stack sits
on top of it. Deliberately usable on its own: nothing here depends on
Consul, Nomad, Vault or anything from another collection.

## Roles

| Role | Purpose |
|---|---|
| [`wireguard`](roles/wireguard/README.md) | WireGuard interface: generated-once key, declarative peers, `wg-quick@` unit |
| [`wireguard_client`](roles/wireguard_client/README.md) | Keys and importable `.conf` files for the devices that connect to it |

## Requirements

- ansible-core >= 2.15
- `community.general` >= 8.0.0 (pulled in automatically)
- Debian or Ubuntu on the managed host. Other distributions get
  `wireguard-tools` from `vars/default.yml` and are untested.
- `ufw` on the managed host, if you leave `wireguard_manage_firewall` on.
  Turn it off and the roles touch no firewall state at all.

## Install

```bash
ansible-galaxy collection install eugene_panin.base
```

Or straight from a tag, which is what you want if you care that a push to
`main` cannot change what you deploy:

```yaml
# requirements.yml
collections:
  - name: https://github.com/eugene-panin/ansible-collection-base.git
    type: git
    version: v0.1.0
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

Handing out client configs is a second role, run either side of the first one:

```yaml
- name: Bring up the mesh and hand out configs
  hosts: all
  become: true
  vars:
    wireguard_client_list:
      - name: laptop
        address: 10.77.0.2/32
    wireguard_client_endpoint: "{{ ansible_host }}:51820"
    wireguard_client_routes: 10.77.0.0/24
  tasks:
    - ansible.builtin.include_role:
        name: eugene_panin.base.wireguard_client

    - ansible.builtin.include_role:
        name: eugene_panin.base.wireguard
      vars:
        wireguard_peers: "{{ wireguard_client_peers }}"

    - ansible.builtin.include_role:
        name: eugene_panin.base.wireguard_client
```

Read [`wireguard_client`](roles/wireguard_client/README.md) before you use it:
it generates client private keys on the host and leaves them there, which is
not how WireGuard is normally set up.

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

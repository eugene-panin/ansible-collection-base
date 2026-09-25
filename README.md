# eugene_panin.base

[![CI](https://github.com/eugene-panin/ansible-collection-base/actions/workflows/ci.yml/badge.svg?branch=main)](https://github.com/eugene-panin/ansible-collection-base/actions/workflows/ci.yml)
[![Galaxy](https://img.shields.io/badge/dynamic/json?url=https%3A%2F%2Fgalaxy.ansible.com%2Fapi%2Fv3%2Fplugin%2Fansible%2Fcontent%2Fpublished%2Fcollections%2Findex%2Feugene_panin%2Fbase%2F&query=%24.highest_version.version&label=galaxy&color=blue&cacheSeconds=3600)](https://galaxy.ansible.com/ui/repo/published/eugene_panin/base/)
[![License: MIT](https://img.shields.io/badge/license-MIT-green)](LICENSE)

Host foundation layer — what a machine needs before any workload stack sits
on top of it. Deliberately usable on its own: nothing here depends on
Consul, Nomad, Vault or anything from another collection.

## Roles

| Role | Purpose |
|---|---|
| [`wireguard`](roles/wireguard/README.md) | WireGuard interface: generated-once key, declarative peers, `wg-quick@` unit |
| [`wireguard_client`](roles/wireguard_client/README.md) | Keys and importable `.conf` files for the devices that connect to it |
| [`firewall`](roles/firewall/README.md) | ufw: incoming denied by default, SSH and declared ports allowed |
| [`docker`](roles/docker/README.md) | Docker Engine only, pinned and held, with a validated `daemon.json` |

## Requirements

- ansible-core >= 2.19; CI runs 2.19 and the latest release
- `community.general` >= 10.0.0 (pulled in automatically)
- Ubuntu 22.04, Ubuntu 24.04 or Debian 12 on the managed host; CI runs every
  role on all three. WireGuard itself comes from the kernel; the roles install
  `wireguard-tools` only, without the packages it recommends, which on these
  systems include a kernel image.
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

Everything runs locally in Docker, the same way CI runs it:

```bash
make deps                                  # community.general
make lint                                  # yamllint and ansible-lint, production profile
make sanity                                # ansible-test sanity
make test ROLE=wireguard                   # every scenario of one role, Ubuntu 24.04
make test ROLE=wireguard SCENARIO='-s firewall'
make matrix ROLE=wireguard_client          # Ubuntu 24.04, 22.04 and Debian 12
make test-all                              # every role
```

The `wireguard` scenarios need the WireGuard kernel module on the Docker host.
Docker Desktop and GitHub's Ubuntu runners have it.

CI runs every role on three distributions and on the oldest supported
ansible-core. A release goes out only from a green run.

## License

MIT — see [LICENSE](LICENSE).

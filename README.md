# eugene_panin.base

Host foundation layer — what a machine needs before any workload stack sits
on top of it. Deliberately usable on its own: nothing here depends on
Consul, Nomad, Vault or anything from another collection.

## Roles

| Role | Purpose |
|---|---|
| `wireguard` | WireGuard interface: generated-once key, declarative peers, `wg-quick@` unit |

## Boundary

Roles in this collection never read variables belonging to another
collection. Anything that has to cross a collection boundary is declared in
the consuming inventory and passed in explicitly — that's what keeps this
collection installable on a host that has nothing else on it.

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

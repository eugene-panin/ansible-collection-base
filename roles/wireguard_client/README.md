# wireguard_client

Issues keys for the people and devices that connect to a WireGuard host, and
writes each one an importable `.conf`.

Run it twice in the same play, either side of the `wireguard` role. The first
pass mints a keypair per client and publishes `wireguard_client_peers`, which
is what you feed to `wireguard_peers`. The second pass renders the
configurations, which it can only do once the host key exists and the
endpoint is known.

```yaml
- name: Bring up the mesh and hand out configs
  hosts: gateway
  become: true
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

## Where the client keys live

On the host, in `/etc/wireguard/clients/`, root-only, and they stay there.

This is not how WireGuard is usually done. Normally a client generates its own
keypair and hands over nothing but the public half, so the host never sees a
key it could impersonate someone with. Here the host generates both, because
the point of the role is to produce a file you can import into a desktop or
phone app without touching a command line.

For a gateway you own, protecting machines you own, that trade is usually
fine — the host already holds a key that decrypts everything crossing the
tunnel. If it is not fine for you, skip this role: declare `wireguard_peers`
by hand with public keys your clients generated themselves.

The generated `.conf` contains a private key. Fetching it off the host is on
you, and so is its file mode once it lands.

## Variables

| Variable | Default | Purpose |
|---|---|---|
| `wireguard_client_list` | `[]` | Clients to issue keys and configs for |
| `wireguard_client_dir` | `/etc/wireguard/clients` | Where keys and configs are kept |
| `wireguard_client_endpoint` | `""` | `host:port` clients dial; nothing is rendered until it is set |
| `wireguard_client_routes` | `10.10.0.0/24` | What clients send through the tunnel, unless they say otherwise |
| `wireguard_client_dns` | `[]` | Resolvers and search domains written into the client |
| `wireguard_client_persistent_keepalive` | `25` | Seconds between keepalives |

Each entry in `wireguard_client_list` needs `name` and `address`, and may
override `routes`.

`routes` is what the client sends into the tunnel. Do not confuse it with a
peer's `allowed_ips` on the host, which is the address that client answers
at — the two point in opposite directions, and the role derives the second
one from `address` for you.

```yaml
wireguard_client_list:
  - name: laptop
    address: 10.77.0.2/32
  - name: phone
    address: 10.77.0.3/32
    routes: 0.0.0.0/0
```

## Notes

- Keys are generated with `creates:`, so a re-run does not mint new ones and
  invalidate configs people are already using. Removing a name from
  `wireguard_client_list` drops the peer from the host but leaves the key and
  config on disk; delete them yourself.
- `routes: 0.0.0.0/0` sends everything through the tunnel, which needs
  IP forwarding and NAT on the host. Neither this role nor `wireguard`
  configures that.
- `wireguard_client_endpoint` is rendered verbatim, so a hostname works as
  well as an address.

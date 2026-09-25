# firewall

ufw that denies incoming by default, allows SSH and the ports you declare, and
allows outgoing.

```yaml
- name: Firewall
  hosts: all
  become: true
  roles:
    - role: eugene_panin.base.firewall
      vars:
        firewall_allow:
          - port: 443
            interface: eth0
            comment: HTTPS
          - port: 25
            interface: eth0
            comment: SMTP
```

SSH is allowed before incoming is denied, so a run over SSH cannot lock
itself out; change `firewall_ssh_port` if SSH listens elsewhere. Rules other
roles add, such as the WireGuard port from the `wireguard` role, are left
alone.

## Tested

The scenario runs two hosts. The target serves two ports and allows one of
them on its interface; the other host checks that the allowed port answers
and the other one does not, and the target checks it still reaches out. A
rule with an interface and no port is covered by the `single_node` scenario
of `eugene_panin.hashistack`, where jobs on Nomad's bridge reach the host
through one.

## What it does not do

- Docker. Docker publishes container ports through its own iptables chains,
  which come before ufw's, so ufw does not stop a published port. Keep
  published ports off the public interface, as the `nomad` role of
  `eugene_panin.hashistack` does with `nomad_network_interface`.
- Rules it did not add are not removed.

## Variables

| Variable | Default | Purpose |
|---|---|---|
| `firewall_ssh_port` | `22` | SSH port, allowed first |
| `firewall_allow` | `[]` | Rules: `port`, `interface` or both, and optionally `proto`, `from`, `comment`; an interface alone allows all of it |

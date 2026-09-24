# docker

Installs Docker Engine and nothing else: `docker-ce`, `docker-ce-cli` and
`containerd.io` from Docker's apt repository. Written for a Nomad client,
whose `docker` driver needs a running daemon and nothing more.

```yaml
- name: Docker
  hosts: nomad_clients
  become: true
  roles:
    - role: eugene_panin.base.docker
```

## What it installs, and what it leaves out

The packages are installed without what they only recommend, so no buildx,
compose or rootless extras. The CLI comes along because the engine package
depends on it.

`docker_version` is an exact release, and the three packages are held, so an
unattended upgrade does not replace the engine and restart every container
behind your back. Changing `docker_version` upgrades in place and restarts
the daemon. The role restarts it itself rather than relying on the package
scripts, which a `policy-rc.d` can silence.

## Daemon configuration

`docker_daemon_config` is rendered to `/etc/docker/daemon.json` and checked
with `dockerd --validate` before it replaces the running one. A configuration
Docker rejects fails the run and the daemon keeps what it had.

The default logging driver is `local`. Docker's own default, `json-file`,
does not rotate, and a chatty container fills the disk.

## Tested

The scenario installs 29.8.0, upgrades to 29.8.1 and checks that the daemon
itself runs the new release. It checks the three packages are held and that
none of the recommended ones is installed, hands the role a configuration
with an unknown option and checks it was refused, then runs a container from
an image built on the spot, with no registry involved, and reads its output
back through the logging driver.

## What it does not do

- No users are added to the `docker` group. Membership in it is root on the
  host.
- Firewall. Docker publishes container ports through its own iptables chains,
  which come before ufw's. A port published on a public address is reachable
  no matter what ufw says. Bind published ports to a private address; for a
  Nomad client, set its `network_interface` to one.
- Debian and Ubuntu only.

## Variables

| Variable | Default | Purpose |
|---|---|---|
| `docker_version` | `29.8.1` | Exact Docker Engine release |
| `docker_daemon_config` | `{log-driver: local}` | Content of `daemon.json` |

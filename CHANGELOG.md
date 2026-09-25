# Changelog

All notable changes to this collection are documented here.
This project adheres to [Semantic Versioning](https://semver.org/).

## [0.4.1] - 2026-09-25

### Fixed

- `docker`: `--check` reported the repository key as changed on every run.
  On ansible-core 2.21 `get_url` downloads the file in check mode even with
  `force: false` and an existing destination. The key is now pinned by its
  SHA-256, `docker_gpg_key_sha256`, so an existing key is left alone, and a
  key other than the pinned one fails the run. Found by `--check` on a live
  host.

## [0.4.0] - 2026-09-25

### Added

- `firewall` role: ufw that denies incoming by default, allows SSH first and
  then the declared ports, optionally per interface, and allows outgoing.
  A rule with an interface and no port allows everything in on that
  interface, such as a container bridge. Tested with a second host that checks
  the allowed port answers and the other one does not.

## [0.3.4] - 2026-09-25

### Fixed

- `wireguard` and `wireguard_client` installed packages that pulled in a
  kernel image or DKMS with a compiler: `wireguard-tools` recommends a kernel
  module package, which on Ubuntu 22.04 and Debian 12 resolves to a
  `linux-image`, and the `wireguard` metapackage the `wireguard` role used on
  Debian and Ubuntu depends on `wireguard-dkms` where the kernel does not
  provide the module. WireGuard is in the kernel on every supported system;
  both roles now install `wireguard-tools` alone, without recommends, and the
  tests check no kernel or DKMS package came along.

## [0.3.3] - 2026-09-25

### Fixed

- `wireguard_client` ran `wg` without installing `wireguard-tools`, so on a
  host where the `wireguard` role had not run first it failed. Its test had
  installed the package in prepare and hid it; prepare now checks the package
  is absent. Found by the `single_node` playbook of `eugene_panin.hashistack`.

## [0.3.2] - 2026-09-24

### Changed

- The Molecule scenarios are no longer shipped in the Galaxy artifact.
- CI actions moved off Node 20: checkout v5, setup-python v6, upload-artifact v6.

## [0.3.1] - 2026-09-24

### Fixed

- `wireguard_client` tests did not catch three regressions they claimed to:
  rendering without the host key, rendering without the endpoint, and
  publishing a public key that does not belong to the client's private key.
  Each guard hid the other, and the key was only checked for its format.
  The scenario now holds back each of the two separately and compares the
  published key with `wg pubkey` of the private one. The role itself did not
  change.

## [0.3.0] - 2026-09-24

### Added

- `docker` role: Docker Engine from Docker's apt repository, and only the
  engine, its CLI and containerd. Pinned to an exact release and held against
  unattended upgrades. `daemon.json` is checked with `dockerd --validate`
  before it replaces the running one; the default logging driver is `local`,
  which rotates.

## [0.2.0] - 2026-09-23

### Breaking changes

- Requires ansible-core 2.19 or later and `community.general` 10.0.0 or later.
  0.1.0 claimed 2.15 and 8.0.0, neither of which was ever tested. CI now runs
  the roles on 2.19 and on the latest release.

### Changed

- CI runs every role on Ubuntu 22.04, Ubuntu 24.04 and Debian 12, the
  platforms the role metadata lists. 0.1.0 was only tested on 24.04.

## [0.1.0] - 2026-09-23

### Added

- `wireguard` role: installs WireGuard, generates the interface private key
  once, renders the interface configuration with declarative peers and
  manages the `wg-quick@<interface>` unit.
- `wireguard_client` role: issues a keypair per client on the WireGuard host
  and renders an importable configuration for each. Run it either side of
  `wireguard`, feeding `wireguard_client_peers` into `wireguard_peers`.
- `wireguard` publishes its interface public key as `wireguard_public_key`,
  which is what the client configurations point at.

### Fixed

- `--check` against a configured host reported the interface configuration as
  changed and queued a restart every time. The private key was not read in
  check mode, so the template rendered an empty `PrivateKey` and never matched
  what was on disk.
- `--check` against a host without WireGuard failed outright: the systemd unit
  does not exist until the package is installed, and the apt cache was left
  unrefreshed, so there was no installation candidate either.
- A failed `wg genkey` left an empty key file behind, which `creates:` then
  accepted on the next run.
- `wireguard_allow_mesh_traffic: false` only skipped the ufw task, so a rule
  added by an earlier run stayed in place.
- Installation gave up immediately when `unattended-upgrades` held the dpkg
  lock, which it does on a freshly built host.
- A peer added to the configuration while a later task failed the play never
  reached the running interface, and no re-run would put it there: handlers do
  not flush on failure, and the next run saw an unchanged template. The role
  now compares the peers the interface is serving against the configured ones
  and applies the difference with `wg syncconf`.

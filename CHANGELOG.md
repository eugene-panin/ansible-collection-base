# Changelog

All notable changes to this collection are documented here.
This project adheres to [Semantic Versioning](https://semver.org/).

## [Unreleased]

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

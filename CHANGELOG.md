# Changelog

All notable changes to this collection are documented here.
This project adheres to [Semantic Versioning](https://semver.org/).

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

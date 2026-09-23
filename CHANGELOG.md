# Changelog

All notable changes to this collection are documented here.
This project adheres to [Semantic Versioning](https://semver.org/).

## [Unreleased]

### Added

- `wireguard` role: installs WireGuard, generates the interface private key
  once, renders the interface configuration with declarative peers and
  manages the `wg-quick@<interface>` unit.

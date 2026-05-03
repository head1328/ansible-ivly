# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

## [0.1.0] - 2026-05-02

First tagged release. Part of the ivly platform v0.1.0 ("Featherweight").
Published to Ansible Galaxy as `head1328.ivly`.

### Added

- Ansible role that deploys the ivly invoicing stack on Raspberry Pi OS
  (Debian 13 / Trixie) using rootless Podman + Quadlet systemd units.
- Component image defaults pinned to the `0.1.0` release of every ivly
  service: `ivly/api:0.1.0`, `ivly/backoffice:0.1.0`,
  `ivly/printer:0.1.0`, `ivly/validation:0.1.0`. Supporting infra at
  `couchdb:3.5.1` and `caddy:2-alpine`.
- Caddy reverse proxy with optional automatic Let's Encrypt
  (`ivly_caddy_domain`) or Caddy internal CA for self-signed certs on
  internal domains (`ivly_caddy_tls_internal`).
- Optional `podman-auto-update` integration
  (`ivly_autoupdate_enabled`, time + random delay knobs) for floating
  image tags. Off by default since pinned tags do not benefit.
- Optional iptables port redirection (`ivly_iptables_enabled`) for
  binding to privileged ports < 1024 without root containers.
- Optional custom CA certificate injection into ivly-api
  (`ivly_custom_certs`) for internal TLS endpoints (e.g. an internal
  Caddy in front of a remote Ollama instance).
- Optional Ollama integration (`ivly_ollama_url`, `ivly_ollama_model`)
  for AI-assisted features.
- Localization knob `ivly_app_locale` (default `de-DE`).
- Molecule test suite using the Podman driver
  (`make test`, `make converge`, `make verify`, `make destroy`,
  `make login`).
- AGPL-3.0 LICENSE file with the full upstream license text plus a
  project-specific copyright block (`AGPL-3.0-or-later` per
  `meta/main.yml`).
- `.github/FUNDING.yml` and `.gitea/FUNDING.yml` listing Liberapay
  (`head1328`) and Patreon (`head1328`) for sponsoring.
- Sponsoring section in the README pointing at Liberapay.

### Dependencies

- `head1328.podman` Ansible role for the rootless Podman base setup
  (declared in `meta/main.yml`).

### Notes

- This role is published under the `head1328` namespace
  (Galaxy: `head1328.ivly`) instead of `ivly` because Ansible Galaxy
  requires GitHub-hosted source repositories and the ivly organisation
  itself only mirrors there from Codeberg. The role's source of truth
  remains <https://codeberg.org/ivly/ansible-ivly>.
- Image tags are pinned to `0.1.0` rather than `:latest` so a stack
  deployed by this role version is reproducible. Bumping individual
  components requires either overriding the `ivly_*_image` variable or
  releasing a new role version with updated defaults.

[Unreleased]: https://codeberg.org/ivly/ansible-ivly/compare/0.1.0...HEAD
[0.1.0]: https://codeberg.org/ivly/ansible-ivly/releases/tag/0.1.0

# Ansible Role: ivly

The official Ansible role that deploys the [ivly](https://codeberg.org/ivly) invoicing stack on Raspberry Pi OS (Debian 13 / Trixie) using containers.

## Description

This Ansible role deploys **ivly** as a set of containers using **Podman Quadlets**:

- **CouchDB** as document store
- **ivly-api** (Go) as backend API server
- **ivly-backoffice** (SvelteKit SPA via nginx) as frontend
- **ivly-printer** (Go + TeX Live) for LaTeX PDF rendering
- **ivly-validation** (Java) for XRechnung/ZUGFeRD/Factur-X XML validation
- **Caddy** as reverse proxy

The setup is intended for **internal networks** and does **not** rely on public
port exposure or Let's Encrypt (though Caddy supports it via `ivly_caddy_domain`).

## Architecture

```
Browser
  |
  v
https://ivly.example.local
  |
  v
Caddy (reverse proxy)
  |
  +-- /api/*   --> ivly-api:3000
  +-- /*       --> ivly-backoffice:8080

ivly-api:3000
  |
  +-- CouchDB        (ivly-couchdb:5984)
  +-- Printer         (ivly-printer:8080)
  +-- Validation      (ivly-validation:8080)
  +-- Ollama          (remote, configurable via ivly_ollama_url)
```

All containers communicate via a shared Podman network (`ivly`).
All images are pulled from **Docker Hub**.

## Assumed topology

The reference setup (see [`example/playbook.yml`](example/playbook.yml)) expects two Raspberry Pis:

| Host | Hardware | Role | Required? |
|------|----------|------|-----------|
| `ivly` | Raspberry Pi 4/5 | Runs the ivly stack via this role | yes |
| `ollama` | Raspberry Pi 5 + Hailo AI HAT+ | Runs Ollama behind its own Caddy via [`head1328.hailo`](https://galaxy.ansible.com/head1328/hailo) and [`head1328.hailo_ollama`](https://galaxy.ansible.com/head1328/hailo_ollama) | optional |

If you don't have a dedicated `ollama` host, point `ivly_ollama_url` at any reachable Ollama-compatible endpoint, or omit the LLM-related variables entirely - ivly will still work, just without LLM-backed assistance features.

## Key Characteristics

- Podman + systemd **Quadlets** (rootless)
- **Automatic container updates** via Podman auto-update (configurable)
- No host networking
- Caddy supports both plain HTTP and automatic Let's Encrypt
- Systemd service dependencies (CouchDB/Printer/Validation -> API -> Caddy)
- Lifecycle managed via **systemd handlers**
- LLM/chat features via remote **Ollama** endpoint (see `head1328.hailo` and `head1328.hailo_ollama` roles for setting up a private Ollama service)

## Requirements

- Podman with Quadlet support
- systemd
- Ansible collections:
  ```
  containers.podman
  community.general
  ```

## Dependencies

This role depends on `head1328.podman` which:
- Installs Podman
- Configures rootless Podman for a dedicated user
- Sets up user namespaces (subuid/subgid)
- Enables systemd user lingering
- Configures resource delegation for containers

The dependency is automatically installed when using `ansible-galaxy` with `requirements.yml`.

## Usage

### Installation

Install the role and its dependencies using Ansible Galaxy:

```bash
ansible-galaxy role install head1328.ivly
```

Or use a `requirements.yml`:

```yaml
---
roles:
  - name: head1328.ivly
```

Then install with:

```bash
ansible-galaxy install -r requirements.yml
```

### Playbook Example

See `example/playbook.yml` for a complete example. Minimal usage:

```yaml
- name: Deploy ivly
  hosts: ivly
  become: true
  roles:
    - role: head1328.ivly
      ivly_couchdb_password: "{{ vault_ivly_couchdb_password }}"
      ivly_caddy_domain: "ivly.example.local"
```

The `head1328.podman` dependency will be automatically applied before this role.

## Variables

Default values are defined in `defaults/main.yml`.

### CouchDB

| Variable | Default | Description |
|----------|---------|-------------|
| `ivly_couchdb_image` | `docker.io/couchdb:3.5.3` | CouchDB container image |
| `ivly_couchdb_user` | `admin` | CouchDB admin username |
| `ivly_couchdb_password` | `changeme` | CouchDB admin password (override!) |

### ivly services

| Variable | Default | Description |
|----------|---------|-------------|
| `ivly_api_image` | `docker.io/ivly/api:0.3.2` | API container image |
| `ivly_backoffice_image` | `docker.io/ivly/backoffice:0.3.2` | Backoffice container image |
| `ivly_printer_image` | `docker.io/ivly/printer:0.3.1` | Printer container image |
| `ivly_validation_image` | `docker.io/ivly/validation:0.2.0` | Validation container image |
| `ivly_ollama_url` | (empty) | Remote Ollama endpoint (optional) |
| `ivly_ollama_model` | (empty) | Ollama model name (optional) |
| `ivly_app_locale` | `de-DE` | Application locale |
| `ivly_banner_text_de` | (empty) | Operator notice in the sidebar, German, one line. A missing language falls back to the other one. |
| `ivly_banner_text_en` | (empty) | Operator notice in the sidebar, English, one line |
| `ivly_banner_level` | `info` | Notice level, `info` or `warning` |

### Caddy / Network

| Variable | Default | Description |
|----------|---------|-------------|
| `ivly_caddy_image` | `docker.io/caddy:2-alpine` | Caddy container image |
| `ivly_caddy_domain` | `:80` | `:80` for plain HTTP, FQDN for Let's Encrypt |
| `ivly_caddy_tls_internal` | `false` | Use Caddy's internal CA (self-signed) instead of Let's Encrypt - required for internal domains like `*.fritz.box`, `*.local`, etc. |
| `ivly_https_port` | `8443` | HTTPS port on host |
| `ivly_http_port` | `8080` | HTTP port on host |

### Podman / Auto-update

| Variable | Default | Description |
|----------|---------|-------------|
| `ivly_podman_user` | `podman` | Rootless Podman user |
| `ivly_autoupdate_enabled` | `false` | Enable automatic image updates (podman-auto-update). Only meaningful with floating tags such as `:latest`, `:1`, `:1.2`. Pinned tags like `:1.2.3` never change digest and would just waste bandwidth on the registry check. |
| `ivly_autoupdate_time` | `04:00` | Update schedule (systemd OnCalendar) |
| `ivly_autoupdate_random_delay` | `900` | Random delay in seconds |

### Custom CA certificates

| Variable | Default | Description |
|----------|---------|-------------|
| `ivly_custom_certs` | `[]` | List of PEM file paths (on the Ansible controller) to trust inside `ivly-api`, e.g. an internal CA in front of the Ollama endpoint |
| `ivly_custom_certs_host_path` | `/etc/ivly/extra-certs` | Where the certs are deposited on the host before being mounted into the container |

Sensitive values (e.g. `ivly_couchdb_password`) should be encrypted with Ansible Vault.

## TLS for Internal Domains

For deployments on internal networks (e.g. behind a Fritzbox using `*.fritz.box` domains, or `*.local` / `*.lan`), Let's Encrypt cannot issue certificates because the ACME servers cannot resolve or reach the domain from the public internet.

In that case, set `ivly_caddy_tls_internal: true`. Caddy will then act as its own CA and issue a self-signed certificate. The browser will show a security warning on first visit because the CA is not trusted by default.

### Trusting Caddy's Internal CA

To get rid of the browser warning, install Caddy's root certificate on each client machine.

#### 1. Extract the root certificate from the host

The certificate is stored inside the rootless Caddy container's data volume. The `podman` user typically has `/bin/nologin` as its shell and a non-readable home directory, so a few workarounds are needed.

From your local workstation (replace `<ivly-host>` with the hostname):

```bash
ssh <ivly-host> "cd /tmp && sudo -u podman bash -c \
  'XDG_RUNTIME_DIR=/run/user/\$(id -u) \
   podman cp ivly-caddy:/data/caddy/pki/authorities/local/root.crt /tmp/caddy-root.crt' \
  && sudo chmod 644 /tmp/caddy-root.crt"

scp <ivly-host>:/tmp/caddy-root.crt ./caddy-root.crt
```

Notes:
- `cd /tmp` before `sudo` avoids `cannot chdir to /home/<user>: Permission denied` when running as a regular user with a private home directory.
- `chmod 644` is required because `podman cp` writes the file with mode `0600`, owned by the `podman` user.

#### 2. Install on macOS

```bash
sudo security add-trusted-cert -d -r trustRoot \
  -k /Library/Keychains/System.keychain caddy-root.crt
```

Or via the GUI: open `caddy-root.crt`, then in **Keychain Access** find the **Caddy Local Authority** entry, double-click it, expand **Trust**, and set *When using this certificate* to **Always Trust**.

#### 3. Install on Linux (Debian/Ubuntu)

```bash
sudo cp caddy-root.crt /usr/local/share/ca-certificates/caddy-root.crt
sudo update-ca-certificates
```

For Firefox (uses its own cert store): **Settings -> Privacy & Security -> Certificates -> View Certificates -> Authorities -> Import**, then enable *Trust this CA to identify websites*.

#### 4. Install on Windows

Double-click `caddy-root.crt`, then **Install Certificate -> Local Machine -> Place all certificates in the following store -> Trusted Root Certification Authorities**.

#### 5. Install on iOS / Android

Mail the `caddy-root.crt` to your device, open it, then trust it via system settings. On iOS additionally enable full trust under **Settings -> General -> About -> Certificate Trust Settings**.

### Notes

- Caddy regenerates the root CA only if the data volume is wiped. Once trusted, the certificate is valid for ~10 years.
- If you redeploy ivly and lose the Caddy data volume, you must redistribute the new root CA.

## Trusting external CAs inside ivly-api

`ivly-api` must validate TLS connections to external services configured via `ivly_ollama_url` (and any future TLS endpoints). When that endpoint is fronted by a private CA - typical for an internal Ollama Caddy using `tls internal`, an enterprise PKI, or a self-signed certificate - the container's default trust store does not contain the issuer and requests fail with `x509: certificate signed by unknown authority`.

To make `ivly-api` trust additional CAs, set `ivly_custom_certs` to a list of PEM file paths on the Ansible controller:

```yaml
- role: head1328.ivly
  ivly_ollama_url: "https://ollama.internal/ollama"
  ivly_custom_certs:
    - "files/ca/internal-ollama-root.crt"
```

The role then:

1. Creates `ivly_custom_certs_host_path` on the host (default `/etc/ivly/extra-certs`).
2. Copies each listed PEM file into that directory.
3. Mounts the directory read-only into `ivly-api` at `/etc/ssl/certs/custom`.
4. Sets `SSL_CERT_DIR=/etc/ssl/certs/custom` in the container.

### Caveat: SSL_CERT_DIR replaces the system trust store

Go (and OpenSSL-style libraries) treat `SSL_CERT_DIR` as a replacement for the default trust paths, not an addition. As soon as `ivly_custom_certs` is non-empty, `ivly-api` only trusts the certificates you list. This is fine as long as `ivly-api` only talks to:

- Internal services on the shared Podman network (CouchDB, Validator, Printer - all over plain HTTP).
- The configured Ollama URL.

If `ivly-api` ever needs to reach a public TLS endpoint (e.g. a hosted LLM, a webhook, a mail gateway), include the relevant public roots (`/etc/ssl/certs/ca-certificates.crt` from a Debian system, or the Mozilla bundle) in `ivly_custom_certs` as well.

## Testing

This role uses [Molecule](https://molecule.readthedocs.io/) for testing with Podman.

### Prerequisites

- `podman` e.g. via Homebrew
- `python3` e.g. via Homebrew
- `pip install molecule "molecule-plugins[podman]"`

### Running Tests

```bash
# Full test suite (create, converge, idempotence, verify, destroy)
make test

# Development workflow
make converge  # Apply the role
make verify    # Run verification tests
make login     # Login to test instance
make destroy   # Clean up test instances

# Code quality
make lint      # Run ansible-lint
```

### Local Development

Link the role for local use without Galaxy:

```bash
make symlink
# Symlinks to ~/.ansible/roles/head1328.ivly
```

### CI/CD

This role uses [Woodpecker CI](https://woodpecker-ci.org/) for automated code quality checks on Codeberg. Linting with `ansible-lint` runs automatically on push and pull requests.

## Sponsoring

If `ansible-ivly` saved you a weekend - or you just like the idea of
self-hosted, container-based e-invoicing running quietly on a Raspberry
Pi - you can support continued maintenance via Liberapay. Donations go
directly into the time I spend on open-source work like this; one-off
or recurring, every bit helps.

[![LiberaPay](https://img.shields.io/liberapay/patrons/head1328.svg?logo=liberapay)](https://liberapay.com/head1328/donate)

## Authors

This role is maintained by Kevin Horst.
See the [AUTHORS](./AUTHORS) file for details.

## License

AGPL-3.0-or-later

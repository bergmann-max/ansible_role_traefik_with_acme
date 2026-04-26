# Ansible Role: Traefik with ACME

[![Ansible](https://img.shields.io/badge/ansible-%3E%3D%202.10-EE0000?logo=ansible&logoColor=white)](https://www.ansible.com/)
[![Platform](https://img.shields.io/badge/platform-Ubuntu-E95420?logo=ubuntu&logoColor=white)](https://ubuntu.com/)
[![Docker](https://img.shields.io/badge/docker-compose-2496ED?logo=docker&logoColor=white)](https://docs.docker.com/compose/)
[![ACME Cloudflare](https://img.shields.io/badge/ACME-Cloudflare-F38020?logo=cloudflare&logoColor=white)](https://www.cloudflare.com/)
[![Auth](https://img.shields.io/badge/auth-Basic%20Auth-grey?logo=webauthn&logoColor=white)]()
[![Auth](https://img.shields.io/badge/auth-Authentik-FD4B2D?logo=authentik&logoColor=white)](https://goauthentik.io/)
[![License](https://img.shields.io/badge/license-Unlicense-blue)](LICENSE)

This Ansible role deploys Traefik as a Docker Compose stack with automatic HTTPS via ACME DNS challenge using **Cloudflare**.

## Features

- Deploys Traefik as a Docker Compose stack
- Configures ACME DNS challenge via Cloudflare
- Creates static and dynamic Traefik configuration files
- Exposes the Traefik dashboard via HTTPS
- Protects the dashboard with HTTP Basic Authentication or Authentik forward auth
- Passes the Cloudflare API token securely via Docker Secret

## Requirements & Supported Platforms

- Ansible >= 2.10
- Ubuntu
- Docker Engine installed on the target host
- Docker Compose plugin available as `docker compose`
- A valid Cloudflare API token set via `traefik_with_acme_cf_token`

## Role Variables

The following variables can be set (see `defaults/main.yml`):

| Variable | Default | Description |
|---|---|---|
| `traefik_with_acme_email` | `mail@example.com` | Email address used for ACME certificate registration |
| `traefik_with_acme_traefik_user` | `""` | Linux system user that owns the deployment files |
| `traefik_with_acme_docker_release` | `traefik:v3.6.12` | Traefik image tag |
| `traefik_with_acme_fqdn` | `traefik.example.com` | Fully qualified domain name for the dashboard |
| `traefik_with_acme_docker_compose_dir` | `/opt/traefik` | Target directory for the Docker Compose project, configs, and ACME storage |
| `traefik_with_acme_cf_token` | `""` | Cloudflare API token |
| `traefik_with_acme_dashboard_users` | `""` | htpasswd string for dashboard Basic Auth — generate with `htpasswd -nb <user> <password>` |
| `traefik_with_acme_use_authentik` | `false` | Set to `true` to use Authentik as forward auth instead of Basic Auth |
| `traefik_with_acme_authentik_url` | `""` | URL of the Authentik forward auth endpoint, e.g. `https://authentik.example.com/outpost.goauthentik.io/auth/traefik` |

The following variable is set in `vars/main.yml` and is not intended to be overridden:

| Variable | Value | Description |
|---|---|---|
| `traefik_with_acme_docker_group` | `docker` | Linux group for file ownership and Docker access |

## Cloudflare Token

The token is written to `cf_dns_api_token.secret` (mode `u=r,g=,o=`) in the deployment directory and mounted into the Traefik container as a Docker Secret. Traefik reads it via the `CF_DNS_API_TOKEN_FILE` environment variable.

## Install
Add to your `requirements.yml`:

```yaml
roles:
  - name: traefik_with_acme
    src: git+ssh://git@github.com/bergmann-max/ansible_role_traefik_with_acme.git
    version: main
    scm: git
```

```bash
$ ansible-galaxy install -r requirements.yml
```

## Example Playbook
```yaml
- name: Apply template role
  hosts: all
  become: true

  roles:
    - role: template
      vars:
        example_var: "value"
```

```yaml
- name: Deploy Traefik with ACME via Cloudflare
  hosts: all
  become: true
  vars:
    traefik_with_acme_email: admin@example.com
    traefik_with_acme_traefik_user: traefik
    traefik_with_acme_fqdn: traefik.example.com
    traefik_with_acme_cf_token: "your-cloudflare-token"
    traefik_with_acme_dashboard_users: "admin:$apr1$..."
  roles:
    - role: ansible_role_traefik_with_acme
```

## Authentik Forward Auth

Instead of Basic Auth, the dashboard can be protected via [Authentik](https://goauthentik.io/) using Traefik's `forwardAuth` middleware.

### Ansible variables

```yaml
traefik_with_acme_use_authentik: true
traefik_with_acme_authentik_url: "https://authentik.example.com/outpost.goauthentik.io/auth/traefik"
```

When `traefik_with_acme_use_authentik` is `true`, the `traefik_with_acme_dashboard_users` variable is ignored.

### What to configure in Authentik

**1. Create a Proxy Provider**

Go to *Applications → Providers → Create* and choose **Proxy Provider**.

- Name: `traefik-dashboard` (or anything you like)
- Authorization flow: your default authorization flow (e.g. `default-provider-authorization-implicit-consent`)
- Mode: **Forward auth (single application)**
- External host: `https://traefik.example.com` (your dashboard URL)

**2. Create an Application**

Go to *Applications → Applications → Create*.

- Name: `Traefik Dashboard`
- Slug: `traefik-dashboard`
- Provider: select the provider you just created

**3. Create or assign an Outpost**

Go to *Applications → Outposts*.

- If you already have an embedded outpost, edit it and add the `traefik-dashboard` application to it.
- Otherwise create a new outpost of type **Proxy**, add the application, and deploy it.

The outpost exposes the forward auth endpoint at:
```
https://<authentik-domain>/outpost.goauthentik.io/auth/traefik
```
This is the URL you set in `traefik_with_acme_authentik_url`.

**4. Make sure Authentik is reachable from Traefik**

Traefik calls the Authentik URL on every request to the dashboard. Both services must be able to reach each other — either via Docker network or over the public URL. If Authentik runs behind the same Traefik instance, make sure it is already up before the dashboard is accessed.

### Usage example with Authentik

```yaml
- name: Deploy Traefik with Authentik forward auth
  hosts: all
  become: true
  vars:
    traefik_with_acme_email: admin@example.com
    traefik_with_acme_traefik_user: traefik
    traefik_with_acme_fqdn: traefik.example.com
    traefik_with_acme_cf_token: "your-cloudflare-token"
    traefik_with_acme_use_authentik: true
    traefik_with_acme_authentik_url: "https://authentik.example.com/outpost.goauthentik.io/auth/traefik"
  roles:
    - role: ansible_role_traefik_with_acme
```

## Tags

| Tag | Purpose |
|---|---|
| `traefik_prepare` | Create project directories and ACME storage |
| `traefik_config` | Template Traefik configuration files |
| `traefik_deploy` | Deploy Docker Compose files and Cloudflare secret file |
| `traefik_start` | Start the Traefik stack via `docker compose up -d` |

## Notes

- The dashboard is exposed on `https://<fqdn>/dashboard/`
- Visiting `https://<fqdn>/` redirects automatically to `/dashboard/`
- The Traefik container publishes ports `80`, `443`, and `8080`

## License

Unlicense

## Author Information

Max Bergmann

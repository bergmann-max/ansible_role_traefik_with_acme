# Ansible Role: ansible_role_traefik_with_acme

[![Ansible](https://img.shields.io/badge/ansible-%3E%3D%202.11-EE0000?logo=ansible&logoColor=white)](https://www.ansible.com/)
[![Platform](https://img.shields.io/badge/platform-Ubuntu-E95420?logo=ubuntu&logoColor=white)](https://ubuntu.com/)
[![Docker](https://img.shields.io/badge/docker-compose-2496ED?logo=docker&logoColor=white)](https://docs.docker.com/compose/)
[![License](https://img.shields.io/badge/license-Unlicense-blue)](LICENSE)

This Ansible role deploys Traefik as a Docker Compose stack with automatic HTTPS via ACME DNS challenge using **Cloudflare**. A systemd unit manages the stack lifecycle on the target host.

## Features

- Deploys Traefik as a Docker Compose stack
- Configures ACME DNS challenge via Cloudflare
- Creates static and dynamic Traefik configuration files
- Exposes the Traefik dashboard via HTTPS
- Protects the dashboard with HTTP Basic Authentication
- Installs a systemd unit for autostart and lifecycle management

## Requirements & Supported Platforms

- Ansible >= 2.11
- Ubuntu
- Docker Engine installed on the target host
- Docker Compose plugin available as `docker compose`
- A valid Cloudflare API token provided in `.env`

## Role Variables

The following variables can be set (see `defaults/main.yml` and `vars/main.yml`):

| Variable | Default | Description |
|---|---|---|
| `ansible_role_traefik_with_acme_email` | `mail@example.com` | Email address used for ACME certificate registration |
| `ansible_role_traefik_with_acme_traefik_user` | `traefik` | Linux system user that owns the deployment files and runs the systemd service |
| `ansible_role_traefik_with_acme_traefik_group` | `docker` | Linux group for file ownership and Docker access |
| `ansible_role_traefik_with_acme_docker_release` | `latest` | Traefik image tag |
| `ansible_role_traefik_with_acme_domain` | `example.com` | Base domain used for dashboard routing |
| `ansible_role_traefik_with_acme_subdomain` | `traefik` | Subdomain for the dashboard, for example `traefik.example.com` |
| `ansible_role_traefik_with_acme_docker_compose_dir` | `/opt/traefik` | Target directory for the Docker Compose project, configs, and ACME storage |

## Usage Example

```yaml
- name: Deploy Traefik with ACME via Cloudflare
  hosts: all
  become: true
  vars:
    ansible_role_traefik_with_acme_email: admin@example.com
    ansible_role_traefik_with_acme_domain: example.com
    ansible_role_traefik_with_acme_subdomain: traefik
  roles:
    - role: ansible_role_traefik_with_acme
```

Before the first run, provide the Cloudflare token in the deployment directory:

```bash
cp /opt/traefik/example.env /opt/traefik/.env
```

Required variable in `.env`:

```dotenv
CF_DNS_API_TOKEN=your-cloudflare-token
```

When no `.env` exists yet, the role copies an `example.env` template into `{{ ansible_role_traefik_with_acme_docker_compose_dir }}`.

## Tags

| Tag | Purpose |
|---|---|
| `traefik_user_setup` | Create the Traefik system user and group |
| `traefik_prepare` | Create project directories and ACME storage |
| `traefik_config` | Template Traefik configuration files |
| `traefik_deploy` | Deploy Docker Compose files and environment template |
| `traefik_service_setup` | Install and enable the systemd service |

## Notes

- The dashboard is exposed on `https://<subdomain>.<domain>/dashboard/`
- The role currently templates a default Basic Auth user in `templates/dashboard.yml.j2`; change it before production use
- The Traefik container publishes ports `80`, `443`, and `8080`

## License

Unlicense

## Author Information

Role maintained by Max Bergmann.

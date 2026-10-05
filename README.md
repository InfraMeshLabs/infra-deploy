# InfraMesh Deployment

InfraMesh Deploy provides Docker Compose based deployment for InfraMesh components. Each component has its own directory with a self-contained Compose project.

| Component | Directory | Status |
| --- | --- | --- |
| Console (with PostgreSQL and Redis) | [`console/`](console) | Available |
| Router | — | Planned |
| Worker | — | Planned |

The Console runs from the official public image — no source checkout, build, or registry login is required.

```text
ghcr.io/inframeshlabs/infra-console
```

## Quick Start (Console)

Requires Docker Engine with Docker Compose v2 (`docker compose`).

```bash
git clone https://github.com/InfraMeshLabs/infra-deploy.git
cd infra-deploy/console

cp .env.example .env

# Edit .env and set the required secrets:
#   POSTGRES_PASSWORD, JWT_SECRET_KEY, ENCRYPTION_KEY, INFRA_ADMIN_PASSWORD

docker compose up -d
```

Then open <http://localhost:8080> and sign in with the bootstrap admin account (`INFRA_ADMIN_USERNAME` / `INFRA_ADMIN_PASSWORD`).

Generate the two keys with:

```bash
openssl rand -base64 48   # JWT_SECRET_KEY
openssl rand -base64 32   # ENCRYPTION_KEY (must be exactly 32 bytes; never change it afterwards)
```

`docker compose up` refuses to start while a required value is empty.

## Deployment Architecture

```text
                InfraMesh Deploy

             ┌──────────────────┐
             │ InfraMesh Console│
             │       :8080      │
             └────────┬─────────┘
                      │
             ┌────────┴────────┐
             │                 │
             ▼                 ▼
        PostgreSQL           Redis
```

| Service | Image | Notes |
| --- | --- | --- |
| `console` | `ghcr.io/inframeshlabs/infra-console:${INFRA_CONSOLE_VERSION:-0.1.1}` | Frontend, REST API and Node WebSocket on HTTP `:8080`. Runs with the `prod` profile. |
| `postgres` | `postgres:16` | Database `infra_mesh`. Data persisted in the `postgres-data` volume. |
| `redis` | `redis:7.4` | Session Affinity and Worker In-Flight Capacity. TTL-based runtime state only, so no volume. |

Only the Console port is published to the host. Change it with `INFRA_CONSOLE_PORT` in `console/.env`.

## Console Version

The Console image tag is set in `console/.env`:

```env
INFRA_CONSOLE_VERSION=0.1.1
```

The default is a pinned version so that a deployment always reproduces the same Console. To update, change the version and run in `console/`:

```bash
docker compose pull
docker compose up -d
```

`INFRA_CONSOLE_VERSION=latest` is also available if you explicitly want to track the newest release. See [docs/console/upgrade.md](docs/console/upgrade.md).

## Connecting AI Runtimes

Inside the Console container, `localhost` is the container itself. Register runtimes with an address that matches where they run:

| Runtime location | Address to register | Example |
| --- | --- | --- |
| Docker host | `http://host.docker.internal:<port>` | Ollama: `http://host.docker.internal:11434`<br>vLLM: `http://host.docker.internal:8000/v1` |
| Same Docker network | `http://<service-name>:<port>` | `http://ollama:11434` |
| Another server | IP or domain of that server | `http://192.168.x.x:11434`, `https://runtime.example.com` |

The Console uses the registered address as-is; it never rewrites `localhost`. See [docs/console/installation.md](docs/console/installation.md#connecting-ai-runtimes).

## HTTPS

The Console serves plain HTTP on `:8080`, and this repository does not bundle a reverse proxy. For production, terminate TLS in front of it:

```text
Internet
   │
   ▼
Nginx / Caddy / Traefik / Load Balancer
   │ HTTPS
   ▼
InfraMesh Console :8080
```

See [docs/console/installation.md](docs/console/installation.md#https-and-reverse-proxy) for proxy requirements.

## Overview

InfraMesh is a distributed AI inference platform designed to connect and utilize heterogeneous GPU and CPU resources across multiple nodes.

The platform is composed of the following components:

```text
                         InfraMesh
                             │
                          Console
                             │
                    ┌────────┴────────┐
                    │                 │
                 Router            Worker
                    │                 │
                    └────────┬────────┘
                             │
                         AI Runtime
```

- **Console** — Control plane and inference orchestration layer.
- **Node SDK** — Common contracts and integrations used by Router and Worker nodes.
- **Router** — Selects an appropriate Worker for an inference request.
- **Worker** — Executes AI inference using the configured runtime.

This repository currently deploys the **Console**. Router and Worker deployments will be added as separate directories next to `console/`.

## Repository Scope

| Repository | Responsible for |
| --- | --- |
| [`infra-console`](https://github.com/InfraMeshLabs/infra-console) | Console application (frontend, backend), Dockerfile, official image build and GHCR publishing |
| `infra-deploy` (this repository) | Docker Compose, environment template, PostgreSQL, Redis, deployment documentation |

This repository never rebuilds the Console image.

## Structure

```text
infra-deploy
├── README.md
├── console/
│   ├── .env.example
│   └── docker-compose.yml
└── docs/
    └── console/
        ├── installation.md
        ├── configuration.md
        └── upgrade.md
```

## Related Projects

| Project | Description |
| --- | --- |
| [`infra-console`](https://github.com/InfraMeshLabs/infra-console) | Control plane and inference orchestration layer |
| [`infra-node`](https://github.com/InfraMeshLabs/infra-node) | Common SDK, contracts, and integrations for InfraMesh nodes |
| [`infra-router`](https://github.com/InfraMeshLabs/infra-router) | Reference implementation for Router nodes |
| [`infra-worker`](https://github.com/InfraMeshLabs/infra-worker) | Reference implementation for Worker nodes |

## Documentation

Deployment files and documentation are grouped per component: `<component>/` and `docs/<component>/`.

### Console

- [Installation](docs/console/installation.md) — first run, runtime addresses, HTTPS, operations
- [Configuration](docs/console/configuration.md) — every `.env` variable
- [Upgrade](docs/console/upgrade.md) — updating the Console version, backups

## License

Licensed under the **Apache License 2.0**.

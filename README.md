# InfraMesh

InfraMesh is a distributed AI inference platform designed to connect and utilize heterogeneous GPU and CPU resources across multiple nodes.

This repository is the official download and deployment entry point for **InfraMesh Console**. It provides:

- Standalone packages for macOS and Windows, published as [GitHub Release](https://github.com/InfraMeshLabs/infra-deploy/releases) assets
- A Docker Compose deployment (Console + PostgreSQL, optional Redis)
- Installation, configuration, upgrade and security documentation

## Deployment Options

### Standalone Distribution

Download, extract and run. Available from Console v0.2.0.

| OS | Architecture | Package |
| --- | --- | --- |
| macOS | Apple Silicon (arm64) | `inframesh-console-<version>-macos-arm64.tar.gz` |
| macOS | Intel (x64) | `inframesh-console-<version>-macos-x64.tar.gz` |
| Windows | x64 | `inframesh-console-<version>-windows-x64.zip` |

Each package ships with a `.sha256` checksum file. There is no Linux standalone package; use Docker on Linux.

### Docker Deployment

Runs on any environment with Docker Engine and Docker Compose v2 — Linux, macOS (Docker Desktop) and Windows (Docker Desktop).

```text
ghcr.io/inframeshlabs/infra-console
```

The image is public; no source checkout, build, or registry login is required.

### Comparison

| | Standalone | Docker |
| --- | --- | --- |
| Java installation | Not required (runtime bundled) | Not required (runtime in image) |
| Docker installation | Not required | Required |
| Database | Embedded H2 (file-based) | PostgreSQL |
| Redis | Optional, not bundled | Optional (`affinity` / `redis` Compose profiles) |
| Supported environments | macOS (arm64, x64), Windows x64 | Any Docker-supported environment |
| Data persistence | `INFRAMESH_HOME` directory | `postgres-data` Docker volume |
| Network binding | `127.0.0.1` by default | Published host port (`INFRA_CONSOLE_PORT`) |
| Installation | Extract the archive and run the launcher | `docker compose up -d` |
| Intended use | A single Console on a local PC | Servers, Linux, shared environments |

Redis is not included in the standalone package. Features that rely on Redis (Session Affinity, shared Worker Capacity across several Consoles) need a separately operated Redis server. Without Redis, the Console runs normally with those features unavailable.

## Quick Start

### Standalone Installation

1. Download the package for your platform and its `.sha256` file from the [Releases page](https://github.com/InfraMeshLabs/infra-deploy/releases).
2. Verify the checksum, extract, and run the launcher.

macOS:

```bash
shasum -a 256 -c inframesh-console-0.2.0-macos-arm64.tar.gz.sha256
tar -xzf inframesh-console-0.2.0-macos-arm64.tar.gz
./inframesh-console-0.2.0/bin/inframesh
```

Windows (after verifying the checksum and extracting the ZIP):

```bat
inframesh-console-0.2.0\bin\inframesh.cmd
```

When the Console is ready, your browser opens `http://127.0.0.1:8080`. On the first launch you create the administrator account in the browser. Press `Ctrl+C` in the terminal to stop the Console.

The packages are not code-signed yet — see [Standalone Installation](docs/console/standalone.md) for Gatekeeper / SmartScreen notes and full instructions.

### Docker Installation

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

Redis is optional infrastructure and is not started by default. It may be used for Session Affinity and for shared Worker Capacity coordination across multiple Consoles — see [Redis](docs/console/installation.md#redis-optional). Without Redis, Session Affinity is unavailable and Worker Capacity is kept in the Console's memory.

See [Docker Installation](docs/console/installation.md) for the full guide.

#### Docker Architecture

```text
             ┌──────────────────┐
             │ InfraMesh Console│
             │       :8080      │
             └────────┬─────────┘
                      │
             ┌────────┴────────┐
             │                 │
             ▼                 ▼
        PostgreSQL           Redis (optional)
```

```text
Required                 Optional
- Console                - Redis
- PostgreSQL                 - Session Affinity
                             - Shared Worker Capacity coordination
```

| Service | Image | Notes |
| --- | --- | --- |
| `console` | `ghcr.io/inframeshlabs/infra-console:${INFRA_CONSOLE_VERSION:-0.1.1}` | Frontend, REST API and Node WebSocket on HTTP `:8080`. Runs with the `prod` profile. |
| `postgres` | `postgres:16` | Database `infra_mesh`. Data persisted in the `postgres-data` volume. |
| `redis` | `redis:7.4` | Optional, only started with `--profile affinity` or `--profile redis`. Holds Session Affinity and shared Worker Capacity state (TTL-based, no volume). |

Only the Console port is published to the host. Change it with `INFRA_CONSOLE_PORT` in `console/.env`.

Router and Worker nodes are not part of this stack. To run actual inference, deploy or connect them separately (see [Related Projects](#related-projects)).

## Configuration

| Deployment | Where | Reference |
| --- | --- | --- |
| Standalone | Environment variables (`INFRAMESH_HOME`, `SERVER_PORT`, `INFRAMESH_OPEN_BROWSER`, …) | [Standalone — Configuration](docs/console/standalone.md#configuration) |
| Docker | `console/.env` | [Configuration](docs/console/configuration.md) |

### Docker Console Version

The Console image tag is set in `console/.env`:

```env
INFRA_CONSOLE_VERSION=0.1.1
```

The default is a pinned version so that a deployment always reproduces the same Console. `INFRA_CONSOLE_VERSION=latest` is also available if you explicitly want to track the newest release. See [Upgrading](docs/console/upgrade.md).

### Connecting AI Runtimes

The address you register for a runtime depends on where the Console runs.

- **Standalone:** the Console runs directly on your PC, so a runtime on the same PC is reachable at `http://localhost:<port>` (for example Ollama at `http://localhost:11434`).
- **Docker:** inside the Console container, `localhost` is the container itself. Register runtimes with an address that matches where they run:

| Runtime location | Address to register | Example |
| --- | --- | --- |
| Docker host | `http://host.docker.internal:<port>` | Ollama: `http://host.docker.internal:11434`<br>vLLM: `http://host.docker.internal:8000/v1` |
| Same Docker network | `http://<service-name>:<port>` | `http://ollama:11434` |
| Another server | IP or domain of that server | `http://192.168.x.x:11434`, `https://runtime.example.com` |

The Console uses the registered address as-is; it never rewrites `localhost`. See [docs/console/installation.md](docs/console/installation.md#connecting-ai-runtimes).

## Data Persistence

| Deployment | Persistent data | Survives |
| --- | --- | --- |
| Standalone | `INFRAMESH_HOME` (default `~/.inframesh`, Windows `%USERPROFILE%\.inframesh`) — kept separate from the extracted package | Restarts, and package upgrades as long as the directory is kept |
| Docker | `postgres-data` volume | `docker compose stop` / `down` / image updates. **`docker compose down -v` deletes it.** |

Back up before every upgrade. See [Standalone — Data Directory](docs/console/standalone.md#data-directory) and [Docker — Backup and Restore](docs/console/installation.md#backup-and-restore).

## Upgrading

- **Standalone:** stop the Console, back up `INFRAMESH_HOME`, extract the new release into a new directory and run it. The existing data directory is reused.
- **Docker:** back up PostgreSQL, change `INFRA_CONSOLE_VERSION`, then `docker compose pull` and `docker compose up -d`.

See [Upgrading](docs/console/upgrade.md).

## Security

- The Console serves plain HTTP. The standalone Console listens on `127.0.0.1` only by default; do not expose either deployment directly to untrusted networks without a reverse proxy that terminates TLS.
- The standalone packages are not code-signed or notarized yet. Download them only from the [Releases page](https://github.com/InfraMeshLabs/infra-deploy/releases) and verify the SHA-256 checksum. A checksum confirms the file matches the published checksum; it does not replace code signing or publisher verification.
- Keep `.env`, the standalone `INFRAMESH_HOME` directory, and backups private — they contain secrets and encrypted keys.

```text
Internet
   │
   ▼
Nginx / Caddy / Traefik / Load Balancer
   │ HTTPS
   ▼
InfraMesh Console :8080
```

See [Security](docs/console/security.md) and [HTTPS and Reverse Proxy](docs/console/installation.md#https-and-reverse-proxy).

## Documentation

### Console

- [Standalone Installation](docs/console/standalone.md) — macOS / Windows packages, first run, configuration, data directory
- [Docker Installation](docs/console/installation.md) — Docker Compose, Redis, runtime addresses, HTTPS, operations, backup
- [Configuration](docs/console/configuration.md) — every Docker `.env` variable
- [Upgrading](docs/console/upgrade.md) — upgrading standalone and Docker deployments
- [Security](docs/console/security.md) — network exposure, TLS, secrets, package integrity

### Repository

- [Releases](docs/releases.md) — release channels, assets, checksums, versioning

## Releases

Official releases are published on the [GitHub Releases page](https://github.com/InfraMeshLabs/infra-deploy/releases). Standalone packages and their `.sha256` files are release assets; they are never committed to this repository. Docker images are published to `ghcr.io/inframeshlabs/infra-console`. See [docs/releases.md](docs/releases.md).

## Overview

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

This repository currently deploys the **Console**.

| Component | Standalone | Docker | Status |
| --- | --- | --- | --- |
| Console | Release assets | [`console/`](console) (with PostgreSQL, optional Redis) | Available |
| Router | — | — | Planned |
| Worker | — | — | Planned |

## Repository Scope

| Repository | Responsible for |
| --- | --- |
| `infra-console` (private) | Console source code, official image build and GHCR publishing, standalone package build and release promotion |
| `infra-deploy` (this repository) | Public releases and downloads, Docker Compose, environment template, deployment documentation |

This repository never rebuilds the Console image or the standalone packages.

## Structure

```text
infra-deploy
├── README.md
├── console/
│   ├── .env.example
│   └── docker-compose.yml
└── docs/
    ├── releases.md
    └── console/
        ├── standalone.md
        ├── installation.md
        ├── configuration.md
        ├── upgrade.md
        └── security.md
```

## Related Projects

| Project | Description |
| --- | --- |
| [`infra-node`](https://github.com/InfraMeshLabs/infra-node) | Common SDK, contracts, and integrations for InfraMesh nodes |
| [`infra-router`](https://github.com/InfraMeshLabs/infra-router) | Reference implementation for Router nodes |
| [`infra-worker`](https://github.com/InfraMeshLabs/infra-worker) | Reference implementation for Worker nodes |

## License

Licensed under the **Apache License 2.0**.

# Installation

Runs InfraMesh Console, PostgreSQL and Redis with Docker Compose. The Compose project lives in [`console/`](../../console); run every `docker compose` command below from that directory.

```text
              InfraMesh Console  (Frontend + REST API + Node WebSocket, :8080)
                 /          \
                ▼            ▼
          PostgreSQL        Redis
```

## Requirements

- Docker Engine with Docker Compose v2 (`docker compose`)
- Network access to `ghcr.io` and Docker Hub to pull images

Not required: an `infra-console` checkout, Java, Node.js, Gradle, a Docker build, `docker login ghcr.io`, or a GitHub Personal Access Token. The Console image is public:

```bash
docker pull ghcr.io/inframeshlabs/infra-console:0.1.1
```

## Install

```bash
git clone https://github.com/InfraMeshLabs/infra-deploy.git
cd infra-deploy/console

cp .env.example .env
```

Set the required values in `.env`:

| Variable | How to set it |
| --- | --- |
| `POSTGRES_PASSWORD` | Any strong password |
| `JWT_SECRET_KEY` | `openssl rand -base64 48` |
| `ENCRYPTION_KEY` | `openssl rand -base64 32` (Base64 of exactly 32 bytes) |
| `INFRA_ADMIN_PASSWORD` | Password of the bootstrap admin account |

Then start the stack:

```bash
docker compose up -d
docker compose ps              # all three services should become healthy
docker compose logs -f console
```

Open `http://localhost:8080` (or the port set in `INFRA_CONSOLE_PORT`) and sign in with `INFRA_ADMIN_USERNAME` / `INFRA_ADMIN_PASSWORD`.

Notes:

- `.env` holds secrets and is ignored by Git. Never commit or share it.
- `docker compose up` fails immediately if a required value is empty.
- The Console starts after PostgreSQL and Redis are healthy, and creates its schema with Flyway on startup. An empty database is fine.
- The image runs with `SPRING_PROFILES_ACTIVE=prod`. Do not override it; the `local` profile and its development defaults are not part of the image.
- PostgreSQL and Redis are not published to the host. Only the Console port is.

See [configuration.md](configuration.md) for every variable.

## Connecting AI Runtimes

Inside the Console container, `localhost` means **the Console container itself**, not the Docker host. The address you register for a runtime depends on where it runs.

### Runtime on the Docker host

```text
Console Container
       │
       ▼
host.docker.internal
       │
       ▼
Ollama / vLLM
```

| Runtime | Address on the host | Address to register in the Console |
| --- | --- | --- |
| Ollama | `http://localhost:11434` | `http://host.docker.internal:11434` |
| vLLM (OpenAI-compatible API) | `http://localhost:8000/v1` | `http://host.docker.internal:8000/v1` |

The `console` service declares `extra_hosts: host.docker.internal:host-gateway`, so the same address works on Linux as well as Docker Desktop.

On Linux, a runtime bound only to `127.0.0.1` is not reachable from a container. Make it listen on the Docker bridge address too (for Ollama: `OLLAMA_HOST=0.0.0.0`). On Docker Desktop (macOS/Windows) a `127.0.0.1` binding is reachable.

### Runtime on the same Docker network

If the runtime runs in the same Compose project or Docker network, use its service name:

```text
http://ollama:11434
```

### Runtime on another server

Use the IP address or domain of that server:

```text
http://192.168.x.x:11434
https://runtime.example.com
```

Only the host part of the address changes between these cases. The Console uses the registered address exactly as entered and never converts `localhost` to `host.docker.internal`.

## HTTPS and Reverse Proxy

The Console image serves HTTP on `:8080`. This repository does not include Nginx, Caddy or any other proxy. If you need HTTPS, put your own reverse proxy or load balancer in front:

```text
Internet
   │
   ▼
Nginx / Caddy / Traefik / Load Balancer
   │ HTTPS
   ▼
InfraMesh Console :8080
```

The proxy must:

- Forward `X-Forwarded-*` headers. The Console uses them to mark the refresh token cookie as `Secure`.
- Pass WebSocket upgrades for `/ws/v1/nodes/connect` (used by OUTBOUND Worker/Router nodes) and keep idle connections open longer than the node heartbeat interval (10 seconds by default).

OUTBOUND nodes connect to the same port as everything else:

```text
OUTBOUND Worker / Router ──WebSocket──▶ ws://<console-host>:8080/ws/v1/nodes/connect
```

## Operations

```bash
docker compose stop          # stop containers, keep data
docker compose down          # remove containers and network, keep the volume
docker compose down -v       # also delete the volume - all database data is lost
```

| Volume | Contents | If deleted |
| --- | --- | --- |
| `postgres-data` | Organizations, teams, accounts, nodes, audit logs and all other persistent data | Everything is lost |

The Console container itself is stateless. Redis holds only TTL-based runtime state (Session Affinity, Worker In-Flight Capacity) that is rebuilt after a restart, so it has no volume.

To update the Console, see [upgrade.md](upgrade.md).

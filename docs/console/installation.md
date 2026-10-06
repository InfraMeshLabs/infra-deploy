# Installation

Runs InfraMesh Console and PostgreSQL with Docker Compose. Redis is [optional](#redis-optional): it is only needed for Session Affinity or shared Worker Capacity. The Compose project lives in [`console/`](../../console); run every `docker compose` command below from that directory.

```text
              InfraMesh Console  (Frontend + REST API + Node WebSocket, :8080)
                 /          \
                ▼            ▼
          PostgreSQL        Redis (optional)
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
- The Console starts after PostgreSQL is healthy (and Redis, when started with the `affinity` profile), and creates its schema with Flyway on startup. An empty database is fine.
- The image runs with `SPRING_PROFILES_ACTIVE=prod`. Do not override it; the `local` profile and its development defaults are not part of the image.
- PostgreSQL and Redis are not published to the host. Only the Console port is.
- The default stack has no Redis. Authentication, organizations and teams, node management, every routing strategy and inference all work without it.

## Redis (optional)

Redis is optional infrastructure. It may be used for:

- **Session Affinity** — sends requests that carry the same `sessionId` to the same Worker when possible.
- **Shared Worker Capacity coordination** — lets several Console instances share each Worker's concurrency limit.

The two are independent settings; enable either, both, or neither. Without Redis, Session Affinity is unavailable and Worker Capacity falls back to the Console's memory.

### Session Affinity

Off by default.

To enable it, set this in `.env`:

```env
INFRAMESH_SESSION_AFFINITY_ENABLED=true
```

and start the stack with the `affinity` profile so Redis runs too:

```bash
docker compose --profile affinity up -d
```

- Do both. With the value set to `true` but no profile, the Console still starts, logs an ERROR that Redis is unreachable, and routes without Session Affinity.
- To avoid typing `--profile` every time, add `COMPOSE_PROFILES=affinity` to `.env`.
- Once enabled, each Team can turn Session Affinity on or off in its settings (on by default). While it is disabled on the Console, the Team setting has no effect but its stored value is kept.
- Use the same profile when stopping so Redis is removed as well: `docker compose --profile affinity down`.
- If Redis fails while Session Affinity is enabled, inference keeps working: the Console logs the failure and routes as if there were no affinity.

### Shared Worker Capacity

Each Worker's concurrency limit is enforced with in-flight reservations. By default (`WORKER_CAPACITY_MODE=MEMORY`) they live in the Console's memory, which needs no Redis but is only accurate with a single Console instance. To share the limit across several Console instances, set:

```env
WORKER_CAPACITY_MODE=REDIS
```

and start Redis with `docker compose --profile redis up -d` (the `affinity` and `redis` profiles start the same Redis).

- Unlike Session Affinity, this does not fail open. If Redis is unreachable in `REDIS` mode, Workers that have a concurrency limit stop receiving requests (`429`) until Redis is back; Workers without a limit keep working. The Console logs an ERROR at startup when it cannot reach Redis.
- There is no automatic mode: the Console never switches stores on its own.

These settings require a Console image that supports `INFRAMESH_SESSION_AFFINITY_ENABLED` and `WORKER_CAPACITY_MODE`. Older images use Redis unconditionally and need Redis to be running.

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

The Console container has no persistent state. Redis (optional) holds only TTL-based state (Session Affinity, and Worker in-flight capacity in `REDIS` mode) that is rebuilt after a restart, so it has no volume. With the default `WORKER_CAPACITY_MODE=MEMORY`, Worker in-flight capacity lives in the Console's memory and is not shared between Console instances.

To update the Console, see [upgrade.md](upgrade.md).

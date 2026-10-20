# Docker Installation

Runs InfraMesh Console and PostgreSQL with Docker Compose. Redis is [optional](#redis-optional): it is only needed for Session Affinity or shared Worker Capacity. The Compose project lives in [`console/`](../../console); run every `docker compose` command below from that directory.

To run the Console on a single macOS or Windows PC without Docker, see [Standalone Installation](standalone.md).

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
docker pull ghcr.io/inframeshlabs/infra-console:0.2.2
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
docker compose ps              # console and postgres should become healthy
docker compose logs -f console
```

Open `http://localhost:8080` (or the port set in `INFRA_CONSOLE_PORT`) and sign in with `INFRA_ADMIN_USERNAME` / `INFRA_ADMIN_PASSWORD`.

Notes:

- `.env` holds secrets and is ignored by Git. Never commit or share it.
- `docker compose up` fails immediately if a required value is empty.
- The Console starts after PostgreSQL is healthy, and creates its schema with Flyway on startup. An empty database is fine.
- When Redis is started with the `affinity` or `redis` profile, Compose also waits for it to become healthy. The Redis dependency is optional (`required: false`), so without a profile it is ignored, and if Redis does not start or become healthy Compose warns instead of failing — check `docker compose ps` and the Console log.
- The image runs with `SPRING_PROFILES_ACTIVE=prod`. Do not override it; the `local` profile and its development defaults are not part of the image.
- PostgreSQL and Redis are not published to the host. Only the Console port is.
- The default stack has no Redis. Authentication, organizations and teams, node management, every routing strategy and inference all work without it.

## Redis (optional)

Redis is optional infrastructure. It may be used for:

- **Session Affinity** — sends requests that carry the same `sessionId` to the same Worker when possible.
- **Shared Worker Capacity coordination** — lets several Console instances share each Worker's concurrency limit.

The two are independent settings; enable either, both, or neither. Without Redis, Session Affinity is unavailable and Worker Capacity uses the Console's memory (`WORKER_CAPACITY_MODE=MEMORY`, the default). Redis is required only when you select `WORKER_CAPACITY_MODE=REDIS`.

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
- With a profile active, `docker compose ps` also lists `redis`, which should become healthy.
- If Redis fails while Session Affinity is enabled, inference keeps working: the Console logs the failure and routes as if there were no affinity.

### Shared Worker Capacity

Each Worker's concurrency limit is enforced with in-flight reservations. By default (`WORKER_CAPACITY_MODE=MEMORY`) they live in the Console's memory, which needs no Redis but is only accurate with a single Console instance. To share the limit across several Console instances, set:

```env
WORKER_CAPACITY_MODE=REDIS
```

and start Redis with `docker compose --profile redis up -d` (the `affinity` and `redis` profiles start the same Redis).

- Unlike Session Affinity, this does not fail open. If Redis is unreachable in `REDIS` mode, Workers that have a concurrency limit stop receiving requests (`429`) until Redis is back; Workers without a limit keep working. The Console logs an ERROR at startup when it cannot reach Redis.
- There is no automatic mode: the Console never switches stores on its own. In particular, it does not fall back to `MEMORY` when Redis becomes unavailable.

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

On Linux, a runtime bound only to `127.0.0.1` is not reachable from a container. Make it listen on the Docker bridge address too (for Ollama: `OLLAMA_HOST=0.0.0.0`). On Docker Desktop (macOS/Windows), `host.docker.internal` resolves to the host machine. However, whether a runtime bound to `127.0.0.1` is reachable depends on Docker Desktop's networking behavior. If the connection fails, configure the runtime to listen on an interface reachable from the container.

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

To let Ollama accept requests from another machine, see [Ollama Remote Access](ollama-remote-access.md).

Only the host part of the address changes between these cases. The Console uses the registered address exactly as entered and never converts `localhost` to `host.docker.internal`.

## HTTPS and Reverse Proxy

The Console image serves HTTP on `:8080`. This repository does not include Nginx, Caddy or any other proxy. If you need HTTPS, put your own reverse proxy or load balancer in front:

```text
Client
   │ HTTPS
   ▼
Nginx / Caddy / Traefik / Load Balancer   (terminates TLS)
   │ HTTP
   ▼
InfraMesh Console :8080
```

The proxy must:

- Set and forward the `X-Forwarded-*` headers, especially `X-Forwarded-Proto: https`. The Console uses them (`server.forward-headers-strategy=framework`) to recognize the original HTTPS request and mark the refresh token cookie as `Secure`; without them, the cookie is issued without `Secure`.
- Overwrite any `Forwarded` / `X-Forwarded-*` headers sent by clients instead of passing them through. The Console accepts these headers from whoever connects to it, so also make sure clients cannot reach the Console port directly, bypassing the proxy (the Compose stack publishes it on all host interfaces).
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

## Backup and Restore

All persistent data is in PostgreSQL. Run these commands from the `console/` directory while the stack is running.

Back up:

```bash
docker compose exec -T postgres sh -c 'pg_dump -U "$POSTGRES_USER" infra_mesh' > backup.sql
```

Make sure the command finishes without errors before treating `backup.sql` as a valid backup. A failed `pg_dump` can leave an incomplete or empty file.

Restore into an empty database (for example on a new host with a fresh `postgres-data` volume), before the Console has started for the first time:

- Do not start the Console before restoring. The Console runs Flyway migrations on startup and creates its tables, so the restore would then conflict with them. Use `docker compose up -d postgres`, which starts only PostgreSQL.
- Do not restore into a database that already contains data.
- Run the commands in the directory that contains `backup.sql`, or adjust the path.

```bash
docker compose up -d postgres
docker compose exec -T postgres sh -c 'psql -U "$POSTGRES_USER" -d infra_mesh' < backup.sql
docker compose up -d
```

- When the Console starts after the restore, Flyway applies any migrations the backup does not have yet (for example, when restoring into a newer Console version).
- A backup is only usable together with the same `ENCRYPTION_KEY`. Keep `.env` (or at least `ENCRYPTION_KEY` and `JWT_SECRET_KEY`) with your backups, and store both securely.
- `backup.sql` contains account data and encrypted keys. Do not commit or share it.

To update the Console, see [upgrade.md](upgrade.md).

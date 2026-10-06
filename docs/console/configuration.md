# Configuration

All Console configuration lives in `console/.env`, next to `console/docker-compose.yml`. Start from the template:

```bash
cd console
cp .env.example .env
```

Do not wrap values in quotes. `.env` is ignored by Git; never commit or share it.

## Deployment

| Variable | Default | Description |
| --- | --- | --- |
| `INFRA_CONSOLE_VERSION` | `0.1.1` | Tag of `ghcr.io/inframeshlabs/infra-console`. Pin a version; `latest` is optional (see [upgrade.md](upgrade.md)). |
| `INFRA_CONSOLE_PORT` | `8080` | Host port for the Console. The container port is always `8080`. |

## Required

| Variable | Description |
| --- | --- |
| `POSTGRES_USER` | PostgreSQL user. Used to create the database and passed to the Console as `SPRING_DB_USERNAME`. |
| `POSTGRES_PASSWORD` | PostgreSQL password. Passed to the Console as `SPRING_DB_PASSWORD`. |
| `JWT_SECRET_KEY` | JWT signing key, a Base64 string — `openssl rand -base64 48` |
| `ENCRYPTION_KEY` | Encryption key for Team Keys / Node API Keys, Base64 of exactly 32 bytes — `openssl rand -base64 32` |
| `INFRA_ADMIN_PASSWORD` | Password of the bootstrap admin account |

Important:

- **The database name is fixed to `infra_mesh`.** The Console has no setting for it.
- **`POSTGRES_USER` / `POSTGRES_PASSWORD` only take effect when the `postgres-data` volume is first created.** Changing them later does not change the existing database credentials, and the Console will fail to connect.
- **Never change `ENCRYPTION_KEY` once set.** Stored Team Keys and Node API Keys can no longer be decrypted.
- Changing `JWT_SECRET_KEY` invalidates every issued token; all users must sign in again.
- **The bootstrap admin** is created on first start only if no account named `INFRA_ADMIN_USERNAME` exists. Changing `INFRA_ADMIN_PASSWORD` afterwards does not change that account's password.

## Optional

| Variable | Default | Description |
| --- | --- | --- |
| `INFRA_ADMIN_USERNAME` | `admin` | Bootstrap admin username |
| `INFRA_ADMIN_NICKNAME` | `admin` | Bootstrap admin nickname |
| `INFRA_ADMIN_EMAIL` | `admin@infra-console.local` | Bootstrap admin email |
| `INFRAMESH_SESSION_AFFINITY_ENABLED` | `false` | Enables Session Affinity on the Console. Redis is only used when this is `true` — start the stack with `--profile affinity`. Teams can then turn it on or off individually. |
| `INFRAMESH_SESSION_AFFINITY_TTL` | `30m` | Session Affinity TTL |
| `WORKER_CAPACITY_MODE` | `MEMORY` | Where Worker in-flight capacity is kept, independent of Session Affinity. `MEMORY`: Console memory, no Redis, single Console only. `REDIS`: shared across Consoles — start the stack with `--profile redis`. |
| `REDIS_PASSWORD` | (none) | Only used when Redis is used. When set, Redis requires this password and the Console uses it. Empty means no authentication. |
| `ALLOWED_ORIGIN_PATTERNS` | `http://localhost:<INFRA_CONSOLE_PORT>` | Only needed when a frontend on a different origin calls the API |
| `JWT_EXPIRATION` | `3600000` | Access token lifetime (ms) |
| `JWT_REFRESH_TOKEN_EXPRATION` | `1209600000` | Refresh token lifetime (ms). The spelling matches the Console setting. |
| `NODE_OUTBOUND_REQUEST_TIMEOUT` | `60s` | How long to wait for an OUTBOUND Worker response |
| `NODE_OUTBOUND_HEARTBEAT_TIMEOUT` | `30s` | OUTBOUND Node heartbeat timeout |
| `NODE_OUTBOUND_STREAM_IDLE_TIMEOUT` | `60s` | Maximum silence between OUTBOUND streaming chunks |
| `NODE_REGISTRATION_TOKEN_TTL` | `10m` | Node Registration Token lifetime |
| `WORKER_CAPACITY_INFLIGHT_TTL` | `5m` | Worker In-Flight reservation lease |

## Fixed by Compose

These Console settings are set in `console/docker-compose.yml` to the internal service addresses and are not read from `.env`:

| Console variable | Value |
| --- | --- |
| `SPRING_DB_HOST` / `SPRING_DB_PORT` | `postgres` / `5432` |
| `REDIS_HOST` / `REDIS_PORT` | `redis` / `6379` (only used when Session Affinity is enabled or `WORKER_CAPACITY_MODE=REDIS`) |
| `SPRING_PROFILES_ACTIVE` | `prod` (set by the image) |

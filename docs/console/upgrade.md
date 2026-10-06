# Upgrade

## Updating the Console

Run these commands from the `console/` directory.

1. Back up PostgreSQL:

   ```bash
   docker compose exec postgres sh -c 'pg_dump -U "$POSTGRES_USER" infra_mesh' > backup.sql
   ```

2. Change the version in `console/.env`:

   ```env
   INFRA_CONSOLE_VERSION=0.1.2
   ```

3. Pull and restart:

   ```bash
   docker compose pull
   docker compose up -d
   ```

   To update only the Console and leave PostgreSQL (and Redis, if used) untouched:

   ```bash
   docker compose pull console
   docker compose up -d console
   ```

Pull the latest `infra-deploy` changes first (`git pull`) when a release changes `console/docker-compose.yml` or `console/.env.example`.

## Version policy

Two kinds of tags are published to `ghcr.io/inframeshlabs/infra-console`:

```text
0.1.1     a specific release
latest    the newest release
```

The default is a pinned version, so the same `.env` always deploys the same Console:

```text
infra-deploy
      │
      └── INFRA_CONSOLE_VERSION=0.1.1
            ↓
      always the same Console version
```

If you explicitly want to keep tracking the newest release, set:

```env
INFRA_CONSOLE_VERSION=latest
```

With `latest`, `docker compose up -d` alone keeps using the image already on disk. Run `docker compose pull` to actually move to a newer release.

## Things to keep in mind

- Never use `docker compose down -v` during an update; it deletes the database volume.
- Schema changes are applied by Flyway when the new Console starts. After the schema has moved forward, rolling back to an older image may not work — restore the backup instead.
- Keep `ENCRYPTION_KEY` and `JWT_SECRET_KEY` unchanged. See [configuration.md](configuration.md).
- When the Console restarts, WebSocket connections from OUTBOUND nodes drop. The Node SDK reconnects automatically, but requests in flight during the restart fail.

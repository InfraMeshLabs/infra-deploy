# Upgrading

Always back up before upgrading. Automatic rollback and database schema downgrades are not supported: once a newer Console has migrated the database, an older version may not start against it. To go back, restore the backup taken before the upgrade.

## Standalone

The installation directory and the data directory (`INFRAMESH_HOME`) are separate. An upgrade replaces only the installation directory; the data directory is reused as-is.

```text
Installation
├── inframesh-console-0.2.1/
└── inframesh-console-0.2.2/

Persistent Data
└── ~/.inframesh/
```

1. **Stop the running Console** with `Ctrl+C` in its terminal. Make sure no Console is still using the data directory.

2. **Back up `INFRAMESH_HOME`** (default `~/.inframesh`, Windows `%USERPROFILE%\.inframesh`). Copy the whole directory, including `config/` and `data/`:

   ```bash
   cp -Rp ~/.inframesh ~/inframesh-backup-$(date +%Y%m%d-%H%M%S)
   ```

   ```powershell
   Copy-Item -Recurse "$env:USERPROFILE\.inframesh" "$env:USERPROFILE\inframesh-backup-$(Get-Date -Format yyyyMMdd-HHmmss)"
   ```

   If you configured a custom `INFRAMESH_HOME`, replace `~/.inframesh` (or `$env:USERPROFILE\.inframesh`) with your data directory. The timestamp in the backup name keeps a new backup from colliding with an earlier one; if you change the name, make sure the destination does not exist yet.

3. **Download the new release** and its `.sha256` file from the [Releases page](https://github.com/InfraMeshLabs/infra-deploy/releases).

4. **Verify the SHA-256 checksum** as described in [Standalone Installation](standalone.md#macos) (macOS) or [Windows](standalone.md#windows).

5. **Extract into a new directory.** Do not extract over the previous version's directory. Each release extracts into its own `inframesh-console-<version>/` directory.

6. **Run the new launcher with the same `INFRAMESH_HOME`.** If you use the default location, no setting is needed. If you set `INFRAMESH_HOME` before, set the same value again.

   ```bash
   ./inframesh-console-0.2.2/bin/inframesh
   ```

   ```bat
   inframesh-console-0.2.2\bin\inframesh.cmd
   ```

7. **Check that your data is still there.** Sign in with your existing administrator account and confirm that organizations, teams and nodes are present. You should not see the Initial Administrator Setup page again; if you do, the Console is using a different (empty) `INFRAMESH_HOME`. Stop it and check the `Home:` line the launcher prints.

8. **Confirm normal operation**, then delete the old installation directory when you no longer need it.

Never delete `INFRAMESH_HOME` during an upgrade. It contains the database and the secrets needed to read it.

To return to the previous version:

1. Stop the Console completely.
2. Move the current `INFRAMESH_HOME` to a separate location. Do not delete it.
3. Restore the backup from step 2 to the original `INFRAMESH_HOME` location.
4. Run the previous version's launcher.
5. Sign in and check that your data is present.

Restoring a pre-upgrade backup discards changes made after that backup. Keep the directory you moved aside in step 2 if you may need those changes later. Do not run the previous version against the upgraded data directory: after the new version's Flyway migrations have run, the older version may not support the schema.

## Docker

Run these commands from the `console/` directory.

1. **Back up PostgreSQL:**

   ```bash
   docker compose exec -T postgres sh -c 'pg_dump -U "$POSTGRES_USER" infra_mesh' > backup.sql
   ```

   Make sure the command finishes without errors before continuing. A failed `pg_dump` can leave an incomplete or empty `backup.sql`.

   Also keep a secure copy of the `ENCRYPTION_KEY` this database uses (it is in `.env`, not in `backup.sql`). Without the original key, Node API Keys in a restored database cannot be decrypted. Store database backups and secrets securely, and do not commit them.

2. **Change the version** in `console/.env`:

   ```env
   INFRA_CONSOLE_VERSION=0.2.2
   ```

3. **Pull the new image and restart:**

   ```bash
   docker compose pull
   docker compose up -d
   ```

   If you run Redis, pass the same profile you started the stack with to both commands. Redis is not part of the default stack, so a command without the profile does not include it.

   ```bash
   docker compose --profile affinity pull
   docker compose --profile affinity up -d
   ```

   Use `--profile redis` instead if that is the profile you use. If you set `COMPOSE_PROFILES` in `.env`, the profile is applied automatically and `--profile` is not needed. Starting Redis does not change any Console setting; Session Affinity and `WORKER_CAPACITY_MODE` are still controlled by `.env`.

   To update only the Console image without starting or recreating its dependencies:

   ```bash
   docker compose pull console
   docker compose up -d --no-deps console
   ```

   `--no-deps` does not start PostgreSQL or Redis. Use it only when PostgreSQL, and Redis if your configuration needs it, are already running. If a release changes the services or dependencies in `docker-compose.yml`, use the full-stack commands above instead.

4. **Check health:**

   ```bash
   docker compose ps                # console and postgres (and redis, with a profile) should be "healthy"
   docker compose logs -f console
   ```

5. **Check that your data is still there** by signing in and confirming that organizations, teams and nodes are present.

Pull the latest `infra-deploy` changes first (`git pull`) when a release changes `console/docker-compose.yml` or `console/.env.example`. Check `git status` for local changes before pulling. `.env` is not tracked by Git and is not updated by the pull: compare `.env.example` with your existing `.env`, add any newly required variables by hand, and keep your existing secrets. Never overwrite `.env` with `.env.example`.

### Version policy

Two kinds of tags are published to `ghcr.io/inframeshlabs/infra-console`:

```text
0.2.2     a specific release
latest    the newest release
```

The default is a pinned version, so the same `.env` always deploys the same Console:

```text
infra-deploy
      │
      └── INFRA_CONSOLE_VERSION=<version>
            ↓
      always the same Console version
```

If you explicitly want to keep tracking the newest release, set:

```env
INFRA_CONSOLE_VERSION=latest
```

With `latest`, `docker compose up -d` alone keeps using the image already on disk. Run `docker compose pull` to actually move to a newer release.

### Things to keep in mind

- Never use `docker compose down -v` during an update; it deletes the database volume.
- Schema changes are applied by Flyway when the new Console starts. After the schema has moved forward, rolling back to an older image may not work — restore the backup instead.
- Keep `ENCRYPTION_KEY` and `JWT_SECRET_KEY` unchanged. See [configuration.md](configuration.md).
- When the Console restarts, WebSocket connections from OUTBOUND nodes drop. The Node SDK reconnects automatically with exponential backoff (up to 30 seconds between attempts by default) and keeps retrying until the node itself is stopped. Requests in flight during the restart fail and are not retried.

## Moving between Standalone and Docker

The two deployments use different databases (embedded H2 and PostgreSQL) and keep separate data. Switching from one to the other starts with an empty Console; this guide does not cover moving data between them.

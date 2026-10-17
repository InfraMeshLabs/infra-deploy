# Upgrading

Always back up before upgrading. Automatic rollback and database schema downgrades are not supported: once a newer Console has migrated the database, an older version may not start against it. To go back, restore the backup taken before the upgrade.

## Standalone

The installation directory and the data directory (`INFRAMESH_HOME`) are separate. An upgrade replaces only the installation directory; the data directory is reused as-is.

```text
Installation
├── inframesh-console-0.2.0/
└── inframesh-console-0.2.1/

Persistent Data
└── ~/.inframesh/
```

1. **Stop the running Console** with `Ctrl+C` in its terminal. Make sure no Console is still using the data directory.

2. **Back up `INFRAMESH_HOME`** (default `~/.inframesh`, Windows `%USERPROFILE%\.inframesh`). Copy the whole directory, including `config/` and `data/`:

   ```bash
   cp -Rp ~/.inframesh ~/inframesh-backup-$(date +%Y%m%d)
   ```

   ```powershell
   Copy-Item -Recurse "$env:USERPROFILE\.inframesh" "$env:USERPROFILE\inframesh-backup-$(Get-Date -Format yyyyMMdd)"
   ```

3. **Download the new release** and its `.sha256` file from the [Releases page](https://github.com/InfraMeshLabs/infra-deploy/releases).

4. **Verify the SHA-256 checksum** as described in [Standalone Installation](standalone.md#macos) (macOS) or [Windows](standalone.md#windows).

5. **Extract into a new directory.** Do not extract over the previous version's directory. Each release extracts into its own `inframesh-console-<version>/` directory.

6. **Run the new launcher with the same `INFRAMESH_HOME`.** If you use the default location, no setting is needed. If you set `INFRAMESH_HOME` before, set the same value again.

   ```bash
   ./inframesh-console-0.2.1/bin/inframesh
   ```

   ```bat
   inframesh-console-0.2.1\bin\inframesh.cmd
   ```

7. **Check that your data is still there.** Sign in with your existing administrator account and confirm that organizations, teams and nodes are present. You should not see the Initial Administrator Setup page again; if you do, the Console is using a different (empty) `INFRAMESH_HOME`. Stop it and check the `Home:` line the launcher prints.

8. **Confirm normal operation**, then delete the old installation directory when you no longer need it.

Never delete `INFRAMESH_HOME` during an upgrade. It contains the database and the secrets needed to read it.

To return to the previous version: stop the Console, replace `INFRAMESH_HOME` with the backup from step 2, and run the previous version's launcher.

## Docker

Run these commands from the `console/` directory.

1. **Back up PostgreSQL:**

   ```bash
   docker compose exec postgres sh -c 'pg_dump -U "$POSTGRES_USER" infra_mesh' > backup.sql
   ```

2. **Change the version** in `console/.env`:

   ```env
   INFRA_CONSOLE_VERSION=0.2.0
   ```

3. **Pull the new image and restart:**

   ```bash
   docker compose pull
   docker compose up -d
   ```

   To update only the Console and leave PostgreSQL (and Redis, if used) untouched:

   ```bash
   docker compose pull console
   docker compose up -d console
   ```

   If you run Redis, include the same profile you started the stack with (for example `docker compose --profile affinity up -d`).

4. **Check health:**

   ```bash
   docker compose ps                # console and postgres should be "healthy"
   docker compose logs -f console
   ```

5. **Check that your data is still there** by signing in and confirming that organizations, teams and nodes are present.

Pull the latest `infra-deploy` changes first (`git pull`) when a release changes `console/docker-compose.yml` or `console/.env.example`.

### Version policy

Two kinds of tags are published to `ghcr.io/inframeshlabs/infra-console`:

```text
0.2.0     a specific release
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
- When the Console restarts, WebSocket connections from OUTBOUND nodes drop. The Node SDK reconnects automatically, but requests in flight during the restart fail.

## Moving between Standalone and Docker

The two deployments use different databases (embedded H2 and PostgreSQL) and keep separate data. Switching from one to the other starts with an empty Console; this guide does not cover moving data between them.

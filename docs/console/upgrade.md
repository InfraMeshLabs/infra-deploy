# Upgrading

Always back up before upgrading. Automatic rollback and database schema downgrades are not supported: once a newer Console has migrated the database, an older version may not start against it. To go back, restore the backup taken before the upgrade.

## Standalone

The installation directory and the data directory (`INFRAMESH_HOME`) are separate. An upgrade replaces only the installation directory; the data directory is reused as-is.

- **Upgrading from v0.2.4 or earlier to v0.3.0** changes the database from the embedded H2 file to PostgreSQL. Follow [Standalone from v0.2.x (H2) to v0.3.0](#standalone-from-v02x-h2-to-v030) instead of the steps below.
- **Upgrading from v0.3.0 or later** to a newer release:

```text
Installation
├── inframesh-console-0.3.0/
└── inframesh-console-<new>/

Settings
└── ~/.inframesh/          (config/standalone.yml)

Data
└── PostgreSQL database
```

1. **Stop the running Console** with `Ctrl+C` in its terminal. Make sure no Console is still using the data directory.
2. **Back up** the PostgreSQL database and `INFRAMESH_HOME/config/` (see [Backup and Restore](standalone.md#backup-and-restore)).
3. **Download the new release** and its `.sha256` file from the [Releases page](https://github.com/InfraMeshLabs/infra-deploy/releases) and **verify the SHA-256 checksum** as described in [Standalone Installation](standalone.md#macos) (macOS) or [Windows](standalone.md#windows).
4. **Extract into a new directory.** Do not extract over the previous version's directory.
5. **Run the new launcher with the same `INFRAMESH_HOME`.** It uses the stored PostgreSQL connection and applies schema changes when it starts.
6. **Check that your data is still there.** Sign in with your existing administrator account and confirm that organizations, teams and nodes are present. If you see the database setup page or the Initial Administrator Setup page, the Console is using a different `INFRAMESH_HOME` or a different database. Stop it and check the `Home:` line the launcher prints.
7. **Confirm normal operation**, then delete the old installation directory when you no longer need it.

To return to the previous version, restore the PostgreSQL backup taken in step 2 into an empty database (see [Backup and Restore](standalone.md#backup-and-restore)) and run the previous version's launcher. Do not run an older version against a database that a newer version has already migrated.

## Standalone from v0.2.x (H2) to v0.3.0

v0.2.0 to v0.2.4 stored all data in an embedded H2 file, `INFRAMESH_HOME/data/console.mv.db`, with the secrets in `INFRAMESH_HOME/config/secrets.yml`. v0.3.0 uses PostgreSQL only: it does not start on the H2 file, and the data is **not** moved automatically. The `inframesh migrate-h2` command copies it into an empty PostgreSQL database.

| Source | H2 schema | Handling |
| --- | --- | --- |
| v0.2.4 | V9 | Read as is |
| v0.2.0 – v0.2.3 | V1 – V7 | Upgraded on a temporary working copy inside `INFRAMESH_HOME/run/`; the original file is not changed |

Data created by v0.2.0 to v0.2.4 packages was migrated with the v0.3.0 packages during release testing (Windows x64: v0.2.0 – v0.2.4; macOS arm64: v0.2.0 and v0.2.4, including an H2 backup ZIP of v0.2.4).

### 1. Prepare

1. **Stop the running Console.** `migrate-h2` refuses to run while a Console or another command uses the same `INFRAMESH_HOME`.
2. **Back up the whole `INFRAMESH_HOME`.** `data/console.mv.db` and `config/secrets.yml` belong together: the encryption key in `secrets.yml` is the only way to read the Node API Keys stored in the H2 file.

   ```bash
   cp -Rp ~/.inframesh ~/inframesh-backup-$(date +%Y%m%d-%H%M%S)
   ```

   ```powershell
   Copy-Item -Recurse "$env:USERPROFILE\.inframesh" "$env:USERPROFILE\inframesh-backup-$(Get-Date -Format yyyyMMdd-HHmmss)"
   ```

3. **Prepare an empty PostgreSQL database** and a user that owns it (see [PostgreSQL](standalone.md#postgresql)). 16 is the verified version.
4. **Install v0.3.0** into a new directory (see [Standalone Installation](standalone.md)). Keep the same `INFRAMESH_HOME`.

### 2. Save the target database

Run the v0.3.0 launcher. Because no PostgreSQL connection is stored yet, the database connection setup page opens. It shows **"An embedded database (H2) of an earlier version exists"** with the path of the H2 file.

1. Enter the connection of the empty database and choose **Test connection**. The result should say that the database does not hold the existing data yet.
2. Choose **Save and exit**. The connection is saved and the Console stops without creating anything in the database.

> Do **not** choose **Save and start** here. That starts a new, empty installation in that database. You can still migrate into it only until you create an administrator or other data; after that, prepare another empty database.

If you give the connection with environment variables (`SPRING_DB_*` or `SPRING_DATASOURCE_*`) instead, set them in the same way for the migration command.

### 3. Check

```bash
"./inframesh-console-${VERSION}/bin/inframesh" migrate-h2 --check
```

```bat
inframesh-console-%VERSION%\bin\inframesh.cmd migrate-h2 --check
```

The check reads the H2 file, the target database and the encryption key, and shows the tables and row counts to be moved. It changes nothing (it does not create tables in the target either).

### 4. Migrate

```bash
"./inframesh-console-${VERSION}/bin/inframesh" migrate-h2
```

The command repeats the checks, shows the summary and asks for confirmation; type `migrate` to start (anything else cancels). In a non-interactive run, use `migrate-h2 --yes`.

```text
[1/8] Checking the source H2 database
[2/8] Checking the target PostgreSQL database
[3/8] Validating the secrets and the encryption key
[4/8] Preparing the PostgreSQL schema (Flyway migrations of this Console, up to V9)
[5/8] Migrating application data
[6/8] Validating migrated data (every row, every column)
[7/8] Finalizing (identity counters, completion record, commit)
[8/8] Migration completed
```

- The copy is one PostgreSQL transaction, and every row is compared after the copy.
- The output is also written to `INFRAMESH_HOME/logs/h2-migration-<time>.log`. It contains no passwords, secrets or user data.
- Accounts and password hashes, organizations, teams, members, member API keys, nodes and node credentials, request logs and audit logs are moved as they are. The existing encryption key and JWT secret are used as they are; nothing is regenerated.
- Team keys of v0.2.0 / v0.2.1 cannot be used after the migration (as for any upgrade to v0.2.2 or later). Issue member API keys instead.

Exit codes: `0` completed, already migrated or check passed; `1` failed or refused; `2` invalid option; `3` the commit result could not be confirmed; `4` cancelled.

### 5. Start and check

Start the Console with the launcher as usual. It starts on PostgreSQL with your existing data.

1. Sign in with your existing accounts. Browsers may have to sign in again.
2. Check organizations, teams, members and nodes.
3. Check that existing member API keys work for inference and that OUTBOUND nodes reconnect with their existing credentials.

### Running the command again

On a database that already holds this migration, `migrate-h2` reports `ALREADY MIGRATED` and changes nothing. A database that holds other data, or data migrated from a different H2 file, is refused.

### If the migration fails

- **A check fails** — nothing has been changed. The message names the cause:

  | Message | Action |
  | --- | --- |
  | `INFRAMESH_HOME is in use` | Stop the Console or the other command first. |
  | `TARGET_NOT_CONFIGURED` | Save the target database with **Save and exit** (step 2). |
  | `TARGET_HAS_DATA` / `TARGET_MIGRATED_FROM_ANOTHER_SOURCE` | Prepare another empty database. Existing data is never merged or deleted. |
  | `SECRETS_MISSING` / `ENCRYPTION_KEY_MISMATCH` | Put the original `config/secrets.yml` of this installation back (from your backup), or remove an `ENCRYPTION_KEY` of another installation from the environment. |
  | `SOURCE_NEWER` / `SOURCE_DAMAGED` / `SOURCE_NOT_INFRAMESH` | The H2 file is newer than this Console, damaged, or not an InfraMesh database. Restore it from your backup. |
  | Connection errors (`TARGET_AUTHENTICATION_FAILED`, `TARGET_CONNECTION_REFUSED`, `TARGET_TLS_FAILED`, …) | Check PostgreSQL and the stored connection (`inframesh db-config`). |

- **The migration fails or is interrupted** before it commits — no data is left in the target. Remove the cause and run the same command again; you do not need to clear the target.
- **Exit code `3`** (the connection dropped while committing) — do not start the Console and do not prepare another database. When PostgreSQL is reachable again, run the same command again: it reports `ALREADY MIGRATED` if the commit succeeded, and migrates otherwise.
- If the original encryption key is lost, the stored Node API Keys cannot be recovered. Start a new installation and register the nodes again.

### H2 backup ZIP only

If you have only an H2 backup ZIP made by v0.2.4 (`inframesh backup` or the administration API of that version), restore it into a **new** `INFRAMESH_HOME` with the v0.3.0 launcher, then follow steps 2 to 5 with that home:

```bash
INFRAMESH_HOME="$HOME/inframesh-restored" "./inframesh-console-${VERSION}/bin/inframesh" restore --input ~/inframesh-backup.zip
INFRAMESH_HOME="$HOME/inframesh-restored" "./inframesh-console-${VERSION}/bin/inframesh"
```

`restore` puts the H2 file and `secrets.yml` of the backup into that home; the ZIP is not changed. If a restore is interrupted, run `inframesh recover`. A restored H2 file is used only as the migration source. v0.3.0 cannot create H2 backups: `inframesh backup` exits with code `1`.

### After the migration

- **Do not delete `config/secrets.yml`.** The migrated installation keeps using the encryption key and JWT secret in it.
- Back up the PostgreSQL database together with `config/secrets.yml` and `config/standalone.yml` from now on (see [Backup and Restore](standalone.md#backup-and-restore)).
- The Console no longer opens the H2 file and does not delete it. Keep it and your backup until you have checked accounts, organizations, teams, nodes and inference; deleting it afterwards is your decision. While it exists, the setup page keeps mentioning it.
- To go back to v0.2.4, stop v0.3.0 and run the v0.2.4 launcher with the backup of `INFRAMESH_HOME` you made in step 1. Changes made in v0.3.0 are not moved back to H2.

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
   INFRA_CONSOLE_VERSION=0.3.0
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
0.3.0     a specific release
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

From v0.3.0 both deployments use PostgreSQL, but each keeps its own database and settings. Switching from one to the other starts with an empty Console unless you move the database yourself; a moved database is usable only together with the same `ENCRYPTION_KEY` (and `JWT_SECRET_KEY` to keep sessions). This guide does not cover that procedure, and it has not been verified.

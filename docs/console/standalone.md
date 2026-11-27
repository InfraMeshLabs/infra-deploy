# Standalone Installation

The Standalone Distribution runs InfraMesh Console directly on a local PC. Each package bundles a Java runtime, so Java and Docker do not need to be installed.

From Console v0.3.0 the Console stores its data in **PostgreSQL**, for the Standalone Distribution as well as for Docker. The package does not include PostgreSQL: prepare a PostgreSQL database before the first start (see [PostgreSQL](#postgresql)). Redis is not required.

> **Upgrading from v0.2.4 or earlier?** Those versions stored their data in an embedded H2 file. v0.3.0 does not start on that file. Follow [Upgrading a Standalone installation from v0.2.x](upgrade.md#standalone-from-v02x-h2-to-v030) to move the data to PostgreSQL. The rest of this page describes v0.3.0 and later.

The [Docker deployment](installation.md) remains available and is recommended for servers and Linux.

## Supported Platforms

| OS | Architecture | Package |
| --- | --- | --- |
| macOS | Apple Silicon (arm64) | `inframesh-console-<version>-macos-arm64.tar.gz` |
| macOS | Intel (x64) | `inframesh-console-<version>-macos-x64.tar.gz` |
| Windows | x64 | `inframesh-console-<version>-windows-x64.zip` |

There is no Linux standalone package. Use the [Docker deployment](installation.md) on Linux.

Pick the macOS package that matches your CPU. To check it, run `uname -m` in Terminal: `arm64` means Apple Silicon, `x86_64` means Intel.

## PostgreSQL

The Console needs one PostgreSQL database that it uses only for itself. The Console does not install PostgreSQL and does not create databases or accounts.

| PostgreSQL version | Console behaviour |
| --- | --- |
| 16 | Verified version. |
| 13 – 15 | Accepted with a warning: older than the verified version. |
| 17 and later | Accepted with a notice: newer than the verified version. |
| 12 and earlier | Refused. |

Install PostgreSQL 16 with the method you prefer (for example the installers from [postgresql.org](https://www.postgresql.org/download/)) or use an existing PostgreSQL server. Then create a user and an empty database owned by that user, for example with `psql` as a PostgreSQL administrator:

```sql
CREATE ROLE infra_mesh LOGIN PASSWORD '<choose a password>';
CREATE DATABASE infra_mesh OWNER infra_mesh ENCODING 'UTF8';
```

- The user only needs to own the database. A superuser is not required.
- The database must be used only by InfraMesh Console. A database that contains another application's tables is refused.
- For PostgreSQL on another machine, use TLS (`verify-full` is recommended in the setup screen).

## Download

Download packages only from the official Releases page:

<https://github.com/InfraMeshLabs/infra-deploy/releases>

Each release provides one package per platform and a matching `.sha256` file:

```text
inframesh-console-<version>-macos-arm64.tar.gz
inframesh-console-<version>-macos-arm64.tar.gz.sha256

inframesh-console-<version>-macos-x64.tar.gz
inframesh-console-<version>-macos-x64.tar.gz.sha256

inframesh-console-<version>-windows-x64.zip
inframesh-console-<version>-windows-x64.zip.sha256
```

Download both the package and its `.sha256` file into the same directory.

The commands on this page use a version variable (`VERSION` in the macOS shell and Command Prompt, `$version` in PowerShell). The value `0.3.0` in the examples is only an example: set it to the release you downloaded. The variable lasts only for the current terminal session; set it again in a new terminal.

## macOS

### 1. Verify the checksum

Run this in the directory that contains both files:

```bash
VERSION=0.3.0

shasum -a 256 -c "inframesh-console-${VERSION}-macos-arm64.tar.gz.sha256"
```

The output must end with `OK`:

```text
inframesh-console-<version>-macos-arm64.tar.gz: OK
```

If it reports `FAILED`, delete the download and download it again from the Releases page. Do not run a package whose checksum does not match. On Intel Macs, replace `arm64` with `x64` in the file names.

### 2. Extract

```bash
tar -xzf "inframesh-console-${VERSION}-macos-arm64.tar.gz"
```

This creates the `inframesh-console-<version>/` directory. It contains only the application; your settings are stored separately (see [Data Directory](#data-directory)).

### 3. Run the launcher

```bash
"./inframesh-console-${VERSION}/bin/inframesh"
```

Keep the terminal open while you use the Console. Press `Ctrl+C` in that terminal to stop it. The first start continues in the browser; see [First Run](#first-run).

### Gatekeeper

The macOS packages are **not code-signed or notarized** yet. macOS Gatekeeper may warn about or block the bundled runtime.

If execution is blocked:

1. Confirm the checksum verification above succeeded.
2. Open **System Settings → Privacy & Security**.
3. Find the message about the blocked item and choose **Open Anyway**, then run the launcher again.

Do not disable Gatekeeper system-wide (for example with `spctl --master-disable`). Only allow the InfraMesh package you downloaded from the official Releases page and verified.

## Windows

### 1. Verify the checksum

In PowerShell, in the directory that contains both files:

```powershell
$version = "0.3.0"

Get-FileHash `
  ".\inframesh-console-$version-windows-x64.zip" `
  -Algorithm SHA256
```

Compare the printed `Hash` with the value at the start of `inframesh-console-<version>-windows-x64.zip.sha256` (open it in Notepad). They must be identical; letter case does not matter.

To compare automatically:

```powershell
$expected = (Get-Content ".\inframesh-console-$version-windows-x64.zip.sha256") -split '\s+' | Select-Object -First 1
$actual = (Get-FileHash ".\inframesh-console-$version-windows-x64.zip" -Algorithm SHA256).Hash
if ($actual -eq $expected) { 'OK' } else { 'MISMATCH - do not run this package' }
```

If the values differ, delete the download and download it again from the Releases page.

### 2. Extract

Extract the ZIP (right-click → **Extract All…**, or `Expand-Archive` in PowerShell):

```powershell
Expand-Archive ".\inframesh-console-$version-windows-x64.zip" -DestinationPath .
```

This creates the `inframesh-console-<version>\` directory.

### 3. Run the launcher

From PowerShell in the directory where you extracted the ZIP:

```powershell
.\inframesh-console-$version\bin\inframesh.cmd
```

Or from Command Prompt:

```bat
set VERSION=0.3.0
inframesh-console-%VERSION%\bin\inframesh.cmd
```

Keep the window open while you use the Console, and press `Ctrl+C` in it to stop.

- Administrator privileges are not required for normal use.
- You do not need to change the PowerShell execution policy. The launcher applies its policy only to its own process.

### SmartScreen

The Windows package is **not code-signed** yet. Windows SmartScreen or your antivirus may show a warning. Before allowing it, confirm that you downloaded the ZIP from the official Releases page and that its SHA-256 checksum matches.

## First Run

```text
Run the launcher
    │
    ▼
No PostgreSQL connection stored yet
    │
    ▼
Database connection setup page (opened in the default browser)
    │   Test connection → Save and start
    ▼
Console creates its tables in the empty database and starts
    │
    ▼
Initial Administrator Setup
```

On the first start the launcher prints:

```text
InfraMesh Console
Home: /Users/<you>/.inframesh

Starting...
Database setup: the PostgreSQL connection has to be entered or checked before the Console can start.
Open this file in a browser on this computer to continue:
/Users/<you>/.inframesh/run/database-setup.html
```

and opens that file in your default browser (set `INFRAMESH_OPEN_BROWSER=false` to open it yourself). The file contains a one-time link to the **Database connection setup** page. The page is available only on this PC (`127.0.0.1`).

1. Enter host, port, database name, user name, password and SSL mode of the database you prepared.
2. Choose **Test connection**. The test checks that PostgreSQL is reachable, the login, the PostgreSQL version, the permissions and whether the database is empty or an existing InfraMesh database. It does not change the database.
3. Choose **Save and start**. The connection is saved, the Console creates its tables and starts.
4. The browser then shows the **Initial Administrator Setup** page. Create the administrator account there.

From then on the launcher starts the Console directly with the saved connection and prints:

```text
InfraMesh Console is ready:
http://127.0.0.1:8080
```

If you change `SERVER_PORT`, use that port instead. The launcher detects the port the Console actually listens on and opens that address.

The launcher waits up to 60 seconds for the Console to become ready (it does not time out while the setup page is open). If the Console exits early (for example, because the port is already in use or another Console is using the same data directory), the launcher prints an error and exits.

Initial Administrator Setup is accepted only from the PC running the Console over a direct loopback connection. Create the administrator before you change `SERVER_ADDRESS` (see [Standalone Network Access](standalone-network.md)).

### If PostgreSQL is not available at startup

If the saved database cannot be used when the Console starts (for example, PostgreSQL is stopped or the password was changed), the Console does not start and the launcher opens the setup page again in **recovery** mode, with the cause. Fix the cause and choose **Try again with the stored settings**, or correct the connection details and save them. The Console never falls back to another database.

### Changing the database connection

Stop the Console and run:

```bash
"./inframesh-console-${VERSION}/bin/inframesh" db-config
```

```bat
inframesh-console-%VERSION%\bin\inframesh.cmd db-config
```

The setup page opens with the stored connection. Changing the connection does not move data: if you switch to another database, the data of the current database stays where it is. The stored password is never shown and is not sent to a different host or over a weaker SSL mode.

## Configuration

Configure the launcher with environment variables:

| Variable | Default | Description |
| --- | --- | --- |
| `INFRAMESH_HOME` | `~/.inframesh` (macOS)<br>`%USERPROFILE%\.inframesh` (Windows) | Settings and log directory |
| `SERVER_PORT` | `8080` | Console HTTP port |
| `INFRAMESH_OPEN_BROWSER` | `true` | Set to `false` to not open the browser automatically. Only `true` or `false` are accepted. |
| `SPRING_PROFILES_ACTIVE` | `prod` | Spring profile. Leave it at the default. |
| `SERVER_ADDRESS` | `127.0.0.1` | Network address to listen on. See [Standalone Network Access](standalone-network.md) and [Security](security.md#standalone) before changing it. |

macOS:

```bash
SERVER_PORT=9090 INFRAMESH_OPEN_BROWSER=false "./inframesh-console-${VERSION}/bin/inframesh"
```

Windows (Command Prompt):

```bat
set SERVER_PORT=9090
set INFRAMESH_OPEN_BROWSER=false
inframesh-console-%VERSION%\bin\inframesh.cmd
```

Windows (PowerShell):

```powershell
$env:SERVER_PORT = '9090'
$env:INFRAMESH_OPEN_BROWSER = 'false'
.\inframesh-console-$version\bin\inframesh.cmd
```

Settings can also be placed in `<INFRAMESH_HOME>/config/application.yml` (optional; it does not exist by default). Environment variables take precedence over that file.

### Giving the database connection without the setup page

Instead of the setup page you can give the connection explicitly, for example for unattended starts:

| Variable | Description |
| --- | --- |
| `SPRING_DB_HOST` / `SPRING_DB_PORT` | PostgreSQL host and port (default port `5432`). The database name is `infra_mesh`. |
| `SPRING_DB_USERNAME` / `SPRING_DB_PASSWORD` | PostgreSQL user and password |
| `SPRING_DATASOURCE_URL` | Full JDBC URL (`jdbc:postgresql://<host>:<port>/<database>`), for another database name or JDBC options. Use with `SPRING_DATASOURCE_USERNAME` / `SPRING_DATASOURCE_PASSWORD`. |
| `JWT_SECRET_KEY` / `ENCRYPTION_KEY` | Required together with an explicit connection, unless the data directory already holds the secrets (`config/standalone.yml` or `config/secrets.yml`). |

- Explicit values take precedence over the stored connection. Giving only part of them is refused.
- With an explicit connection, a failed connection stops the Console with an error instead of opening the setup page.
- Only PostgreSQL is accepted. An H2 JDBC URL or `INFRAMESH_DATABASE_MODE=embedded` is refused at startup.

### Redis

Redis is not bundled with the standalone package and is not required to start the Console or for basic inference. The Console connects to Redis only when a Redis-backed feature is enabled:

| Feature | Redis |
| --- | --- |
| Basic inference | Not required |
| Worker Capacity, `MEMORY` mode (default) | Not required |
| Worker Capacity, `REDIS` mode (shared across several Consoles) | Required |
| Session Affinity (disabled by default) | Required when enabled |

Both features are off by default. Using them requires a separately operated Redis server; see [Redis](installation.md#redis-optional) for the settings. The standalone package is intended for a single Console instance, so the default in-memory Worker Capacity is usually sufficient.

If Redis becomes unavailable while these features are enabled:

- **Session Affinity:** requests are routed normally without affinity.
- **Worker Capacity `REDIS` mode:** Workers with a concurrency limit reject requests (HTTP 429) until Redis recovers; Workers without a limit keep working. The Console still starts if Redis is unreachable at startup, but logs an error and behaves the same way.

## Data Directory

The extracted package directory (the **installation directory**) contains only the application. Accounts, organizations, teams, nodes and logs of requests are stored in **PostgreSQL**. The Console's own settings are stored in a separate **data directory**, `INFRAMESH_HOME`:

| OS | Default |
| --- | --- |
| macOS | `~/.inframesh` |
| Windows | `%USERPROFILE%\.inframesh` |

```text
Installation (replaceable)
└── inframesh-console-<version>/
    ├── bin/        launcher
    ├── app/        Console application
    └── runtime/    bundled Java runtime

Settings (keep)
└── ~/.inframesh/
    ├── config/     standalone.yml (database connection and generated secrets)
    │               application.yml (optional)
    ├── data/       empty (database files of v0.2.4 or earlier, if any)
    ├── logs/       console.log
    └── run/        runtime state of the running Console (port, setup page)

Data (keep)
└── PostgreSQL database
```

- `config/standalone.yml` holds the PostgreSQL connection **including its password**, and the JWT signing key and encryption key the Console generated on the first start with a new database. The file is created with owner-only permissions; it is **not encrypted**. Change the connection only with `inframesh db-config`.
- **The encryption key belongs to the database.** It decrypts the Node API Keys stored in the database. Back up `config/standalone.yml` together with the database. A database restored without its key cannot be used: the Console refuses to start rather than generating a new key. Member API Keys are stored as hashes and cannot be recovered from their stored values.
- Installations that were moved from v0.2.4 or earlier keep their secrets in `config/secrets.yml`. Keep that file; it is still used.
- Directories are created with owner-only permissions on macOS.
- Logs are written to the terminal and to `logs/console.log`. The log file rolls over when it reaches 10 MB or the date changes (`console.log.<yyyy-MM-dd>.<n>.gz`). Rolled files are kept for 14 days, up to 200 MB in total. These are defaults; standard Spring Boot `logging.*` settings (for example in `config/application.yml`) override them.
- The Console does not create database files in `INFRAMESH_HOME`. `data/` stays empty unless v0.2.4 or earlier stored its H2 database there; `backups/` exists only if an earlier version created H2 backups. The Console does not open, change or delete those files.

### Using a different data directory

Set `INFRAMESH_HOME` before running the launcher:

```bash
INFRAMESH_HOME="$HOME/inframesh-data" "./inframesh-console-${VERSION}/bin/inframesh"
```

```powershell
$env:INFRAMESH_HOME = 'D:\inframesh-data'
.\inframesh-console-$version\bin\inframesh.cmd
```

Use the same value every time. A different `INFRAMESH_HOME` has no stored connection and opens the setup page again.

### One Console per data directory

Only one Console instance can use a given `INFRAMESH_HOME` at a time. A second instance using the same directory fails to start. To run several Consoles, give each its own `INFRAMESH_HOME`, `SERVER_PORT` and database.

## Backup and Restore

A complete backup has two parts:

| Part | How |
| --- | --- |
| The PostgreSQL database | PostgreSQL tools, for example `pg_dump` / `pg_restore`. The Console does not back up or restore PostgreSQL. |
| `INFRAMESH_HOME/config/` | Copy `standalone.yml` (and `secrets.yml` if it exists) while the Console is stopped. |

Example with `pg_dump` (custom format):

```bash
pg_dump -h <host> -U infra_mesh -d infra_mesh -Fc -f inframesh-$(date +%Y%m%d).dump
cp -p ~/.inframesh/config/standalone.yml ~/inframesh-standalone-$(date +%Y%m%d).yml
```

To restore, create an empty database, restore the dump into it **before** the Console starts against it, and put the matching `config/standalone.yml` (or `config/secrets.yml`) back:

```bash
pg_restore -h <host> -U infra_mesh -d infra_mesh --no-owner inframesh-20260101.dump
```

- Make sure `pg_dump` finishes without errors. A failed dump can leave an incomplete file.
- Keep the dump and the settings files private. The dump contains account data and encrypted keys, and the settings files contain the database password and the encryption key.
- Do not use `--no-comments`: it drops the record that a database was migrated from H2 (relevant only for databases moved from v0.2.x).

`inframesh backup` was removed in v0.3.0. It changes nothing and exits with code `1` and an explanation. `inframesh restore` and `inframesh recover` remain only to restore H2 backups made by v0.2.4 or earlier for migration — see [Upgrading](upgrade.md#standalone-from-v02x-h2-to-v030).

## Upgrading

Extract each new release into a new directory and run it with the same `INFRAMESH_HOME`. Never delete `INFRAMESH_HOME` when upgrading. See [Upgrading](upgrade.md#standalone).

## Known Limitations

- The packages are not code-signed, and the macOS packages are not notarized.
- No Linux standalone package.
- PostgreSQL is not bundled and is not installed by the Console.
- Only PostgreSQL 16 is a verified version.
- One Console instance per `INFRAMESH_HOME`.
- No built-in TLS. See [Standalone Network Access](standalone-network.md) and [Security](security.md).

## Uninstalling

1. Stop the Console.
2. Delete the extracted `inframesh-console-<version>` directory.
3. Only if you no longer need any data, delete `INFRAMESH_HOME` and drop the PostgreSQL database. This permanently deletes the accounts, the data and the secrets.

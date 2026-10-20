# Standalone Installation

The Standalone Distribution runs InfraMesh Console directly on a local PC. Each package bundles a Java runtime and uses an embedded H2 file database, so Java, Docker, PostgreSQL and Redis do not need to be installed.

The Standalone Distribution is available from Console v0.2.0. It is an additional deployment option; the [Docker deployment](installation.md) remains available and is recommended for servers and Linux.

## Supported Platforms

| OS | Architecture | Package |
| --- | --- | --- |
| macOS | Apple Silicon (arm64) | `inframesh-console-<version>-macos-arm64.tar.gz` |
| macOS | Intel (x64) | `inframesh-console-<version>-macos-x64.tar.gz` |
| Windows | x64 | `inframesh-console-<version>-windows-x64.zip` |

There is no Linux standalone package. Use the [Docker deployment](installation.md) on Linux.

Pick the macOS package that matches your CPU. To check it, run `uname -m` in Terminal: `arm64` means Apple Silicon, `x86_64` means Intel.

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

The commands on this page use a version variable (`VERSION` in the macOS shell and Command Prompt, `$version` in PowerShell). The value `0.2.2` in the examples is only an example: set it to the release you downloaded. The variable lasts only for the current terminal session; set it again in a new terminal.

## macOS

### 1. Verify the checksum

Run this in the directory that contains both files:

```bash
VERSION=0.2.2

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

This creates the `inframesh-console-<version>/` directory. It contains only the application; your data is stored separately (see [Data Directory](#data-directory)).

### 3. Run the launcher

```bash
"./inframesh-console-${VERSION}/bin/inframesh"
```

The launcher prints the data directory, starts the Console, and opens your default browser once the Console is ready:

```text
InfraMesh Console
Home: /Users/<you>/.inframesh

Starting...

InfraMesh Console is ready:
http://127.0.0.1:8080
```

Keep the terminal open while you use the Console. Press `Ctrl+C` in that terminal to stop it.

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
$version = "0.2.2"

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
set VERSION=0.2.2
inframesh-console-%VERSION%\bin\inframesh.cmd
```

The launcher starts the Console and opens your default browser once it is ready. Keep the window open while you use the Console, and press `Ctrl+C` in it to stop.

- Administrator privileges are not required for normal use.
- You do not need to change the PowerShell execution policy. The launcher applies its policy only to its own process.

### SmartScreen

The Windows package is **not code-signed** yet. Windows SmartScreen or your antivirus may show a warning. Before allowing it, confirm that you downloaded the ZIP from the official Releases page and that its SHA-256 checksum matches.

## First Run

```text
Run the launcher
    │
    ▼
Initialize INFRAMESH_HOME (directories, generated secrets)
    │
    ▼
Initialize the embedded H2 database
    │
    ▼
Start InfraMesh Console
    │
    ▼
Wait until the Console reports ready
    │
    ▼
Open the default browser
    │
    ▼
Initial Administrator Setup
```

On the first launch, the browser shows the **Initial Administrator Setup** page. Create the administrator account there; it is stored in the data directory and kept across restarts and upgrades.

The default address is:

```text
http://127.0.0.1:8080
```

If you change `SERVER_PORT`, use that port instead. The launcher detects the port the Console actually listens on and opens that address.

The launcher waits up to 60 seconds for the Console to become ready; if it is not ready by then, the launcher stops the Console and exits with an error. If the Console exits early (for example, because the port is already in use or another Console is using the same data directory), the launcher prints an error and exits.

## Configuration

The launcher takes no command-line arguments. Configure it with environment variables:

| Variable | Default | Description |
| --- | --- | --- |
| `INFRAMESH_HOME` | `~/.inframesh` (macOS)<br>`%USERPROFILE%\.inframesh` (Windows) | Data and configuration directory |
| `SERVER_PORT` | `8080` | Console HTTP port |
| `INFRAMESH_OPEN_BROWSER` | `true` | Set to `false` to not open the browser automatically. Only `true` or `false` are accepted. |
| `SPRING_PROFILES_ACTIVE` | `prod` | Spring profile. Leave it at the default. |
| `SERVER_ADDRESS` | `127.0.0.1` | Network address to listen on. See [Security](security.md#standalone) before changing it. |

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

The extracted package directory (the **installation directory**) contains only the application. All persistent data is stored in a separate **data directory**, `INFRAMESH_HOME`:

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

Persistent Data (keep)
└── ~/.inframesh/
    ├── config/     secrets.yml (generated), application.yml (optional)
    ├── data/       embedded H2 database
    ├── logs/       created, currently unused
    └── run/        runtime state of the running Console (port)
```

- The H2 database in `data/` holds accounts (including the administrator), organizations, teams, nodes and all other Console data. It is kept when the Console restarts and when you upgrade the package.
- On the first launch, the Console generates its JWT signing key and encryption key and stores them in `config/secrets.yml`. **The database and `secrets.yml` belong together** and must be backed up together: the encryption key is required to decrypt encrypted credentials such as Node API Keys. Member API Keys are stored as hashes and cannot be recovered from their stored values. If the database exists but `secrets.yml` is missing, the Console refuses to start rather than generating new keys.
- Directories are created with owner-only permissions on macOS.
- Logs are written to the terminal. The `logs/` directory is created but not used yet.

### Using a different data directory

Set `INFRAMESH_HOME` before running the launcher:

```bash
INFRAMESH_HOME="$HOME/inframesh-data" "./inframesh-console-${VERSION}/bin/inframesh"
```

```powershell
$env:INFRAMESH_HOME = 'D:\inframesh-data'
.\inframesh-console-$version\bin\inframesh.cmd
```

Use the same value every time. A different `INFRAMESH_HOME` starts with an empty database.

### One Console per data directory

Only one Console instance can use a given `INFRAMESH_HOME` at a time. A second instance using the same directory fails to start. To run several Consoles, give each its own `INFRAMESH_HOME` and `SERVER_PORT`.

### Backup

1. Stop the Console (`Ctrl+C`). Do not copy the database while the Console is running.
2. Copy the **entire** `INFRAMESH_HOME` directory — at least `config/` and `data/` together.

```bash
cp -Rp ~/.inframesh ~/inframesh-backup-$(date +%Y%m%d)
```

```powershell
Copy-Item -Recurse "$env:USERPROFILE\.inframesh" "$env:USERPROFILE\inframesh-backup-$(Get-Date -Format yyyyMMdd)"
```

- A backup contains secrets (`secrets.yml`) and the database. Store it with the same care as passwords.
- Restoring `data/` without the matching `config/secrets.yml` (or with one from a different installation) leaves the Console unable to start or unable to decrypt stored keys.
- To restore, stop the Console and replace the `INFRAMESH_HOME` directory with the backup.

## Upgrading

Extract each new release into a new directory and run it with the same `INFRAMESH_HOME`. Never delete `INFRAMESH_HOME` when upgrading. See [Upgrading](upgrade.md#standalone).

## Known Limitations

- The packages are not code-signed, and the macOS packages are not notarized.
- No Linux standalone package.
- One Console instance per `INFRAMESH_HOME`.
- No built-in TLS. See [Security](security.md).
- Logs are written to the terminal only.

## Uninstalling

1. Stop the Console.
2. Delete the extracted `inframesh-console-<version>` directory.
3. Only if you no longer need any data, delete `INFRAMESH_HOME`. This permanently deletes the database, accounts and secrets.

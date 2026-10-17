# Security

This page describes the security characteristics of the standalone and Docker deployments and what you are responsible for when operating them.

## Network Exposure

### Standalone

By default the standalone Console listens only on the loopback address:

```text
127.0.0.1
```

It is reachable only from the same PC. This is the intended way to use the standalone package.

`SERVER_ADDRESS` changes the listen address (for example `0.0.0.0` to accept connections from other hosts). When it is set to a non-loopback address, the Console logs a warning at startup. Do not do this on untrusted networks. If other machines must reach the Console, prefer the [Docker deployment](installation.md) behind a reverse proxy; at minimum, put TLS, a firewall and network access controls in front of it.

### Docker

The Compose stack publishes only the Console port (`INFRA_CONSOLE_PORT`, default `8080`) on all host interfaces. PostgreSQL and Redis are not published. Restrict access to the Console port with your firewall or security groups, and put a reverse proxy in front for HTTPS.

## TLS

Neither deployment terminates TLS. The Console serves plain HTTP.

- Do not expose the Console directly to the internet or other untrusted networks.
- If external access is required, terminate TLS in a reverse proxy or load balancer (Nginx, Caddy, Traefik, …) in front of the Console. See [HTTPS and Reverse Proxy](installation.md#https-and-reverse-proxy) for the proxy requirements (forwarded headers, WebSocket upgrades).

```text
Client ──HTTPS──▶ Reverse Proxy ──HTTP──▶ InfraMesh Console
```

## Secrets and Credentials

Treat the following as secrets:

| Item | Standalone | Docker |
| --- | --- | --- |
| Administrator and user accounts | Stored in the H2 database under `INFRAMESH_HOME/data` | Stored in PostgreSQL |
| Bootstrap admin password | Not used by default — the first administrator is created in the browser | `INFRA_ADMIN_PASSWORD` in `.env` |
| JWT signing key | Generated into `INFRAMESH_HOME/config/secrets.yml` | `JWT_SECRET_KEY` in `.env` |
| Encryption key (Team Keys, Node API Keys) | Generated into `INFRAMESH_HOME/config/secrets.yml` | `ENCRYPTION_KEY` in `.env` |
| Node API Keys and Team Keys | Stored encrypted in the database | Stored encrypted in the database |
| Database | `INFRAMESH_HOME/data` | `postgres-data` volume, `POSTGRES_PASSWORD` in `.env` |

Recommendations:

- Never commit or share `.env`, `secrets.yml`, database files or database dumps. `.env` is ignored by Git in this repository.
- Keep backups of `INFRAMESH_HOME`, `backup.sql` and `.env` in a location only you or your operators can read.
- Use strong, unique passwords for administrator accounts.
- Never change the encryption key once data exists: previously stored keys can no longer be decrypted. Changing the JWT signing key signs every user out.
- When setting secrets through environment variables, remember that other processes run by the same user, shell history and process listings may expose them.
- Do not paste real keys, passwords or Node API Keys into issues, logs or screenshots. The examples in this repository never contain real credentials.

## Package Integrity

Current limitations of the v0.2.0 standalone packages:

- macOS packages are not code-signed.
- macOS packages are not notarized.
- The Windows package is not code-signed.

As a result, macOS Gatekeeper and Windows SmartScreen may warn about or block the packages. Therefore:

1. Download packages only from the official Releases page: <https://github.com/InfraMeshLabs/infra-deploy/releases>
2. Verify the SHA-256 checksum before running a package (see [Standalone Installation](standalone.md)).
3. Do not disable Gatekeeper or SmartScreen system-wide. Allow only the package you verified.

A SHA-256 check confirms that the file you downloaded is identical to the file the published checksum describes — that it was not corrupted or altered in transit. Because the checksum is published in the same place as the package, it does **not** prove who published the package, and it does not replace code signing or publisher verification.

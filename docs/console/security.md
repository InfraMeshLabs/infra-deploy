# Security

This page describes the security characteristics of the standalone and Docker deployments and what you are responsible for when operating them.

## Network Exposure

### Standalone

By default the standalone Console listens only on the loopback interface:

```text
127.0.0.1:8080
```

It accepts requests only from the same PC. This is the intended way to use the standalone package.

`SERVER_ADDRESS` changes the listen address so that the Console also accepts requests on other network interfaces. `SERVER_ADDRESS=0.0.0.0` accepts requests on every IPv4 interface of the machine, so anyone who can reach the PC over the network can reach the Console. When it is set to a non-loopback address, the Console logs a warning at startup. Do not do this on untrusted networks.

Needing remote access does not mean you have to change `SERVER_ADDRESS`:

- For private remote access from your own devices or a small private group, keep the default address and use [Tailscale Serve](#private-remote-access-with-tailscale).
- For servers that expose the Console to other users or networks, prefer the [Docker deployment](installation.md) behind a reverse proxy; at minimum, put TLS, a firewall and network access controls in front of it.

```text
Standalone
    │
    ├── Local Access
    │   └── http://127.0.0.1:8080
    │
    └── Private Remote Access
        └── Tailscale Serve
            └── HTTPS (Tailnet)
```

#### Private Remote Access with Tailscale

[Tailscale](https://tailscale.com) builds a private network (a *tailnet*) between the users and devices you authorize. [Tailscale Serve](https://tailscale.com/kb/1312/serve) can publish a local HTTP service to that tailnet over HTTPS. This is optional — InfraMesh does not depend on Tailscale.

With Serve:

- The Console keeps running on `127.0.0.1:8080`. You do not change `SERVER_ADDRESS`.
- Port `8080` is not opened to your LAN or to the internet.
- Tailscale Serve terminates HTTPS and forwards requests to the Console over loopback HTTP.
- Who can actually reach the Console is decided by your tailnet's [grants / access control policy](https://tailscale.com/kb/1018/acls).

```text
Mac / Windows
│
├── InfraMesh Console
│   └── 127.0.0.1:8080 (HTTP)
│
└── Tailscale Serve
    └── HTTPS
         ▲
         │
         │ Tailnet
         │
    Authorized Devices
```

**Prerequisites**

- The standalone Console is installed and running.
- Tailscale is installed on the PC running the Console, and you are logged in.
- The devices you want to connect from are logged in to the same tailnet.
- HTTPS certificates are enabled for your tailnet, as required by Tailscale Serve (see the [Serve documentation](https://tailscale.com/kb/1312/serve)).
- Your tailnet's grants / ACL policy allows those users or devices to reach the Console PC.

**Step 1. Start the Console**

Start the standalone Console as usual and confirm that it opens locally:

```text
http://127.0.0.1:8080
```

**Step 2. Start Tailscale Serve**

On the PC running the Console:

```bash
tailscale serve --bg 8080
```

This proxies HTTPS requests on the tailnet to `http://127.0.0.1:8080` and keeps the configuration running in the background, including after the terminal is closed. The CLI has changed between Tailscale versions; if the command is rejected, check `tailscale serve --help` and the [Serve CLI reference](https://tailscale.com/kb/1242/tailscale-serve).

**Step 3. Find the address**

```bash
tailscale serve status
```

The output shows the HTTPS address, for example:

```text
https://your-device.your-tailnet.ts.net
```

This is only an example. Your machine name and tailnet name will differ.

**Step 4. Connect from another device**

On an authorized device in the tailnet, open that HTTPS address in a browser and sign in to the Console as usual.

```text
Browser
   │
   │ HTTPS
   ▼
Tailscale Serve
   │
   │ HTTP (Loopback)
   ▼
InfraMesh Console
127.0.0.1:8080
```

**Step 5. Remove the configuration**

```bash
tailscale serve reset
```

`reset` clears **all** Serve configuration on this device, including anything else you serve with it. To remove only the Console, turn off its handler instead:

```bash
tailscale serve --https=443 off
```

The command syntax may differ between Tailscale versions. Check `tailscale serve --help` if the command is not supported.

**Security notes**

- **Serve is not Funnel.** Tailscale Funnel publishes a service to the public internet. Do not use Funnel for the Console; this page covers Serve only.
- Being on the tailnet does not grant access to everything. Use grants / ACLs to allow only the users and devices that need the Console.
- Keep `SERVER_ADDRESS` at its default `127.0.0.1`. Serve reaches the Console over loopback, so there is no reason to listen on other interfaces.
- **Direct Tailscale IPv4 access:** You can also access the Console directly over HTTP using its Tailscale IPv4 address if the Console is bound to that address. Bind to the Tailscale IPv4 address itself rather than `0.0.0.0`, which listens on every IPv4 interface. However, this is not the recommended configuration. Direct HTTP access does not provide a browser Secure Context, so some browser features may be unavailable. Prefer Tailscale Serve for private HTTPS access without changing the default loopback binding.
- HTTPS ends at Tailscale Serve; the Console still receives plain HTTP over loopback.
- Serve does not replace InfraMesh authentication or authorization. Console accounts and API Key authentication still apply to every request.
- Do not share the administrator password, Member API Keys or Node API Keys with other people, even inside your tailnet.

### Docker

The Compose stack publishes only the Console port (`INFRA_CONSOLE_PORT`, default `8080`) on all host interfaces. PostgreSQL and Redis are not published. Restrict access to the Console port with your firewall or security groups, and put a reverse proxy in front for HTTPS.

The Console uses forwarded headers (`Forwarded`, `X-Forwarded-*`) to determine the original request scheme, which also decides whether refresh token cookies get the `Secure` attribute. It does not check which client sent these headers. When deploying the Console behind a reverse proxy:

- Restrict direct access to the Console port so that clients cannot bypass the reverse proxy. This matters even with a correctly configured proxy.
- Configure the reverse proxy to overwrite `Forwarded` and `X-Forwarded-*` headers supplied by clients instead of passing them through.
- Set the appropriate forwarded headers, including `X-Forwarded-Proto: https`, when the proxy terminates TLS.

See [HTTPS and Reverse Proxy](installation.md#https-and-reverse-proxy).

### Ollama

Ollama has no API key authentication by default. Connecting it from another PC requires binding it to a network interface (`OLLAMA_HOST`), after which any client that can reach that address can call the Ollama API.

- Never expose the Ollama API directly to the internet. Prefer binding to a specific LAN or Tailscale address over `0.0.0.0`, and allow only the Console PC through the firewall.
- InfraMesh Node API Keys do not protect the Ollama API, and the Console sends no credentials when it calls Ollama directly.

See [Ollama Remote Access](ollama-remote-access.md) for setup and further notes.

## TLS

Neither deployment terminates TLS. The Console itself serves plain HTTP; HTTPS must be provided by something in front of it, such as a reverse proxy, a load balancer or Tailscale Serve.

- Do not expose the Console directly to the internet or other untrusted networks.
- If external access is required, terminate TLS in a reverse proxy or load balancer (Nginx, Caddy, Traefik, …) in front of the Console. See [HTTPS and Reverse Proxy](installation.md#https-and-reverse-proxy) for the proxy requirements (forwarded headers, WebSocket upgrades).

```text
Client ──HTTPS──▶ Reverse Proxy ──HTTP──▶ InfraMesh Console
```

- For private remote access to the standalone Console, [Tailscale Serve](#private-remote-access-with-tailscale) can terminate HTTPS inside your tailnet and forward requests over loopback. It is not a replacement for a reverse proxy or load balancer in a general service deployment.

```text
Tailnet Client ──HTTPS──▶ Tailscale Serve ──HTTP (Loopback)──▶ InfraMesh Console
```

## Secrets and Credentials

Treat the following as secrets:

| Item | Standalone | Docker |
| --- | --- | --- |
| Administrator and user accounts | Stored in the H2 database under `INFRAMESH_HOME/data` | Stored in PostgreSQL |
| Bootstrap admin password | Not used by default — the first administrator is created in the browser | `INFRA_ADMIN_PASSWORD` in `.env` |
| JWT signing key | Generated into `INFRAMESH_HOME/config/secrets.yml` | `JWT_SECRET_KEY` in `.env` |
| Encryption key (Node API Keys) | Generated into `INFRAMESH_HOME/config/secrets.yml` | `ENCRYPTION_KEY` in `.env` |
| Member API Keys | Stored as SHA-256 hashes in the database | Stored as SHA-256 hashes in the database |
| Node API Keys | Stored encrypted in the database | Stored encrypted in the database |
| OUTBOUND node credentials and registration tokens | Stored as SHA-256 hashes in the database | Stored as SHA-256 hashes in the database |
| Database | `INFRAMESH_HOME/data` | `postgres-data` volume, `POSTGRES_PASSWORD` in `.env` |

Recommendations:

- Never commit or share `.env`, `secrets.yml`, database files or database dumps. `.env` is ignored by Git in this repository.
- Keep backups of `INFRAMESH_HOME`, `backup.sql` and `.env` in a location only you or your operators can read.
- Use strong, unique passwords for administrator accounts.
- Never change the encryption key once data exists: previously stored keys can no longer be decrypted. Changing the JWT signing key signs every user out.
- When setting secrets through environment variables, remember that other processes run by the same user, shell history and process listings may expose them.
- Do not paste real keys, passwords or Node API Keys into issues, logs or screenshots. The examples in this repository never contain real credentials.

Hashed values cannot be recovered: a Member API Key is shown only once when it is created. Encrypted values can be decrypted with the encryption key, which is why the encryption key, the database and their backups must be protected together.

### Credential Compromise

If a credential is exposed:

- **Member API Key:** revoke it (the key owner or a team administrator can) and create a new one. A revoked key cannot be restored.
- **Node API Key:** regenerate it on the node's detail panel (**Regenerate**), then update the key configured on the node. The previous key stops working immediately.
- **OUTBOUND node credential:** there is no rotation for a registered node's credential. Delete the node and register it again with a new registration command.
- **User password:** have an administrator reset the password; this also revokes the account's refresh tokens and forces a new sign-in. Changing your own password does not sign out existing sessions. In both cases, access tokens already issued remain valid until they expire.
- **Activity review:** administrators can review the **Audit Logs** page for unexpected activity, such as key creation, revocation or rotation.
- **JWT signing key:** replace it and restart the Console. Every issued token becomes invalid and every user has to sign in again.

## Package Integrity

Current limitations of the v0.2.2 standalone packages:

- macOS packages are not code-signed.
- macOS packages are not notarized.
- The Windows package is not code-signed.

As a result, macOS Gatekeeper and Windows SmartScreen may warn about or block the packages. Therefore:

1. Download packages only from the official Releases page: <https://github.com/InfraMeshLabs/infra-deploy/releases>
2. Verify the SHA-256 checksum before running a package (see [Standalone Installation](standalone.md)).
3. Do not disable Gatekeeper or SmartScreen system-wide. Allow only the package you verified.

A SHA-256 check confirms that the file you downloaded is identical to the file the published checksum describes — that it was not corrupted or altered in transit. Because the checksum is published in the same place as the package, it does **not** prove who published the package, and it does not replace code signing or publisher verification.

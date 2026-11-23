# Standalone Network Access

This page explains how to run the standalone Console so that other devices can reach it: browsers on other PCs, and OUTBOUND Worker / Router nodes running on other machines.

The standalone Console serves **plain HTTP only**. It has no built-in TLS. Every setup on this page that leaves the local PC must therefore be planned with that in mind.

| Setup | Who can reach the Console | Transport | When to use |
| --- | --- | --- | --- |
| [Local only](#local-only-default) (default) | The same PC | HTTP over loopback | Personal use, development, evaluation |
| [LAN / VPN](#lan--vpn-access) | Devices on a trusted private network | HTTP (not encrypted) | Small trusted networks, short-term use |
| [TLS reverse proxy](#external-access-with-a-tls-reverse-proxy) | Clients you allow through the proxy | HTTPS / WSS to the proxy, HTTP over loopback to the Console | Any network you do not fully trust |

For private remote access from your own devices, [Tailscale Serve](security.md#private-remote-access-with-tailscale) is another option that keeps the default loopback binding.

## Local Only (Default)

By default the standalone Console listens on the loopback interface only:

```text
http://127.0.0.1:8080
```

- Only the same PC can open the Console or connect nodes to it. Other devices on the network cannot reach it.
- No reverse proxy, certificate or firewall change is needed.
- At startup the Console logs: `Standalone Console listens on 127.0.0.1 only. Set server.address to accept connections from other hosts.`

Keep this default unless another device actually needs to reach the Console.

## Before Exposing the Console

Complete these steps on the PC itself, while the Console still listens on `127.0.0.1`:

1. **Create the administrator account.** Initial Setup is accepted only from the PC running the Console, over a direct loopback connection. It is rejected (`SETUP_LOOPBACK_ONLY`) when the request comes from another host or through a reverse proxy, and loopback is not reachable at all once the Console is bound to a specific non-loopback address.
2. Use a strong administrator password. Anyone who can reach the Console can try to sign in.
3. Back up `INFRAMESH_HOME` (see [Backup](standalone.md#backup)).

## LAN / VPN Access

`SERVER_ADDRESS` changes the address the Console listens on.

Bind to the **specific private address** that other devices use (for example, the PC's LAN or VPN address) rather than `0.0.0.0`:

macOS:

```bash
SERVER_ADDRESS=<private-ip> "./inframesh-console-${VERSION}/bin/inframesh"
```

Windows (PowerShell):

```powershell
$env:SERVER_ADDRESS = '<private-ip>'
.\inframesh-console-$version\bin\inframesh.cmd
```

You can also set `server.address` in `<INFRAMESH_HOME>/config/application.yml`; environment variables take precedence over that file.

What changes when `SERVER_ADDRESS` is set:

- The Console listens only on that address. `127.0.0.1` is no longer reachable, so the browser opened by the launcher uses `http://<private-ip>:<port>`.
- The Console logs a warning at startup: `Standalone Console is bound to a non-loopback address (...). Configure TLS, firewall, and network access controls before exposing it to untrusted networks.`
- `SERVER_ADDRESS=0.0.0.0` listens on **every** IPv4 interface, including Wi-Fi, VPN and virtual adapters. Avoid it unless you have checked every interface it exposes.
- If the address is not assigned to the PC, the Console fails to start (`java.net.BindException: Can't assign requested address`) and the launcher reports that the Console exited before becoming ready.

**Plain HTTP is not encrypted, even on a private network.** Passwords, access tokens, Member API Keys, prompts and responses travel in clear text and can be read by anything that can observe the network path. Being on a LAN or a VPN does not make HTTP safe by itself. Use this setup only on networks you trust, and prefer the [TLS reverse proxy](#external-access-with-a-tls-reverse-proxy) setup otherwise. Browsers also do not treat `http://<private-ip>` as a secure context.

Restrict who can reach the port with a firewall — see [Firewall Recommendations](#firewall-recommendations).

## External Access with a TLS Reverse Proxy

Use a reverse proxy that terminates TLS whenever clients or nodes connect over a network you do not fully trust.

```text
Browser / API client / OUTBOUND node
               │
          HTTPS / WSS
               │
     TLS reverse proxy (same PC)
               │
     HTTP / WS over loopback
               │
  InfraMesh Standalone Console (127.0.0.1)
```

- Run the reverse proxy **on the same PC** and keep the Console on its default `127.0.0.1`. The proxy reaches the Console over loopback, and nothing else can reach the Console port directly.
- **Never expose the Console's HTTP port to the internet.** Only the proxy's HTTPS port should be reachable from other networks.
- The reverse proxy is optional and not bundled. You operate it, its certificate and its configuration.

### Requirements

The Console relies on the proxy for the following:

| Requirement | Why |
| --- | --- |
| Terminate TLS and redirect HTTP to HTTPS | The Console itself serves HTTP only. |
| Send `X-Forwarded-Proto: https` | The Console marks the refresh token cookie `Secure` (and sends `Strict-Transport-Security`) only when it sees an HTTPS request. |
| Send `Host` and `X-Forwarded-Host` with the public host **and port** that browsers use | The Console compares the browser's `Origin` with the request's scheme, host and port. If they do not match, browser sign-in fails with HTTP 403. |
| **Overwrite** `Forwarded` and all `X-Forwarded-*` headers sent by clients | The Console uses these headers as-is and does not check who sent them. A client that can set them can change what the Console believes about the request. |
| Pass WebSocket upgrades for `/ws/v1/nodes/connect` | OUTBOUND Worker / Router nodes keep a WebSocket connection to the Console. |
| Read timeout longer than 60 seconds | Non-streaming inference waits up to 60 seconds for a Worker (`NODE_OUTBOUND_REQUEST_TIMEOUT`), and streaming responses may pause up to 60 seconds between chunks (`NODE_OUTBOUND_STREAM_IDLE_TIMEOUT`). OUTBOUND nodes send a heartbeat every 10 seconds by default. A shorter proxy timeout cuts these off with a proxy error (for example, HTTP 504). |
| Disable response buffering | Streaming inference responses (`text/event-stream`) should reach the client as each chunk arrives. |
| Allow request bodies of at least 10 MB | The Console accepts inference request bodies up to 10 MB (`INFERENCE_MAX_REQUEST_BODY_SIZE`). Nginx rejects bodies larger than 1 MB by default, before the Console sees them. |

### Example: Nginx

The following is an example for Nginx, the only proxy this page gives a configuration for. Replace `console.example.com` and the certificate paths with your own. It assumes the Console runs on the same PC on the default port `8080`.

```nginx
map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;
}

server {
    listen 80;
    server_name console.example.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name console.example.com;

    ssl_certificate     /etc/ssl/inframesh/fullchain.pem;
    ssl_certificate_key /etc/ssl/inframesh/privkey.pem;

    # At least the Console inference limit (10 MB) - Nginx's default is 1 MB.
    client_max_body_size 11m;

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_http_version 1.1;

        # Overwrite forwarded headers - never pass client-supplied values through.
        proxy_set_header Host              $http_host;
        proxy_set_header X-Forwarded-Host  $http_host;
        proxy_set_header X-Forwarded-Proto https;
        proxy_set_header X-Forwarded-For   $remote_addr;
        proxy_set_header X-Forwarded-Port  "";
        proxy_set_header X-Real-IP         "";
        proxy_set_header Forwarded         "";

        # WebSocket for OUTBOUND nodes (/ws/v1/nodes/connect).
        proxy_set_header Upgrade    $http_upgrade;
        proxy_set_header Connection $connection_upgrade;

        # Streaming responses.
        proxy_buffering off;
        proxy_request_buffering off;

        # Longer than the Console's 60-second request / stream idle timeouts.
        proxy_read_timeout 90s;
        proxy_send_timeout 90s;
    }
}
```

Notes:

- `$http_host` is the `Host` header the browser sent, including the port if it is not the default. Keep it when the proxy listens on a non-standard HTTPS port, otherwise browser sign-in fails with 403.
- `X-Forwarded-For` is set to the connecting address instead of appending to a client-supplied value.
- Use a certificate that clients already trust (for example, from a public CA or your organization's CA). With a self-signed or private-CA certificate, every browser and every OUTBOUND node must be configured to trust it.

## Firewall Recommendations

- **Local only / TLS reverse proxy:** no inbound rule is needed for the Console port, because it listens on `127.0.0.1`. Allow inbound HTTPS (443) to the proxy only from the networks or addresses that need it, and allow port 80 only if you want the HTTP-to-HTTPS redirect.
- **LAN / VPN:** allow inbound TCP to the Console port (default `8080`) only from the specific devices or subnet that need it.
- Never forward the Console port from a router or cloud security group to the internet.
- After changing the setup, check from another device that only the intended port answers.

How to create the rules depends on your operating system and network equipment. The operating system firewall may also ask whether to allow incoming connections the first time the Console listens on a non-loopback address.

## OUTBOUND Node Connectivity

OUTBOUND Worker and Router nodes open a connection **to** the Console, so the Console must be reachable from the node's machine. With the default `127.0.0.1` binding, only nodes on the same PC can connect.

1. **Choose the Console URL the node will use:**

   | Setup | `console-url` |
   | --- | --- |
   | Same PC | `http://127.0.0.1:8080` |
   | LAN / VPN | `http://<private-ip>:8080` |
   | TLS reverse proxy | `https://console.example.com` |

   The node derives the WebSocket address from this URL: `http://` becomes `ws://`, `https://` becomes `wss://`, and the path is always `/ws/v1/nodes/connect`. Use the same scheme, host and port that a browser would use. Do not add the credential or any other value to the URL.

2. **Register the node.** In the Console, create the node with the `OUTBOUND` connection mode and issue a registration token. Send the token once to `POST <console-url>/api/v1/nodes/register` (request body: `registrationToken`, `name`, `nodeType`); the response contains the node's `nodeId` and `credential`. Registration uses the same URL as the browser, so with a reverse proxy it also goes through the proxy. A registration token is short-lived (10 minutes by default), can be used once, and can be issued only for a node that has not been registered yet. The credential is returned only in this response — store it right away.

3. **Configure the node** with the issued values. The property names are the same for Worker and Router:

   ```yaml
   inframesh:
     node:
       connection-mode: OUTBOUND
       console-url: ${INFRAMESH_CONSOLE_URL}
       node-id: ${INFRAMESH_NODE_ID}
       credential: ${INFRAMESH_NODE_CREDENTIAL}
   ```

   Keep the credential in an environment variable or a secret store, not in source control. The node sends it in request headers during the WebSocket handshake, never in the URL.

4. **Trust the certificate.** With a TLS reverse proxy, the node must trust the proxy's certificate. Certificates from a public CA usually need no extra configuration. For a private CA or a self-signed certificate, add it to the Java trust store the node uses.

5. **Check the connection.** The node appears as connected in the Console's node list, and the Console log shows `Node connection established`. Then send a real inference request through the Console.

## Security Warnings

- The Console has no built-in TLS. Without a TLS reverse proxy, everything — including passwords and API keys — crosses the network in clear text.
- Do not expose the Console's HTTP port to the internet, even behind a password.
- Clients must not be able to bypass the reverse proxy. The Console trusts `Forwarded` / `X-Forwarded-*` headers from whoever sends them; a client that reaches the Console port directly can claim that its request arrived over HTTPS.
- Initial Setup works only from the PC itself. Do not try to open it to other hosts.
- Console accounts, Member API Keys and Node credentials still protect every request behind the proxy. Do not share them, and revoke and reissue them if they may have been exposed (see [Credential Compromise](security.md#credential-compromise)).
- `0.0.0.0` exposes the Console on every IPv4 interface of the PC. Prefer a specific address.

## Troubleshooting

| Symptom | Likely cause | What to check |
| --- | --- | --- |
| Other devices cannot connect at all | The Console still listens on `127.0.0.1` | Set `SERVER_ADDRESS` (LAN / VPN), or connect through the reverse proxy. Check the startup log line. |
| Console exits at startup with `Can't assign requested address` | `SERVER_ADDRESS` is not an address of this PC | Use an address shown by the operating system's network settings. |
| Initial Setup fails with `SETUP_LOOPBACK_ONLY` | Setup was opened from another host or through the proxy | Run it on the PC itself while the Console listens on `127.0.0.1`. |
| Browser sign-in through the proxy fails with HTTP 403 | The proxy does not pass the public scheme, host and port (`X-Forwarded-Proto`, `Host` / `X-Forwarded-Host`) | Compare the proxy headers with the [example](#example-nginx), including the port. |
| Large requests fail with HTTP 413 and an HTML error page | The proxy's own body limit (Nginx: 1 MB by default) | Set `client_max_body_size` to at least 10 MB. A JSON 413 (`COMMON_PAYLOAD_TOO_LARGE`) comes from the Console's own 10 MB limit. |
| Slow inference fails with HTTP 504 from the proxy | Proxy read timeout shorter than the Worker response time | Use a read timeout longer than 60 seconds. |
| Streaming responses arrive all at once | Proxy response buffering | Disable buffering (`proxy_buffering off`). |
| OUTBOUND node keeps reconnecting, handshake returns HTTP 401 | Wrong `node-id` or `credential`, or the node is not in `OUTBOUND` mode | Re-check the issued values; reissue the credential if needed. |
| OUTBOUND node cannot connect through the proxy (not 401) | WebSocket upgrade not passed, wrong scheme / port in `console-url`, or the node does not trust the certificate | Check the `Upgrade` / `Connection` headers in the proxy, the `console-url`, and the node's trust store. |

# Ollama Remote Access

The Console can use [Ollama](https://ollama.com) directly as a Worker runtime. This page explains how to connect an Ollama server that runs on a **different PC** from the Console, and how to troubleshoot the connection.

InfraMesh never changes Ollama's settings. You configure Ollama's network binding yourself on the PC that runs it.

## Connection Layouts

### Same PC

When Ollama runs on the same PC as a standalone Console, the default Ollama settings work as-is. Ollama listens on `127.0.0.1:11434`.

```text
PC A
 ├── InfraMesh Console
 │
 └── Ollama
      └── 127.0.0.1:11434
```

Endpoint to register:

```text
http://127.0.0.1:11434
```

If the Console runs in a Docker container, `localhost` / `127.0.0.1` means the Console container itself, not the PC. See [Docker](#docker).

### Different PC

When Ollama runs on another PC, the Console reaches it over the network. Ollama must accept requests on a network interface, not only on loopback.

```text
PC A                          PC B
┌──────────────────┐          ┌──────────────────┐
│ InfraMesh        │          │ Ollama           │
│ Console          │          │                  │
│                  │  HTTP    │ 0.0.0.0:11434    │
│                  │ ───────► │                  │
└──────────────────┘          └──────────────────┘
```

Example:

```text
Console IP  : 192.168.0.10
Ollama IP   : 192.168.0.100
Ollama Port : 11434
```

Endpoint to register:

```text
http://192.168.0.100:11434
```

## Allowing Network Access in Ollama

Ollama reads its listen address from the `OLLAMA_HOST` environment variable. The default is `127.0.0.1:11434`, which accepts requests only from the same PC.

To accept requests on every IPv4 interface:

```text
OLLAMA_HOST=0.0.0.0:11434
```

To accept requests only on one LAN address, use the IP of the Ollama PC itself:

```text
OLLAMA_HOST=192.168.0.100:11434
```

The address must be assigned to a network interface of the PC that runs Ollama; otherwise Ollama fails to start. Binding to a specific address limits exposure to that network, so prefer it over `0.0.0.0` when the address is stable.

> [!WARNING]
> Ollama has no authentication by default. Anyone who can reach the bound address can call the Ollama API. Read [Security](#security) before changing `OLLAMA_HOST`.

Setting the variable does not affect an Ollama process that is already running. Restart Ollama after every change.

### Windows

Set a user environment variable in PowerShell:

```powershell
[Environment]::SetEnvironmentVariable(
    "OLLAMA_HOST",
    "0.0.0.0:11434",
    "User"
)
```

Then restart Ollama completely:

1. Right-click the Ollama icon in the system tray (taskbar notification area) and choose **Quit Ollama**.
2. Start Ollama again from the Start menu.

Already running processes and open terminals keep the old environment. If you run `ollama serve` in a terminal, open a new terminal after setting the variable.

### macOS

If you use the Ollama app:

```bash
launchctl setenv OLLAMA_HOST "0.0.0.0:11434"
```

Then quit Ollama from the menu bar and start it again. A value set with `launchctl setenv` does not survive a reboot; run the command again after restarting the Mac.

If you run the server from a terminal:

```bash
OLLAMA_HOST=0.0.0.0:11434 ollama serve
```

Quit the Ollama app (or any other running `ollama serve`) first; otherwise the port is already in use and the new server fails to start.

### Linux

If you run the server from a terminal:

```bash
OLLAMA_HOST=0.0.0.0:11434 ollama serve
```

If Ollama runs as a systemd service (the default for the official install script):

```bash
sudo systemctl edit ollama.service
```

Add:

```ini
[Service]
Environment="OLLAMA_HOST=0.0.0.0:11434"
```

Apply the change:

```bash
sudo systemctl daemon-reload
sudo systemctl restart ollama
```

Check the service:

```bash
sudo systemctl status ollama
```

## Testing the Connection

Run these checks in order. The first failing step tells you where the problem is.

### 1. On the Ollama PC, over loopback

```bash
curl http://127.0.0.1:11434/api/tags
```

A running Ollama returns its model list:

```json
{
  "models": []
}
```

If models are installed, the `models` array contains them.

### 2. On the Ollama PC, over its LAN IP

```bash
curl http://192.168.0.100:11434/api/tags
```

If step 1 works but this fails, Ollama is still bound to loopback only. Check `OLLAMA_HOST` and restart Ollama.

### 3. From the Console PC

```bash
curl http://192.168.0.100:11434/api/tags
```

An HTTP 200 response with the model list means basic API connectivity works. If step 2 works but this fails, the problem is between the two PCs: firewall, IP address or network. See [Troubleshooting](#troubleshooting).

### Windows PowerShell

In Windows PowerShell, `curl` can be an alias of `Invoke-WebRequest`, which takes different arguments. If you see a prompt such as `Supply values for the following parameters: Uri:`, you are running the alias. Use `curl.exe` instead:

```powershell
curl.exe -i http://192.168.0.100:11434/api/tags
```

Or:

```powershell
Invoke-RestMethod -Uri "http://192.168.0.100:11434/api/tags"
```

## Checking the Listen Address

To see which address Ollama is actually listening on:

Windows:

```powershell
netstat -ano | findstr 11434
```

macOS / Linux:

```bash
lsof -nP -iTCP:11434 -sTCP:LISTEN
```

| Output | Meaning |
| --- | --- |
| `127.0.0.1:11434` | Loopback only — reachable from the same PC only |
| `0.0.0.0:11434` (`*:11434` in `lsof`) | All IPv4 interfaces |
| `192.168.0.100:11434` | That LAN address only |

A non-loopback binding only means Ollama accepts connections on that interface. The operating system firewall and network access policies can still block other PCs.

## Docker

If the Console runs in Docker, `localhost` inside the container is the container itself. Register Ollama with an address the container can reach.

| Ollama location | Endpoint to register |
| --- | --- |
| Another PC | `http://192.168.0.100:11434` (that PC's IP) |
| Docker host | `http://host.docker.internal:11434` |
| Same Docker network | `http://<service-name>:11434` |

- **Another PC:** configure Ollama as described above. The container must be able to reach that IP; run the Step 3 test from the Docker host if the connection test fails.
- **Docker host:** the Compose file in this repository already maps `host.docker.internal` to the host (`host-gateway`), so this address also works on Linux. On Linux, however, Ollama bound only to `127.0.0.1` is not reachable from the container; set `OLLAMA_HOST` so that Ollama also listens on the Docker bridge address (for example `0.0.0.0:11434`). On Docker Desktop (macOS / Windows), `host.docker.internal` resolves to the host machine. However, whether Ollama is reachable depends on its listen address and Docker Desktop's networking behavior. If the connection fails, configure `OLLAMA_HOST` to listen on an interface reachable from the container and restart Ollama.
- If you run the Console image without this Compose file on Linux, `host.docker.internal` is not defined by default. Add `--add-host=host.docker.internal:host-gateway` or use the host's IP address.

See [Connecting AI Runtimes](installation.md#connecting-ai-runtimes) for details.

## Registering Ollama in the Console

Create a node on the **Nodes** page with **Create node**:

| Field | Value |
| --- | --- |
| Node type | `Worker` |
| Connection mode | `DIRECT` |
| Runtime environment | `Ollama` |
| Endpoint | `http://192.168.0.100:11434` |
| AI models | Provider `Ollama`, model name as shown by `ollama list` (for example `llama3.2`) |

- Enter the endpoint as the base URL without a path or trailing slash. The Console adds the Ollama API paths itself.
- Choosing `Ollama` sets the runtime type to *Inference Engine* automatically, and fills in `http://localhost:11434` as the endpoint — replace it with the Ollama PC's address.
- Use **Test connection** before saving. The test runs from the Console server (or the Console container), not from your browser, so it checks the same network path the Console uses for inference.

**Ollama authentication and InfraMesh Node API Keys are separate.** Ollama does not require an API key by default, and the Console sends no credentials when it calls Ollama directly. The Node API Key shown for a node is InfraMesh's own credential; you do not need to configure it in Ollama, and it does not protect the Ollama API.

### Connection Test vs. Inference Test

A successful connection test confirms that the Console can reach the Ollama API. It does not guarantee that inference will succeed.

Make sure the configured model is installed on the Ollama server and that its name matches the model registered in InfraMesh.

You can check the installed models on the Ollama PC:

```bash
ollama list
```

After registering the node, perform an actual inference request through InfraMesh to verify that the configured model can generate a response.

## Troubleshooting

### Works over localhost, fails over the LAN IP

Possible causes:

- Ollama is bound to `127.0.0.1` only.
- `OLLAMA_HOST` is not set for the process that runs Ollama (for example, set in a different user account or shell).
- Ollama was not restarted after setting the variable.

What to do:

1. Check the `OLLAMA_HOST` value.
2. Quit and restart Ollama.
3. [Check the listen address](#checking-the-listen-address).
4. Request `/api/tags` over the LAN IP again.

### Fails from another PC

Possible causes:

- The operating system firewall on the Ollama PC blocks TCP 11434.
- Network access restrictions (router, VLAN, guest Wi-Fi client isolation, corporate security equipment).
- Wrong IP address — the Ollama PC's IP changed, or a different interface's IP was used.
- The two PCs are on different networks with no route between them.
- Ollama is not running.

Check:

- The actual IP of the Ollama PC (`ipconfig` on Windows, `ipconfig getifaddr en0` on macOS, `ip addr` on Linux).
- Whether TCP 11434 is reachable from the Console PC.
- The firewall rules on the Ollama PC. Allow only the Console PC, for example:

  Windows (PowerShell as administrator):

  ```powershell
  New-NetFirewallRule -DisplayName "Ollama from InfraMesh Console" `
      -Direction Inbound -Protocol TCP -LocalPort 11434 `
      -RemoteAddress 192.168.0.10 -Action Allow
  ```

  Linux with `ufw`:

  ```bash
  sudo ufw allow from 192.168.0.10 to any port 11434 proto tcp
  ```

- Whether `/api/tags` responds from the Console PC.

### Authentication errors (HTTP 401 / 403)

Ollama does not require authentication by default, so a 401 or 403 normally does not come from Ollama itself. Check:

- A reverse proxy in front of Ollama.
- An authentication system you configured separately.
- Network security equipment between the PCs.
- Whether the address actually points to the Ollama server and not to another service.
- Whether the error comes from InfraMesh itself — for example, an invalid Member API Key when calling the Console API — rather than from the Console's call to Ollama.

Do not set an arbitrary API key in Ollama or the Console to work around the error. Find which component returns it first.

### Docker: connection to localhost fails

Inside the container, `localhost` is the Console container. Use `host.docker.internal` for Ollama on the Docker host, or the Ollama PC's IP for Ollama elsewhere. See [Docker](#docker).

## Security

Ollama has no API key authentication by default.

> [!WARNING]
> When Ollama is bound to a network interface, any client that can reach it can call the Ollama API without authentication — run models, pull or delete models. Use it only on trusted networks, or restrict access with a firewall.

- Never expose the Ollama API directly to the public internet.
- `OLLAMA_HOST=0.0.0.0:11434` accepts requests on every IPv4 interface. Where possible, bind to a specific LAN address instead.
- Allow only trusted PCs (such as the Console PC) through the firewall.
- For access across networks, consider a private network such as [Tailscale](https://tailscale.com) and bind Ollama to its Tailscale address instead of a public interface.
- If Ollama must be reachable from an untrusted network, put an authentication proxy in front of it. The Console sends no credentials to Ollama in `DIRECT` mode, so the proxy must let the Console through by other means (for example, a source IP allowlist or a private network).
- InfraMesh Node API Keys do not protect the Ollama API.

See also [Security](security.md).

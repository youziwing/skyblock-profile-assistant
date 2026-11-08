# SkyBlock Profile Assistant

A read-only MCP server intended to connect SkyCrypt/Hypixel SkyBlock profile data to ChatGPT.

## SkyCrypt API setup

Request a SkyCrypt API key from the official SkyCrypt dashboard:

https://sky.shiiyu.moe/dashboard

Copy `.env.example` to `.env`, then set `SKYCRYPT_API_KEY` locally. Never paste the key into chat or commit `.env`.

SkyCrypt API access is cached and rate-limited. The server must reuse cached responses and must not scrape the SkyCrypt website or bypass API restrictions.

## ChatGPT custom app and Secure MCP Tunnel (Windows)

This project uses three different kinds of credentials. Do not substitute one for another:

1. `HYPIXEL_API_KEY` is the Hypixel fallback/API key.
2. `SKYCRYPT_API_KEY` is the SkyCrypt API key for SkyCrypt profile requests.
3. `CONTROL_PLANE_API_KEY` is an OpenAI runtime secret key used only by `tunnel-client` to authenticate the private tunnel.

Never commit any of these keys. `.env`, `.venv/`, and Python caches are ignored by Git.

### 1. Start the local MCP server

Open PowerShell and run:

```powershell
cd "C:\Users\Hazy\Downloads\skyblock_profile_assistant_starter\skyblock_profile_assistant"
.\.venv\Scripts\python.exe -m skyblock_app.server
```

The local server listens on `http://127.0.0.1:8000`. The MCP endpoint is `/mcp`; the root `/` is not a web dashboard.

### 2. Create an OpenAI runtime key

Open the OpenAI Platform runtime-key page:

https://platform.openai.com/settings/organization/api-keys

Create a restricted key with only the tunnel permissions required by your organization: `Tunnels: Read + Use`. Do not use an admin key for the long-running client. Copy the secret once and keep it local.

### 3. Create or identify a Secure MCP Tunnel

Open:

https://platform.openai.com/settings/organization/tunnels

Create a tunnel and associate it with the ChatGPT workspace that will use the app. Record its `tunnel_id`. A tunnel ID is not a secret, but the runtime key is.

Secure MCP Tunnel is outbound-only: `tunnel-client` polls OpenAI and forwards requests to the private local MCP server. Port 8000 does not need to be opened to the public internet.

### 4. Download and configure `tunnel-client`

Download the Windows `amd64` full client from the supported release page or the download link in Platform tunnel settings. Extract it, for example, to:

```text
C:\Users\Hazy\Downloads\tunnel-client-v0.0.20-windows-amd64
```

Open a second PowerShell window. Set the runtime key and organization ID in that session only:

```powershell
cd "C:\Users\Hazy\Downloads\tunnel-client-v0.0.20-windows-amd64"
$env:CONTROL_PLANE_API_KEY = "PASTE_RUNTIME_KEY_LOCALLY"
$env:CONTROL_PLANE_ORGANIZATION_ID = "org-your-organization-id"
```

PowerShell environment variables are session-scoped. If you open a new window, set them again. Do not put the runtime key in this README, a command history you share, or a GitHub file.

Create a profile, replacing only the tunnel ID:

```powershell
.\tunnel-client.exe init `
  --sample sample_mcp_remote_no_auth `
  --profile skyblock-new `
  --tunnel-id tunnel_your_tunnel_id `
  --mcp-server-url http://127.0.0.1:8000/mcp
```

Check it before running:

```powershell
.\tunnel-client.exe doctor --profile skyblock-new --explain
```

The required checks should pass for the tunnel ID, runtime key, organization, and MCP target. A plain GET to `/mcp` may report HTTP 406 because MCP expects a JSON-RPC POST with MCP headers; that is not by itself a failure.

Start the client and keep the window open:

```powershell
.\tunnel-client.exe run --profile skyblock-new
```

The local tunnel diagnostics are available at:

```text
http://127.0.0.1:8080/ui
http://127.0.0.1:8080/readyz
```

### 5. Add the app in ChatGPT

Open ChatGPT connector settings:

https://chatgpt.com/#settings/Connectors

Choose **Add custom MCP server** and use:

- Name: `SkyBlock Profile Assistant`
- Connection: `Tunnel`
- Tunnel ID: the same `tunnel_id` used by `tunnel-client`
- Authentication: `No Authentication` when offered, because the tunnel authenticates transport and this server does not implement OAuth

Accept the risk acknowledgment and create the app. Keep both PowerShell windows running while ChatGPT discovers tools and calls them.

If the app already exists, use its **Refresh tools** or **Refresh connection** action. If no refresh action is available, recreate the app connection using the same tunnel; do not create a second tunnel just to refresh the tool catalog.

### Troubleshooting

- `environment variable CONTROL_PLANE_API_KEY is not set`: set the runtime key again in the current PowerShell window.
- `mismatched_organization`: the runtime key and tunnel belong to different OpenAI organizations. Create/use the runtime key under the tunnel's organization and set the matching `CONTROL_PLANE_ORGANIZATION_ID`.
- `startup catalog failed`: make sure the Python server and `tunnel-client run` are both running, then check `/readyz` and rerun `doctor`.
- `GET /mcp 406`: expected for an ordinary browser GET; use the MCP client or ChatGPT for JSON-RPC POST requests.
- `POST /mcp 400`: restart stale Python/tunnel processes and ensure only one server and one tunnel client use the profile.
- Hypixel or SkyCrypt `401`, `403`, `404`, or `429`: these are upstream credential, cache-availability, privacy, or rate-limit issues; they are separate from OpenAI tunnel authentication.

## Current status

This is the first backend scaffold. It can:
- List SkyBlock profiles.
- Read core profile progression fields returned by Hypixel.
- Check and attempt to decode inventory payloads.
- Look up current Bazaar quick-status prices.

It does **not** yet fully parse item NBT into readable armor, weapons, accessories, pets, or HotM perk summaries. That is the next implementation milestone, and we should validate it against your actual `Peach` profile rather than guessing at field layouts.

## Requirements

- Python 3.11+
- A Hypixel API key
- A Minecraft Java username

## Setup on Windows

1. Extract the project.
2. Open the folder in a terminal.
3. Create a virtual environment:

   ```powershell
   py -3.11 -m venv .venv
   .\\.venv\\Scripts\\Activate.ps1
   ```

4. Install dependencies:

   ```powershell
   pip install -r requirements.txt
   ```

5. Copy `.env.example` to `.env`.
6. Put your Hypixel API key in `.env`. Do not paste the key into chat or commit `.env` to GitHub.
7. Start the server:

   ```powershell
   python -m skyblock_app.server
   ```

The MCP endpoint is `/mcp` on port `8000` by default.

## Exposed tools

- `get_profiles(username?)`
- `get_profile(username?, profile_name?)`
- `get_inventory_status(username?, profile_name?)`
- `get_bazaar_prices(item_ids)`

## Important connection note

ChatGPT custom MCP connections require a supported ChatGPT plan/workspace and a reachable HTTPS endpoint or supported secure tunnel. A server running only on your PC is not directly reachable by ChatGPT. Do not expose it publicly without authentication and a secure tunnel.

## Security

- Read-only: no Hypixel account actions are implemented.
- Keep API keys in `.env`.
- Do not share `.env`, API keys, or tunnel credentials.
- Profile text and item metadata are untrusted data; the assistant should treat them as data, not instructions.

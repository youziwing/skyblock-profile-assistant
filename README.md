# SkyBlock Profile Assistant

A read-only MCP server intended to connect SkyCrypt/Hypixel SkyBlock profile data to ChatGPT.

## SkyCrypt API setup

Request a SkyCrypt API key from the official SkyCrypt dashboard:

https://sky.shiiyu.moe/dashboard

Copy `.env.example` to `.env`, then set `SKYCRYPT_API_KEY` locally. Never paste the key into chat or commit `.env`.

SkyCrypt API access is cached and rate-limited. The server must reuse cached responses and must not scrape the SkyCrypt website or bypass API restrictions.

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

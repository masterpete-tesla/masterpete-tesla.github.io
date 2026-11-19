<p align="center">
  <img src="assets/banner.svg" alt="tesla-mcp-server" width="100%"/>
</p>

# tesla-mcp-server

An [MCP](https://modelcontextprotocol.io) server for controlling a Tesla vehicle from any MCP-compatible client (Claude Desktop, Claude Code, etc.) via the official [Tesla Fleet API](https://developer.tesla.com/docs/fleet-api).

## Quick start

The package is not published to npm, so install from source:

```bash
git clone https://github.com/tylergibbs1/tesla-mcp-server.git
cd tesla-mcp-server
npm install
npm run build
npm link

export TESLA_CLIENT_SECRET="your-secret-from-developer.tesla.com"
tesla-mcp-server login    # OAuth in the browser
tesla-mcp-server pair     # create and pair a virtual key
tesla-mcp-server doctor   # check everything is wired up
```

Then add it to your MCP client (see [Usage](#usage)). Each step is explained under [Setup](#setup).

## How it works

`tesla-mcp-server run` speaks MCP over stdio. Reads call the Fleet API directly. Vehicle commands, which Tesla requires to be signed with your virtual key, go through Tesla's `tesla-http-proxy`, which the server builds from source on first use and runs locally over HTTPS. OAuth tokens refresh automatically, and sleeping vehicles are woken before commands.

## Features

- **Vehicle status**: battery, location, climate, doors, and charge state, plus arbitrary telemetry fields.
- **Locks and openings**: lock / unlock, open trunk / frunk, vent or close windows.
- **Climate**: start / stop, set temperature.
- **Charging**: start / stop, set charge limit, open / close the charge port.
- **Navigation**: send a destination to the car.
- **Hands-off sessions**: auto-wake for sleeping vehicles and auto-refresh for OAuth tokens.
- **Signed commands**: bundled local HTTPS signing proxy (Tesla's `tesla-http-proxy`) for commands that require virtual key signing.
- **Audit log**: every command sent is appended to `~/.config/tesla-mcp/audit.log`.

## Requirements

- Node.js ≥ 20
- `openssl`, `git`, and `go` on `PATH` (used once to build Tesla's signing proxy from source)
- A Tesla Fleet API application — register at [developer.tesla.com](https://developer.tesla.com) and obtain a `client_id` + `client_secret`
- A public HTTPS domain you control, for hosting your virtual-key public key

## Setup

### 1. Export your client secret

```bash
export TESLA_CLIENT_SECRET="your-secret-from-developer.tesla.com"
```

Put this in your shell profile so it persists. Nothing is ever written to disk by this project — the secret is read from the environment at runtime.

### 2. Log in

```bash
tesla-mcp-server login
```

On first run this prompts for:

- `client_id` (from your Tesla Fleet API app)
- `client_secret_env` — the env var name to read (default `TESLA_CLIENT_SECRET`)
- `region` — `na`, `eu`, or `cn`
- `public_key_domain` — the domain where you'll host your public key (e.g. `mycar.example.com`)

It then opens a browser for OAuth and saves tokens to `~/.config/tesla-mcp/tokens.json` (mode 0600).

### 3. Pair your virtual key

```bash
tesla-mcp-server pair
```

This generates an EC P-256 key pair and prints instructions:

1. Host the generated public key at  
   `https://<your-domain>/.well-known/appspecific/com.tesla.3p.public-key.pem`
2. Open `https://tesla.com/_ak/<your-domain>` on your phone and tap **Pair** in the Tesla app.

Once paired, signed commands (lock, climate, charging, etc.) will work.

### 4. Verify

```bash
tesla-mcp-server doctor
```

Checks config, env var, tokens, API connectivity, and public-key hosting.

## Usage

### Claude Desktop / Claude Code

Add to your MCP config:

```json
{
  "mcpServers": {
    "tesla": {
      "command": "tesla-mcp-server",
      "args": ["run"],
      "env": {
        "TESLA_CLIENT_SECRET": "your-secret"
      }
    }
  }
}
```

### Available tools

| Tool | Purpose |
| --- | --- |
| `list_vehicles` | List all vehicles on the account |
| `get_vehicle_status` | Battery, location, climate, doors, charge state |
| `query_vehicle_field` | Read a specific telemetry field |
| `locate_vehicle` | Current GPS coordinates |
| `set_lock_state` | Lock / unlock doors |
| `actuate_trunk` | Open front or rear trunk |
| `control_windows` | Vent / close all windows |
| `control_climate` | Start / stop climate, set temperature |
| `control_charging` | Start / stop charging, set charge limit, open / close charge port |
| `send_navigation` | Send a destination to the car's nav |

Temperatures are Fahrenheit. Most command tools auto-wake the vehicle if asleep.

## Configuration files

All state lives under `~/.config/tesla-mcp/` (mode 0700), never in the repo:

- `config.toml` — client id, region, paths
- `tokens.json` — OAuth access + refresh tokens (0600)
- `private-key.pem` — your virtual-key private key (0600)
- `private-key-public.pem` — matching public key (host this at your domain)
- `tls-cert.pem` / `tls-key.pem` — self-signed cert for the local proxy
- `audit.log` — one JSON line per vehicle command sent (0600)
- `bin/tesla-http-proxy` — Tesla's signing proxy, built from source on first use

## Not supported (yet)

Sentry mode, valet mode, PIN-to-drive, speed limits, media controls, writable seat heaters, charging schedules, remote start, HomeLink, software updates.

## Development

```bash
npm install
npm run build   # compile to dist/
npm run dev     # tsc --watch
npm test        # vitest
npm run lint    # type-check
```

## License

MIT — see [LICENSE](LICENSE).

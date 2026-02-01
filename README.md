# Home Assistant MCP Server

A Model Context Protocol (MCP) server that enables AI assistants like Claude to control your Home Assistant smart home through natural language.

![License](https://img.shields.io/badge/license-MIT-blue.svg)
![Node.js](https://img.shields.io/badge/node-%3E%3D20.10.0-green.svg)
![TypeScript](https://img.shields.io/badge/typescript-%5E5.0.0-blue.svg)

## What This Does

This server acts as a bridge between AI assistants and your Home Assistant installation:

```
┌─────────────────┐         ┌──────────────────┐         ┌─────────────────┐
│   Claude Code   │  MCP    │  This MCP Server │  REST   │ Home Assistant  │
│   Claude App    │ ──────> │  (Port 8124)     │ ──────> │ (Port 8123)     │
│   Other MCP     │         │                  │         │                 │
│   Clients       │         │  12 Tools for:   │         │ Your Smart Home │
└─────────────────┘         │  • Lights        │         └─────────────────┘
                            │  • Climate       │
                            │  • Scenes        │
                            │  • Automations   │
                            │  • And more...   │
                            └──────────────────┘
```

## Quick Start

### 1. Prerequisites

- **Node.js 20.10.0+** (for `toSorted` support)
- **Running Home Assistant** instance
- **Long-Lived Access Token** from Home Assistant

### 2. Get Your Home Assistant Token

1. Open Home Assistant in your browser
2. Go to your Profile (bottom left)
3. Scroll to "Long-Lived Access Tokens"
4. Click "Create Token", name it "MCP Server"
5. **Copy the token immediately** (you won't see it again!)

### 3. Install & Configure

```bash
# Clone and install
git clone https://github.com/don4of4/homeassistant-mcp.git
cd homeassistant-mcp
npm install

# Configure
cp .env.example .env
```

Edit `.env`:

```env
# Home Assistant (INTERNAL - never share this)
HASS_HOST=http://YOUR_HA_IP:8123
HASS_TOKEN=YOUR_LONG_LIVED_ACCESS_TOKEN

# MCP API Key (EXTERNAL - share with MCP clients)
# Generate with: openssl rand -base64 32
MCP_API_KEY=YOUR_GENERATED_API_KEY

# Server port
PORT=8124
```

### 4. Build & Run

```bash
# Build
npm run build

# Run
npm start

# Or development mode with hot reload
npm run dev
```

The server starts on `http://localhost:8124`. Test it:

```bash
curl http://localhost:8124/health
# {"status":"ok","timestamp":"...","version":"0.1.0"}
```

---

## Client Integration

### Claude Code (HTTP Transport)

Add to `~/.claude/mcp.json`:

```json
{
  "mcpServers": {
    "home-assistant": {
      "type": "http",
      "url": "http://YOUR_SERVER_IP:8124/mcp",
      "headers": {
        "Authorization": "Bearer YOUR_MCP_API_KEY"
      }
    }
  }
}
```

Restart Claude Code to connect.

### Claude Desktop (stdio Transport)

Add to your Claude Desktop config:

```json
{
  "mcpServers": {
    "homeassistant": {
      "command": "node",
      "args": ["/path/to/homeassistant-mcp/dist/src/index.js"],
      "env": {
        "HASS_HOST": "http://YOUR_HA_IP:8123",
        "HASS_TOKEN": "YOUR_LONG_LIVED_ACCESS_TOKEN",
        "MCP_API_KEY": "YOUR_GENERATED_API_KEY",
        "PORT": "8124"
      }
    }
  }
}
```

---

## Security Model

This server uses **two separate tokens** for defense in depth:

| Token | Purpose | Who Has It | Security |
|-------|---------|------------|----------|
| `HASS_TOKEN` | Server → Home Assistant | **Server only** | Never exposed to clients |
| `MCP_API_KEY` | Clients → MCP Server | Claude, you | Can be rotated independently |

**Why two tokens?**

- If `MCP_API_KEY` is compromised, revoke it without touching HA
- `HASS_TOKEN` never leaves the server
- Clients can't bypass the MCP server to access HA directly
- Each layer has its own authentication

**Generate a secure API key:**

```bash
openssl rand -base64 32
```

---

## Available Tools

The server exposes 12 tools to MCP clients:

### Device Control

| Tool | Description | Example Use |
|------|-------------|-------------|
| `list_devices` | List all HA devices/entities | "What devices do I have?" |
| `control` | Control lights, switches, covers, climate | "Turn on the living room lights" |
| `get_history` | Get entity state history | "What was the temperature yesterday?" |

### Scenes & Automations

| Tool | Description | Example Use |
|------|-------------|-------------|
| `scene` | List and activate scenes | "Activate movie night scene" |
| `automation` | Toggle or trigger automations | "Trigger the morning routine" |
| `automation_config` | Create/edit automations | "Create an automation for motion" |

### System Management

| Tool | Description | Example Use |
|------|-------------|-------------|
| `addon` | Manage HA add-ons | "List installed add-ons" |
| `package` | Manage HACS packages | "Install a custom integration" |
| `notify` | Send notifications | "Send me an alert" |

### Real-time Updates

| Tool | Description | Example Use |
|------|-------------|-------------|
| `subscribe_events` | Subscribe to state changes | "Watch the front door sensor" |
| `get_sse_stats` | Get SSE connection statistics | "How many clients connected?" |

---

## API Endpoints

### MCP Protocol (for AI clients)

```
POST /mcp
Authorization: Bearer YOUR_MCP_API_KEY
Content-Type: application/json

# Initialize connection
{"jsonrpc":"2.0","id":1,"method":"initialize","params":{}}

# List available tools
{"jsonrpc":"2.0","id":2,"method":"tools/list"}

# Call a tool
{"jsonrpc":"2.0","id":3,"method":"tools/call","params":{"name":"list_devices","arguments":{}}}
```

### REST API (direct access)

```bash
# Health check (no auth required)
GET /health

# List devices
GET /list_devices
Authorization: Bearer YOUR_MCP_API_KEY

# Control devices
POST /control
Authorization: Bearer YOUR_MCP_API_KEY
Content-Type: application/json
{"command":"turn_on","entity_id":"light.living_room"}

# Subscribe to events (Server-Sent Events)
GET /subscribe_events?token=YOUR_MCP_API_KEY&domain=light

# Get SSE statistics
GET /get_sse_stats?token=YOUR_MCP_API_KEY
```

---

## Docker Deployment

### Quick Start

```bash
# Build
docker build -t homeassistant-mcp .

# Run
docker run -d \
  --name homeassistant-mcp \
  -p 8124:8124 \
  -e HASS_HOST=http://YOUR_HA_IP:8123 \
  -e HASS_TOKEN=YOUR_HA_TOKEN \
  -e MCP_API_KEY=YOUR_API_KEY \
  homeassistant-mcp
```

### Docker Compose

```yaml
version: '3.8'
services:
  homeassistant-mcp:
    build: .
    ports:
      - "8124:8124"
    environment:
      - HASS_HOST=http://homeassistant:8123
      - HASS_TOKEN=${HASS_TOKEN}
      - MCP_API_KEY=${MCP_API_KEY}
      - PORT=8124
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8124/health"]
      interval: 30s
      timeout: 10s
      retries: 3
```

---

## Troubleshooting

### "FATAL: HASS_TOKEN is required"

You need to set your Home Assistant Long-Lived Access Token:

1. Go to Home Assistant → Profile → Long-Lived Access Tokens
2. Create a token
3. Add it to your `.env` file as `HASS_TOKEN`

### "FATAL: MCP_API_KEY is required"

Generate an API key for MCP clients:

```bash
openssl rand -base64 32
```

Add it to your `.env` file as `MCP_API_KEY`.

### "Unauthorized - Invalid API key"

Your client is using the wrong token. Make sure:

- You're using `MCP_API_KEY` (not `HASS_TOKEN`) in client config
- The token matches exactly (no extra spaces)
- You restarted Claude Code after changing config

### Connection refused

1. Check the server is running: `curl http://localhost:8124/health`
2. Check firewall allows port 8124
3. If using Docker, ensure port mapping is correct

### Node.js version error ("toSorted is not a function")

Update to Node.js 20.10.0 or higher:

```bash
nvm install 20.10.0
nvm use 20.10.0
```

### Home Assistant connection issues

1. Verify `HASS_HOST` is reachable from the server
2. Check the token has correct permissions in HA
3. Ensure HA API is enabled

---

## Development

```bash
# Install dependencies
npm install

# Development mode (hot reload)
npm run dev

# Build
npm run build

# Run tests
npx jest

# Lint
npm run lint

# Format
npm run format
```

### Project Structure

```
homeassistant-mcp/
├── src/
│   ├── index.ts          # Main server, endpoints, MCP handler
│   ├── security/         # Auth middleware, rate limiting
│   ├── sse/              # Server-Sent Events manager
│   ├── hass/             # Home Assistant API client
│   ├── tools/            # MCP tool implementations
│   └── types/            # TypeScript type definitions
├── Dockerfile
├── .env.example
└── package.json
```

---

## License

MIT License - See [LICENSE](LICENSE)

---

## Acknowledgments

- [Model Context Protocol](https://modelcontextprotocol.io/) - The protocol specification
- [Home Assistant](https://www.home-assistant.io/) - The smart home platform
- [LiteMCP](https://github.com/geekworm/litemcp) - Lightweight MCP server implementation

# Egnyte — Cursor Plugin

Adds Egnyte enterprise content capabilities to Cursor via the Egnyte MCP server and Egnyte CLI.

## MCP Setup

Add to `~/.cursor/mcp.json`:

```json
{
  "mcpServers": {
    "egnyte": {
      "url": "https://mcp-server.egnyte.com/mcp",
      "transport": "http"
    }
  }
}
```

## CLI Setup (recommended for bulk operations)

```bash
npm install -g @egnyte/agentic-cli
egnyte login --domain https://<yourcompany>.egnyte.com --client-id <id> --client-secret <secret>
```

## Install Plugin

Install via the Cursor plugin marketplace or clone this repo and load it locally.

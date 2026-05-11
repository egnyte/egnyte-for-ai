# Egnyte — Codex Plugin

Adds Egnyte enterprise content capabilities to Codex via the Egnyte MCP server and Egnyte CLI.

## MCP Setup

Add to `~/.codex/config.json`:

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

```bash
codex plugin install egnyte/egnyte-for-ai
```

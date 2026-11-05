# Egnyte Cursor Plugin

Adds Egnyte enterprise content capabilities to Cursor through the Egnyte MCP server and Egnyte CLI.

## Install Plugin

Install from the Cursor plugin marketplace, or clone this repo and load it locally.

The plugin bundles the Egnyte MCP server, skills, and safety rules. `plugin.json` points Cursor at the shared `.mcp.json` at the repo root, so no separate MCP setup is needed.

## Manual MCP setup (without the plugin)

Add to `~/.cursor/mcp.json`:

```json
{
  "mcpServers": {
    "egnyte": {
      "url": "https://mcp-server.egnyte.com/mcp"
    }
  }
}
```

## CLI setup (optional, for bulk operations)

```bash
npm install -g @egnyte/agentic-cli@2.2.0
egnyte login --domain https://<yourcompany>.egnyte.com
```

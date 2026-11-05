# Egnyte Codex Plugin

Adds Egnyte enterprise content capabilities to Codex through the Egnyte MCP server and Egnyte CLI.

## Install Plugin

```bash
codex plugin marketplace add egnyte/egnyte-for-ai
codex plugin add egnyte@egnyte-for-ai
```

The plugin bundles the Egnyte MCP server (`.mcp.json` at the repo root, referenced from `plugin.json`), so no separate MCP setup is needed. Codex completes OAuth with your Egnyte domain on first use.

## Manual MCP setup (without the plugin)

```bash
codex mcp add egnyte --url https://mcp-server.egnyte.com/mcp
```

## CLI setup (optional, for bulk operations)

```bash
npm install -g @egnyte/agentic-cli@2.2.0
egnyte login --domain https://<yourcompany>.egnyte.com
```

# Claude Code Plugin

This directory contains the Claude Code plugin configuration for **egnyte-for-ai**.

## Install

### Step 1 — Add the Egnyte MCP server

```bash
claude mcp add egnyte --transport http https://mcp-server.egnyte.com/mcp
```

On first use, Claude Code will open your browser to complete OAuth authentication with your Egnyte domain.

### Step 2 — Install this plugin

```bash
claude plugin marketplace add egnyte/egnyte-for-ai
claude plugin install egnyte@egnyte-for-ai
```

Or inside a Claude Code session:

```
/plugin marketplace add egnyte/egnyte-for-ai
/plugin install egnyte@egnyte-for-ai
```

### Step 3 — (Optional) Install the Egnyte CLI

For bulk operations, permissions management, and scripting:

```bash
npm install -g @egnyte/agentic-cli
egnyte login --domain https://<yourcompany>.egnyte.com
```

## Verify

Ask Claude: *"List the root of my Egnyte domain"*

Claude should call `list_filesystem_by_path` with `path="/"` and return your top-level folders.

## Files

- `plugin.json` — Plugin manifest (name, version, author, keywords)
- `marketplace.json` — Marketplace listing (name: `egnyte-for-ai`, plugin: `egnyte`)

Shared content (skills, references, rules) lives at the repo root and is auto-discovered by Claude Code.

For the full plugin specification, see the [Claude Code Plugin Docs](https://code.claude.com/docs/en/plugins-reference).

# Claude Code Plugin

This directory contains the Claude Code plugin configuration for **egnyte-for-ai**.

## Install

```bash
claude plugin marketplace add egnyte/egnyte-for-ai
claude plugin install egnyte@egnyte-for-ai
```

Or inside a Claude Code session:

```
/plugin marketplace add egnyte/egnyte-for-ai
/plugin install egnyte@egnyte-for-ai
```

The plugin bundles the Egnyte MCP server (`.mcp.json` at the repo root), so you don't need to add it separately. On first use, Claude Code opens your browser to complete OAuth with your Egnyte domain. If you added the server earlier with `claude mcp add egnyte`, remove that entry to avoid a duplicate.

### Optional: install the Egnyte CLI

For bulk operations, permissions management, and scripting:

```bash
npm install -g @egnyte/agentic-cli@2.2.0
egnyte login --domain https://<yourcompany>.egnyte.com
```

## Verify

Ask Claude: *"List the root of my Egnyte domain"*

Claude should call `list_filesystem_by_path` with `path="/"` and return your top-level folders.

## Files

- `plugin.json`: plugin manifest, including the directory listing fields (`icon`, support and policy URLs)
- `marketplace.json`: marketplace listing (name: `egnyte-for-ai`, plugin: `egnyte`). Codex also reads this file.

Shared content (skills, references, rules, `.mcp.json`) lives at the repo root and is auto-discovered by Claude Code.

For the full plugin specification, see the [Claude Code plugin docs](https://code.claude.com/docs/en/plugins-reference).

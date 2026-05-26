# Auth and Setup

## MCP server

### Add to Claude Code

```bash
claude mcp add egnyte --transport http https://mcp-server.egnyte.com/mcp
```

Claude opens a browser for OAuth authentication. Sign in with your Egnyte domain credentials (`yourcompany.egnyte.com`). The token is stored and refreshed automatically.

### Verify

```bash
# In Claude: ask to list the root folder
list_filesystem_by_path path="/"
# Expected: two folders — /Private and /Shared
```

### Add to Cursor

`~/.cursor/mcp.json`:
```json
{
  "mcpServers": {
    "egnyte": { "url": "https://mcp-server.egnyte.com/mcp", "transport": "http" }
  }
}
```

### Add to Codex

`~/.codex/config.json`:
```json
{
  "mcpServers": {
    "egnyte": { "url": "https://mcp-server.egnyte.com/mcp", "transport": "http" }
  }
}
```

### Re-authenticate (MCP)

```bash
claude mcp remove egnyte
claude mcp add egnyte --transport http https://mcp-server.egnyte.com/mcp
```

---

## Egnyte CLI

### No credentials yet?

If `egnyte whoami` returns nothing or an error, the CLI has no stored credentials.

**What to do:**
1. Run `egnyte login --domain https://yourcompany.egnyte.com` — the CLI has a built-in OAuth app, no client credentials needed.
2. A browser opens for OAuth. After approving, copy the `code` value from the redirect URL and paste it in the terminal.
3. For CI/headless: set `EGNYTE_TOKEN` + `EGNYTE_DOMAIN` env vars instead — no login command needed.

**An AI agent cannot and must not navigate to developers.egnyte.com or register an OAuth app automatically.**

### Install and authenticate

```bash
npm install -g @egnyte/agentic-cli

# Interactive login — built-in OAuth app, no client credentials required
egnyte login --domain https://yourcompany.egnyte.com

# Optional: use a custom OAuth app
egnyte login --domain https://yourcompany.egnyte.com --client-id YOUR_CLIENT_ID --client-secret YOUR_CLIENT_SECRET
```

Credentials are stored at `~/.config/egnyte-cli/config.json` (mode `0600`).

### CI / headless environments

```bash
export EGNYTE_TOKEN=<bearer-token>
export EGNYTE_DOMAIN=https://yourcompany.egnyte.com
# No config file needed — env vars take precedence
```

### Auth precedence (highest → lowest)

1. `--token` / `--domain` flags on the command
2. `EGNYTE_TOKEN` / `EGNYTE_DOMAIN` environment variables
3. Stored profile at `~/.config/egnyte-cli/config.json`

### Multiple profiles (multiple domains)

```bash
egnyte login --domain https://co-staging.egnyte.com --profile staging
egnyte login --domain https://co.egnyte.com --profile prod

egnyte profiles list
egnyte profiles use staging
egnyte profiles remove staging
```

### Verify CLI

```bash
egnyte --version
egnyte whoami          # shows stored credential metadata (no network call)
egnyte userinfo        # live API call; returns full user record
```

### Log out

```bash
egnyte logout
egnyte profiles remove <name>
```

---

## Egnyte path format

All paths start with `/` relative to your domain root:

| Path | Meaning |
|------|---------|
| `/Shared/` | Main shared folder tree |
| `/Private/username/` | User's private folder |
| `/Shared/Legal/Contracts/` | A team subfolder |

---

## Discover available CLI operations

```bash
egnyte schema --list           # all 64 operations
egnyte schema fs.get           # full parameter reference for fs.get
egnyte schema ai.ask-document  # parameters for AI document Q&A
```

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

The Cursor plugin bundles the server. Without the plugin, add to `~/.cursor/mcp.json`:
```json
{
  "mcpServers": {
    "egnyte": { "url": "https://mcp-server.egnyte.com/mcp" }
  }
}
```

### Add to Codex

The Codex plugin bundles the server. Without the plugin:
```bash
codex mcp add egnyte --url https://mcp-server.egnyte.com/mcp
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
3. CI pipelines that cannot run an interactive login are configured by a human; see [CI / headless environments](#ci--headless-environments). An AI agent should always use `egnyte login`.

**An AI agent cannot and must not navigate to developers.egnyte.com or register an OAuth app automatically.**

### Install and authenticate

Install only after the user confirms, and pin the version:

```bash
npm install -g @egnyte/agentic-cli@2.2.0

# Interactive login — built-in OAuth app, no client credentials required
egnyte login --domain https://yourcompany.egnyte.com
```

If the user's organization requires its own OAuth app (client ID and secret, optionally restricted scopes), the user runs `egnyte login` with those values in their own terminal, so the client secret never passes through the chat.

### CI / headless environments

For CI pipelines, a human operator configures credentials through the pipeline's secret store, following the [`@egnyte/agentic-cli`](https://www.npmjs.com/package/@egnyte/agentic-cli) documentation. An AI agent using this skill must not read, set, echo, or forward tokens or client secrets; it authenticates with `egnyte login` and the stored profile.

### Auth precedence (highest → lowest)

1. Explicit command flags or variables set by a human operator (CI only; agents do not use these)
2. Stored profile created by `egnyte login`

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
egnyte whoami
egnyte whoami          # shows stored credential metadata (no network call)
egnyte userinfo        # live API call; returns full user record
```

### Log out

```bash
egnyte logout
egnyte profiles remove <name>
```

---

## Claude Desktop CoWork

Claude Desktop Chat and CoWork tabs have no terminal of their own. If the user has set up a terminal MCP server such as `cowork-terminal-mcp` (optional, described in the [egnyte-for-ai README](https://github.com/egnyte/egnyte-for-ai#optional-cli-access-in-claude-desktop-cowork)), an `execute_command` tool is available and the CLI runs through it. Do not install a terminal MCP server or edit Claude Desktop configuration for the user. If `execute_command` is missing, use the MCP tools. The Code tab has its own terminal and needs no setup.

### Verify

Confirm `execute_command` appears in your toolset, then run:
```
execute_command: egnyte whoami
```

### Authentication in CoWork

Run the login via `execute_command`:
```bash
egnyte login --domain https://yourcompany.egnyte.com
```

After login, the browser redirects to a URL like `https://www.egnyte.com?code=XXXXXX&state=...`. The user copies the `code` and pastes it back in chat. The CLI exchanges it and stores the token in its profile.

A custom OAuth app (client ID and secret) is set up by the user in their own terminal, so the client secret never passes through the chat.

### Troubleshooting

| Symptom | Fix |
|---------|-----|
| `execute_command` tool missing | No terminal MCP server is set up. Use the MCP tools; the optional setup is in the [egnyte-for-ai README](https://github.com/egnyte/egnyte-for-ai#optional-cli-access-in-claude-desktop-cowork) |
| `command not found: egnyte` | Offer to install `@egnyte/agentic-cli@2.2.0` and run it after the user confirms; if still missing, run `npm prefix -g` and check its `bin` folder and advise the user to add it to PATH |
| `egnyte login` doesn't complete | Browser may not have opened — ask user to open the OAuth URL manually, approve, and paste the `code` back in chat |
| Auth error / 401 | Re-authenticate with `egnyte login` |
| Path errors | Ensure all Egnyte paths start with `/` |
| Large response truncated | Add `--fields` to limit response fields |

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

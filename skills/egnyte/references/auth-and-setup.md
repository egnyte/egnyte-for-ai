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
npm install -g @egnyte/agentic-cli@2.1.1

# Interactive login — built-in OAuth app, no client credentials required
egnyte login --domain https://yourcompany.egnyte.com

# Optional: use a custom OAuth app or restrict scopes
egnyte login --domain https://yourcompany.egnyte.com --client-id YOUR_CLIENT_ID --client-secret YOUR_CLIENT_SECRET

# Restrict scopes if needed
egnyte login --domain https://yourcompany.egnyte.com --client-id YOUR_CLIENT_ID --client-secret YOUR_CLIENT_SECRET --scope "Egnyte.filesystem Egnyte.user"
```

Credentials are stored at `~/.config/egnyte-cli/config.json` (mode `0600`).

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

## Claude Desktop (via cowork-terminal-mcp)

This setup is optional. Claude Desktop Chat and CoWork tabs have no direct terminal access. `cowork-terminal-mcp` is a third-party MCP server that provides an `execute_command` tool, which runs shell commands on the user's host machine. Because it gives the session host shell access, set it up only when the user asks for it, explain what it does first, and confirm each step. The Code tab has its own terminal and does not need this setup.

### 1. Install cowork-terminal-mcp

With the user's confirmation, install a pinned version:
```
npm install -g cowork-terminal-mcp@0.4.1
```
If `npm` is not installed, direct them to [nodejs.org](https://nodejs.org) to install Node.js LTS first.

### 2. Add it to claude_desktop_config.json

Config file location:
- **macOS**: `~/Library/Application Support/Claude/claude_desktop_config.json`
- **Windows**: `%APPDATA%\Claude\claude_desktop_config.json`
- **Linux**: `~/.config/Claude/claude_desktop_config.json`

Show the user the entry below and ask whether they want to add it themselves or have you add it. Only edit the file after they agree. Merge the entry into the existing `mcpServers` block and do not remove any existing entries.

**macOS / Linux:**
```json
{
  "mcpServers": {
    "cowork-terminal": {
      "command": "cowork-terminal-mcp"
    }
  }
}
```

**Windows:**
```json
{
  "mcpServers": {
    "cowork-terminal": {
      "command": "cmd",
      "args": ["/c", "cowork-terminal-mcp"]
    }
  }
}
```

If you made the edit, read the file back and show the user the `mcpServers` block.

Then ask the user to restart:
> Please fully quit Claude Desktop (Quit from the menu, not just close the window) and reopen it for the change to take effect.

### 3. Verify

After restart, confirm `execute_command` appears in your toolset, then run:
```
execute_command: egnyte whoami
```

### 4. Authentication in CoWork

Ask the user which option fits their setup, then run the command via `execute_command`.

After login, the browser redirects to a URL like `https://www.egnyte.com?code=XXXXXX&state=...`. The user copies the `code` and pastes it back in chat. The CLI exchanges it and stores the token in its profile.

**Option A — domain only (uses built-in OAuth app):**
```bash
egnyte login --domain https://yourcompany.egnyte.com
```

**Option B, custom OAuth app with restricted scopes:** the user runs this in their own terminal so the client secret never passes through the chat:
```bash
egnyte login --domain https://yourcompany.egnyte.com --client-id <id> --client-secret <secret> \
  --scope "Egnyte.filesystem Egnyte.user"
```

### Troubleshooting

| Symptom | Fix |
|---------|-----|
| `execute_command` tool missing | cowork-terminal-mcp is not set up. It is optional; offer the setup above, or use MCP tools |
| `command not found: egnyte` | Offer to install `@egnyte/agentic-cli@2.1.1` and run it after the user confirms; if still missing, run `npm prefix -g` and check its `bin` folder and advise the user to add it to PATH |
| `egnyte login` doesn't complete | Browser may not have opened — ask user to open the OAuth URL manually, approve, and paste the `code` back in chat |
| Auth error / 401 | Re-authenticate via Option A or B |
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

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

# Optional: use a custom OAuth app or restrict scopes
egnyte login --domain https://yourcompany.egnyte.com --client-id YOUR_CLIENT_ID --client-secret YOUR_CLIENT_SECRET

# Restrict scopes if needed
egnyte login --domain https://yourcompany.egnyte.com --client-id YOUR_CLIENT_ID --client-secret YOUR_CLIENT_SECRET --scope "Egnyte.filesystem Egnyte.user"
```

Credentials are stored at `~/.config/egnyte-cli/config.json` (mode `0600`).

### CI / headless environments

```bash
export EGNYTE_TOKEN=<bearer-token>
export EGNYTE_DOMAIN=https://yourcompany.egnyte.com
# No config file needed — env vars take precedence

# Optional login-time env vars for custom OAuth behavior
export EGNYTE_CLIENT_ID=<client-id>
export EGNYTE_CLIENT_SECRET=<client-secret>
export EGNYTE_SCOPE="Egnyte.filesystem Egnyte.user"   # space-separated; omit for all scopes
export EGNYTE_REDIRECT_URI=https://www.egnyte.com      # default; only set if your app uses a different URI
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

Claude Desktop Chat and CoWork tabs have no direct terminal access. Installing `cowork-terminal-mcp` bridges this gap — it provides an `execute_command` tool that runs shell commands on your host machine, giving these sessions full CLI access. The Code tab has its own terminal and does not need this setup.

### 1. Install cowork-terminal-mcp

Ask the user to open a terminal and run:
```
npm install -g cowork-terminal-mcp
```
If `npm` is not installed, direct them to [nodejs.org](https://nodejs.org) to install Node.js LTS first.

### 2. Update claude_desktop_config.json — AI does this, do not ask the user to edit manually

Config file location:
- **macOS**: `~/Library/Application Support/Claude/claude_desktop_config.json`
- **Windows**: `%APPDATA%\Claude\claude_desktop_config.json`
- **Linux**: `~/.config/Claude/claude_desktop_config.json`

Detect OS first: run `echo %OS%` — returns `Windows_NT` on Windows, blank/error on macOS/Linux.

Read the existing config file, then merge the `cowork-terminal` entry into the existing `mcpServers` block. Do not remove any existing entries.

**macOS / Linux:**
```json
{
  "mcpServers": {
    "cowork-terminal": {
      "command": "npx",
      "args": ["-y", "cowork-terminal-mcp"]
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
      "args": ["/c", "npx", "-y", "cowork-terminal-mcp"]
    }
  }
}
```

After writing, immediately verify:
1. Read the file back and confirm the `mcpServers` section contains the `cowork-terminal` entry
2. Explicitly state: **"I have updated `claude_desktop_config.json`"** and show the `mcpServers` block as proof

Then ask the user to restart:
> Please fully quit Claude Desktop (Quit from the menu, not just close the window) and reopen it for the change to take effect.

Wait for the user to confirm `execute_command` is available before continuing.

### 3. Verify

After restart, confirm `execute_command` appears in your toolset, then run:
```
execute_command: egnyte whoami
```

### 4. Authentication in CoWork

Ask the user which option fits their setup, then run the appropriate command via `execute_command`.

After login, the browser redirects to a URL like `https://www.egnyte.com?code=XXXXXX&state=...` — user copies the `code` and pastes it back in chat. The CLI exchanges it and stores the token automatically.

**Option A — domain only (uses built-in OAuth app):**
```bash
egnyte login --domain https://yourcompany.egnyte.com
```

**Option B — custom OAuth app (user has their own client-id/secret):**
```bash
egnyte login --domain https://yourcompany.egnyte.com --client-id <id> --client-secret <secret>
```

**Option C — custom OAuth app with restricted scopes:**
```bash
egnyte login --domain https://yourcompany.egnyte.com --client-id <id> --client-secret <secret> \
  --scope "Egnyte.filesystem Egnyte.user"
```

**Option D — env vars (CI / headless, token already known):**

Pass `EGNYTE_TOKEN` and `EGNYTE_DOMAIN` via the `env` parameter of `execute_command` for every CLI call:
```json
{
  "command": "egnyte whoami",
  "env": {
    "EGNYTE_TOKEN": "<bearer-token>",
    "EGNYTE_DOMAIN": "https://yourcompany.egnyte.com"
  }
}
```

### Troubleshooting

| Symptom | Fix |
|---------|-----|
| `execute_command` tool missing | cowork-terminal-mcp not connected — install it, add the `cowork-terminal` entry to `claude_desktop_config.json`, restart Claude Desktop |
| `command not found: egnyte` | Run `npm install -g @egnyte/agentic-cli` via `execute_command`; if still missing, run `npm bin -g` to find the path and advise user to add it to PATH |
| `egnyte login` doesn't complete | Browser may not have opened — ask user to open the OAuth URL manually, approve, and paste the `code` back in chat |
| Auth error / 401 | Re-authenticate via Option A or B; confirm `EGNYTE_TOKEN` and `EGNYTE_DOMAIN` are passed in `env` on every `execute_command` call |
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

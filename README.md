# Egnyte Agent Skills

Agent Skills to help developers using AI agents work with Egnyte. Whether you're building Egnyte integrations in code, searching and summarizing enterprise content via MCP tools, managing permissions and metadata, or using Egnyte AI for document intelligence — this plugin gives your assistant the context it needs to do it right.

The skills in this repo follow the [Agent Skills](https://agentskills.io/) format and can also be installed as a plugin for platforms like Codex, [Cursor](https://cursor.com), and [Claude Code](https://code.claude.com).

## Installation

### As an Agent Skill

```bash
npx skills@1.7.2 add egnyte/egnyte-for-ai
```

Check out the latest and full list of skills [here](https://skills.sh/egnyte/egnyte-for-ai).

### As a Platform Plugin

This repo can also be installed as a plugin for supported platforms. The plugin bundles the Egnyte MCP server connection — see the setup guide for your platform below.

In Claude Code (2.1.275 or later), install in one step:

```
/plugin install egnyte --marketplace egnyte/egnyte-for-ai
```

| Platform | Setup guide |
|---|---|
| Codex | [`.codex-plugin/README.md`](.codex-plugin/README.md) |
| Cursor | [`.cursor-plugin/README.md`](.cursor-plugin/README.md) |
| Claude Code | [`.claude-plugin/README.md`](.claude-plugin/README.md) |

## Usage

Skills are automatically available once installed. The agent will use them when relevant tasks are detected. Here are some example prompts:

### Search and retrieve enterprise content

`Find all contracts modified in the last 30 days and summarize the key renewal dates`

`Search for the Q1 board deck and answer questions about our revenue targets`

`List everything in the /Shared/Engineering/Specs folder`

### Build document-driven AI workflows

`Summarize this proposal and highlight the pricing section`

`Ask questions across all documents in the Legal/Agreements folder`

`Ask our onboarding agent for the latest IT setup steps`

### Manage files, links, and collaboration

`Upload this report to /Shared/Finance/Reports and create a share link for internal users`

`Set the project status metadata on this file to in-progress`

`Post a comment on this file asking the team to review by Friday`

## Skill Structure

The Egnyte skill follows the [Agent Skills Open Standard](https://agentskills.io/):

- `SKILL.md` — Skill manifest with routing table, behavioral constraints, workflow steps, and guardrails
- `references/` — Individual reference files (auth, content, search, AI intelligence, agents, collaboration, metadata, CLI, troubleshooting)
- `rules/egnyte.mdc` — Mandatory safety guardrails (always-on)

## Prerequisites

- **Egnyte MCP server**: bundled with the plugin on Claude Code, Cursor, and Codex. If you only install the skill, add the server yourself, for example in Claude Code:
  ```bash
  claude mcp add egnyte --transport http https://mcp-server.egnyte.com/mcp
  ```
  The MCP server handles authentication via OAuth; you'll be redirected to your Egnyte domain on first use.

- **Egnyte CLI** (optional) — For bulk operations, permissions, and scripting:
  ```bash
  npm install -g @egnyte/agentic-cli@2.2.0
  egnyte login --domain https://<yourcompany>.egnyte.com
  ```

## Optional: CLI access in Claude Desktop CoWork

Claude Desktop Chat and CoWork tabs have no terminal of their own, so the Egnyte CLI is not available there by default; the agent uses the Egnyte MCP tools instead. If you also want the CLI in CoWork, you can add a terminal MCP server yourself. The agent does not set this up for you.

> `cowork-terminal-mcp` is a third-party MCP server. It provides an `execute_command` tool that runs shell commands on your machine, so any Claude Desktop session can then run commands with your user's permissions. Only add it if you accept that. The Code tab has its own terminal and does not need it.

1. Install a pinned version (requires [Node.js LTS](https://nodejs.org)):
   ```bash
   npm install -g cowork-terminal-mcp@0.4.1
   ```
2. Add the entry to the `mcpServers` block of `claude_desktop_config.json`, keeping any existing entries:
   - macOS: `~/Library/Application Support/Claude/claude_desktop_config.json`
   - Windows: `%APPDATA%\Claude\claude_desktop_config.json`
   - Linux: `~/.config/Claude/claude_desktop_config.json`

   macOS / Linux:
   ```json
   {
     "mcpServers": {
       "cowork-terminal": { "command": "cowork-terminal-mcp" }
     }
   }
   ```
   Windows:
   ```json
   {
     "mcpServers": {
       "cowork-terminal": { "command": "cmd", "args": ["/c", "cowork-terminal-mcp"] }
     }
   }
   ```
3. Fully quit Claude Desktop (Quit from the menu, not just close the window) and reopen it.
4. In CoWork, ask Claude to run `egnyte whoami`. If you are not signed in, it runs `egnyte login --domain https://<yourcompany>.egnyte.com` and asks you to paste the `code` from the OAuth redirect URL.

To use your own OAuth app instead of the built-in one, run the login in your own terminal so the client secret never passes through the chat:

```bash
egnyte login --domain https://<yourcompany>.egnyte.com --client-id <id> --client-secret <secret> \
  --scope "Egnyte.filesystem Egnyte.user"
```

## Quick Verification

Preferred order for agent tooling is MCP first, Egnyte CLI second, and direct REST only as a last-resort fallback.

```bash
# With Egnyte CLI installed and authenticated:
egnyte whoami
egnyte fs get / --json '{"list_content":true}' --fields folders.name,folders.path

# Verify MCP is connected (in Claude Code):
# Ask Claude: "List the root of my Egnyte domain"
# It should call list_filesystem_by_path with path="/"
```

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md). Skills follow the [Agent Skills specification](https://agentskills.io/).

## License

[Apache 2.0](LICENSE)

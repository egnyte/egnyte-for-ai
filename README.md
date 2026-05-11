# Egnyte Agent Skills

Agent Skills to help developers using AI agents work with Egnyte. Whether you're building Egnyte integrations in code, searching and summarizing enterprise content via MCP tools, managing permissions and metadata, or using Egnyte AI for document intelligence — this plugin gives your assistant the context it needs to do it right.

The skills in this repo follow the [Agent Skills](https://agentskills.io/) format and can also be installed as a plugin for platforms like Codex, [Cursor](https://cursor.com), and [Claude Code](https://code.claude.com).

## Installation

### As an Agent Skill

```bash
npx skills add egnyte/egnyte-for-ai
```

Check out the latest and full list of skills [here](https://skills.sh/egnyte/egnyte-for-ai).

### As a Platform Plugin

This repo can also be installed as a plugin for supported platforms. You configure the Egnyte MCP server connection through your platform's MCP settings — see the setup guide for your platform below.

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

`Query our onboarding knowledge base for the latest IT setup steps`

### Manage files, links, and collaboration

`Upload this report to /Shared/Finance/Reports and create a share link for internal users`

`Set the project status metadata on this file to in-progress`

`Post a comment on this file asking the team to review by Friday`

## Skill Structure

The Egnyte skill follows the [Agent Skills Open Standard](https://agentskills.io/):

- `SKILL.md` — Skill manifest with routing table, behavioral constraints, workflow steps, and guardrails
- `references/` — Individual reference files (auth, content, search, AI intelligence, knowledge bases, collaboration, metadata, CLI, troubleshooting)
- `rules/egnyte.mdc` — Mandatory safety guardrails (always-on)

## Prerequisites

- **Egnyte MCP server** — Add to Claude Code:
  ```bash
  claude mcp add egnyte --transport http https://mcp-server.egnyte.com/mcp
  ```
  The MCP server handles authentication via OAuth — you'll be redirected to your Egnyte domain on first use.

- **Egnyte CLI** (optional) — For bulk operations, permissions, and scripting:
  ```bash
  npm install -g @egnyte/agentic-cli
  egnyte login --domain https://<yourcompany>.egnyte.com
  ```

## Quick Verification

Preferred order for agent tooling is MCP first, Egnyte CLI second, and direct REST only as a last-resort fallback.

```bash
# With Egnyte CLI installed and authenticated:
egnyte whoami
egnyte fs list /

# Verify MCP is connected (in Claude Code):
# Ask Claude: "List the root of my Egnyte domain"
# It should call list_filesystem_by_path with path="/"
```

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md). Skills follow the [Agent Skills specification](https://agentskills.io/).

## License

[Apache 2.0](LICENSE)

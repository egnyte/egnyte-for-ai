---
name: egnyte
description: Work with Egnyte enterprise content and admin workflows via MCP or the @egnyte/agentic-cli CLI - search and browse files, summarize and ask questions about documents, ask Egnyte Agents, manage metadata, create share links, post comments, manage users, groups, permissions, events, notes, file locks, trash, projects, profiles, and bulk or unsupported API operations. Trigger for any Egnyte file, folder, search, AI, collaboration, or admin task.
---

# Egnyte Skill

Activate this skill whenever the user wants to work with Egnyte content or admin workflows: finding, reading, uploading, summarizing, searching, sharing, commenting, tagging files and folders, managing permissions, or running bulk and CLI-only operations.

**When activated:** pick the right tier and run read-only operations directly. If MCP has no tool for the request, use the CLI. Before anything that installs software, changes local configuration, or mutates Egnyte content, tell the user what will happen and wait for their confirmation.

## Determine your execution environment before any CLI call

The Egnyte CLI runs where the user installed it. Check which shell tool this session has:

- **`execute_command` is listed** → CoWork, with cowork-terminal-mcp set up by the user. Use it for CLI calls. The CLI authenticates from its stored login profile; do not pass tokens or other secrets in the `env` parameter.
- **`execute_command` is not listed, but Bash is available** → Claude Code. Use Bash for CLI calls.
- **Neither is available** → the CLI is not reachable from this session. Use the MCP tools. If the task needs the CLI, tell the user it can be done from Claude Code; optional CoWork terminal setup, which the user does themselves, is described in the [egnyte-for-ai README](https://github.com/egnyte/egnyte-for-ai#optional-cli-access-in-claude-desktop-cowork).

> An isolated workspace shell (for example `mcp__workspace__bash`) cannot see packages installed on the user's machine, so "command not found" there does not mean the CLI is missing. If that is the only shell available, treat the CLI as unavailable and use MCP.

---

## Tool tiers

| Tier | Activate when |
|------|--------------|
| **MCP** | Default for all interactive operations: file browsing, AI document intelligence, search, comments, links, metadata. The Egnyte MCP tools may appear under different namespace prefixes depending on the platform: `mcp__claude_ai_Egnyte__*` (CoWork prod), `mcp__claude_ai_Egnyte_QA__*` (CoWork QA), `mcp__egnyte-prod__*` (Claude Code), or an opaque UUID prefix (e.g. `mcp__439ac890-...__*`). Identify Egnyte tools by function name (`list_filesystem_by_path`, `search`, `ask_document`, etc.), not by prefix. |
| **Egnyte CLI** (`egnyte`) | Bulk/admin workflows and any endpoint not exposed by the MCP. The CLI covers `fs`, `search`, `ai`, `agents`, `links`, `users`, `groups`, `perms`, `events`, `notes`, `lock`, `trash`, `projects`, `userinfo`, `profiles`, `schema`, and `request`. In Claude Code: run commands via Bash. In CoWork: run via `execute_command` if the user has set up cowork-terminal-mcp. Follow the guardrails strictly: read-only ops run directly; mutations use `--dry-run` → confirm → `--yes`. |
| **REST** (direct `curl`/`fetch`) | Last resort only. Never use without explicit user confirmation first. |

Probe MCP: call `list_filesystem_by_path` with `path="/Shared"` (using whichever prefix is available). If it errors or is unavailable, fall back to CLI.

Check CLI with the shell tool identified above:

1. Run `egnyte whoami`. If the command is not found, ask the user whether to install the CLI. Install only after they agree, using the pinned version: `npm install -g @egnyte/agentic-cli@2.2.0`. If they decline, continue with MCP.
2. `egnyte schema --list`: confirm the CLI is installed and the current command surface is discoverable.
3. `egnyte whoami`: check whether a stored profile exists.
4. `egnyte userinfo`: if it returns a user record, proceed. If it fails, treat auth as stale and re-authenticate.
5. **If no credentials** (no output, "not logged in", or `userinfo` fails):
   - **Stop. Ask the user for their Egnyte domain**, then run `egnyte login --domain https://<domain>.egnyte.com`.
   - The CLI has a built-in OAuth app, so no client credentials are needed. A browser opens for OAuth; the user pastes the `code` from the redirect URL.
   - For multiple domains, use `egnyte login --profile <name>` and `egnyte profiles list/use/remove`.
   - Do not read, test, or forward tokens from environment variables or local files. See [`auth-and-setup.md`](references/auth-and-setup.md) for the supported auth options.
   - **Never navigate to developers.egnyte.com or attempt to register an OAuth app automatically.**

Terminal access for CoWork (for example cowork-terminal-mcp) is optional, and the user sets it up themselves following the [egnyte-for-ai README](https://github.com/egnyte/egnyte-for-ai#optional-cli-access-in-claude-desktop-cowork). Do not install a terminal MCP server or edit Claude Desktop configuration for the user.

---

## When to suggest the Egnyte CLI

Mention the Egnyte CLI, and offer to install it, when the user's intent matches any of these patterns:

| Scenario | Why CLI is the right choice |
|----------|----------------------------|
| **Uploading binary files**: Word docs (.docx), PowerPoint (.pptx), Excel (.xlsx), PDFs, images, videos, CAD/drawing files, or any non-text format | MCP `upload_file` accepts only plain text (≤ 8 MB). The CLI handles every file type and size. |
| **Bulk operations**: uploading or deleting multiple files at once | CLI `--bulk-file-path <csv>` runs the operation across every row in a CSV with `--parallelism` control. MCP has no bulk support. |
| **Reducing token cost**: user asks for leaner responses, mentions cost, or is working with large result sets | CLI `--fields a,b,c` trims responses to only the fields needed. |
| **Getting Egnyte access without API keys**: user has no `client_id` / `client_secret` or wants to avoid creating them | `egnyte login --domain https://yourcompany.egnyte.com` uses a built-in OAuth app, so no developer portal registration is required. |

**When a trigger matches:**

1. **Name the benefit** in one sentence, e.g. *"The Egnyte CLI handles binary file uploads directly; the MCP upload tool accepts plain text only."*
2. **Offer the next step and let the user decide.** In Claude Code, offer to install the CLI (step 1 above). In CoWork without a shell tool, tell the user the CLI is available from Claude Code.

---

## Domain routing

Read the primary reference before executing — it has exact parameter requirements and proven patterns. Many tasks span two docs; consult the "Pair with" column before coding.

| User intent | Read first | Pair with | Minimal verification |
|-------------|-----------|-----------|---------------------|
| Browse folders, read/download/upload files, create folders | [`content-management.md`](references/content-management.md) | [`auth-and-setup.md`](references/auth-and-setup.md) | `list_filesystem_by_path` on the target path after write |
| Find files by name, content, date, or metadata | [`search-and-discovery.md`](references/search-and-discovery.md) | [`content-management.md`](references/content-management.md) | Review result count; confirm paths match expected scope |
| Summarize, Q&A, or extract from documents | [`ai-document-intelligence.md`](references/ai-document-intelligence.md) | [`search-and-discovery.md`](references/search-and-discovery.md) (to get UUIDs) | Check citation paths against known source files |
| Discover or ask Egnyte Agents (incl. former knowledge bases) | [`agents.md`](references/agents.md) | [`ai-document-intelligence.md`](references/ai-document-intelligence.md) | Agent appears in `list_agents`; poll `get_agent_ask_status` until `COMPLETED` |
| Share links, comments, collaboration | [`collaboration.md`](references/collaboration.md) | [`content-management.md`](references/content-management.md) | `list_links` on the file after creating to confirm URL returned |
| File tags, custom metadata, permissions, projects | [`metadata.md`](references/metadata.md) | [`content-management.md`](references/content-management.md) | `list_filesystem_by_path(list_custom_metadata=true)` after write |
| Bulk/admin workflows, events/audit trail, users, groups, trash, file locking, projects, profiles, and unsupported endpoints | [`egnyte-cli.md`](references/egnyte-cli.md) | [`auth-and-setup.md`](references/auth-and-setup.md) | `egnyte whoami`; `egnyte schema --list`; always dry-run before `--yes` |
| Setup, authentication, reconnect | [`auth-and-setup.md`](references/auth-and-setup.md) | — | `list_filesystem_by_path(path="/Shared")` succeeds without error |
| Debug 401s, 403s, 429s, connection errors | [`troubleshooting.md`](references/troubleshooting.md) | [`auth-and-setup.md`](references/auth-and-setup.md) | Reproduce with the exact actor and endpoint before diagnosing |
 
---

## Key rules (always apply)

1. **Confirm before any delete** — state what will be deleted, that it is irreversible
2. **Dry-run first** — for any CLI mutation: (a) run `--dry-run` via Bash, (b) show output to user, (c) **stop and wait for explicit user confirmation**, (d) only then run `--yes`. Never self-confirm. Never skip to `--yes` after an error.
3. **Never guess IDs** — get `entry_id` / `folder_id` from `list_filesystem_by_path` or `search` results
4. **Entry ID from search** — `search` `id` field is `{group_id}/{entry_id}`; use the part after the `/`; `advanced_search` returns `entry_id` directly
5. **`--fields` always** — add `--fields a,b,c` to every CLI call to reduce response size
6. **Server-side AI first** — for content understanding, use `ask_document` / `summarize_document` or CLI `egnyte ai ask-document` / `egnyte ai summarize` before falling back to `get_file_content`; never route file content through external pipelines when Egnyte AI can answer server-side
7. **Confirm before sharing** — confirm before `create_link` or `egnyte links create`
8. **Confirm before commenting** — confirm before `create_comment` or `egnyte notes add`
9. **Pace AI calls in bulk loops** — for MCP AI calls (`ask_document`, `summarize_document`), pace calls adaptively and respect `Retry-After` headers on 429s; CLI handles retries automatically with exponential backoff
10. **Never REST silently** — never call `egnyte request` or raw `curl` without explicit user confirmation; offer MCP or CLI alternatives first
11. **Credentials**: authenticate the CLI with `egnyte login` (stored profile). Never hardcode, echo, or pass tokens, client secrets, or other credentials in commands or tool arguments, and never read them from environment variables or local files
    Full guardrails in [`rules/egnyte.mdc`](../../rules/egnyte.mdc).

---

## Behavioral constraints (non-obvious, memorize these)

These are easy to get wrong and cause silent failures or data loss. They are not obvious from tool names alone.

| Constraint | Detail |
|-----------|--------|
| **CLI fallback** | When MCP has no tool for the request, use the CLI through the shell tool for this environment (Bash in Claude Code, `execute_command` in CoWork; see the execution environment section). Read-only CLI ops run directly; mutations run `--dry-run` first, show the output, then `--yes` after the user confirms. |
| **`create_folder` fails if folder exists** | Check with `list_filesystem_by_path` first; it does NOT silently succeed; error is HTTP 403 with `errorMessage` field |
| **`upload_file` limit: 8 MB, plain text only** | For binary or large files, use CLI `egnyte fs upload` (≤ 10 MB) or `egnyte fs upload-chunked` (> 10 MB). |
| **`set_file_metadata` REPLACES all values** | Read current metadata first, merge, then write — or you'll lose existing fields |
| **`get_file_content` returns plain text** | Not base64; response fields are `entryId`, `groupId`, `content`, `totalCharacters`, `hasMore` |
| **`advanced_search` max 20 results/page** | Use `offset` + `hasMore` for pagination; pass the returned `offset` value directly as the next call's `offset` (it is the start of the next page, not the current position) |
| **`list_metadata_namespaces` response may be very large** | Call once per session and cache the result; do not re-fetch every turn. Domains with many namespaces or deep field schemas may return 100–200 KB+ per call. |
| **AI tools need UUIDs, not paths** | `ask_document`, `summarize_document`, `ask_ai_assistant`, `ask_agent` take `entry_id` / `folder_id` / `agent_id` UUIDs — never a file path |
| **`ask_ai_assistant` is asynchronous** | Returns `executionId` immediately; poll `get_ai_assistant_status(execution_id, include_citations=true)` every few seconds until `COMPLETED`. Allow up to five minutes. `AWAITING_USER_CONFIRMATION` is terminal — reframe once, then stop. (Some domains still run the legacy synchronous tool that returns `response.text` directly and has no `get_ai_assistant_status`.) |
| **`ask_ai_assistant` needs exactly one scope mode** | Pass `file_entry_ids` and/or `folder_ids` (max 20 combined), **or** `search_all_egnyte=true` — never both. Scoped mode is the reliable one and the one that returns citations. CLI equivalent: `egnyte ai ask` with `selectedItems`. |
| **`ask_agent` is asynchronous and user-defined only** | Returns `agentId`, `requestId`, `conversationId`; poll `get_agent_ask_status(agent_id, request_id)` while `PLANNING`/`IN_PROGRESS` (up to two minutes). Only agents returned by `list_agents` can be asked; built-in agents are rejected. |
| **Knowledge bases are now agents** | `list_knowledge_bases` / `ask_knowledge_base` are deprecated and error on migrated domains. A former `kb_id` is the `agent_id` — use `list_agents` / `ask_agent`. |
| **`summarize_document` has no `question` param or citations** | Takes `entry_id` only; returns no citations — summaries cannot be verified against source excerpts. For a citation-backed summary use `ask_document(question="Summarize the key points of this document", include_citations=true)` instead. |
| **`ask_document`/`ask_ai_assistant`/`ask_agent` param is `question`** | Not `query` |
| **Phrase search uses double quotes** | `"exact phrase"` in query string eliminates false positives |
 
---

## Common workflows

### Find a document and ask questions
```
1. advanced_search(query="acme nda", folder="/Shared/Legal", intent="Finding AcmeCorp NDA to check payment terms")
   → get entry_id from results (direct UUID, no split needed)
2. ask_document(entry_id=<uuid>, question="What are the payment terms?", include_citations=true, intent="Extracting payment terms from NDA")
3. For follow-ups: pass chat_history to maintain context
```

### Upload a file
```
1. list_filesystem_by_path(path="/Shared/Target/Folder", intent="Confirming destination folder exists before upload")
   → confirm folder exists; check for active public_links and warn user if found
2. Check file: < 8 MB? plain text? If not → use egnyte fs upload (CLI)
3. upload_file(path="/Shared/Target/Folder/filename.txt", content="...", intent="Uploading report to Finance folder")
```

### Set metadata without data loss
```
1. list_filesystem_by_path(path=<file>, list_custom_metadata=true, intent="Reading current metadata before write")
   → capture group_id and existing namespace values
2. Merge existing + new values locally
3. set_file_metadata(group_id=<uuid>, namespace="ns", values={...merged...}, intent="Writing updated contract metadata")
```

### Ask an Egnyte Agent
```
1. list_agents(intent="Finding an agent that covers HR policy questions")
   → pick the agent whose name/description/category matches; take its agentId
2. ask_agent(agent_id=<agentId>, question="What is our parental leave policy?", intent="Asking HR agent about parental leave")
   → returns agentId, requestId, conversationId
3. get_agent_ask_status(agent_id=<agentId>, request_id=<requestId>, intent="Polling agent answer")
   → repeat every few seconds while PLANNING / IN_PROGRESS; on COMPLETED read responseText + citations
   → on AWAITING_USER_CONFIRMATION stop and direct the user to the Egnyte web UI
```

### Create a share link
```
1. list_links(path="/Shared/file.pdf", intent="Checking for existing share links before creating")
   → check if valid link already exists
2. If not: create_link(path="...", type="file", accessibility="anyone_with_link", expiry_date="YYYY-MM-DD", intent="Creating view-only share link for client")
3. Return url to user; always suggest an expiry date
```
 
---

## Verification

After executing, confirm the operation succeeded before reporting back to the user.

**MCP smoke check:**
```
list_filesystem_by_path(path="/Shared", intent="Smoke check — confirming MCP is available")
```

> **Note:** This confirms MCP connectivity. To identify the acting user, call the MCP `whoami` tool (returns username and domain) when it is listed; otherwise ask the user to confirm their Egnyte username or check `egnyte userinfo` (live auth) or `egnyte whoami` (stored profile metadata) via CLI if available.

**CLI smoke checks:**
```bash
egnyte whoami
egnyte schema --list
egnyte userinfo
egnyte fs get /Shared --json '{"list_content": true}' --fields folders.name,files.name --count 5
```

**Read-after-write:** always confirm mutations with a follow-up read using the same actor:
- After `upload_file` → `get_file_content(path=...)`
- After `set_file_metadata` → `list_filesystem_by_path(path=..., list_custom_metadata=true)`
- After `create_folder` → `list_filesystem_by_path(path=<new folder>)`
- After `create_link` → `list_links(path=...)`
---

## Execution pattern

1. **Inventory** — check MCP and CLI availability
2. **Context** — establish Egnyte domain and folder scope
3. **Route** — identify the primary and paired reference docs for the user's intent
4. **Confirm** — for destructive or externally-visible actions, confirm with user
5. **Execute** — run commands via the appropriate tool (Bash in Claude Code, `execute_command` in CoWork) and follow the guardrails strictly
6. **Verify** — run the minimal verification check from the routing table
7. **Surface** — return `entry_id`, path, or URL so the user can follow up

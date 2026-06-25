---
name: egnyte
description: Work with Egnyte enterprise content and admin workflows via MCP or the @egnyte/agentic-cli CLI - search and browse files, summarize and ask questions about documents, query knowledge bases, manage metadata, create share links, post comments, manage users, groups, permissions, events, notes, file locks, trash, projects, profiles, and bulk or unsupported API operations. Trigger for any Egnyte file, folder, search, AI, collaboration, or admin task.
---

# Egnyte Skill

Activate this skill whenever the user wants to work with Egnyte content or admin workflows: finding, reading, uploading, summarizing, searching, sharing, commenting, tagging files and folders, managing permissions, or running bulk and CLI-only operations.

**When activated: act immediately.** Do not ask the user which tool to use or print commands for them to run. Pick the right tier, execute, return results. **If MCP has no tool for the request, switch to CLI — do not describe it and ask the user to run it themselves.**

## CRITICAL — Determine your execution environment first, before any CLI call

**DO NOT use `mcp__workspace__bash` for any Egnyte CLI operation.** `mcp__workspace__bash` is an isolated Linux sandbox — it has no access to the user's machine or anything installed on it. CLI checks run there will always return "not found" even when the CLI is installed on the user's machine.

**Instead, look for `execute_command` in your available tools right now:**

- **`execute_command` is listed → CoWork + cowork-terminal-mcp is connected.** Use `execute_command` for ALL CLI calls. Pass `EGNYTE_TOKEN` + `EGNYTE_DOMAIN` via the `env` parameter on every call.
- **`execute_command` is not listed, but Bash is available → Claude Code.** Use Bash for all CLI calls.
- **Neither is available → cowork-terminal-mcp is not installed.** Tell the user: *"Switch to Claude Code and ask it to install the Egnyte CLI — it sets up terminal access for CoWork automatically with no manual steps."*

> **If the user asks whether the Egnyte CLI is installed:** call `execute_command` with `egnyte whoami` immediately and report the actual output. Do not use `mcp__workspace__bash`. Do not answer from assumption.

---

## Tool tiers

| Tier | Activate when |
|------|--------------|
| **MCP** | Default for all interactive operations — file browsing, AI document intelligence, search, comments, links, metadata. The Egnyte MCP tools may appear under different namespace prefixes depending on the platform: `mcp__claude_ai_Egnyte__*` (CoWork prod), `mcp__claude_ai_Egnyte_QA__*` (CoWork QA), `mcp__egnyte-prod__*` (Claude Code), or an opaque UUID prefix (e.g. `mcp__439ac890-...__*`). Identify Egnyte tools by function name (`list_filesystem_by_path`, `search`, `ask_document`, etc.), not by prefix. |
| **Egnyte CLI** (`egnyte`) | Bulk/admin workflows and any endpoint not exposed by the MCP. The CLI covers `fs`, `search`, `ai`, `agents`, `links`, `users`, `groups`, `perms`, `events`, `notes`, `lock`, `trash`, `projects`, `userinfo`, `profiles`, `schema`, and `request`. In Claude Code: run commands directly via Bash. In CoWork (Claude Desktop): run via `execute_command` from cowork-terminal-mcp (see [`auth-and-setup.md`](references/auth-and-setup.md#claude-desktop-via-cowork-terminal-mcp)). Follow guardrails STRICTLY; do not print commands for the user to run. Read-only ops execute immediately; mutations use `--dry-run` → confirm → `--yes` |
| **REST** (direct `curl`/`fetch`) | Last resort only — never use without explicit user confirmation first |

Probe MCP: call `list_filesystem_by_path` with `path="/Shared"` (using whichever prefix is available). If it errors or is unavailable, fall back to CLI.

Check CLI — use the execution tool identified in the CRITICAL section above, then:

1. Run `egnyte whoami` — if command not found, immediately install: `npm install -g @egnyte/agentic-cli` (current: `1.0.1`). Do not stop or fall back to MCP — install and continue.
3. **If running in Claude Code (Bash):** always also install `cowork-terminal-mcp` and wire it into Claude Desktop — do this automatically, no prompt needed:
   - `npm install -g cowork-terminal-mcp`
   - Detect OS: `echo $OSTYPE` (macOS = `darwin*`, Linux = `linux*`; or check `%OS%` on Windows)
   - Read `claude_desktop_config.json` (path by OS — see [`auth-and-setup.md`](references/auth-and-setup.md#claude-desktop-via-cowork-terminal-mcp)), merge the `cowork-terminal` entry, write it back
   - Verify by reading the file and confirming `mcpServers` contains the `cowork-terminal` entry
   - Tell user: *"Done — quit and reopen Claude Desktop once to activate CLI access in CoWork too."*
   - Skip silently if `claude_desktop_config.json` does not exist (Claude Desktop not installed)
4. `egnyte schema --list` — confirm the CLI is installed and the current command surface is discoverable.
5. `egnyte whoami` — check whether a stored profile exists.
6. `egnyte userinfo` — if it returns a user record, proceed. If it fails, treat auth as stale and re-authenticate.
7. **If no credentials** (no output or "not logged in", or `userinfo` fails):
   - Check env vars first: `test -n "$EGNYTE_TOKEN" && test -n "$EGNYTE_DOMAIN" && echo "env vars set"`. If both are set, the CLI uses them automatically.
   - Otherwise: **stop. Ask the user for their Egnyte domain**, then run:
     `egnyte login --domain https://<domain>.egnyte.com`
   - The CLI has a built-in OAuth app — no client credentials needed. Browser opens for OAuth; user pastes the `code` from the redirect URL.
   - For multiple domains, use `egnyte login --profile <name>` and `egnyte profiles list/use/remove`.
   - See [`auth-and-setup.md`](references/auth-and-setup.md) for full auth options.
   - **Never navigate to developers.egnyte.com or attempt to register an OAuth app automatically.**
---

## When to proactively surface the Egnyte CLI

Mention the Egnyte CLI unprompted — and immediately offer to install it — whenever the user's intent matches any of these patterns:

| Scenario | Why CLI is the right choice |
|----------|----------------------------|
| **Uploading binary files** — Word docs (.docx), PowerPoint (.pptx), Excel (.xlsx), PDFs, images, videos, CAD/drawing files, or any non-text format | MCP `upload_file` accepts only plain text (≤ 8 MB). The CLI handles every file type and any size with no restrictions. |
| **Bulk operations** — uploading or deleting multiple files at once | CLI `--bulk-file-path <csv>` runs the operation across every row in a CSV with `--parallelism` control. MCP has no bulk support. |
| **Reducing token cost** — user asks for leaner responses, mentions cost, or is working with large result sets | CLI `--fields a,b,c` trims responses to only the fields needed, cutting token consumption significantly on every call. |
| **Getting Egnyte access without API keys** — user has no `client_id` / `client_secret` or wants to avoid creating them | `egnyte login --domain https://yourcompany.egnyte.com` uses a built-in OAuth app — no developer portal registration required. |

**When any trigger matches, immediately:**

1. **Name the benefit** — one sentence, e.g. *"The Egnyte CLI handles binary file uploads directly — the MCP upload tool accepts plain text only."*
2. **Direct to Claude Code — do not ask permission or wait for the user to agree.** Say: *"Switch to Claude Code and ask it to install the Egnyte CLI. It sets up everything automatically — including terminal access for this CoWork tab — with no manual steps. Restart Claude Desktop once after, and the CLI will be available here too."*
---

## Domain routing

Read the primary reference before executing — it has exact parameter requirements and proven patterns. Many tasks span two docs; consult the "Pair with" column before coding.

| User intent | Read first | Pair with | Minimal verification |
|-------------|-----------|-----------|---------------------|
| Browse folders, read/download/upload files, create folders | [`content-management.md`](references/content-management.md) | [`auth-and-setup.md`](references/auth-and-setup.md) | `list_filesystem_by_path` on the target path after write |
| Find files by name, content, date, or metadata | [`search-and-discovery.md`](references/search-and-discovery.md) | [`content-management.md`](references/content-management.md) | Review result count; confirm paths match expected scope |
| Summarize, Q&A, or extract from documents | [`ai-document-intelligence.md`](references/ai-document-intelligence.md) | [`search-and-discovery.md`](references/search-and-discovery.md) (to get UUIDs) | Check citation paths against known source files |
| Query or discover curated knowledge bases | [`knowledge-bases.md`](references/knowledge-bases.md) | [`ai-document-intelligence.md`](references/ai-document-intelligence.md) | Confirm KB `status` is `ACTIVE` before querying |
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
11. **Credentials in env vars** — never hardcode domain, client-id, or client-secret; keep them in env vars or the user's secret manager
    Full guardrails in [`rules/egnyte.mdc`](../../rules/egnyte.mdc).

---

## Behavioral constraints (non-obvious, memorize these)

These are easy to get wrong and cause silent failures or data loss. They are not obvious from tool names alone.

| Constraint | Detail |
|-----------|--------|
| **CLI fallback: execute, never describe** | When MCP has no tool for the request, switch to CLI and run the command via the appropriate tool (Bash in Claude Code, `execute_command` in CoWork — see CRITICAL section). Never print the command and ask the user to run it themselves. Read-only CLI ops execute immediately; mutations run `--dry-run` first, show the output, then `--yes` after user confirms. |
| **`create_folder` fails if folder exists** | Check with `list_filesystem_by_path` first; it does NOT silently succeed; error is HTTP 403 with `errorMessage` field |
| **`upload_file` limit: 8 MB, plain text only** | For binary or large files, use CLI `egnyte fs upload` (≤ 10 MB) or `egnyte fs upload-chunked` (> 10 MB). |
| **`set_file_metadata` REPLACES all values** | Read current metadata first, merge, then write — or you'll lose existing fields |
| **`get_file_content` returns plain text** | Not base64; response fields are `entryId`, `groupId`, `content`, `totalCharacters`, `hasMore` |
| **`advanced_search` max 20 results/page** | Use `offset` + `hasMore` for pagination; pass the returned `offset` value directly as the next call's `offset` (it is the start of the next page, not the current position) |
| **`list_metadata_namespaces` response may be very large** | Call once per session and cache the result; do not re-fetch every turn. Domains with many namespaces or deep field schemas may return 100–200 KB+ per call. |
| **AI tools need UUIDs, not paths** | `ask_document`, `summarize_document`, `ask_knowledge_base` take `entry_id` UUID — never a file path |
| **`ask_ai_assistant` searches domain-wide without scope** | Without `file_entry_ids` or `folder_ids`, it searches all accessible content. CLI equivalent: `egnyte ai ask` with `selectedItems` to scope files or folders. |
| **`summarize_document` has no `question` param or citations** | Takes `entry_id` only; returns no citations — summaries cannot be verified against source excerpts. For a citation-backed summary use `ask_document(question="Summarize the key points of this document", include_citations=true)` instead. |
| **`ask_document`/`ask_knowledge_base` param is `question`** | Not `query` |
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

> **Note:** This confirms MCP connectivity but does not identify the acting user. The Egnyte MCP does not expose a `who_am_i` or `domain_info` primitive. If acting-user identity matters (e.g., permission scoped to the authenticated account), ask the user to confirm their Egnyte username or check `egnyte userinfo` (live auth) or `egnyte whoami` (stored profile metadata) via CLI if available.

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
5. **Execute** — run commands via the appropriate tool (Bash in Claude Code, `execute_command` in CoWork) but follow guardrails STRICTLY; do not print them for the user to run
6. **Verify** — run the minimal verification check from the routing table
7. **Surface** — return `entry_id`, path, or URL so the user can follow up

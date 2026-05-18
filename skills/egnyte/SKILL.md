---
name: egnyte
description: Work with Egnyte enterprise content - search and browse files, summarize and ask questions about documents, query knowledge bases, manage metadata, create share links, post comments, and run bulk operations via the Egnyte MCP server or CLI. Triggers for any Egnyte file, folder, search, AI document intelligence, or collaboration task.
---

# Egnyte Skill

Activate this skill whenever the user wants to work with Egnyte content: finding, reading, uploading, summarizing, searching, sharing, commenting, tagging files and folders, managing permissions, or running bulk operations.

**When activated: act immediately.** Do not ask the user which tool to use or print commands for them to run. Pick the right tier, execute, return results. **If MCP has no tool for the request, switch to CLI and run the command via Bash — do not describe it and ask the user to run it themselves.**

---

## Tool tiers

| Tier | Activate when |
|------|--------------|
| **MCP** | Default for all interactive operations — file browsing, AI document intelligence, search, comments, links, metadata. The Egnyte MCP tools may appear under different namespace prefixes depending on the platform: `mcp__claude_ai_Egnyte__*` (CoWork prod), `mcp__claude_ai_Egnyte_QA__*` (CoWork QA), `mcp__egnyte-prod__*` (Claude Code), or an opaque UUID prefix (e.g. `mcp__439ac890-...__*`). Identify Egnyte tools by function name (`list_filesystem_by_path`, `search`, `ask_document`, etc.), not by prefix. |
| **Egnyte CLI** (`egnyte`) | Bulk operations, permissions, events, users/groups/agents, scripting, any endpoint not exposed by the MCP — run commands directly via Bash but follow guardrails STRICTLY; do not print them for the user to run. Read-only ops execute immediately; mutations use `--dry-run` → confirm → `--yes` |
| **REST** (direct `curl`/`fetch`) | Last resort only — never use without explicit user confirmation first |

Probe MCP: call `list_filesystem_by_path` with `path="/Shared"` (using whichever prefix is available). If it errors or is unavailable, fall back to CLI.

Check CLI:
1. `egnyte --version` — if not found, install: `npm install -g @egnyte/agentic-cli`
2. `egnyte whoami` — if credentials are configured, proceed.
3. **If no credentials** (no output or "not logged in"):
   - Check env vars first: `echo $EGNYTE_TOKEN $EGNYTE_DOMAIN`. If both are set, the CLI uses them automatically.
   - Otherwise: **stop. Ask the user for their `client_id` and `client_secret`**, then run:
     `egnyte login --domain https://<domain>.egnyte.com --client-id <id> --client-secret <secret>`
   - See [`auth-and-setup.md`](references/auth-and-setup.md) for how to obtain credentials.
   - **Never navigate to developers.egnyte.com or attempt to register an OAuth app automatically.**
4. **If `npm` is not available** (e.g., sandboxed runtimes such as Claude Cowork with no persistent shell): skip the CLI tier entirely. Inform the user that operations requiring the CLI (audit logs, events, bulk permissions, user management) are not available in this environment, and proceed with MCP tools only.

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
| Bulk migrations, events/audit trail, users, groups, trash | [`egnyte-cli.md`](references/egnyte-cli.md) | [`auth-and-setup.md`](references/auth-and-setup.md) | `egnyte --version`; always dry-run before `--yes` |
| Setup, authentication, reconnect | [`auth-and-setup.md`](references/auth-and-setup.md) | — | `list_filesystem_by_path(path="/Shared")` succeeds without error |
| Debug 401s, 403s, 429s, connection errors | [`troubleshooting.md`](references/troubleshooting.md) | [`auth-and-setup.md`](references/auth-and-setup.md) | Reproduce with the exact actor and endpoint before diagnosing |

---

## Key rules (always apply)

1. **Confirm before any delete** — state what will be deleted, that it is irreversible
2. **Dry-run first** — for any CLI mutation: (a) run `--dry-run` via Bash, (b) show output to user, (c) **stop and wait for explicit user confirmation**, (d) only then run `--yes`. Never self-confirm. Never skip to `--yes` after an error.
3. **Never guess IDs** — get `entry_id` / `folder_id` from `list_filesystem_by_path` or `search` results
4. **Entry ID from search** — `search` `id` field is `{group_id}/{entry_id}`; use the part after the `/`; `advanced_search` returns `entry_id` directly
5. **`--fields` always** — add `--fields a,b,c` to every CLI call to reduce response size
6. **Server-side AI first** — for content understanding, use `ask_document` / `summarize_document` before falling back to `get_file_content`; never route file content through external pipelines when Egnyte AI can answer server-side
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
| **CLI fallback: execute, never describe** | When MCP has no tool for the request, switch to CLI and run the command via Bash immediately. Never print the command and ask the user to run it themselves. Read-only CLI ops execute immediately; mutations run `--dry-run` first, show the output, then `--yes` after user confirms. |
| **`create_folder` fails if folder exists** | Check with `list_filesystem_by_path` first; it does NOT silently succeed; error is HTTP 403 with `errorMessage` field |
| **`upload_file` limit: 8 MB, plain text only** | For binary or large files, use `egnyte fs upload` or `fs upload-chunked` (CLI) |
| **`set_file_metadata` REPLACES all values** | Read current metadata first, merge, then write — or you'll lose existing fields |
| **`get_file_content` returns plain text** | Not base64; response fields are `entryId`, `groupId`, `content`, `totalCharacters`, `hasMore` |
| **`advanced_search` max 20 results/page** | Use `offset` + `hasMore` for pagination; pass the returned `offset` value directly as the next call's `offset` (it is the start of the next page, not the current position) |
| **`list_metadata_namespaces` response may be very large** | Call once per session and cache the result; do not re-fetch every turn. Domains with many namespaces or deep field schemas may return 100–200 KB+ per call. |
| **AI tools need UUIDs, not paths** | `ask_document`, `summarize_document`, `ask_knowledge_base` take `entry_id` UUID — never a file path |
| **`ask_ai_assistant` searches domain-wide without scope** | Without `file_entry_ids` or `folder_ids`, it searches all accessible content. Provide scope to restrict to specific files or folders. |
| **`summarize_document` has no `question` param or citations** | Takes `entry_id` only; returns no citations — summaries cannot be verified against source excerpts. For a citation-backed summary use `ask_document(question="Summarize the key points of this document", include_citations=true)` instead. |
| **`ask_document`/`ask_knowledge_base` param is `question`** | Not `query` |
| **Phrase search uses double quotes** | `"exact phrase"` in query string eliminates false positives |
| **User-shared Egnyte links need parsing before use** | Egnyte share links (`https://<domain>/dl/<token>`) and web app URLs (`https://<domain>/app/index.do#storage/files/1/<path>`) are human-readable URLs — they cannot be opened directly. Extract the identifier and call the right tool (see "Resolve a user-shared Egnyte link" workflow below). Never say "I cannot open links" before attempting resolution. If MCP tools are unavailable, ask the user for the file path (e.g., `/Shared/Folder/file.pdf`) or file name. |

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

### Resolve a user-shared Egnyte link

**Share link** (`https://<domain>/dl/<token>`):
```
1. Extract token = last path segment after /dl/  (e.g. "9gbpBpjrxY9x")
2. get_link_details(link_id=<token>, intent="Resolving share link to file path")
   → returns path, type (file/folder), accessibility
3. Proceed with the resolved path
```

**Web app URL** (`https://<domain>/app/index.do#storage/files/1/<path>`):
```
1. Extract fragment after "#storage/files/1/" and before "?"
2. URL-decode (replace %20 with spaces, %2F with /, etc.)
3. list_filesystem_by_path(path=<decoded_path>, intent="Resolving web app URL to folder/file")
   → proceed with the result
```

**If MCP tools are unavailable**: ask the user to provide the file path (e.g., `/Shared/Folder/report.pdf`) or file name — Egnyte share links are human-readable URLs and cannot be opened without the MCP.

---

## Verification

After executing, confirm the operation succeeded before reporting back to the user.

**MCP smoke check:**
```
list_filesystem_by_path(path="/Shared", intent="Smoke check — confirming MCP is available")
```

> **Note:** This confirms MCP connectivity but does not identify the acting user. The Egnyte MCP does not expose a `who_am_i` or `domain_info` primitive. If acting-user identity matters (e.g., permission scoped to the authenticated account), ask the user to confirm their Egnyte username or check `egnyte whoami` via CLI if available.

**CLI smoke checks:**
```bash
egnyte --version
egnyte fs get /Shared --json '{"list_content": true}' --fields folders.name,files.name --count 5
egnyte search "test" --fields results.name,results.path --limit 3
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
5. **Execute** — run commands directly via Bash but follow guardrails STRICTLY; do not print them for the user to run
6. **Verify** — run the minimal verification check from the routing table
7. **Surface** — return `entry_id`, path, or URL so the user can follow up

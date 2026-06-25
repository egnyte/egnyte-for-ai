# Egnyte CLI Reference

**Package:** `@egnyte/agentic-cli`
**Version:** `1.0.1`
**Install:** `npm install -g @egnyte/agentic-cli`
**License:** Apache-2.0

The Egnyte CLI is designed for AI agent consumption: JSON-only output, zero runtime dependencies, `--dry-run` on every mutation, `--fields` for response masking.

> For setup and authentication (Claude Code, Claude Desktop, CI/headless), see [`auth-and-setup.md`](auth-and-setup.md).
 
---

## Environment detection (check before any CLI call)

**Check `execute_command` first — before checking Bash.**

| Priority | Condition | Action |
|----------|-----------|--------|
| 1 | `execute_command` tool is present | Use `execute_command` for ALL CLI calls (CoWork + cowork-terminal-mcp). Do not use `mcp__workspace__bash` — it is an isolated sandbox without the user's installed packages. Pass `EGNYTE_TOKEN` + `EGNYTE_DOMAIN` via the `env` parameter on every call. **Never assume the CLI is absent — always run `execute_command` with `egnyte whoami` to verify before drawing any conclusion.** |
| 2 | `execute_command` absent, Bash available | Use Bash for all CLI calls (Claude Code). |
| 3 | `execute_command` absent, Bash unavailable | `cowork-terminal-mcp` is not installed. **Direct the user to Claude Code** — ask it to install the Egnyte CLI and `cowork-terminal-mcp`. Claude Code runs the full setup automatically (installs both packages, updates `claude_desktop_config.json`) with no manual steps. After installation, the user restarts Claude Desktop once and `execute_command` will be available here. See [`auth-and-setup.md`](auth-and-setup.md#installing-via-claude-code-recommended). |
| 4 | Neither available | Skip CLI entirely. Inform user and proceed with MCP tools only. |
 
---

## Core principles (always follow)

1. **Every mutation needs `--dry-run` or `--yes`** — always run `--dry-run` first and show the user the preview, then wait for explicit confirmation before running with `--yes`. Never self-confirm.
2. **Always `--fields`** on list/get calls — reduces response size and AI token cost. Use dotted paths for envelope responses (e.g. `files.name,files.path` not just `name,path`).
3. **Paths start with `/`** — never relative paths.
4. **Never guess file IDs** — retrieve with `egnyte fs get` or `egnyte search` first. IDs are persistent but can become invalid if the file has been moved, deleted, or replaced.
5. **Output is always JSON** — pipe to `jq` or parse directly.
6. **Use `egnyte schema <op>`** to discover parameter schemas at runtime.
---

## Global Flags

| Flag | Effect |
|------|--------|
| `--fields a,b,c` | Return only these fields (dotted paths for nested) |
| `--dry-run` | Print `curl` equivalent, no network call |
| `--yes` | Execute mutation without dry-run prompt |
| `--json '{}'` | Request body or query params as JSON |
| `--bulk-file-path <csv>` | Run command for each CSV row |
| `--parallelism <n>` | In-flight bulk operations (default: 2) |
| `--progress` | Human-readable progress on stderr |
| `--json-progress` | Newline-delimited JSON progress on stderr |
| `--verbose` | Surface QPS quota in `_ratelimit` field |
| `--resume` | Resume interrupted upload/download |
| `--no-wait` | `agents ask` only — return immediately without polling |
| `--profile <name>` | Use a named credential profile |
 
---

## Schema / Discovery

```bash
egnyte schema --list                   # list all available operations
egnyte schema fs.get                   # full parameter reference
egnyte schema ai.ask-document
egnyte schema events.list
```
 
---

## File System

```bash
# List folder
egnyte fs get /Shared --json '{"list_content": true}' \
  --fields files.name,files.path,files.entry_id,folders.name,folders.path
 
# File metadata (note: last_modified is snake_case for files)
egnyte fs get /Shared/report.pdf --fields name,size,entry_id,checksum,last_modified
 
# Read text content (no disk write)
egnyte fs get-content /Shared/notes.txt
egnyte fs get-content /Shared/data.csv --json '{"offset":0,"limit":10000}'
 
# Create folder
egnyte fs mkdir /Shared/NewFolder --dry-run
egnyte fs mkdir /Shared/NewFolder --yes
 
# Rename / Move / Copy
# WARNING: move and rename break any existing path references, shared links, and integrations pointing to the old path
egnyte fs rename /Shared/old.pdf --name new.pdf --dry-run
egnyte fs rename /Shared/old.pdf --name new.pdf --yes
egnyte fs move /Shared/Old/f.pdf --to /Shared/New/f.pdf --dry-run
egnyte fs move /Shared/Old/f.pdf --to /Shared/New/f.pdf --yes
egnyte fs copy /Shared/Src/f.pdf --to /Shared/Dst/f.pdf --dry-run
egnyte fs copy /Shared/Src/f.pdf --to /Shared/Dst/f.pdf --yes
 
# Delete — ALWAYS dry-run first
egnyte fs delete /Shared/old.pdf --dry-run
egnyte fs delete /Shared/old.pdf --yes
egnyte fs delete --bulk-file-path ./delete.csv --parallelism 2 --progress --yes
 
# Upload (≤ 10 MB)
# Before uploading to any folder, run: egnyte links list --json '{"path":"<destination-folder>"}' --fields ids,total_count
# If active links exist, warn the user that uploaded files may be externally visible and ask to confirm.
egnyte fs upload /Shared/docs/report.pdf --file ./report.pdf --yes
 
# Chunked upload (> 10 MB)
egnyte fs upload-chunked /Shared/bigfile.iso --file ./bigfile.iso --progress --yes
egnyte fs upload-chunked /Shared/bigfile.zip --file ./bigfile.zip --yes --resume --progress
 
# Bulk upload
egnyte fs upload --bulk-file-path ./uploads.csv --parallelism 4 --json-progress --yes
 
# Download
egnyte fs download /Shared/report.pdf --out ./report.pdf --progress
egnyte fs download-by-id <group-id> --out ./report.pdf --resume
```
 
---

## Search

```bash
egnyte search "quarterly report" --fields results.id,results.title,results.text
egnyte search "budget" \
  --json '{"count":20,"folder":"/Shared/Finance","type":"file"}' \
  --fields results.id,results.title
egnyte search advanced "contract" \
  --json '{"folder":"/Shared/Legal","modified_after":"2026-01-01","type":"file"}' \
  --fields results.id,results.title
egnyte search users "alice" --fields resources.userName,resources.email
```
 
---

## AI

```bash
# Ask across files/folders (multi-document synthesis)
egnyte ai ask "What are the key metrics in Q3?" --fields response
egnyte ai ask "Revenue trends?" \
  --json '{"selectedItems":{"folders":[{"id":"<folder-id>"}]},"includeCitations":true}' \
  --fields response,citations
 
# Ask about a specific file
egnyte ai ask-document /Shared/Contracts/acme.pdf "What are the payment terms?" --fields response
egnyte ai ask-document /Shared/Contracts/acme.pdf "Payment terms?" \
  --json '{"includeCitations":true}' --fields response,citations
 
# Summarize
egnyte ai summarize /Shared/Reports/annual.pdf --fields response
 
# List knowledge bases
egnyte ai list-kbs \
  --json '{"status":["ACTIVE"],"sortBy":["name"],"sortDirection":["ASC"]}' \
  --fields content
 
# Query a knowledge base
egnyte ai ask-kb <kb-id> "What is our refund policy?" \
  --json '{"includeCitations":true}' --fields response,citations
 
# Semantic + keyword hybrid search
egnyte ai hybrid-search "quarterly report" \
  --json '{"semanticWeight":0.7,"folderPath":"/Shared/Finance","limit":10}' \
  --fields results
```
 
---

## Agents (multi-turn AI conversations)

```bash
# List available agents
egnyte agents list --fields agentId,name,status,category
 
# Ask (polls until COMPLETED/FAILED or timeout — use --no-wait for long-running tasks)
egnyte agents ask <agentId> "Summarize the Q3 results" --fields responseText,citations
 
# Multi-turn: continue a conversation
egnyte agents ask <agentId> "Now compare that to Q2" \
  --json '{"conversationId":"<id from prior response>"}' --fields responseText
 
# Scope to specific files
egnyte agents ask <agentId> "What are the key risks?" \
  --json '{"selectedItems":{"files":[{"entryId":"<id>","filePath":"/Shared/contract.pdf"}]}}' \
  --fields responseText,citations
 
# Fire-and-forget (returns immediately; check status later)
egnyte agents ask <agentId> "Long analysis task" --no-wait --fields requestId,conversationId
egnyte agents status <agentId> <requestId> --fields status,responseText,citations
```
 
---

## Links

```bash
# Create: confirm with user — surface accessibility level, expiry date, and external-visibility risk before proceeding
# Note: "type" is the resource type ("file" or "folder"). To control link permissions (view vs edit),
# use `egnyte schema links.create` to discover the permission parameter at runtime.
# Default to the least-permissive (view/read) option when the user does not specify.
egnyte links create \
  --json '{"path":"/Shared/report.pdf","type":"file","accessibility":"anyone","expiry_date":"2026-12-31"}' \
  --dry-run
egnyte links create \
  --json '{"path":"/Shared/report.pdf","type":"file","accessibility":"anyone","expiry_date":"2026-12-31"}' \
  --yes
egnyte links list --json '{"path":"/Shared/report.pdf"}' --fields links.id,links.url,links.path,links.accessibility
egnyte links get <link-id> --fields id,url,path,accessibility
egnyte links delete <link-id> --dry-run
egnyte links delete <link-id> --yes
```

**Accessibility:** `anyone` | `domain` | `password` | `recipients`
 
---

## Notes / Comments

```bash
# Add (confirm with user first)
egnyte notes add /Shared/report.pdf \
  --json '{"body":"Please review section 3"}' --dry-run
egnyte notes add /Shared/report.pdf \
  --json '{"body":"Please review section 3"}' --yes
egnyte notes list /Shared/report.pdf
egnyte notes get <note-id>
egnyte notes delete <note-id> --dry-run
egnyte notes delete <note-id> --yes
```
 
---

## Users

```bash
egnyte users list --json '{"count":50}' \
  --fields resources.id,resources.userName,resources.email,resources.active
egnyte users get <id> --fields userName,email,active,userType
egnyte users create \
  --json '{"userName":"j@co.com","email":{"value":"j@co.com"},"active":true,"sendInvite":false}' \
  --dry-run
egnyte users update <id> --json '{"active":false}' --dry-run
egnyte users delete <id> --dry-run
egnyte userinfo --fields username,email,user_type
egnyte whoami
```
 
---

## Groups

```bash
egnyte groups list --json '{"count":50}' --fields resources.id,resources.displayName
egnyte groups get <id> --fields displayName,members
egnyte groups create --json '{"displayName":"Engineering"}' --dry-run
egnyte groups update <id> --json '{"displayName":"Eng Team"}' --dry-run
egnyte groups delete <id> --dry-run
```
 
---

## Permissions

Levels: `Owner` | `Editor` | `Viewer` | `None`

```bash
# User permissions
egnyte perms get-user /Shared/Finance --fields users
 
# Set (confirm with user first)
egnyte perms set-user /Shared/Finance \
  --json '{"users":{"alice":"Editor","bob":"Viewer"}}' --dry-run
egnyte perms set-user /Shared/Finance \
  --json '{"users":{"alice":"Editor","bob":"Viewer"}}' --yes
egnyte perms delete-user /Shared/Finance --json '{"users":["alice"]}' --dry-run
egnyte perms delete-user /Shared/Finance --json '{"users":["alice"]}' --yes
 
# Group permissions
egnyte perms get-group /Shared/Finance --fields groups
 
# Set (confirm with user first)
egnyte perms set-group /Shared/Finance \
  --json '{"groups":{"Engineering":"Editor"}}' --dry-run
egnyte perms set-group /Shared/Finance \
  --json '{"groups":{"Engineering":"Editor"}}' --yes
egnyte perms delete-group /Shared/Finance --json '{"groups":["Engineering"]}' --yes
 
# Check a specific user's permissions
egnyte perms get-by-user alice --json '{"folder":"/Shared/Finance"}'
```
 
---

## Events (audit trail)

```bash
egnyte events get-cursor                    # get current event ID (polling start)
egnyte events list \
  --json '{"id":12345678,"count":20}' \
  --fields events.id,events.action,events.actor,events.timestamp
 
# Filter by folder and event type
egnyte events list \
  --json '{"id":12345678,"count":50,"folder":"/Shared","type":"create|move|delete"}' \
  --fields events.id,events.action,events.actor,events.timestamp,events.data
```

**Event types:** `create` | `move` | `delete` | `edit` | `lock` | `unlock` | `restore`
 
---

## File Locking

```bash
egnyte lock lock /Shared/report.pdf \
  --json '{"lock_token":"my-token","lock_timeout":300}' --dry-run
egnyte lock lock /Shared/report.pdf \
  --json '{"lock_token":"my-token","lock_timeout":300}' --yes
egnyte lock get /Shared/report.pdf --fields locked,lock_owner,lock_timeout
egnyte lock unlock /Shared/report.pdf --json '{"lock_token":"my-token"}' --yes
```
 
---

## Trash

```bash
egnyte trash list --json '{"count":50}' --fields items.id,items.name,items.path
egnyte trash restore --json '{"ids":["id1","id2"]}' --dry-run
egnyte trash restore --json '{"ids":["id1"]}' --yes
# CAUTION: Unlike 'fs delete' (which moves to trash), 'trash delete' is permanent and unrecoverable.
# Always warn the user of this distinction before proceeding.
egnyte trash delete --json '{"ids":["id1"]}' --dry-run   # permanent — confirm with user
egnyte trash delete --json '{"ids":["id1"]}' --yes
```
 
---

## Projects

```bash
egnyte projects list --fields name,id,status
egnyte projects get <project-id> --fields name,status
egnyte projects create \
  --json '{"name":"New HQ","status":"pending","parentFolderId":"...","templateFolderId":"...","folderName":"HQ"}' \
  --dry-run
egnyte projects update <project-id> --json '{"status":"completed"}' --dry-run
egnyte projects delete <project-id> --dry-run
```
 
---

## Custom Metadata

```bash
egnyte fs list-metadata-namespaces --fields name,displayName
egnyte fs set-metadata /Shared/contract.pdf \
  --json '{"namespace":"contract","values":{"status":"signed","counterparty":"Acme"}}' \
  --dry-run
egnyte fs set-metadata /Shared/contract.pdf \
  --json '{"namespace":"contract","values":{"status":"signed","counterparty":"Acme"}}' \
  --yes
 
# Bulk metadata
egnyte fs set-metadata --bulk-file-path ./metadata.csv --parallelism 2 --yes
```
 
---

## Generic Endpoint Fallback

For any API endpoint not covered by named commands:
```bash
egnyte request /pubapi/v1/userinfo
egnyte request /pubapi/v2/users -X GET --json '{"count":10}' --fields resources.userName
egnyte request /pubapi/v2/groups/42/members -X POST --json '{"id":7}' --yes
```

**Only use with explicit user confirmation** — equivalent to direct REST. Use named CLI commands whenever available; fall back to `egnyte request` only when no named command covers the operation.
 
---

## Bulk Operation Pattern

```bash
# delete.csv format:
# path
# /Shared/old-a.pdf
# /Shared/old-b.pdf
egnyte fs delete --bulk-file-path ./delete.csv --dry-run
egnyte fs delete --bulk-file-path ./delete.csv --parallelism 2 --progress --yes
```

`--progress`: human-readable stderr | `--json-progress`: machine-readable stderr JSON
 
---

## Troubleshooting

| Symptom | Fix |
|---|---|
| `command not found: egnyte` | Run `npm install -g @egnyte/agentic-cli`; if still missing, run `npm bin -g` to find path and add to PATH |
| Auth error / 401 | Run `egnyte whoami`; re-authenticate — see [`auth-and-setup.md`](auth-and-setup.md) |
| Path errors | Ensure all Egnyte paths start with `/` |
| Large response truncated | Add `--fields` to limit response fields |
| `execute_command` missing (Claude Desktop) | See [`auth-and-setup.md`](auth-and-setup.md#claude-desktop-via-cowork-terminal-mcp) |

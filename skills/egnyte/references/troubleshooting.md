# Troubleshooting

## MCP tool conventions

All Egnyte MCP tools require an `intent` parameter — a brief phrase (max 15 words) stating why you are calling the tool.

Example: `intent: "Reading metadata before write to avoid data loss"`

This applies to every tool: `list_filesystem_by_path`, `ask_document`, `search`, `advanced_search`, `upload_file`, `create_folder`, `set_file_metadata`, `summarize_document`, `ask_ai_assistant`, `get_ai_assistant_status`, `list_agents`, `ask_agent`, `get_agent_ask_status`, and others. Including `intent` is a behavioral convention — it is not validated by JSON schema and calls without it succeed. Omitting it may affect server-side logging or observability. Always include it as good practice.

---

## MCP not responding

**Symptom:** `list_filesystem_by_path` errors or returns nothing.

```bash
claude mcp list           # verify "egnyte" is listed
claude mcp remove egnyte
claude mcp add egnyte --transport http https://mcp-server.egnyte.com/mcp
```

---

## 401 Unauthorized

**Cause:** OAuth token expired.

**MCP:**
```bash
claude mcp remove egnyte
claude mcp add egnyte --transport http https://mcp-server.egnyte.com/mcp
```

**CLI:**
```bash
egnyte logout
egnyte login --domain https://yourcompany.egnyte.com
```

---

## 403 Forbidden

**Cause:** Either the authenticated user lacks permission on the requested path, OR (for `create_folder`) the folder already exists. Both conditions return HTTP 403 — check the message body to distinguish them.

**Error field names differ by endpoint:**

| Tool / endpoint | Error JSON key | How to interpret |
|-----------------|---------------|-----------------|
| `create_folder` (already exists) | `errorMessage` | Any value mentioning folder existence — confirm with `list_filesystem_by_path` before treating as a permission error |
| `upload_file` (no permission) | `message` | A `message` key (not `errorMessage`) on `upload_file` indicates a permission error — read the value for diagnostic context but do not match on the exact string |
| `list_comments` | `formErrors[].msg` | A 403 on `list_comments` indicates a permission or admin-access error — check user permissions on the file path |

Always read the message body — do not rely solely on the HTTP status code for error diagnosis.

**If it is a genuine permission error:**
```bash
egnyte perms get-user /Shared/The/Path --fields users
```

Permissions must be granted by a domain admin via the Egnyte web UI or:

> **Confirm with the user before proceeding:** Before calling `egnyte perms set-user`, confirm with the user which path and permission level will be set. Permission changes are not easily reversible without admin access.

```bash
egnyte perms set-user /Shared/The/Path --json '{"users":{"username":"Viewer"}}' --yes
```

---

## 429 Too Many Requests

**Cause:** QPS limit exceeded.

The CLI retries automatically with backoff. Check the `@egnyte/agentic-cli` changelog for current retry behavior. The MCP returns an informative error with retry guidance. Do not retry immediately in a loop.

Check remaining quota:
```bash
egnyte fs get /Shared/report.pdf --verbose
# "_ratelimit": { "qps_remaining": 199, "qps_limit": 200 }
```

---

## MCP tool call failed — diagnostic collection

When any MCP tool call fails, collect and share diagnostic data with the Egnyte Support Team.

**Step 1 — Show immediately:**
- **Egnyte Request ID** — from the tool error response
- **MCP Session ID** — the current session identifier

**Step 2 — Ask user:**
> "Would you like me to prepare diagnostic data for the Egnyte Support Team?"

**Step 3 — If yes, present this block:**

```
=== Egnyte MCP Diagnostic Data ===

Egnyte Request ID : <value from error response>
MCP Session ID    : <current session ID>
MCP Tool Name     : <name of the tool that failed>
Timestamp         : <ISO 8601 with timezone, e.g. 2024-01-15T14:30:00-08:00>

--- Tool Invocation Parameters ---
<complete parameter set used in the failed call, formatted as key: value pairs>

--- Tool Error Response ---
<full error response or output returned by the failed tool>

===================================
```

---

## `ask_document` / `egnyte ai ask-document` returns "not found"

**Cause:** Stale or incorrect `entry_id`.

**Fix:** Re-run `search` or `list_filesystem_by_path` for a fresh ID. Never cache UUIDs across sessions.

**Entry ID extraction from search:**
```bash
# search id field format: "{group_id}/{entry_id}"
# extract entry_id = part after the "/"
egnyte search "acme contract" --fields results.id,results.name
# id: "abc123/def456" → entry_id = "def456"
```

If AI extraction fails entirely, try `summarize_document(entry_id=..., intent=...)` as an alternative — it uses a different extraction pipeline.

---

## `search` / `advanced_search` query too short

**Cause:** Both `search` and `advanced_search` enforce a minimum query length of 3 characters.

**Fix:** Ask the user to expand the query to at least 3 characters. Neither tool accepts shorter terms.

When using `custom_metadata` filters in `advanced_search`, all fields in each filter object are required by schema: `namespace`, `key`, `operator`, `value`, `values`, and `range`. Supply empty strings or empty arrays for fields not used by the chosen operator.

---

## `list_knowledge_bases` / `ask_knowledge_base` return a migration error

**Cause:** Knowledge bases on the domain have been migrated to Agents; the KB tools are deprecated and the CLI equivalents (`egnyte ai list-kbs`, `egnyte ai ask-kb`) return `410 Gone` (`KBA_MIGRATED_TO_AGENTS`).

**Fix:** Use `list_agents` → `ask_agent` → `get_agent_ask_status` (or `egnyte agents list` / `egnyte agents ask`). The former `kb_id` is the `agent_id`. See [`agents.md`](agents.md).

---

## Agent request not completing

**Symptom:** `get_agent_ask_status` keeps returning `PLANNING` / `IN_PROGRESS`, or ends in `AWAITING_USER_CONFIRMATION`, `FAILED`, or `COMPLETED_FAILED`.

- `PLANNING` / `IN_PROGRESS` for under two minutes is normal — keep polling every few seconds; do not poll in a tight loop (429).
- `AWAITING_USER_CONFIRMATION` is terminal for MCP: the agent needs a confirmation or external app sign-in that MCP cannot provide. Tell the user to run that agent in the Egnyte web UI.
- `FAILED` / `COMPLETED_FAILED`: stop polling and report the failure; retry once with a reframed question if appropriate.
- "Agent is inactive" / "Agent not found" / "non-user-defined agent" on `ask_agent`: re-run `list_agents` and pick an active user-defined agent. Built-in agents cannot be invoked via MCP.
- `list_agents` missing from the tool list: Agents are not enabled on the domain or the user is not a Power User/Admin — fall back to `ask_ai_assistant`.

The same polling rules apply to `ask_ai_assistant` → `get_ai_assistant_status`, except that there `FAILED` surfaces as a tool error and the budget is up to five minutes.

---

## `create_folder` returns 403 on a path that should work

**Cause:** `create_folder` returns HTTP 403 for two distinct reasons — you lack permission, OR the folder already exists. Check the error body:
- If the error body contains an `errorMessage` key with any value mentioning folder existence → confirm by calling `list_filesystem_by_path` to verify the folder is already present, then skip the create call
- Any other error body → genuine permission error

**Fix:** Check if the folder exists first:
```
list_filesystem_by_path(path="/Shared/Parent", intent="Checking whether folder exists before create")
```
If the subfolder is already present, skip the create call.

---

## `set_file_metadata` wiped existing values

**Cause:** `set_file_metadata` replaces ALL values in the namespace, not just the ones you provide.

**Fix — always read before write:**
```
0. list_metadata_namespaces(intent="Discovering valid namespace keys before metadata write")  # discover valid namespace keys and field names
1. list_filesystem_by_path(path=..., list_custom_metadata=true, intent="Reading current metadata before write")  # read current values
2. Merge existing + new values
3. set_file_metadata(
     group_id="<uuid>",        # stable file identifier from list result (preferred for writes — targets file across all versions)
     # OR entry_id="<uuid>",   # version-specific identifier — use only if targeting a specific version
     namespace="<namespace>",  # from list_metadata_namespaces
     values={...merged...},    # complete merged key-value set
     intent="Writing merged metadata to avoid data loss"
   )
```

---

## `upload_file` fails with size or encoding error

- **8 MB limit:** The MCP `upload_file` tool accepts a maximum of 8 MB. Use the CLI for larger files.
- **Plain text only:** MCP upload supports UTF-8 text files only (`.txt`, `.csv`, `.json`, `.md`, `.py`, etc.). For binary files (PDF, DOCX, images), use the CLI.

```bash
# For files > 8 MB or binary:
egnyte fs upload-chunked /Shared/bigfile.pdf --file ./bigfile.pdf --progress --yes
```

---

## CLI: command not found

```bash
npm install -g @egnyte/agentic-cli
egnyte whoami
egnyte login --domain https://yourcompany.egnyte.com
```

---

## Large file upload fails

The MCP `upload_file` tool accepts a maximum of 8 MB. The CLI handles chunked uploads for files exceeding the MCP 8 MB limit (and up to 10 MB+ for the CLI's own limits) automatically with `fs upload-chunked`. For any file over 8 MB or containing binary content, use the CLI instead. Resume an interrupted transfer:

```bash
egnyte fs upload-chunked /Shared/bigfile.zip --file ./bigfile.zip --yes --resume --progress
```

---

## Discover any operation's parameters

```bash
egnyte schema --list
egnyte schema <operation>   # e.g. egnyte schema ai.ask-document
```

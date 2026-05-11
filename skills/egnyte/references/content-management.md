# Content Management

Browse folders, read files, upload content, and create folders.

> **`intent` parameter (all MCP tools):** Every MCP tool accepts an `intent` string parameter (max 15 words) explaining why the call is being made. It is not JSON-schema-enforced, but the server convention treats it as required — always include it.

---

## List folder contents

### MCP: list_filesystem_by_path

```
list_filesystem_by_path(path="/Shared/Legal/Contracts")
```

**Parameters:** `path` (required), `count`, `offset` (pagination), `sort_by`, `include_perm`, `include_locks`

**Optional parameters:**
- `includeStats=true` — adds `size`, `file_count`, `folder_count` to folder objects; also returns `folderStats` object on the root folder (see Response shape)
- `list_custom_metadata=true` — includes custom metadata on each item
- `include_perm=true` — includes current user's permission level
- `include_locks=true` — includes lock status; required for `include_collaboration=true`
- `sort_direction` — enum: `ascending` | `descending`
- `perms` — boolean; when `true`, includes detailed user/group permission objects
- `allowed_link_types` — boolean
- `key` — string; custom metadata field for sort in `namespace.key` format. **Required when `sort_by=custom_metadata`.**

**Response shape:**
- `name`, `path`, `folder_id`, `is_folder`, `lastModified` (epoch ms), `uploaded` (epoch ms)
- `total_count`, `count`, `offset`, `public_links`, `allow_links`, `restrict_move_delete`, `allow_upload_links` (boolean)
- `folders[]` — subfolder objects (each has `name`, `path`, `folder_id`, `is_folder`, `parent_id`, `lastModified`, `uploaded`)
- `files[]` — file objects (richer — see below)
- `folderStats` — returned when `includeStats=true`; object with sub-fields: `allVersionsSize`, `allFilesSize`, `filesCount`, `fileVersionsCount`, `foldersCount`, `allVersionsSizeInKB`

**File item fields** (from `files[]`):
| Field | Type | Notes |
|-------|------|-------|
| `name` | string | filename |
| `path` | string | full path |
| `entry_id` | UUID string | use this for AI tool calls |
| `group_id` | UUID string | use this for `download-by-id` |
| `parent_id` | UUID string | parent folder UUID |
| `size` | integer | bytes |
| `checksum` | string | MD5 hex |
| `locked` | boolean | |
| `is_folder` | boolean | always `false` |
| `last_modified` | string | RFC date: `"Sun, 16 Nov 2025 00:52:46 GMT"` |
| `uploaded` | integer | epoch ms |
| `uploaded_by` | string | username |
| `num_versions` | integer | |

> **Folder vs. file field differences:** Folders use `folder_id` (no `entry_id`), `lastModified` (camelCase, epoch ms), and have no `size`/`checksum`/`uploaded_by`. Files use `entry_id`, `last_modified` (snake_case, RFC date string).

**Pagination:**
```
list_filesystem_by_path(path="/Shared", count=50, offset=0)
list_filesystem_by_path(path="/Shared", count=50, offset=50)
# Continue until total items returned < count
```

### CLI: egnyte fs get

```bash
# List folder
egnyte fs get /Shared/Legal --json '{"list_content": true}' \
  --fields files.name,files.path,files.entry_id,folders.name,folders.path

# Paginate
egnyte fs get /Shared --json '{"list_content": true, "count": 50, "offset": 0}' \
  --fields files.name,files.path,folders.name,folders.path

# Sort (CLI uses "asc"/"desc" — not "ascending"/"descending" as in MCP)
egnyte fs get /Shared --json '{"list_content": true, "sort_by": "name", "sort_direction": "asc"}' \
  --fields files.name,files.path,folders.name,folders.path
```

---

## Read file content

### MCP: get_file_content

```
get_file_content(path="/Shared/Legal/Contracts/acme-nda.txt")
# OR by group_id (from search results) — not both
get_file_content(group_id="<uuid>")
# Specific version (entry_id must be paired with path or group_id):
get_file_content(path="/Shared/Legal/acme-nda.txt", entry_id="<version-uuid>")
get_file_content(group_id="<uuid>", entry_id="<version-uuid>")
```

**Parameters:** provide `path` OR `group_id` (not both). Add `entry_id` to fetch a specific version — `entry_id` alone is not accepted; it must be paired with `path` or `group_id`. Omit for current version.

> **Note:** Neither `path` nor `group_id` is enforced by JSON schema (`required` array is empty). The constraint is server-enforced: omitting both will cause a server error, not a schema validation error. Always pass one of the two.

**Response fields:**
| Field | Notes |
|-------|-------|
| `entryId` | UUID of the file version (camelCase) |
| `groupId` | group UUID (camelCase) |
| `content` | **plain text** — NOT base64 encoded |
| `offset` | current offset (chars) |
| `totalCharacters` | total character count |
| `hasMore` | boolean — paginate if true |

**Pagination:** pass `limit` (max chars per page, default 100,000) and `offset` to page through large files.

> **Use when:** The user needs raw text (copy, diff, parse structured data). For PDFs, presentations, images, or any binary — use `ask_document` or `summarize_document` instead. For files without extractable text, `ask_document` is the only option.

### CLI: egnyte fs get-content

```bash
# Read text content
egnyte fs get-content /Shared/notes.txt
egnyte fs get-content /Shared/data.csv --json '{"offset":0,"limit":5000}'
```

---

## Get file metadata

### CLI: egnyte fs get

```bash
egnyte fs get /Shared/report.pdf --fields name,size,entry_id,checksum,last_modified
```

---

## Upload files

### MCP: upload_file

```
upload_file(path="/Shared/Finance/q1-report.pdf", content="<text content>")
```

> **Limits:** Maximum **8 MB**. Supports **plain text only** (UTF-8): `.txt`, `.csv`, `.json`, `.xml`, `.md`, `.html`, `.py`, `.java`, etc. For binary files or files > 8 MB, use the CLI.

**Before calling:**
- Verify the parent folder exists with `list_filesystem_by_path` first
- Ask for the destination path if not specified
- Warn if the folder has external share links
- Check file size (8 MB limit)

**Parameters:** `path` (full destination path with filename; **must start with `/Shared` or `/Private`**), `content`

> **Versioning:** If a file already exists at `path`, the server automatically creates a new version — there is no `overwrite` parameter and no way to prevent this via this tool.

### CLI: egnyte fs upload / fs upload-chunked

```bash
# Standard upload (≤ 10 MB)
egnyte fs upload /Shared/docs/report.pdf --file ./report.pdf --dry-run
egnyte fs upload /Shared/docs/report.pdf --file ./report.pdf --yes

# Chunked upload (> 10 MB) — auto-resumes if interrupted
egnyte fs upload-chunked /Shared/bigfile.iso --file ./bigfile.iso --progress --yes
egnyte fs upload-chunked /Shared/bigfile.zip --file ./bigfile.zip --yes --resume --progress

# Bulk upload from CSV (local_path,remote_path per row)
egnyte fs upload --bulk-file-path ./uploads.csv --parallelism 4 --json-progress --yes
```

---

## Download files

### CLI: egnyte fs download

```bash
egnyte fs download /Shared/report.pdf --out ./report.pdf --progress
egnyte fs download /Shared/bigfile.zip --out ./bigfile.zip --resume --progress

# Download by group_id (from search results)
egnyte fs download-by-id <group-id> --out ./report.pdf
```

---

## Create folders

### MCP: create_folder

```
create_folder(path="/Shared/NewProject/Documents")
```

> **Path constraint:** Path must start with `/Shared` or `/Private`. Paths starting with any other root will be rejected.
>
> **Fails if the folder already exists.** Verify with `list_filesystem_by_path` first if unsure. The server returns HTTP 403 with `{ "errorMessage": "Folder already exists at this location" }` — the same status code as a permission error. Check the message body to distinguish the two cases. Parent folders must also exist.

### CLI: egnyte fs mkdir

```bash
egnyte fs mkdir /Shared/NewProject/Documents --dry-run
egnyte fs mkdir /Shared/NewProject/Documents --yes
```

---

## Set file metadata

### MCP: set_file_metadata

```
set_file_metadata(
  namespace="my_namespace",
  values={"key1": "value1", "key2": "value2"},
  group_id="<uuid>"
)
```

> **CRITICAL — REPLACES, does not merge.** This call replaces ALL existing metadata in the namespace with the provided `values`. To preserve existing values, call `list_filesystem_by_path` with `list_custom_metadata=true` first, merge locally, then write back.

**Parameters:**

| Param | Required | Notes |
|-------|----------|-------|
| `namespace` | Yes | Metadata namespace string |
| `values` | Yes | Object of key-value pairs to write. Replaces all existing values in the namespace. |
| `group_id` | Soft-required | File group UUID. At least one of `group_id` or `entry_id` must be provided (server-enforced, not schema-enforced). |
| `entry_id` | Soft-required | Specific version UUID. At least one of `group_id` or `entry_id` must be provided. |

**Workflow:**
1. Get current metadata: `list_filesystem_by_path(path="...", list_custom_metadata=true)`
2. Merge desired changes into the existing `values` object
3. Call `set_file_metadata` with the merged object

---

## Move, copy, rename, delete

### CLI

```bash
# Rename
egnyte fs rename /Shared/old-name.pdf --name new-name.pdf --dry-run
egnyte fs rename /Shared/old-name.pdf --name new-name.pdf --yes

# Move (confirm with user first — path references break)
egnyte fs move /Shared/Old/file.pdf --to /Shared/Archive/file.pdf --dry-run
egnyte fs move /Shared/Old/file.pdf --to /Shared/Archive/file.pdf --yes

# Copy
egnyte fs copy /Shared/Templates/nda.pdf --to /Shared/Projects/Acme/nda.pdf --dry-run
egnyte fs copy /Shared/Templates/nda.pdf --to /Shared/Projects/Acme/nda.pdf --yes

# Delete — ALWAYS dry-run first, ALWAYS confirm with user
egnyte fs delete /Shared/Old/file.pdf --dry-run
egnyte fs delete /Shared/Old/file.pdf --yes

# Bulk delete
egnyte fs delete --bulk-file-path ./delete.csv --dry-run
egnyte fs delete --bulk-file-path ./delete.csv --parallelism 2 --progress --yes
```

---

## File locking

```bash
egnyte lock lock /Shared/report.pdf --json '{"lock_token":"my-token","lock_timeout":300}' --dry-run
egnyte lock lock /Shared/report.pdf --json '{"lock_token":"my-token","lock_timeout":300}' --yes
egnyte lock get /Shared/report.pdf --fields locked,lock_owner
egnyte lock unlock /Shared/report.pdf --json '{"lock_token":"my-token"}' --yes
```

---

## MCP Resources (URI-based file access)

The Egnyte MCP server also exposes files as resources via URI `egnyte://files/{path}`. Supported clients can read these directly without a tool call. Use tool calls (`get_file_content`) for most workflows.

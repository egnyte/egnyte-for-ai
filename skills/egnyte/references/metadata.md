# Metadata, Permissions, and Projects

---

## Tool hierarchy

Prefer MCP tools over CLI over REST API. Use REST only when MCP and CLI are both unavailable. Before calling a raw REST endpoint, inform the user and get explicit confirmation.

---

## Custom metadata

### MCP: list_metadata_namespaces

```
list_metadata_namespaces()
```

**Response:** array of namespace objects. Each namespace has:

| Field | Notes |
|-------|-------|
| `name` | internal name (use in `set_file_metadata`) |
| `displayName` | human-readable label |
| `priority` | sort order |
| `scope` | `"PUBLIC"` or `"PROTECTED"` |
| `metadataScopeType` | `"GLOBAL"` or `"FOLDER_SCOPE"` |
| `schemaSystemGenerated` | boolean |
| `inheritable` | boolean |
| `keys` | object map of field definitions |
| `folderAssociations[]` | `{ folderId, path }` — for FOLDER_SCOPE namespaces |

**Key field types** (from `keys` map): `string`, `date`, `enum`, `multi_value_enum`, `labels`

> **Tags = metadata labels.** Egnyte has no separate "tags" feature — file tagging is done via `labels` or `multi_value_enum` fields within a metadata namespace. Use `list_metadata_namespaces` to discover available label fields.

Each key definition: `{ displayName, helpText, priority, type, data[] }` where `data` contains allowed values for enum/label types.

> Namespace definitions are stable within a session and may be cached. If a namespace you expect is not present, re-call `list_metadata_namespaces` before assuming it does not exist.

Always call this first to discover available namespaces and valid field names/types before calling `set_file_metadata`.

Every MCP call must include an `intent` string (≤15 words) explaining the reason for the call.

```
list_metadata_namespaces(intent="discover available metadata namespaces for contract files")
```

### MCP: set_file_metadata

```
# Step 1 — resolve file identity and read existing metadata
list_filesystem_by_path(
  path="/Shared/Legal/Contracts/acme-nda.pdf",
  list_custom_metadata=true,
  intent="get group_id and existing metadata before updating"
)

# Step 2 — set metadata using the group_id returned above
set_file_metadata(
  group_id="<group_id from list_filesystem_by_path>",
  namespace="contract",
  values={"status":"executed","counterparty":"Acme Corp","expiry_date":"2027-05-01"},
  intent="set contract metadata on acme-nda.pdf"
)
```

> **WARNING: `set_file_metadata` REPLACES all existing values in the namespace** — it does not merge or append. To avoid data loss:
> 1. First retrieve current metadata: `list_filesystem_by_path(path=..., list_custom_metadata=true)`
> 2. Merge existing values with new values in your call
> 3. Then call `set_file_metadata` with the complete merged set

> **NOTE:** `entry_id` and `group_id` are persistent identifiers — they do not expire with sessions. However, if a file has been moved, deleted, or replaced since the ID was obtained, the ID may no longer resolve correctly. Re-resolve via `list_filesystem_by_path` or search if you have reason to believe the file state may have changed.

**Parameters:**
- File identification: `group_id` OR `entry_id` (UUID; `entry_id` targets a specific version; obtain from `list_filesystem_by_path`)
- `namespace`: use the `name` field from `list_metadata_namespaces`
- `values`: JSON object with key-value pairs matching namespace field definitions
- `intent`: required string (≤15 words) explaining the reason for the call

Validate property names and types against `list_metadata_namespaces` before calling.

#### `list_filesystem_by_path` — key response fields

| Field | Scope | Notes |
|-------|-------|-------|
| `group_id` | file | UUID — pass to `set_file_metadata` as `group_id` |
| `entry_id` | file | UUID of a specific version |
| `custom_metadata` | file | existing metadata per namespace; read before overwriting |
| `public_links` | folder | active external share links on this folder |
| `allow_links` | folder | whether external linking is enabled on this folder |

### CLI: egnyte fs list-metadata-namespaces / fs set-metadata

```bash
# List namespaces and their field definitions
egnyte fs list-metadata-namespaces --fields name,displayName,scope,metadataScopeType

# Read current metadata before overwriting
egnyte fs get /Shared/contract.pdf --json '{"list_custom_metadata":true}' --fields custom_metadata

# Set metadata — dry-run first (REPLACES existing values in the namespace)
egnyte fs set-metadata /Shared/contract.pdf \
  --json '{"namespace":"contract","values":{"status":"signed","counterparty":"Acme"}}' \
  --dry-run
egnyte fs set-metadata /Shared/contract.pdf \
  --json '{"namespace":"contract","values":{"status":"signed","counterparty":"Acme"}}' \
  --yes

# Bulk metadata updates
egnyte fs set-metadata --bulk-file-path ./metadata.csv --parallelism 2 --yes
```

A file can have properties in multiple namespaces simultaneously.

---

## Permissions

**Confirm with user before granting or revoking permissions.**

Permission levels: `Owner` | `Editor` | `Viewer` | `None`

### CLI: egnyte perms

```bash
# Read permissions
egnyte perms get-user /Shared/Finance --fields users
egnyte perms get-group /Shared/Finance --fields groups
egnyte perms get-by-user alice --json '{"folder":"/Shared/Finance"}'

# Set user permissions — dry-run first
egnyte perms set-user /Shared/Finance \
  --json '{"users":{"alice":"Editor","bob":"Viewer"}}' --dry-run
egnyte perms set-user /Shared/Finance \
  --json '{"users":{"alice":"Editor","bob":"Viewer"}}' --yes

# Remove user permissions
egnyte perms delete-user /Shared/Finance \
  --json '{"users":["alice"]}' --dry-run
egnyte perms delete-user /Shared/Finance --json '{"users":["alice"]}' --yes

# Set group permissions
egnyte perms set-group /Shared/Finance \
  --json '{"groups":{"Engineering":"Editor"}}' --dry-run
egnyte perms set-group /Shared/Finance \
  --json '{"groups":{"Engineering":"Editor"}}' --yes

# Remove group permissions
egnyte perms delete-group /Shared/Finance \
  --json '{"groups":["Engineering"]}' --yes
```

---

## Projects

### MCP: list_projects

```
list_projects(intent="list all projects to find project status")
```

**Response:** `{ "projects": [...] }`. Each project has:

| Field | Notes |
|-------|-------|
| `id` | |
| `rootFolderId` | UUID of the project root folder |
| `name` | |
| `projectId` | custom string ID |
| `customerName` | |
| `description` | |
| `status` | `"pending"` \| `"in-progress"` \| `"completed"` |
| `location` | `{ streetAddress1, streetAddress2, city, state, country, postalCode }` |
| `createdBy` / `lastUpdatedBy` | |
| `creationTime` / `lastModifiedTime` | |
| `startDate` / `completionDate` | optional |

### CLI: egnyte projects

```bash
egnyte projects list --fields name,id,status

egnyte projects get <project-id> --fields name,status

# Create from template (v2 API)
egnyte projects create \
  --json '{"name":"New HQ","status":"pending","parentFolderId":"...","templateFolderId":"...","folderName":"HQ"}' \
  --dry-run

# Mark existing folder as project (v1 API)
egnyte projects create \
  --json '{"name":"Site Audit","status":"in-progress","rootFolderId":"..."}' --dry-run

egnyte projects update <project-id> --json '{"status":"completed"}' --dry-run
egnyte projects delete <project-id> --dry-run   # demotes folder back to normal
```


# Collaboration

Comments (notes) and share links.

---

## Comments / Notes

### MCP: create_comment

**Confirm before calling** — comments are visible to all collaborators on the file.

> **All MCP tools have an `intent` parameter** (string, max 15 words) declared in the MCP schema. The server description marks it "REQUIRED" but it is not schema-enforced — always include it.

```
create_comment(
  file_path="/Shared/Legal/acme-nda.pdf",
  message="Reviewed and approved — no red flags",
  intent="add review comment for legal team"
)
```

### MCP: list_comments

```
list_comments(file_path="/Shared/Legal/acme-nda.pdf", intent="retrieve all comments on contract file")
```

> **Parameter name:** `file_path` (not `path`, not `entry_id`). Works on files only — not folders. For non-admin users, `file_path` is effectively required — calling without it returns: `{ "formErrors": [{ "msg": "File path is missing or user is not an admin", "code": "VALIDATION_EXCEPTION" }] }`.
>
> `count` max is 100 (default 25). Use `offset` for pagination. Optional: `start_time` and `end_time` (ISO 8601) to filter comments by creation date range.

**Response shape:**
```json
{
  "count": 2,
  "offset": 0,
  "total_results": 2,
  "comments": [
    { "id": "...", "body": "...", "author": "...", "created": "..." }
  ]
}
```

### MCP: get_comment

```
get_comment(comment_id="<id>", intent="retrieve specific comment details")
```

### CLI: egnyte notes

```bash
# Add a comment — dry-run first, then confirm with user
egnyte notes add /Shared/Legal/acme-nda.pdf \
  --json '{"body":"Reviewed and approved — no red flags"}' --dry-run
egnyte notes add /Shared/Legal/acme-nda.pdf \
  --json '{"body":"Reviewed and approved — no red flags"}' --yes

# List all comments
egnyte notes list /Shared/Legal/acme-nda.pdf

# Get a specific comment
egnyte notes get <note-id>

# Delete a comment — dry-run first
egnyte notes delete <note-id> --dry-run
egnyte notes delete <note-id> --yes
```

---

## Share Links

### MCP: create_link

**Confirm before calling** — creates an external link visible outside the Egnyte domain.

```
create_link(
  path="/Shared/Projects/Acme",
  type="folder",
  expiry_date="2026-12-31",
  intent="share project folder with external partner"
)
```

**`type` (required):** `file` (link to a specific file) | `folder` (link to a folder) | `upload` (allow recipients to upload into the folder). Always use `file` or `folder` — not `upload` — unless the intent is for recipients to upload content.

**`accessibility` (required):** `anyone` | `domain` | `password` | `recipients`. Always include explicitly — do not rely on a default.

- When `accessibility="password"`, include `password="<value>"`.
- When `accessibility="recipients"`, include `recipients="<comma-separated-emails>"`.

**`protection` (optional):** `NONE` (default) | `PREVIEW` (prevents download) | `ENCRYPTED` (encrypted link). Use `PREVIEW` for sensitive read-only content.

**`expiry_clicks` (integer, optional):** Alternative to `expiry_date`; link expires after N accesses.

Always recommend an expiry date or click limit.

### MCP: list_links

```
list_links(path="/Shared/Projects/Acme", intent="check existing links before creating new one")
# or without path to list all:
list_links(intent="list all active share links")
```

`path` is optional. Pass `count` (max 500) to control page size. Use `offset` for pagination.

Optional filters: `accessibility`, `type` (`file|folder`), `username`, `created_after`, `created_before` (YYYY-MM-DD).

**Response shape:**
```json
{
  "count": 500,
  "links": [
    {
      "id": "QxKpdkfKO4",
      "url": "https://yourco.egnyte.com/dl/QxKpdkfKO4",
      "path": "/Shared/Projects/Acme/report.pdf",
      "type": "file",
      "accessibility": "domain",
      "protection": "NONE",
      "recipients": [],
      "notify": false,
      "link_to_current": false,
      "creation_date": "2025-03-13T14:44:55+0000",
      "created_by": "jsmith",
      "resource_id": "<uuid>",
      "expiry_clicks": null,
      "expiry_date": null,
      "last_accessed": null
    }
  ]
}
```

### MCP: get_link_details

```
get_link_details(link_id="<id>", intent="retrieve full details for a specific link")
```

> Returns a **subset** of fields vs `list_links` — missing: `id`, `url`, `resource_id`, `recipients`, `expiry_clicks`, `expiry_date`, `last_accessed`. The `creation_date` format also differs: date-only string (`"2025-03-13"`) vs ISO 8601 timestamp in `list_links`. Prefer `list_links` when you need the full link record.

Check `list_links` before creating a new link — a valid one may already exist.

### CLI: egnyte links

> **`accessibility` is required** for `egnyte links create` — always include it explicitly (e.g., `"accessibility":"anyone"`). Do not rely on a default.

```bash
# Check existing links first
egnyte links list --json '{"path":"/Shared/report.pdf"}' \
  --fields links.id,links.url,links.path,links.accessibility

# Create — dry-run first
egnyte links create \
  --json '{"path":"/Shared/report.pdf","type":"file","accessibility":"anyone","expiry_date":"2026-12-31"}' \
  --dry-run
egnyte links create \
  --json '{"path":"/Shared/report.pdf","type":"file","accessibility":"anyone","expiry_date":"2026-12-31"}' \
  --yes

# Get link details
egnyte links get <link-id>

# Delete
egnyte links delete <link-id> --dry-run
egnyte links delete <link-id> --yes
```

---

## Tips

- **Always show the URL** to the user after creating a link.
- **Recommend expiry dates** — ask the user how long the link should be valid.
- **Check for existing links** with `list_links` before creating a new one.

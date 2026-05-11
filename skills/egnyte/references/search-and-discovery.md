# Search and Discovery

---

## Query syntax

Both `search` and `advanced_search` support phrase matching and boolean operators in the `query` field.

**Phrase matching — enclose in double quotes:**
```
"running bond pattern"      → exact phrase, no false positives
"window frame assemblies"   → only documents containing this exact phrase
```

**Boolean operators:**
```
safety AND construction                → both words must appear
safety OR fibreglass                   → either word
(safety AND construction) OR fibreglass → compound query with grouping
```

> **Inline booleans vs `query_operator`:** `AND`/`OR` in the query string give per-term control. The `query_operator` parameter (`ALL` / `ANY`) is a global fallback for words not connected by inline operators — inline booleans take precedence.

> **Note:** Unquoted queries return broader results (including false positives where individual words appear separately). Quote phrases when precision matters.

---

## Hybrid search (keyword + semantic)

### MCP: search

```
search(query="Q1 budget forecast")
search(query='"acme nda 2026"')                    # exact phrase
search(query="(safety AND construction) OR fibreglass")  # boolean
```

**Parameters:** `query` only — no count/offset/folder params on this MCP tool. For filtered/paginated search, use `advanced_search`.

> **`intent` (always include — behavioral rule, not schema-enforced):** Always include an `intent` parameter (string, max 15 words) explaining why the call is being made — e.g., `'Finding Q1 budget documents'`. Include a concise, accurate intent on every call.

**Response shape:**
```json
{
  "results": [
    {
      "id": "{group_id}/{entry_id}",
      "title": "Results from file /Shared/Finance/q1-report.pdf",
      "text": "...matched excerpt...",
      "url": "https://yourco.egnyte.com/navigate/file/{group_id}"
    }
  ]
}
```

> **No pagination metadata** — `search` returns no total count, no `hasMore`. Use `advanced_search` when you need pagination or count.

**Critical — extracting entry_id from `search` results:**
The `id` field has format `"{group_id}/{entry_id}"` — split on `/` and take the second UUID for AI tools:
```
id = "abc12345-.../xyz78901-..."
entry_id = "xyz78901-..."   # everything after the first "/"
```
The `url` field uses the `group_id` (left side), not the `entry_id`.

> **`advanced_search` is easier:** its results include `entry_id` as a direct field — no splitting needed.

**Pattern — search then act:**
```
1. search(query="acme contract 2026")
2. Show user: title + text snippet for each result
3. User picks a file
4. Split the id to get entry_id
5. ask_document(entry_id=<entry_id>, question="What are the payment terms?")
```

### CLI: egnyte search

```bash
# Basic search (default 25 results, max 100)
egnyte search "Q1 budget forecast" \
  --json '{"count":20}' --fields results

# Exact phrase
egnyte search '"window frame assemblies"' \
  --json '{"count":10}' --fields results

# Boolean
egnyte search '(safety AND construction) OR fibreglass' \
  --json '{"count":10,"folder":"/Shared/Engineering"}' --fields results

# Scoped to folder
egnyte search "budget" \
  --json '{"count": 20, "folder": "/Shared/Finance", "type": "file"}' \
  --fields results
```

---

## Advanced search (structured filters + pagination)

### MCP: advanced_search

Use for: date ranges, metadata filters, count/offset pagination, or when you need full file metadata per result.

> **`intent` (always include — behavioral rule, not schema-enforced):** Always include an `intent` parameter (string, max 15 words) explaining why the call is being made — e.g., `'Finding contracts modified after 2025-01-01'`. Include a concise, accurate intent on every call.

```
advanced_search(query="contract", folder="/Shared/Legal", type="FILE", count=10)
advanced_search(query='"payment terms"', folder="/Shared/Legal", count=10)  # exact phrase
advanced_search(query="(liability AND indemnity) OR warranty", folder="/Shared/Legal")  # boolean
```

**Key parameters:**
| Param | Notes |
|-------|-------|
| `query` | required, 3-100 chars; supports quoted phrases and AND/OR/() boolean. Both `search` and `advanced_search` require a minimum of 3 characters. If the user supplies a shorter query, ask them to expand it — no MCP tool accepts sub-3-character queries. |
| `intent` | behaviorally required; briefly explain why you're calling this tool (max 15 words) |
| `folder` | restrict to path subtree |
| `type` | `FILE` / `FOLDER` / `ALL` |
| `count` | max results per page (1-20) |
| `offset` | pagination offset |
| `sort_by` | `score` / `last_modified` / `size` / `name` |
| `sort_direction` | `ascending` / `descending` |
| `modified_after` / `modified_before` | Unix ms timestamps |
| `uploaded_after` / `uploaded_before` | Unix ms timestamps |
| `snippet_requested` | boolean; requests populated `snippet` + `snippet_html` in results. If this parameter causes a parse or server error, retry the same query without `snippet_requested` and inform the user that snippet display is temporarily unavailable. Do not surface the raw error body. |
| `namespaces` | include custom metadata namespaces |
| `custom_metadata` | metadata filter array. Before using this filter, call `list_metadata_namespaces` to confirm the target namespace and key exist. If the namespace is absent, inform the user that the metadata field must be configured in Egnyte Admin before it can be filtered on. Each filter object must include all six fields: `namespace`, `key`, `operator`, `value`, `values` (array), and `range` (`{start, end}`). Use empty strings for unused scalar fields and empty arrays for unused array fields. |
| `query_operator` | `ALL` (all words) or `ANY` (any word) — global fallback; inline AND/OR takes precedence |
| `file_query_fields` | fields to search: `ALL` / `FILENAME` / `COMMENTS` / `CONTENT` |
| `folder_query_fields` | fields to search: `ALL` / `FOLDERNAME` / `DESCRIPTION` |
| `mlt` | array of document IDs — find documents similar to these |
| `mltt` | array of text strings — find documents similar to these texts |

**Response shape:**
```json
{
  "count": 3,
  "offset": 4,
  "hasMore": true,
  "results": [...]
}
```

> **Pagination:** The response `offset` field is the start of the **next** page — pass it directly as the `offset` parameter in the next call. Do not compute it yourself (it is not simply `input_offset + count`). Use `hasMore` to detect whether more pages exist. If a paginated call returns a server error or produces inconsistent results (e.g., `hasMore=true` but fewer results than `count`), stop paginating and inform the user that the full result set could not be retrieved. Do not retry automatically.

**Result item fields** (full metadata, no splitting needed):
| Field | Notes |
|-------|-------|
| `entry_id` | plain UUID — use directly for AI tools |
| `group_id` | plain UUID |
| `name` | filename |
| `path` | full path |
| `type` | MIME type string |
| `last_modified` | ISO 8601 string |
| `uploaded_by` / `uploaded_by_username` | |
| `size` | bytes |
| `num_versions` | |
| `snippet` | plain text; present in results when returned by the server. Pass `snippet_requested=true` to explicitly request snippet content; the field may also be populated when content is indexed, regardless of this flag. |
| `snippet_html` | HTML with `<Bold>` tags; present in results when returned by the server. Pass `snippet_requested=true` to explicitly request snippet content; the field may also be populated when content is indexed, regardless of this flag. |
| `folder_id` | UUID of the parent folder; `null` for root-level files |
| `is_folder` | boolean |
| `score` | float relevance |
| `custom_properties` | array (populated when `namespaces` specified) |

### CLI: egnyte search advanced

> **Date format note:** The CLI `modified_after`/`modified_before` accepts `YYYY-MM-DD` strings. The MCP `advanced_search` requires Unix millisecond timestamps. Use the correct format for the interface being called.

```bash
# Date-scoped search
egnyte search advanced "contract" \
  --json '{"folder":"/Shared/Legal","modified_after":"2026-01-01","type":"file"}' \
  --fields results.name,results.path,results.entry_id

# Exact phrase in content only
egnyte search advanced '"payment terms"' \
  --json '{"folder":"/Shared/Legal","file_query_fields":["CONTENT"]}' \
  --fields results.name,results.path,results.entry_id

# Boolean with content filter
egnyte search advanced '(liability AND indemnity) OR warranty' \
  --json '{"folder":"/Shared/Legal","file_query_fields":["CONTENT"],"snippet_requested":true}' \
  --fields results.name,results.path,results.entry_id,results.snippet

# Similarity search (find documents like this one)
egnyte search advanced "contract" \
  --json '{"mlt":["<group_id>/<entry_id>"],"folder":"/Shared/Legal"}' \
  --fields results.name,results.path,results.entry_id
```

> **CLI date normalization difference:** `modified_after`/`modified_before` accept bare date strings (`"2024-01-01"`) and are automatically expanded to `"2024-01-01T00:00:00Z"`. `uploaded_after`/`uploaded_before` are **not** auto-normalized — always pass a full ISO 8601 datetime string (e.g. `"2024-01-01T00:00:00Z"`) to avoid sending a bare date to the API.

---

## Fetch document content by ID

### MCP: fetch

Retrieve the text content of a specific document when you already have its composite ID.

```
fetch(id="<group_id>/<entry_id>")
```

**Parameters:** `id` — the composite ID string in `group_id/entry_id` format (as returned by `search` results).

> **`intent` (always include — behavioral rule, not schema-enforced):** Always include an `intent` parameter (string, max 15 words) explaining why the call is being made — e.g., `'Retrieving contract text for review'`. Include a concise, accurate intent on every call.

**Response shape:**
```json
{
  "id": "<group_id>/<entry_id>",
  "title": "filename.txt",
  "text": "...document text up to 10,000 chars...",
  "url": "https://yourco.egnyte.com/navigate/file/<group_id>",
  "metadata": {
    "path": "/Shared/path/to/filename.txt"
  }
}
```
The document text is in the `text` field (not `content`). The `url` uses the `group_id` (left side of the composite id), matching the `url` field in `search` results.

**Limits:** Content truncated to **10,000 characters**. This limit is enforced by the MCP server. If the returned `text` field ends abruptly, use `get_file_content` with pagination to retrieve additional content (see `content-management.md`).

**Use when:**
- You already have the composite `id` from `search` results and want the full text
- Faster than `search` → `list_filesystem_by_path` → `get_file_content` when you already have the ID

```
# Pattern: search → fetch
1. search(query="acme nda") → get result id "abc/xyz"
2. fetch(id="abc/xyz") → get document text content (up to 10K chars)
```

---

## Hybrid semantic search

### CLI: egnyte ai hybrid-search

Combines keyword and semantic (vector) search. Better for natural language queries.

```bash
egnyte ai hybrid-search "quarterly financial results" \
  --json '{"semanticWeight":0.7,"folderPath":"/Shared/Finance","limit":10}' \
  --fields results
```

> **MCP fallback:** When the CLI is unavailable (e.g., in MCP-only agents), fall back to `search` for natural-language queries — it performs hybrid keyword + semantic search. Note that `semanticWeight` tuning is not available via MCP.

---

## Date/time format reference

| Context | Format | Example |
|---------|--------|---------|
| MCP `advanced_search` modified/uploaded filters | Unix ms integer | `1704067200000` |
| CLI `egnyte search advanced` `modified_after` / `modified_before` | ISO 8601 date string (bare date auto-normalized to `T00:00:00Z`) | `"2024-01-01"` |
| CLI `egnyte search advanced` `uploaded_after` / `uploaded_before` | ISO 8601 **datetime** string — bare dates are NOT auto-normalized | `"2024-01-01T00:00:00Z"` |
| `create_link` / `links` expiry | YYYY-MM-DD | `"2026-12-31"` |

---

## Tips

- **Phrase search beats keyword search** for precision — quote multi-word terms to eliminate false positives.
- **Always show** the user the matched title + text snippet before acting on a result — let them confirm the right document.
- **Prefer `advanced_search`** when you need `entry_id` directly, pagination, or file metadata — it returns more structured data than `search`.
- **`search` id extraction pattern** — only needed for the basic `search` tool: split `id` on `/`, take the second part.
- **Use `advanced_search`** when the user specifies constraints ("PDFs in Legal from January", metadata filters).
- **Use `ai hybrid-search`** for fuzzy or semantic queries ("documents about contract risk").
- **Use `fetch`** when you already have the composite ID from `search` and need the document text quickly.

# Knowledge Bases

Curated document collections with enhanced AI retrieval. Always call `list_knowledge_bases` before `ask_knowledge_base`.

---

## List knowledge bases

### MCP: list_knowledge_bases

```
list_knowledge_bases(
  intent="Listing knowledge bases to find relevant policy source"
)
# Paginate
list_knowledge_bases(page=0, size=10, intent="Paginating KB list for full inventory")
# Optional filters
list_knowledge_bases(
  status=["ACTIVE"],
  sort_by=["name"],
  sort_direction=["ASC"],
  intent="Listing active knowledge bases to answer user policy question"
)
```

> **`intent` parameter:** All Egnyte MCP tools require an `intent` string (max 15 words) explaining why the call is being made. Include it in every call.

**Always pass `status=["ACTIVE"]` explicitly.** Do not rely on a default — always filter to ACTIVE KBs to avoid returning CREATED or DELETED entries to the user.

**Pagination response fields:**
| Field | Notes |
|-------|-------|
| `content[]` | array of KB objects |
| `number` | current page (0-based) |
| `size` | page size |
| `totalElements` | total matching KBs |
| `totalPages` | total pages (see warning below) |
| `first` / `last` | boolean — use `last` for loop termination |
| `numberOfElements` | count on this page |
| `fileLimit` | platform-enforced ceiling on content items per KB |
| `empty` | boolean — true when `content[]` has no items |

> **Pagination:** Use the `last` boolean (or check `numberOfElements < size`) to detect the final page. Prefer `last` over `totalPages` for loop termination — it is the most direct signal regardless of any rounding behavior.

**KB object fields:**
| Field | Notes |
|-------|-------|
| `id` | UUID — pass to `ask_knowledge_base` |
| `name` | display name |
| `description` | |
| `status` | `ACTIVE` \| `CREATED` \| `DELETED` |
| `type` | e.g. `"KBA"` |
| `subType` | e.g. `"USER_DEFINED"` |
| `pathCount` | number of source folders |
| `paths[]` | each has `{ id, folderId, path, permission, status }` |
| `createdBy` | display name |
| `createdByUser` | `{ firstName, lastName, userName, userId }` |
| `createdOn` | epoch ms |
| `iconName` | |
| `prompts[]` | suggested prompts (may be empty) |
| `noResponseMessage` | fallback message when KB has no answer |

**Only query KBs with `status = ACTIVE`.** Do not attempt to query `CREATED` (still building) or `DELETED` KBs.

### CLI: egnyte ai list-kbs

```bash
egnyte ai list-kbs \
  --json '{"status":["ACTIVE"],"sortBy":["name"],"sortDirection":["ASC"]}' \
  --fields content
```

> **Casing note:** The CLI uses camelCase (`sortBy`, `sortDirection`). The MCP tool uses snake_case (`sort_by`, `sort_direction`). Use the correct casing for the interface being called.

---

## Query a knowledge base

### MCP: ask_knowledge_base

```
ask_knowledge_base(
  kb_id="<uuid>",
  question="What is our parental leave policy?",
  include_citations=true,
  intent="Answering user question about parental leave policy"
)
```

**Response shape:**
```json
{
  "response": { "text": "The parental leave policy provides..." },
  "conversationId": "<uuid>",
  "citations": [
    {
      "filename": "hr-handbook.pdf",
      "entryId": "<uuid>",
      "objectId": "33.<uuid>",
      "previewUrl": "/navigate/file/<uuid>",
      "chunks": [{ "chunkId": "<uuid>", "sourceText": "...excerpt..." }]
    }
  ]
}
```
The answer is in `response.text`. `conversationId` is returned in the response — its use for follow-up calls has not been confirmed in the current schema. `citations` is top-level (not inside `response`).

**Multi-turn:** The `ask_knowledge_base` schema does not include a `chat_history` or `conversationId` input parameter. Multi-turn is not supported by the current schema.

### CLI: egnyte ai ask-kb

```bash
egnyte ai ask-kb <kb-id> "What is our vacation policy?" \
  --json '{"includeCitations":true}' --fields response,citations
```

---

## Decision guide

| Scenario | Use |
|----------|-----|
| Policy, handbook, or reference question | `ask_knowledge_base` (if a relevant ACTIVE KB exists) |
| Specific file in mind | `ask_document` / `summarize_document` |
| Searching for documents | `search` / `advanced_search` |
| No relevant KB | `ask_ai_assistant` with `folder_ids` or `file_entry_ids` |

Always check `list_knowledge_bases` first for policy and reference questions — a KB gives more accurate, curated results than ad-hoc search.

**`ask_ai_assistant` fallback example (no active KB):**
```
ask_ai_assistant(
  question="What is the expense reimbursement process?",
  folder_ids=["<folder-uuid>"],       # scope to a folder
  file_entry_ids=["<file-uuid>"],     # or scope to specific files
  include_citations=true,
  intent="Fallback query — no active KB covers this topic"
)
```

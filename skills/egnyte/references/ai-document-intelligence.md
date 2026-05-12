# AI Document Intelligence

Three MCP tools and their CLI equivalents for document analysis.

> **All MCP AI tools have an `intent` parameter** declared in the MCP schema. The server description marks it "REQUIRED" but it is not schema-enforced — always include it (≤15 words). MCP-layer only; does not apply to CLI calls.

---

## Ask questions about a document

### MCP: ask_document

```
ask_document(
  intent="answer question about document",
  entry_id="<uuid>",
  question="What are the payment terms?",
  include_citations=true
)
```

**Response shape:**
```json
{
  "response": {
    "text": "Payment is due net-30 from the invoice date..."
  }
}
```
The answer is in `response.text`. No `conversationId` is returned by this tool.

**Citations** (when `include_citations=true`):
```json
{
  "response": { "text": "..." },
  "citations": [
    {
      "filename": "acme-nda.pdf",
      "entryId": "<uuid>",
      "chunks": [
        { "chunkId": "<uuid>", "sourceText": "...excerpt..." }
      ]
    }
  ]
}
```
Citations are a top-level array, not nested inside `response`. Note: `ask_document` citations include `filename`, `entryId`, and `chunks` only — `objectId` and `previewUrl` are not present (they appear in `ask_knowledge_base` and `ask_ai_assistant` citations).

> **Note:** Citations may not be returned for image or binary file types even when `include_citations=true` is set. Check whether `citations` is present before accessing it.

**Multi-turn conversation:**
```
ask_document(
  intent="follow-up question about document",
  entry_id="<uuid>",
  question="What are the termination clauses?",
  include_citations=true,
  chat_history=[
    {"role":"user","content":"What are the payment terms?"},
    {"role":"assistant","content":"Payment is due net-30..."}
  ]
)
```

> **`chat_history` for MCP `ask_document`:** Pass as a JSON array of `{role, content}` objects (see above). This parameter is accepted by the MCP server but does not appear in the formal tool schema — verify behavior in your environment before relying on it.
>
> **`chatHistory` for CLI `egnyte ai ask-document`:** Pass as an object with a `messages` array: `{"messages":[{"role":"user","content":"..."}]}`. This is a different shape from the MCP array format.

**Getting entry_id:**
- From `advanced_search` results: `entry_id` field is a direct UUID.
- From `search` results: split `id` on `/`, take the second part.
- From `list_filesystem_by_path` results: `files[].entry_id`.

### CLI: egnyte ai ask-document

```bash
# Path is auto-resolved to entry_id
egnyte ai ask-document /Shared/Contracts/acme.pdf "What are the payment terms?" \
  --fields response
egnyte ai ask-document /Shared/Contracts/acme.pdf "Payment terms?" \
  --json '{"includeCitations":true}' --fields response,citations
```

> **CLI param names use camelCase:** `includeCitations` (not `include_citations`), `chatHistory` (not `chat_history`, shape: `{"messages":[]}`)

---

## Summarize a document

### MCP: summarize_document

```
summarize_document(
  intent="summarize document contents",
  entry_id="<uuid>"
)
```

**Parameters:** `entry_id` only — no `question`, no `include_citations` parameter. The MCP tool does not support `chat_history`.

**Response shape:**
```json
{
  "response": {
    "text": "This document is a non-disclosure agreement between..."
  }
}
```
The summary is in `response.text`.

> **No citations:** `summarize_document` does not support `include_citations`. Summaries cannot be verified against source excerpts. For a citation-backed summary, use `ask_document(question="Summarize the key points of this document", include_citations=true)` instead — it returns the same summary-style answer with verifiable source chunks attached.

### CLI: egnyte ai summarize

```bash
egnyte ai summarize /Shared/Reports/annual.pdf --fields response

# CLI supports chatHistory for iterative summarization (MCP tool does not):
egnyte ai summarize /Shared/Reports/annual.pdf \
  --json '{"chatHistory":{"messages":[{"role":"user","content":"Focus on the financials"}]}}' \
  --fields response
```

---

## Ask across multiple documents or folders

### MCP: ask_ai_assistant

Best for synthesizing across a set of documents or a whole folder. Without `file_entry_ids` or `folder_ids`, `ask_ai_assistant` searches across content accessible to the current user. Provide scope parameters to restrict results to specific files or folders.

```
ask_ai_assistant(
  intent="find common risk factors across contracts",
  question="What are the common risk factors across these contracts?",
  file_entry_ids=["uuid1", "uuid2", "uuid3"],
  include_citations=true
)
```

```
ask_ai_assistant(
  intent="summarize Q1 meeting notes",
  question="Summarize all meeting notes from Q1",
  folder_ids=["<folder-uuid>"],
  include_citations=true
)
```

**Response shape (base — no `include_citations`):**
```json
{ "response": { "text": "..." }, "conversationId": "<uuid>" }
```

**Response shape (with `include_citations=true`):**
```json
{ "response": { "text": "..." }, "citations": [...], "conversationId": "<uuid>" }
```

`citations` is only present when `include_citations=true`. Citations from `ask_ai_assistant` include `objectId` and `previewUrl` fields (unlike `ask_document` citations).

**Getting `folder_ids`:** Call `list_filesystem_by_path` — each folder in the response has a `folder_id` field. Pass as an array: `folder_ids=["<folder_id>"]`.

### CLI: egnyte ai ask

```bash
# Ask across a folder
egnyte ai ask "What are the key metrics in Q3?" --fields response
egnyte ai ask "Revenue trends?" \
  --json '{"selectedItems":{"folders":[{"id":"<folder-id>"}]},"includeCitations":true}' \
  --fields response,citations

# Scope to specific files
egnyte ai ask "Compare liability clauses" \
  --json '{"selectedItems":{"files":[{"entryId":"<id1>"},{"entryId":"<id2>"}]},"includeCitations":true}' \
  --fields response,citations
```

---

## List and query knowledge bases

### MCP: list_knowledge_bases

```
list_knowledge_bases(
  intent="find available knowledge bases",
  status=["ACTIVE"],
  sort_direction=["DESC"]
)
```

**Optional parameters:**
- `status` — filter by KB status (e.g., `"ACTIVE"`)
- `sort_direction` — `["ASC"]` or `["DESC"]`; defaults to `["DESC"]` per MCP server description
- `include_placeholder_data` — boolean; include placeholder entries
- `include_processing_statistics` — boolean; include ingestion stats (useful for debugging KB readiness)
- `include_prompts` — boolean; expose prompt configuration for the KB

> **CLI vs MCP default conflict:** The CLI (`egnyte ai list-kbs`) defaults to `sortDirection: ['ASC']` per source. The MCP `list_knowledge_bases` defaults to `['DESC']` per server description. Pass `sort_direction` / `sortDirection` explicitly to avoid relying on either default.

**Response shape (paginated envelope):**
```json
{
  "content": [ { "kbId": "<uuid>", "name": "...", "status": "ACTIVE" } ],
  "totalElements": 5,
  "totalPages": 1,
  "number": 0,
  "size": 20,
  "first": true,
  "last": true,
  "empty": false,
  "numberOfElements": 5
}
```
`content` is the array of KB objects. Check `totalElements > 0` before calling `ask_knowledge_base`.

### MCP: ask_knowledge_base

```
ask_knowledge_base(
  intent="query knowledge base for policy information",
  kb_id="<uuid>",
  question="What is the expense reimbursement policy?",
  include_citations=true
)
```

**Response shape** (unverified — based on MCP server guide; live KB unavailable in test environment):
```json
{
  "response": { "text": "..." },
  "conversationId": "<uuid>"
}
```
With `include_citations=true`, a `citations` array is expected. Citations are expected to include `objectId` and `previewUrl` fields. `conversationId` is returned and may be used for conversation continuity.

---

## Multi-turn agent conversations

### CLI: egnyte agents ask

For complex, multi-step analysis with conversation continuity:

```bash
egnyte agents list --fields agentId,name,status,category

# Start a conversation (polls until COMPLETED, up to 5 min)
egnyte agents ask <agentId> "Summarize the Q3 board deck" --fields responseText,citations

# Continue the conversation
egnyte agents ask <agentId> "Now compare that to Q2" \
  --json '{"conversationId":"<id from prior response>"}' --fields responseText

# Scope to specific files
egnyte agents ask <agentId> "What are the key risks?" \
  --json '{"selectedItems":{"files":[{"entryId":"<id>","filePath":"/Shared/contract.pdf"}]}}' \
  --fields responseText,citations
```

---

## Decision guide

| Scenario | Use |
|----------|-----|
| Q&A about one specific document | `ask_document` / `egnyte ai ask-document` |
| Summarize one document | `summarize_document` / `egnyte ai summarize` |
| Get document text when you already have the composite ID | `fetch` (see search-and-discovery.md) |
| Synthesize across multiple docs or a folder | `ask_ai_assistant` / `egnyte ai ask` |
| Multi-turn conversation with continuity (CLI) | `egnyte agents ask` — pass `conversationId` from prior response |
| Multi-turn via MCP | Use `conversationId` from `ask_ai_assistant` response in subsequent `chat_history` calls. For `ask_document`, pass prior exchanges in `chat_history` (array format; not formally in MCP schema — see note above). |
| List and query knowledge bases | `list_knowledge_bases` / `ask_knowledge_base` |
| Semantic search across all content | `egnyte ai hybrid-search` |

---

## Tips

- For **PDFs, presentations, images**: always use AI tools. `get_file_content` returns binary for these.
- Set `include_citations=true` whenever the user needs to verify sources.
- **Chain pattern**: `summarize_document` first, then `ask_document` for specific questions.
- For multi-turn follow-up questions: MCP uses `chat_history` (array of `{role, content}` objects); CLI uses `chatHistory` (object: `{"messages":[{"role":"user","content":"..."}]}`). These are different shapes — do not conflate them.
- **Scope `ask_ai_assistant` for precision** — without `file_entry_ids` or `folder_ids` it searches across content accessible to the current user; provide scope to focus on specific documents.
- **Multi-turn with `ask_ai_assistant`**: `conversationId` is returned in the response. The exact mechanism for passing it as an MCP input to continue a conversation is not declared in the MCP schema — verify before relying on it. For CLI, pass `conversationId` as a top-level `--json` param to `egnyte agents ask`.

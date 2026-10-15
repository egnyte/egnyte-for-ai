# AI Document Intelligence

MCP tools and their CLI equivalents for document analysis: `ask_document`, `summarize_document`, and the asynchronous `ask_ai_assistant` / `get_ai_assistant_status` pair. For purpose-built Egnyte Agents (and former knowledge bases) see [`agents.md`](agents.md).

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
Citations are a top-level array, not nested inside `response`. Note: `ask_document` citations include `filename`, `entryId`, and `chunks` only — `objectId` and `previewUrl` are not present (they appear in `get_ai_assistant_status` and `get_agent_ask_status` citations).

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

### MCP: ask_ai_assistant → get_ai_assistant_status

Best for synthesizing across a set of documents or a whole folder, comparing documents, understanding what a folder is for, reading spreadsheets or scanned PDFs, extracting structured output, or generating a document back into Egnyte. **Asynchronous:** `ask_ai_assistant` returns an `executionId` immediately; the answer comes from polling `get_ai_assistant_status`.

**Scoping — exactly one mode is required:**
- `file_entry_ids` and/or `folder_ids` — maximum 20 UUIDs combined. The reliable mode, and the one that returns citations.
- `search_all_egnyte=true` — search all content the user can access.
- Combining ids with `search_all_egnyte=true` returns an error. Calling with neither currently falls back to an all-Egnyte search, but that fallback is deprecated and will become an error — always pass one.

```
# Step 1 — submit (scoped)
ask_ai_assistant(
  intent="find common risk factors across contracts",
  question="What are the common risk factors across these contracts?",
  file_entry_ids=["uuid1", "uuid2", "uuid3"]
)
→ { "executionId": "<uuid>", "status": "IN_PROGRESS", "conversationId": "<uuid-or-null>" }

# Step 1 — submit (all Egnyte)
ask_ai_assistant(
  intent="locate the latest travel policy",
  question="What is the current travel reimbursement limit?",
  search_all_egnyte=true
)

# Step 2 — poll
get_ai_assistant_status(
  intent="poll AI assistant answer",
  execution_id="<executionId>",
  include_citations=true
)
→ { "executionId": "<uuid>", "status": "IN_PROGRESS" }
→ { "executionId": "<uuid>", "status": "COMPLETED", "responseText": "...", "citations": [...] }
```

**`get_ai_assistant_status` fields:** `status` is `IN_PROGRESS`, `COMPLETED`, or `AWAITING_USER_CONFIRMATION`. `responseText` and `citations` are present only on `COMPLETED`; `include_citations` defaults to true — pass `false` to suppress citations. A `FAILED` run surfaces as a tool error, not a status value.

**Polling rules:**
- Poll every few seconds. Scoped questions usually complete in 20–60 seconds; document generation or several large files can take two to four minutes. Allow at least five minutes before treating a call as stalled.
- `AWAITING_USER_CONFIRMATION` is terminal: retry once with a reframed `ask_ai_assistant` question; if it happens again, stop and tell the user the request cannot be completed via MCP.
- On a rate-limit error, back off and increase the delay between retries. Put related files into one call instead of one call per file.

**Other behavior:**
- Each call is an independent conversation with no follow-up turn — put the full, self-contained question in one call.
- When a document is generated or converted, the `COMPLETED` response states the folder and filename actually used; read it from the response rather than assuming the requested path.
- Files up to 64 MB. Web search is available and cited when the question needs public information.

> **Legacy domains:** Some domains are still being moved to the asynchronous tools. There, `ask_ai_assistant` answers in a single call with `{ "response": { "text": "..." }, "citations": [...], "conversationId": "<uuid>" }`, accepts `include_citations`, and `get_ai_assistant_status` is not listed. Check the tool list: if `get_ai_assistant_status` is absent, treat `ask_ai_assistant` as synchronous.

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

## Egnyte Agents

Purpose-built agents (and all former knowledge bases) are covered in [`agents.md`](agents.md): `list_agents` → `ask_agent` → `get_agent_ask_status` via MCP, or `egnyte agents list` / `egnyte agents ask` via CLI (the CLI polls for you and supports multi-turn with `conversationId`).

---

## Decision guide

| Scenario | Use |
|----------|-----|
| Q&A about one specific document | `ask_document` / `egnyte ai ask-document` |
| Summarize one document | `summarize_document` / `egnyte ai summarize` |
| Get document text when you already have the composite ID | `fetch` (see search-and-discovery.md) |
| Synthesize across multiple docs or a folder, compare documents, read spreadsheets/scans, extract structured data, generate a document | `ask_ai_assistant` → `get_ai_assistant_status` / `egnyte ai ask` |
| Multi-turn conversation with continuity (CLI) | `egnyte agents ask` — pass `conversationId` from prior response |
| Multi-turn via MCP | `ask_ai_assistant` and `ask_agent` have no follow-up turn — put the full question in one call. For `ask_document`, pass prior exchanges in `chat_history` (array format; not formally in MCP schema — see note above). |
| Question for a purpose-built agent or a former knowledge base | `list_agents` → `ask_agent` → `get_agent_ask_status` / `egnyte agents ask` (see [`agents.md`](agents.md)) |
| Semantic search across all content | `egnyte ai hybrid-search` |

---

## Tips

- For **PDFs, presentations, images**: always use AI tools. `get_file_content` returns binary for these.
- Set `include_citations=true` whenever the user needs to verify sources.
- **Chain pattern**: `summarize_document` first, then `ask_document` for specific questions.
- For multi-turn follow-up questions: MCP uses `chat_history` (array of `{role, content}` objects); CLI uses `chatHistory` (object: `{"messages":[{"role":"user","content":"..."}]}`). These are different shapes — do not conflate them.
- **Scope `ask_ai_assistant` for precision** — pass `file_entry_ids` / `folder_ids` (max 20) for citations and reliability; use `search_all_egnyte=true` only when the user genuinely wants the whole domain searched.
- **No multi-turn with MCP `ask_ai_assistant` or `ask_agent`** — each call is independent. For conversation continuity use CLI `egnyte agents ask` with `conversationId`.

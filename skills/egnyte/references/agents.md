# Agents

Egnyte Agents are AI assistants configured for a specific purpose, with their own instructions and content sources. Depending on configuration, an agent may answer questions about content, search Egnyte and the web, create and modify files, generate and upload documents, or operate on Excel files. Always call `list_agents` before `ask_agent`.

> **Knowledge Bases are deprecated and migrated to Agents.** Each Knowledge Base keeps its file sources, behavior, settings, and its ID. The former `kb_id` is now the `agent_id`. On migrated domains `list_knowledge_bases` and `ask_knowledge_base` return an error that points to the agent tools — never call them; use `list_agents` and `ask_agent` instead.

> **`intent` parameter:** All Egnyte MCP tools accept an `intent` string (max 15 words) explaining why the call is being made. Include it in every call.

---

## List agents

### MCP: list_agents

```
list_agents(intent="Listing agents to find the right one for a policy question")
```

No parameters. Returns the **active, user-defined** agents the user created or that are shared with them. Egnyte's built-in agents are not included.

**Response:** a bare JSON array (not a paginated envelope). Each agent has:

| Field | Notes |
|-------|-------|
| `agentId` | UUID — pass to `ask_agent` as `agent_id` |
| `name` | display name |
| `description` | what the agent is configured to do |
| `category` | agent category |
| `status` | `ACTIVE` |

Read `name`, `description`, and `category` to pick the agent whose purpose matches the question. If no agent fits, fall back to `ask_ai_assistant` (see [`ai-document-intelligence.md`](ai-document-intelligence.md)).

> **Availability:** Agent tools appear only on domains where Agents are enabled, and only for Power Users and Admins. If `list_agents` is not in the tool list, tell the user agents are not available on their account and use `ask_ai_assistant` instead.

### CLI: egnyte agents list

```bash
egnyte agents list --fields agentId,name,status,category
```

---

## Ask an agent

### MCP: ask_agent → get_agent_ask_status

`ask_agent` is **asynchronous**. It returns immediately with identifiers; the answer comes from polling `get_agent_ask_status`.

```
# Step 1 — submit
ask_agent(
  agent_id="<agentId from list_agents>",
  question="What is our parental leave policy?",
  intent="Asking the HR policy agent about parental leave"
)
→ { "agentId": "<uuid>", "requestId": "<uuid>", "conversationId": "<uuid>" }

# Step 2 — poll until terminal
get_agent_ask_status(
  agent_id="<agentId>",
  request_id="<requestId>",
  intent="Polling agent response for parental leave question"
)
→ { "status": "IN_PROGRESS" }
→ { "status": "COMPLETED", "responseText": "...", "citations": [...] }
```

**`ask_agent` parameters:**

| Param | Required | Notes |
|-------|----------|-------|
| `agent_id` | Yes | UUID from `list_agents`. Only user-defined agents are accepted; built-in agents are rejected. |
| `question` | Yes | Up to 35,000 characters. Each call is an independent conversation — put the full question in one call. |
| `folder_ids` | No | Folder UUIDs **added** to the agent's configured scope for this question only (from `list_filesystem_by_path` `folder_id`). |
| `file_entry_ids` | No | File `entry_id` UUIDs added to the agent's scope for this question only. |

**`get_agent_ask_status` parameters:** `agent_id` and `request_id`, both from the `ask_agent` response.

**Status values:**

| Status | Meaning | Action |
|--------|---------|--------|
| `PLANNING`, `IN_PROGRESS` | still running | keep polling every few seconds; responses can take up to two minutes |
| `COMPLETED` | answer ready | read `responseText` and `citations` |
| `FAILED`, `COMPLETED_FAILED` | request did not succeed | stop polling; report the failure |
| `AWAITING_USER_CONFIRMATION` | **terminal for MCP** — the agent needs a confirmation or an external app sign-in that cannot be given through MCP; `responseText` is typically empty | stop polling; tell the user to run that agent in the Egnyte web UI |

Do not poll in a tight loop — excessive requests are rate-limited (429). Allow at least two minutes before treating a request as stalled.

**Errors from `ask_agent`:**
- "Agent is inactive" / "Agent not found" → re-run `list_agents` and pick an active agent; never guess an `agent_id`.
- "agent_id refers to a non-user-defined agent" → built-in agents cannot be invoked via MCP; choose a user-defined agent or use `ask_ai_assistant`.

### CLI: egnyte agents ask / agents status

The CLI polls for you by default (up to 5 minutes) and supports multi-turn via `conversationId`:

```bash
# Blocks until complete
egnyte agents ask <agentId> "Summarize the Q3 board deck" --fields responseText,citations

# Continue the conversation
egnyte agents ask <agentId> "Now compare that to Q2" \
  --json '{"conversationId":"<id from prior response>"}' --fields responseText

# Scope to specific files
egnyte agents ask <agentId> "What are the key risks?" \
  --json '{"selectedItems":{"files":[{"entryId":"<id>","filePath":"/Shared/contract.pdf"}]}}' \
  --fields responseText,citations

# Fire-and-forget, then check later
egnyte agents ask <agentId> "Long analysis task" --no-wait --fields requestId,conversationId
egnyte agents status <agentId> <requestId> --fields status,responseText,citations
```

> **Deprecated CLI commands:** `egnyte ai list-kbs` and `egnyte ai ask-kb` stop working once the domain is migrated (the API returns `410 Gone`, error code `KBA_MIGRATED_TO_AGENTS`). Use `egnyte agents list` / `egnyte agents ask` with the former KB ID as `agentId`.

---

## Decision guide

| Scenario | Use |
|----------|-----|
| Policy, handbook, or reference question that a configured agent covers | `list_agents` → `ask_agent` → `get_agent_ask_status` |
| User names an agent, or has a legacy `kb_id` | `ask_agent(agent_id=<id>)` directly (verify it appears in `list_agents` first) |
| Specific file in mind | `ask_document` / `summarize_document` |
| Searching for documents | `search` / `advanced_search` |
| No relevant agent, or agents unavailable on the domain | `ask_ai_assistant` scoped with `folder_ids` / `file_entry_ids` |
| Multi-turn follow-ups with the same agent | CLI `egnyte agents ask` with `conversationId` — MCP `ask_agent` has no follow-up input |

Agents give curated, purpose-specific answers; prefer them over ad-hoc search for questions they are configured to handle.

# enowX ChatUI Memory + Tools System — Design Spec

**Date:** 2026-09-02  
**Status:** DRAFT — awaiting owner review before implementation  
**Author:** Claude (audit + design)

---

## 1. Goal

Build a complete agentic layer for enowX ChatUI:

1. **Persistent sessions** — chat history in SQLite (`chatui.db`), survives browser clear
2. **Memory system** — summarise turns into durable facts, recall across sessions
3. **Tool use** — Claude can use 8 tools (web search, file ops, terminal, memory)
4. **MCP client** — connect to external MCP servers (filesystem, fetch, time, etc.)
5. **Skills** — inject lightweight skill hints into system prompt
6. **Plugin middleware** — pre/post hooks for approval gates and output filtering
7. **Token counting** — count tokens before sending, budget guard to prevent context overflow
8. **Native vision** — user can attach images to messages (drag-drop or paste)
9. **Silent compaction** — auto-summarise old turns when context window fills up (like ChatGPT)

All of this is built **on top of** the existing enowX proxy layer — zero changes to
the proxy, pool, SSE pipeline, or any existing handler.

---

## 2. Hard Rules (Non-Negotiable)

```
NEVER touch:
  - enowX core proxy (/anthropic/v1/*, /v1/*)
  - Account pool, model routing, retry logic
  - SSE streaming pipeline
  - Port 1430 (production)
  - /usr/local/bin/enx
  - enowX store/sqlite/ (existing enowx.db)
  - Antares codebase (zero dependency, zero import)
```

Antares = **architecture reference only**. We study its patterns and build
our own simpler implementation. No `github.com/enowdev/antares` import.

---

## 3. Two-Layer Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│  ChatUI Agentic Layer (new code in enowX)                       │
│                                                                  │
│  Tools, MCP, Skills, Memory all use Anthropic SDK:             │
│                                                                  │
│    anthropic.NewClient(                                         │
│      option.WithAPIKey("dummy"),                                │
│      option.WithBaseURL("http://localhost:<ENOWX_PORT>"),       │
│    )                                                            │
│                                                                  │
│  SDK → /anthropic/v1/messages on enowX                         │
│  SDK never knows about account pool                             │
│  enowX pool handles routing + retry + usage tracking           │
│                                                                  │
└────────────────────────┬────────────────────────────────────────┘
                         │ POST /anthropic/v1/messages
                         │ (standard Anthropic wire format)
┌────────────────────────▼────────────────────────────────────────┐
│  enowX proxy (UNTOUCHED)                                        │
│  Account pool · model routing · retry · SSE pipeline            │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4. System Diagram

```
Browser
  ChatUI SPA (/chatui/)
    useSessions ←── REST ──→ /chatui/api/sessions/*
    useChat     ←── SSE  ──→ /chatui/api/chat/stream   (new agentic endpoint)
                             (internally uses SDK → enowX proxy)

enowX Go server (:1430/:1431)
  ┌─ NEW: server/handlers/chatui_sessions.go
  │    GET/POST/PATCH/DELETE /chatui/api/sessions/*
  │    POST /chatui/api/sessions/:id/messages
  │
  ├─ NEW: server/handlers/chatui_chat.go
  │    POST /chatui/api/chat/stream  ← agentic loop (SDK + tools)
  │
  ├─ NEW: server/handlers/chatui_memory.go
  │    POST /chatui/api/sessions/:id/index
  │    GET  /chatui/api/sessions/:id/memories
  │    GET  /chatui/api/memories?q=...
  │
  ├─ NEW: core/chatui/
  │    agent.go       — agentic loop (SDK streaming + tool dispatch)
  │    tools.go       — 8 built-in tool implementations
  │    mcp.go         — MCP client (stdio + HTTP transports)
  │    skills.go      — skill loader + system prompt injection
  │    plugins.go     — plugin middleware (pre/post hooks)
  │    memory.go      — memory indexer (background goroutine)
  │
  └─ NEW: store/chatui/
       store.go       — ChatStore interface
       sqlite.go      — modernc SQLite implementation
       migrations.go  — embedded DDL
       types.go       — Session, Message, Memory, Skill types

chatui.db  (~/.enowx/chatui.db, separate from enowx.db)
  sessions · messages · memories · skills · kv
```

---

## 5. Antares Reference Map

What we studied in Antares and what we take from each domain:

| Domain | Antares has | What we build in ChatUI |
|---|---|---|
| **Store** | `sessions` + `messages` tables | Same field shapes, own DDL, own migrations |
| **Memory** | `memories` table: scope/scope_key/key/content + FTS | Identical table design, own migration |
| **Memory indexing** | `indexUserTurn()` → LLM summarise → write memories | Same pipeline, Anthropic SDK for LLM call |
| **Tools** | ~81 built-in tools, typed schemas, approval gates | 8 tools (scoped for conversational chat) |
| **MCP client** | stdio + HTTP transports, JSON-RPC 2.0, tools/list | Same protocol, simpler client (6 servers) |
| **Skills** | Markdown files, YAML front matter, PromptBlock injection | Same concept: `.md` files injected as hints |
| **Plugins** | pre/post hooks, stdin/stdout JSON protocol | Simplified: Go interface, no subprocess |
| **Agent loop** | max 50 turns, parallel tool calls, approval gate | Simpler: max 10 turns, serial tool calls |
| **System prompt** | 13-section builder, SOUL.md, RAG injection | Simpler 5-section builder (no SOUL, no RAG in P2) |
| **Migration** | Re-runs all DDL every startup (no version table) | `schema_version` KV key — run only new migrations |

---

## 6. Tool System

### 6.1 Tool inventory (8 tools, Phase 2)

| Tool | Description | Max output tokens | Guard |
|---|---|---|---|
| **`memory`** | Save / search / list facts across sessions. Actions: `save`, `search`, `list`, `delete`. | 500 | None |
| **`web_search`** | Web search, 8 results default. Returns title + snippet + URL per result. | 1500 | None |
| **`web_fetch`** | Fetch a URL, convert to plain text. Hard cap: 15K chars output. | 3000 | Size cap |
| **`read_file`** | Read a file from workspace. Default 200 lines, hard cap 400 lines. | 2000 | Size cap |
| **`write_file`** | Write or create a file. Returns confirmation only. | 100 | Approval |
| **`list_files`** | List directory contents. Skips `.git`, `node_modules`. Max depth 3. | 500 | None |
| **`grep`** | Regex search across files. Max 50 matches returned. | 1000 | None |
| **`terminal`** | Execute shell command. Timeout 30s. Output truncated to 3000 tokens. | 3000 | Approval + timeout |

**Total tool schema tokens in system prompt:** ~800 tokens (all 8 tool descriptions)

### 6.2 Approval gate

Tools marked **Approval** (`write_file`, `terminal`) require explicit user confirmation
before execution. In the agentic loop:

```
1. Claude proposes tool call
2. Server sends SSE event: {"type": "tool_approval_required", "tool": "terminal", "args": {...}}
3. Frontend shows approval dialog to user
4. User approves → server executes → result injected into context
5. User denies  → server injects tool_result with "User denied execution"
```

### 6.3 Output truncation

Every tool result is truncated before injecting into the context window:

```go
const (
    MaxToolOutputTokens = 3000  // hard cap per tool call
    MaxTotalToolTokens  = 8000  // hard cap per turn (sum of all tool results)
)
```

If truncated, a note is appended: `[output truncated — 3000 token limit reached]`

### 6.4 Tool registration

Each tool implements:

```go
type Tool interface {
    Name() string
    Description() string
    InputSchema() json.RawMessage   // JSON Schema for Anthropic tool_use
    Execute(ctx context.Context, input json.RawMessage) (string, error)
    RequiresApproval() bool
}
```

Tools are registered at startup and their schemas are included in every
Anthropic SDK request via `anthropic.ToolParam{}` array.

---

## 7. MCP Client

### 7.1 Protocol

- JSON-RPC 2.0 over stdio (subprocess) or HTTP/SSE
- Protocol version: `2024-11-05`
- Handshake: `initialize` → `notifications/initialized` → `tools/list`
- Tool naming: `mcp__<serverId>__<toolName>`

### 7.2 Supported servers (Phase 3)

| Server ID | Transport | What it provides |
|---|---|---|
| `filesystem` | stdio | Read/write files under scoped dirs |
| `fetch` | stdio | URL fetcher (robots-aware) |
| `memory` | stdio | Persistent entity/relation knowledge graph |
| `sequential-thinking` | stdio | Branching reasoning scratchpad |
| `time` | stdio | Current time + timezone conversion |
| `brave-search` | stdio | Web search (requires Brave API key) |

MCP tools appear in the tool list alongside built-in tools. They go through the
same approval gate and output truncation.

### 7.3 Config

MCP servers are configured in `chatui.db` `kv` table under key `mcp.servers`.
Both stdio and HTTP transports are supported:

```json
[
  {
    "id": "filesystem",
    "transport": "stdio",
    "command": "npx",
    "args": ["-y", "@modelcontextprotocol/server-filesystem", "/home/user"]
  },
  {
    "id": "my-remote-server",
    "transport": "http",
    "url": "http://localhost:3000/mcp"
  }
]
```

Both transports use JSON-RPC 2.0. stdio: newline-delimited on stdin/stdout.
HTTP: `POST /mcp` for requests, `GET /mcp` with `Accept: text/event-stream` for
streaming. Tools from both transports land in the registry as
`mcp__<serverId>__<toolName>` and go through the same approval gate and
output truncation as built-in tools.

---

## 8. Skills System

### 8.1 What skills are

Lightweight Markdown files with YAML front matter. Injected as a compact
hint block in the system prompt — **not** full content, just name + description + triggers.

```markdown
---
name: systematic-debugging
description: Diagnose bugs step by step
triggers: [debug, error, exception, traceback, not working]
enabled: true
---
# Systematic Debugging
...full content only loaded on demand...
```

### 8.2 System prompt injection

At session start, enabled skills inject as ~15 tokens each:

```
## Your skills

- systematic-debugging: Diagnose bugs step by step (use when: debug, error, exception)
- code-review: Review code for bugs and issues (use when: review, PR, quality)
- web-research: Research a topic thoroughly (use when: research, find out, learn about)
```

**Token cost:** 6 default skills × ~20 tokens = **~120 tokens** — negligible.

### 8.3 Built-in skills (6 defaults, Phase 3)

| Skill | Triggers |
|---|---|
| `systematic-debugging` | debug, error, traceback, not working |
| `code-review` | review, PR, quality, audit |
| `web-research` | research, find out, learn about, explain |
| `writing-clearly` | write, document, explain, summarize |
| `test-driven-development` | test, TDD, coverage, spec |
| `git-workflow` | git, commit, branch, merge |

### 8.4 Storage

Skills stored as `.md` files in `$ENOWX_RUNTIME_DIR/chatui/skills/`.
Enable/disable state stored in `chatui.db` `kv` table:
```
kv_key = "skills.enabled"
kv_value = ["systematic-debugging", "code-review", ...]
```

---

## 9. Plugin Middleware

### 9.1 What plugins are (vs Antares)

Antares plugins = external subprocess with stdin/stdout JSON protocol.
ChatUI plugins = **Go interface** — no subprocess, no IPC overhead.

Simpler, faster, sufficient for ChatUI's needs.

### 9.2 Plugin interface

```go
type Plugin interface {
    Name() string
    BeforeToolCall(ctx context.Context, toolName string, args json.RawMessage) (PluginDecision, error)
    AfterToolCall(ctx context.Context, toolName string, args json.RawMessage, result string) (string, error)
}

type PluginDecision struct {
    Allow     bool
    DenyReason string     // if !Allow
    ModifiedArgs json.RawMessage  // optional: replace args
}
```

### 9.3 Built-in plugins (Phase 2)

| Plugin | Hook | What it does |
|---|---|---|
| `ApprovalGate` | Before `write_file`, `terminal` | Blocks execution until user confirms |
| `OutputTruncator` | After all tools | Truncates output to `MaxToolOutputTokens` |
| `WebFetchCleaner` | After `web_fetch` | Strips HTML remnants, compresses whitespace |

### 9.4 Plugin chain

Plugins run in order. `BeforeToolCall`: first denier wins (stops chain).
`AfterToolCall`: result is threaded through all plugins sequentially.

---

## 10. Agentic Chat Loop

### 10.1 New endpoint

```
POST /chatui/api/chat/stream
Body: {
  "session_id": "...",
  "message": "...",
  "model": "claude-opus-4-5",
  "system_extra": "..."   // optional extra system prompt
}
Response: SSE stream
```

This replaces the current direct `/anthropic/v1/messages` call from `useChat.ts`
for sessions that have tool use enabled. Sessions without tools continue using
the existing endpoint directly.

### 10.2 Loop structure (inspired by Antares `agent.Run()`)

```
ChatLoop(ctx, sessionID, userMessage, attachments[]):
  1. buildSystemPrompt(sessionID)     // 5-section prompt (see §12)
  2. loadMessages(sessionID)          // load history from chatui.db
  3. maybeCompact(messages)           // silent compaction if near context limit (see §10.4)
  4. buildUserMessage(userMessage, attachments)  // text + optional ImageBlockParams
  5. tokenCount = SDK.CountTokens(systemPrompt, messages, tools)  // see §10.5
     if tokenCount > modelContextLimit * 0.9 → force compact now
  6. Loop (max 10 turns):
     a. SDK.Messages.NewStreaming(ctx, {tools: allTools, messages: history})
     b. Stream response tokens to client via SSE
     c. if no tool_use blocks → done, persist assistant message
     d. for each tool_use block:
        - runPlugins.BeforeToolCall(tool, args)
          → if denied: inject tool_result "denied: <reason>", continue
          → if needs approval: send SSE approval_required event, wait
        - tool.Execute(ctx, args)
        - runPlugins.AfterToolCall(tool, args, result) → truncate
        - inject tool_result into messages
     e. continue loop with tool results
  7. persistMessages(sessionID, userMsg, assistantMsg)
  8. fire-and-forget: indexMemory(sessionID)
```

### 10.3 Loop limits

| Parameter | Value | Reason |
|---|---|---|
| Max turns | 10 | Conversational chat, not autonomous agent |
| Max parallel tool calls | 1 (serial) | Simpler approval flow |
| Max tool output (per call) | 3000 tokens | Context budget protection |
| Max tool output (per turn) | 8000 tokens | Context budget protection |
| Terminal timeout | 30s | User-facing, must feel responsive |
| web_fetch size cap | 15K chars | ~3750 tokens |
| Compaction threshold | 90% of model context limit | Trigger before hitting hard wall |
| Compaction target | Keep last 10 messages intact | Preserve recent context |

### 10.4 Silent Compaction

When the context window approaches the model's limit, ChatUI silently summarises
old messages in-place — exactly like ChatGPT. The user never sees a hard error;
the conversation just continues.

```
maybeCompact(messages, systemPromptTokens):
  1. Estimate total tokens: systemPromptTokens + sum(message tokens)
  2. If total < modelContextLimit * 0.85 → skip, nothing to do
  3. Identify compaction boundary:
       - Keep: last 10 messages always (recent context is precious)
       - Compact: everything before that boundary
  4. SDK call (same model as session):
       prompt: "Summarise this conversation history concisely.
                Preserve: key decisions, facts established, code written,
                errors encountered, current task state.
                Output: dense factual summary, no filler."
       input: messages[0..boundary]
  5. Replace messages[0..boundary] with single synthetic message:
       role: "user"
       content: "[Conversation summary]\n{summary}"
  6. Persist compacted flag on those rows in chatui.db:
       messages.compacted = true, messages.content = summary (for first row)
       remaining compacted rows: deleted from DB, not just flagged
  7. Send SSE event to frontend: {"type": "compacted", "removed": N, "summary_tokens": M}
     Frontend shows subtle indicator: "Earlier messages summarised"
```

**Why silent:** User is mid-conversation. Hard stopping with "context full" breaks flow.
The summary preserves all meaningful context. This is the same approach Antares uses
(`compaction` in `agent.go`) and what ChatGPT does.

**Compaction model:** same as session model (follows Decision 3 — memory indexing model).

### 10.5 Token Counting

Before every SDK call, count tokens to:
1. Decide whether compaction is needed (pre-flight check)
2. Show token usage in UI (optional, Phase 3)
3. Guard against accidentally sending a request that will fail at the API

```go
// core/chatui/agent.go
count, err := sdkClient.Messages.CountTokens(ctx, anthropic.MessageCountTokensParams{
    Model:    session.Model,
    System:   []anthropic.TextBlockParam{{Text: systemPrompt}},
    Messages: messages,
    Tools:    toolParams,
})
// count.InputTokens = total tokens that would be sent
// Compare against model's known context limit
```

Model context limits stored in `kv` table as `model.context.<modelID>` (populated
from `/chatui/api/config` response which already returns model list). Fallback: 100K
if unknown.

**Note on prompt caching:** The SDK supports `cache_control` blocks for Anthropic-native
providers (saves cost on repeated system prompts). This is **deferred** — requires
per-provider detection in enowX proxy to strip `cache_control` for non-Anthropic
providers. Noted as future improvement when proxy layer supports it.

---

## 11. Database Schema

### 11.1 `sessions`

```sql
CREATE TABLE IF NOT EXISTS sessions (
    id            TEXT PRIMARY KEY,
    title         TEXT NOT NULL DEFAULT 'New Chat',
    model         TEXT NOT NULL DEFAULT 'bob/premium',
    message_count INTEGER NOT NULL DEFAULT 0,
    tokens_in     INTEGER NOT NULL DEFAULT 0,
    tokens_out    INTEGER NOT NULL DEFAULT 0,
    archived      BOOLEAN NOT NULL DEFAULT 0,
    created_at    DATETIME NOT NULL DEFAULT (datetime('now')),
    updated_at    DATETIME NOT NULL DEFAULT (datetime('now'))
);
-- No tools_enabled column — tool use is a global setting (kv: chatui.tools_enabled)
CREATE INDEX IF NOT EXISTS idx_sessions_updated ON sessions(updated_at DESC);
```

### 11.2 `messages`

```sql
CREATE TABLE IF NOT EXISTS messages (
    id           TEXT PRIMARY KEY,
    session_id   TEXT NOT NULL REFERENCES sessions(id) ON DELETE CASCADE,
    seq          INTEGER NOT NULL,
    role         TEXT NOT NULL CHECK(role IN ('user','assistant','system','tool')),
    content      TEXT NOT NULL DEFAULT '',
    reasoning    TEXT,
    tool_calls   TEXT,        -- JSON array of tool_use blocks
    tool_results TEXT,        -- JSON array of tool_result blocks
    attachments  TEXT,        -- JSON array of {type:"image", media_type, data_b64|url}
    compacted    BOOLEAN NOT NULL DEFAULT 0,  -- true if this row is a compaction summary
    model        TEXT,
    tokens_in    INTEGER NOT NULL DEFAULT 0,
    tokens_out   INTEGER NOT NULL DEFAULT 0,
    created_at   DATETIME NOT NULL DEFAULT (datetime('now')),
    UNIQUE(session_id, seq)
);
CREATE INDEX IF NOT EXISTS idx_messages_session ON messages(session_id, seq);

-- FTS5 for cross-session history recall (porter stemmer: "build"/"built"/"building" all match)
CREATE VIRTUAL TABLE IF NOT EXISTS messages_fts USING fts5(
    content,
    reasoning,
    content='messages',
    content_rowid='rowid',
    tokenize='porter unicode61'
);

-- Keep FTS in sync automatically
CREATE TRIGGER IF NOT EXISTS messages_ai AFTER INSERT ON messages BEGIN
    INSERT INTO messages_fts(rowid, content, reasoning)
    VALUES (new.rowid, new.content, coalesce(new.reasoning,''));
END;
CREATE TRIGGER IF NOT EXISTS messages_ad AFTER DELETE ON messages BEGIN
    INSERT INTO messages_fts(messages_fts, rowid, content, reasoning)
    VALUES ('delete', old.rowid, old.content, coalesce(old.reasoning,''));
END;
CREATE TRIGGER IF NOT EXISTS messages_au AFTER UPDATE ON messages BEGIN
    INSERT INTO messages_fts(messages_fts, rowid, content, reasoning)
    VALUES ('delete', old.rowid, old.content, coalesce(old.reasoning,''));
    INSERT INTO messages_fts(rowid, content, reasoning)
    VALUES (new.rowid, new.content, coalesce(new.reasoning,''));
END;
```

### 11.3 `memories`

```sql
CREATE TABLE IF NOT EXISTS memories (
    id         TEXT PRIMARY KEY,
    scope      TEXT NOT NULL CHECK(scope IN ('session','user','global')),
    scope_key  TEXT NOT NULL,
    key        TEXT NOT NULL,
    content    TEXT NOT NULL,
    source     TEXT NOT NULL DEFAULT 'chatui',
    created_at DATETIME NOT NULL DEFAULT (datetime('now')),
    updated_at DATETIME NOT NULL DEFAULT (datetime('now')),
    UNIQUE(scope, scope_key, key)
);
CREATE INDEX IF NOT EXISTS idx_memories_scope ON memories(scope, scope_key);

CREATE VIRTUAL TABLE IF NOT EXISTS memories_fts USING fts5(
    key, content,
    content='memories', content_rowid='rowid'
);
```

### 11.4 `kv`

```sql
CREATE TABLE IF NOT EXISTS kv (
    kv_key   TEXT PRIMARY KEY,
    kv_value TEXT NOT NULL
);
-- Keys used:
--   schema_version             "1", "2", ...
--   chatui.active_session      last active session ID
--   chatui.tools_enabled       "true" | "false"  (global tool use toggle, default "false")
--   chatui.workspace_dir       absolute path, default $HOME
--   mcp.servers                JSON array of MCP server configs
```

### 11.5 `skills`

```sql
CREATE TABLE IF NOT EXISTS skills (
    name        TEXT PRIMARY KEY,
    description TEXT NOT NULL DEFAULT '',
    triggers    TEXT NOT NULL DEFAULT '[]',   -- JSON array of trigger keywords
    category    TEXT NOT NULL DEFAULT '',
    enabled     BOOLEAN NOT NULL DEFAULT 1,
    source      TEXT NOT NULL DEFAULT 'builtin',  -- 'builtin' | 'user'
    file_path   TEXT NOT NULL DEFAULT '',          -- absolute path to .md file
    updated_at  DATETIME NOT NULL DEFAULT (datetime('now'))
);
-- Populated/synced from .md files on startup and on /chatui/api/skills/reload
-- content lives in the .md file; DB stores metadata + enable state only
```

### 11.6 Migration strategy

```go
// On startup: run only new migrations
version := kv.Get("schema_version") // "" on fresh install
if version < currentVersion {
    tx.runMigrationsFrom(version+1, currentVersion)
    kv.Set("schema_version", currentVersion)
}
```

This avoids Antares's "re-run all DDL every startup" risk.

---

## 12. System Prompt Builder

5-section prompt (simplified from Antares's 13-section builder):

```
Section 1: Identity (~50 tokens)
  "You are an AI assistant in enowX ChatUI. Today is {date}."

Section 2: Tool notes (~200 tokens, only if tools_enabled)
  Brief description of available tools and when to use them.
  Approval gate reminder for terminal and write_file.

Section 3: Skills block (~120 tokens, Phase 3)
  "## Your skills\n- skill-name: description (use when: triggers)\n..."

Section 4: Knowledge (~1000 tokens max, Phase 2)

  Section 4a: Curated memories (~200 tokens)
    "## What you remember
     - key: fact (curated by LLM indexer, high signal)"
    Source: memories table, last 10 user-scoped + 5 session-scoped

  Section 4b: Relevant past context (~800 tokens, only if FTS hits)
    "## Relevant past context
     [date | session title] (role) message excerpt..."
    Source: messages_fts FTS5 search over all sessions
    Frozen at session start — not re-queried mid-session

Section 5: Extra (~variable)
  session.system_extra if set (user-provided persona/context).
```

**Frozen snapshot pattern (from Hermes):** Section 4 is built once at
session start and never mutated mid-session. This keeps context stable
across all turns. Mid-session memory writes (from indexer) persist to DB
but do NOT update the current session's prompt — they appear in the
NEXT session's Section 4.

**Total system prompt (Phase 2, tools + knowledge):** ~800–1200 tokens
**Total system prompt (Phase 3, tools + knowledge + skills):** ~900–1400 tokens

---

## 13. Session History as Knowledge — FTS-based Recall

### 13.1 Design decision: FTS5, not vector search

Embedding/vector search requires an embedding model. Bob gateway returns
`403 Access Denied` on `/inference/v1/embeddings` and exposes zero embedding
models in its catalog. No local GPU. No external embedding provider.

**FTS5 is the right answer** for this use case:
- User searches their own history using the same words they used when building
- "sidebar bug" → hits exact conversation → no semantic gap to bridge
- SQLite FTS5: built-in, zero infra, already in schema

Pattern inspired by Hermes Agent's frozen snapshot: relevant history is
retrieved at session start and injected as a stable context block —
not re-queried every turn.

### 13.2 FTS schema

```sql
-- messages_fts: virtual table over messages content
CREATE VIRTUAL TABLE IF NOT EXISTS messages_fts USING fts5(
    content,
    reasoning,
    content='messages',
    content_rowid='rowid',
    tokenize='porter unicode61'   -- porter stemming: "building" matches "build"
);

-- Keep FTS in sync via triggers
CREATE TRIGGER IF NOT EXISTS messages_ai AFTER INSERT ON messages BEGIN
    INSERT INTO messages_fts(rowid, content, reasoning)
    VALUES (new.rowid, new.content, coalesce(new.reasoning,''));
END;
CREATE TRIGGER IF NOT EXISTS messages_ad AFTER DELETE ON messages BEGIN
    INSERT INTO messages_fts(messages_fts, rowid, content, reasoning)
    VALUES ('delete', old.rowid, old.content, coalesce(old.reasoning,''));
END;
CREATE TRIGGER IF NOT EXISTS messages_au AFTER UPDATE ON messages BEGIN
    INSERT INTO messages_fts(messages_fts, rowid, content, reasoning)
    VALUES ('delete', old.rowid, old.content, coalesce(old.reasoning,''));
    INSERT INTO messages_fts(rowid, content, reasoning)
    VALUES (new.rowid, new.content, coalesce(new.reasoning,''));
END;
```

Porter stemmer means "building" matches "build", "built" matches "build" —
handles natural language variation without any embedding model.

### 13.3 Recall pipeline

```
New session starts (or new turn arrives):
  1. Extract query terms from user's first message
     (or re-use session title as query)
  2. FTS search across ALL sessions:
       SELECT m.content, m.role, s.title, s.created_at, s.model
       FROM messages_fts fts
       JOIN messages m ON m.rowid = fts.rowid
       JOIN sessions s ON s.id = m.session_id
       WHERE messages_fts MATCH 'query terms'
         AND m.compacted = 0
         AND m.role IN ('user', 'assistant')
       ORDER BY rank, s.updated_at DESC
       LIMIT 8
  3. Group results by session, take top-2 sessions
  4. Format as frozen context block (Hermes pattern):
       "## Relevant past context
        [2026-09-01 | session: ChatUI Sidebar] (assistant) Built ChatSidebar
        with GlideMenu. Collapsed width 52px, expanded 224px. Toggle moved
        to floating header...
        [2026-08-30 | session: enowX deploy] (user) How do I deploy to :1431?"
  5. Inject into Section 4 of system prompt (replaces per-turn memories)
  6. Frozen for the session — not re-queried mid-session (preserves context
     stability, same as Hermes frozen snapshot pattern)
```

**Token budget for history context:** hard cap 800 tokens. Each chunk
truncated to 200 chars. If FTS returns nothing: section omitted entirely.

### 13.4 Memory indexing pipeline (still runs, complementary)

FTS search over raw messages handles recall. The LLM-based memory indexer
writes **curated facts** to the `memories` table — higher signal, smaller
footprint. Both layers work together:

```
indexMemory(sessionID) — background goroutine, fire-and-forget:
  1. Rate-limit: last indexed < 30s ago? skip.
  2. Load last user + assistant message pair.
  3. Load session.model.
  4. SDK call (base_url → enowX proxy, model = session.model):
       max_tokens: 300
       prompt: "Extract 2-3 durable facts worth remembering.
                Return JSON: [{"key":"...","fact":"..."}]
                Skip: code snippets, temporary state, obvious facts."
  5. Upsert to memories table:
       scope="session", scope_key=sessionID
  6. Upsert one-line user summary:
       scope="user", scope_key="local", key="summary-{sessionID}"
  7. On error: log, discard — never blocks UI
```

### 13.5 Two-layer recall in system prompt

```
Section 4a: Curated memories (~200 tokens)
  "## What you remember
   - user prefers TypeScript strict mode
   - ChatSidebar uses GlideMenu, collapsed=52px
   - deploy target: :1431 dev, :1430 prod"
  Source: memories table (LLM-curated facts)

Section 4b: Relevant history (~800 tokens, only if FTS hits)
  "## Relevant past context
   [yesterday | ChatUI Sidebar] ..."
  Source: messages_fts (raw conversation chunks)
```

Total Section 4 budget: **~1000 tokens max**, hard-capped.

---

## 14. API Specification

All under `/chatui/api/`, bypass `dash.Require`. Response: `{"data": T}` / `{"error": "..."}`.

### Sessions

```
GET    /chatui/api/sessions               list (limit, offset)
POST   /chatui/api/sessions               create {title, model}
GET    /chatui/api/sessions/:id           get + messages
PATCH  /chatui/api/sessions/:id           update {title, model, archived, tools_enabled}
DELETE /chatui/api/sessions/:id           delete
POST   /chatui/api/sessions/:id/messages  append message
```

### Chat (agentic)

```
POST   /chatui/api/chat/stream            SSE agentic loop
  SSE event types:
    {"type": "text_delta",              "delta": "..."}
    {"type": "thinking_delta",          "delta": "..."}
    {"type": "tool_start",              "tool": "web_search", "args": {...}}
    {"type": "tool_result",             "tool": "web_search", "result": "..."}
    {"type": "tool_approval_required",  "tool": "terminal", "args": {...}, "approval_id": "..."}
    {"type": "compacted",               "removed": N, "summary_tokens": M}
    {"type": "done",                    "tokens_in": N, "tokens_out": N}
    {"type": "error",                   "message": "..."}

POST   /chatui/api/chat/approve/:approval_id   approve a pending tool call
POST   /chatui/api/chat/deny/:approval_id      deny a pending tool call
```

### Memory

```
POST   /chatui/api/sessions/:id/index     trigger memory indexing
GET    /chatui/api/sessions/:id/memories  list session memories
GET    /chatui/api/memories?q=...         FTS search all memories
POST   /chatui/api/memories               save a memory manually
DELETE /chatui/api/memories/:id           delete a memory
```

### MCP (Phase 3)

```
GET    /chatui/api/mcp/servers            list configured MCP servers
POST   /chatui/api/mcp/servers            add server
DELETE /chatui/api/mcp/servers/:id        remove server
POST   /chatui/api/mcp/servers/:id/test   test connection
```

### Skills (Phase 3)

```
GET    /chatui/api/skills                 list all skills (from DB)
PATCH  /chatui/api/skills/:name           toggle enabled {enabled: bool}
POST   /chatui/api/skills/reload          resync DB from .md files on disk
```

### Settings

```
GET    /chatui/api/settings               get all settings
  Response: {"data": {"tools_enabled": bool, "workspace_dir": string}}

PATCH  /chatui/api/settings               update settings
  Body: {"tools_enabled"?: bool, "workspace_dir"?: string}
  Response: {"data": {"tools_enabled": bool, "workspace_dir": string}}
```

---

## 15. Frontend Changes

### 15.1 `useChat.ts` — switch to agentic endpoint (Phase 2)

```typescript
// If global tools_enabled:
//   POST /chatui/api/chat/stream (new agentic SSE)
//   Handle SSE event types: text_delta, thinking_delta, tool_start, tool_result,
//                           tool_approval_required, compacted, done, error
// Else:
//   Continue using /anthropic/v1/messages directly (existing, untouched)
```

### 15.2 `useSessions.ts` — hybrid localStorage + API (Phase 1)

```typescript
// Write-through: API first, localStorage as fallback cache
// POST /chatui/api/sessions on create
// POST /chatui/api/sessions/:id/messages after each turn
```

### 15.3 Native vision — image attachments (Phase 2)

User can attach images via drag-drop, clipboard paste (Ctrl+V), or file picker.

```typescript
// ChatInput.tsx additions:
// - onPaste: detect image/* → read as base64, add to attachments[]
// - onDrop: same
// - file picker button: accept="image/png,image/jpeg,image/webp,image/gif"
// - attachment preview: thumbnails above input, × to remove
// - size limit: 5 MB per image, max 3 images per message (frontend enforced)

// Sent to /chatui/api/chat/stream:
{
  session_id: "...",
  message: "What's in this image?",
  attachments: [{ type: "image", media_type: "image/png", data_b64: "..." }]
}
```

Backend converts to `anthropic.ImageBlockParam` before SDK call. Images stored in
`messages.attachments` JSON column and re-sent on context reload for follow-up turns.

### 15.4 New components (Phase 2)

| Component | Purpose |
|---|---|
| `ToolCallBlock.tsx` | Show tool name + args + result inline in message |
| `ApprovalDialog.tsx` | Approval gate UI for terminal + write_file |
| `ImageAttachment.tsx` | Thumbnail preview + remove button in input area |
| `CompactedBanner.tsx` | Subtle "Earlier messages summarised" indicator |
| `MemoryPanel.tsx` | Sidebar panel showing session memories |

---

## 16. Phased Rollout

### Phase 1 — Persistent sessions
- `store/chatui/` package (types, migrations, SQLite)
- Session + message REST API (6 endpoints)
- Frontend: `useSessions.ts` + `useChat.ts` hybrid API/localStorage
- **Result:** history survives browser clear

### Phase 2 — Memory + Tools + Vision + Compaction
- Anthropic SDK (`github.com/anthropics/anthropic-sdk-go`)
- `core/chatui/` package: agent loop, 8 tools, plugins, memory indexer
- **Token counting** — pre-flight check before every SDK call
- **Silent compaction** — auto-summarise when context hits 85% of limit
- **Native vision** — image attachments via drag-drop/paste
- New `/chatui/api/chat/stream` agentic endpoint
- Tool UI: `ToolCallBlock`, `ApprovalDialog`, `ImageAttachment`, `CompactedBanner`
- Memory indexer (background goroutine) + `MemoryPanel` UI
- **Result:** Claude can use tools + remember things across sessions

### Phase 3 — MCP + Skills
- MCP client (stdio transport, 6 servers)
- Skills system (6 built-in skills, user-configurable)
- System prompt: skills block + memory recall
- MCP + skills management APIs
- **Result:** extensible via MCP, guided by skills

### Phase 4 (future, separate spec)
- Semantic memory (RAG/vector search)
- User-installable MCP servers from hub
- Soul/persona system
- Multi-user identity

---

## 17. Security & Safety

| Concern | Mitigation |
|---|---|
| `terminal` tool | Approval gate: user must confirm before execution. 30s timeout. 3000 token output cap. |
| `write_file` tool | Approval gate. Path scoped to workspace directory only (no `../` traversal). |
| `web_fetch` untrusted output | Output wrapped in context marker, size-capped at 15K chars. |
| chatui.db location | `$ENOWX_RUNTIME_DIR/chatui.db` — never /tmp |
| API endpoints bypass dash.Require | Same as `/chatui/api/config` today. Local machine only. |
| Memory indexer LLM calls | Via SDK → enowX proxy → account pool. Rate-limited 1/session/30s. |
| enowX proxy untouched | All new code in `core/chatui/` and `store/chatui/`. Zero edits to existing handlers. |
| Port 1430 | Never touched. All dev on :1431. |

---

## 18. Risks & Mitigations (from Antares audit)

| Antares Risk | Our Mitigation |
|---|---|
| No migration version table → re-runs all DDL on startup | `schema_version` KV key → only new migrations run |
| Background indexer + HTTP handler write contention on SQLite | WAL mode + serialized writes via channel/mutex; no silent error discard |
| PromptBlock token bomb: counts skills not chars | Char budget guard: skills block hard-capped at 500 tokens |
| RAG full table scan O(n) | Not in Phase 1-3; Phase 4 uses proper index |
| `delegate_task` sub-agent cascade | Not implementing sub-agents in ChatUI |
| `browser` tool token bomb (3000-8000 tokens/snapshot) | Not implementing browser tool |
| MCP tool output unbounded | All MCP tool results go through same `MaxToolOutputTokens` truncation as built-in tools |

---

## 19. New Files in enowX

```
core/chatui/
  agent.go        — ChatLoop: agentic turn loop (SDK + tool dispatch)
  tools.go        — 8 built-in tool implementations
  tools_web.go    — web_search + web_fetch
  tools_file.go   — read_file + write_file + list_files + grep
  tools_term.go   — terminal (exec.CommandContext, 30s timeout)
  tools_memory.go — memory tool (read/write memories table)
  recall.go       — FTS-based cross-session history recall (§13)
  mcp.go          — MCP client (Phase 3)
  skills.go       — skill loader + PromptBlock (Phase 3)
  plugins.go      — plugin middleware chain
  memory.go       — background memory indexer goroutine
  prompt.go       — 5-section system prompt builder (frozen snapshot pattern)

store/chatui/
  store.go        — ChatStore interface
  sqlite.go       — modernc SQLite implementation
  migrations.go   — embedded DDL
  types.go        — Session, Message, Memory types

server/handlers/
  chatui_sessions.go — session CRUD + message append
  chatui_chat.go     — POST /chatui/api/chat/stream (SSE)
  chatui_memory.go   — memory read/write/search
  chatui_settings.go — GET/PATCH /chatui/api/settings (tools_enabled, workspace_dir)
  chatui_mcp.go      — MCP server management (Phase 3)
  chatui_skills.go   — skill enable/disable + reload (Phase 3)

web/src/chatui/
  ToolCallBlock.tsx  — tool call display in message thread
  ApprovalDialog.tsx — approval gate UI
  MemoryPanel.tsx    — session memory sidebar panel (Phase 2)
  useChat.ts         — MODIFIED: handle agentic SSE events
  useSessions.ts     — MODIFIED: hybrid API/localStorage
```

### NOT modified

```
/usr/local/bin/enx          — production binary
Antares codebase            — zero changes, zero dependency
enowX core/                 — proxy, pool, model routing — untouched
enowX store/sqlite/         — existing enowx.db — untouched
server/handlers/v1.go       — Anthropic proxy handler — untouched
server/handlers/chatui.go   — existing config handler — untouched
Any other existing file     — untouched
```

---

## 20. Decisions (Resolved)

| # | Question | Decision |
|---|---|---|
| 1 | tools_enabled per-session or global? | **Global** — one toggle for all sessions, stored in `kv` table as `chatui.tools_enabled` |
| 2 | Terminal workspace scoping? | **User-configurable** — stored in `kv` as `chatui.workspace_dir`, default `$HOME`. Shown in ChatUI settings. |
| 3 | Memory indexing model? | **Follows session model** — whatever model the session was using, same model for indexing that turn |
| 4 | MCP transport — stdio only or HTTP also? | **Both** — stdio for local subprocesses, HTTP/SSE for remote servers. Both in Phase 3. |
| 5 | Skills storage — files only or also DB? | **Both** — `.md` files in `$ENOWX_RUNTIME_DIR/chatui/skills/` for content, `skills` table in `chatui.db` for metadata + enable/disable state |

### Implications of decisions

**Decision 1 (global tools_enabled):**
- Remove `tools_enabled` column from `sessions` table
- Add `kv` key `chatui.tools_enabled = "true"|"false"` (default `"false"`)
- Settings UI shows one global toggle: "Enable tool use"
- All new sessions inherit the global setting

**Decision 2 (user-configurable workspace):**
- `kv` key `chatui.workspace_dir` — default `$HOME`
- `read_file`, `write_file`, `list_files`, `grep` are scoped to this dir (no `../` traversal)
- `terminal` CWD is set to this dir; still requires approval gate
- Configurable via `/chatui/api/settings` endpoint (see §14 addition below)

**Decision 3 (memory indexing follows session model):**
- Memory indexer receives `model` from the session record in chatui.db
- Uses that same model for the summarisation call via SDK → enowX proxy
- No hardcoded model name — whatever model was active when the turn ended

**Decision 4 (MCP: both stdio and HTTP):**
- `transport` field in MCP server config: `"stdio"` or `"http"`
- stdio: `{"transport": "stdio", "command": "npx", "args": [...]}`
- HTTP: `{"transport": "http", "url": "http://localhost:3000/mcp"}`
- Both transports share the same JSON-RPC 2.0 + tools/list handshake
- Both land tools in registry as `mcp__<serverId>__<toolName>`

**Decision 5 (skills: files + DB table):**
- New `skills` table in `chatui.db` (see updated §11 below)
- `.md` files are the source of truth for content
- DB table stores: name, enabled, category, triggers (denormalized for fast query)
- On startup: scan skill files → upsert `skills` table (sync)
- User edits `.md` file → restart or POST `/chatui/api/skills/reload` to resync

# enowX ChatUI Memory System — Design Spec

**Date:** 2026-09-02  
**Status:** DRAFT — awaiting owner review before implementation  
**Author:** Claude (audit + design)

---

## 1. Goal

Give enowX ChatUI persistent, searchable chat history backed by a dedicated
SQLite store (`chatui.db`), plus a memory layer that summarises conversations
into durable facts — so a future memory system can read all history by session
or by user.

The ChatUI today stores everything in `localStorage`. This spec migrates that to
a proper DB layer and adds REST endpoints to enowX that the ChatUI frontend calls.

---

## 2. What This Spec Is (and Is Not)

### This spec is:
- A persistent session + message store for ChatUI, backed by SQLite
- A memory indexing pipeline that uses the **Anthropic SDK** to summarise turns
- A set of REST endpoints added to enowX Go server for ChatUI to call
- A reference to Antares architecture patterns (not a port of Antares code)

### This spec is NOT:
- A port of Antares code into enowX
- A change to enowX's core proxy, SSE pipeline, account pool, or model routing
- An import of `github.com/enowdev/antares` as a dependency
- Anything that touches port 1430 (production)

**Hard rule:** enowX proxy layer (`/anthropic/v1/messages`, `/v1/messages`, account pool,
model routing, SSE pipeline) is **never touched**. It runs as-is on `:1430` and `:1431`.

---

## 3. Architecture

### 3.1 The two layers

```
┌─────────────────────────────────────────────────────────────────┐
│  ChatUI Backend (new code in enowX Go server)                   │
│                                                                  │
│  Memory indexing uses Anthropic SDK:                            │
│                                                                  │
│    anthropic.NewClient(                                         │
│      option.WithAPIKey("enx-..."),                              │
│      option.WithBaseURL("http://localhost:<ENOWX_PORT>"),       │
│    )                                                            │
│                                                                  │
│  SDK calls /anthropic/v1/messages on enowX.                     │
│  SDK never knows about account pool.                            │
│  enowX pool handles routing, retry, usage tracking.             │
│                                                                  │
└────────────────────────┬────────────────────────────────────────┘
                         │ POST /anthropic/v1/messages
                         │ (Anthropic wire format)
┌────────────────────────▼────────────────────────────────────────┐
│  enowX proxy (UNTOUCHED)                                        │
│  Account pool, model routing, retry, SSE pipeline               │
│  Handles /anthropic/v1/* and /v1/* as-is today                  │
└─────────────────────────────────────────────────────────────────┘
```

The SDK is configured with `base_url` pointing at enowX itself — so all LLM calls
from the memory indexer go through the same account pool as chat, with zero changes
to the proxy layer.

### 3.2 Full system diagram

```
┌──────────────────────────────────────────────────────────────┐
│  Browser                                                      │
│  enowX ChatUI (React SPA at /chatui/)                        │
│    useSessions  ←─ REST ─→  /chatui/api/sessions/*           │
│    useChat      ←─ SSE  ─→  /anthropic/v1/messages           │
└──────────────────────────────────────────────────────────────┘
         │ HTTP (same origin)
┌────────▼─────────────────────────────────────────────────────┐
│  enowX Go server (:1430 / :1431)                             │
│                                                               │
│  NEW: server/handlers/chatui_sessions.go                     │
│    GET    /chatui/api/sessions          list                  │
│    POST   /chatui/api/sessions          create                │
│    GET    /chatui/api/sessions/:id      get + messages        │
│    PATCH  /chatui/api/sessions/:id      update title/model   │
│    DELETE /chatui/api/sessions/:id      delete               │
│    POST   /chatui/api/sessions/:id/messages  append msg      │
│                                                               │
│  NEW: server/handlers/chatui_memory.go                       │
│    POST   /chatui/api/sessions/:id/index     memory indexing │
│    GET    /chatui/api/sessions/:id/memories  session memories│
│    GET    /chatui/api/memories?q=...         search all      │
│                                                               │
│  EXISTING: /chatui/api/config           key + models         │
│  EXISTING: /anthropic/v1/messages       LLM proxy (untouched)│
│                                                               │
│  NEW: store/chatui/                                          │
│    store.go          — Store interface                        │
│    sqlite.go         — modernc SQLite implementation          │
│    migrations.go     — embedded DDL                           │
│    types.go          — Session, Message, Memory types         │
└──────────────────────────────────────────────────────────────┘
         │ SQLite (WAL mode)
┌────────▼─────────────────────────────────────────────────────┐
│  chatui.db  (separate file — NOT enowX's enowx.db)           │
│  Default: ~/.enowx/chatui.db  (same dir as enowx.db)         │
│  Dev:     ~/.enowx-dev/chatui.db                             │
│                                                               │
│  Tables (own DDL, inspired by Antares schema design):        │
│    sessions       (id, title, model, message_count, ...)     │
│    messages       (id, session_id, seq, role, content, ...)  │
│    memories       (id, scope, scope_key, key, content, ...)  │
│    kv             (kv_key, kv_value)                         │
└──────────────────────────────────────────────────────────────┘
```

---

## 4. Why Separate `chatui.db`

1. **No migration coupling** — enowX DB (`enowx.db`) uses numbered migrations managed
   by enowX. ChatUI has different schema needs and different migration cadence.
2. **No Antares dependency** — we write our own DDL inspired by Antares patterns
   (same table shapes, same field names where it makes sense), but we don't import
   any Antares package.
3. **Independent lifecycle** — ChatUI DB can be wiped, backed up, or migrated
   without touching the core enowX DB.
4. **Single file, one concern** — all ChatUI state (sessions, messages, memories) in
   one place, easy to reason about.

---

## 5. Antares as Architecture Reference

We studied Antares's implementation across these domains and extracted the patterns
we want to replicate — without importing the code:

| Domain | Antares pattern we reference | What we do in enowX ChatUI |
|---|---|---|
| **Store** | `sessions` + `messages` tables, dual-dialect (SQLite/Postgres) | Own SQLite-only DDL with same field shapes |
| **Memory** | `memories` table: scope/scope_key/key/content, FTS | Same table design, own migration |
| **Memory indexing** | `indexUserTurn()` → LLM summarise → write `memories` rows | Same pipeline, but uses Anthropic SDK via enowX proxy |
| **RAG** | `rag_chunks` + `UserCollection(platform, userID)` | Deferred to Phase 3; same collection naming concept |
| **Migration** | Re-runs all DDL on startup (Antares has no version table) | We add a `schema_version` KV key — run only new migrations |
| **Skills injection** | Loads skill text from `skills` table, injects into system prompt | Phase 2+; concept borrowed, implementation simpler |

### What we do NOT port from Antares:
- MCP protocol implementation
- Plugin sidecar lifecycle
- Role/soul system (24 roles)
- Agent loop (`ragcontext.go` full implementation)
- Roles, pairings, social accounts
- Antares server/HTTP layer
- Postgres support (SQLite only for ChatUI)

---

## 6. Anthropic SDK Integration

### Why SDK (not hand-rolled HTTP)?

| | Hand-rolled HTTP | Anthropic SDK |
|---|---|---|
| SSE streaming | Write `bufio.Scanner` + `data:` parser | Built-in, edge cases handled |
| Tool use | Manual JSON marshal/unmarshal | Typed structs |
| Extended thinking | Manual `thinking` block parse | First-class field |
| Error types | `if status == 429` | `anthropic.RateLimitError` |
| Retry | Implement from scratch | Built-in exponential backoff |
| Future Claude versions | Re-implement for each | Auto-supported |

### How it routes through enowX without touching the pool

```go
// store/chatui/indexer.go

import anthropic "github.com/anthropics/anthropic-sdk-go"
import "github.com/anthropics/anthropic-sdk-go/option"

func newIndexerClient(enowxBaseURL string) *anthropic.Client {
    return anthropic.NewClient(
        option.WithAPIKey("dummy"),           // enowX ignores the key for local calls
        option.WithBaseURL(enowxBaseURL),     // → http://localhost:1431 (or 1430 in prod)
    )
}

// SDK sends: POST http://localhost:1431/anthropic/v1/messages
// enowX proxy receives it as a normal Anthropic-format request
// Routes through account pool, picks a real API key, forwards upstream
// Streams response back to SDK
// SDK never knows about the pool
```

**Critical:** `enowxBaseURL` comes from config (`ENOWX_PORT` env var), not hardcoded.
In production this points at `:1430`; in dev at `:1431`.

### What the SDK is used for (Phase 2 only)

Memory indexing only — not for the main chat flow. The main chat in ChatUI already
works via the existing `/anthropic/v1/messages` SSE endpoint. The SDK is used by
the background memory indexer goroutine to summarise conversation turns.

---

## 7. Database Schema

### 7.1 `sessions` table

```sql
CREATE TABLE IF NOT EXISTS sessions (
    id          TEXT PRIMARY KEY,
    title       TEXT NOT NULL DEFAULT 'New Chat',
    model       TEXT NOT NULL DEFAULT 'bob/premium',
    message_count INTEGER NOT NULL DEFAULT 0,
    tokens_in   INTEGER NOT NULL DEFAULT 0,
    tokens_out  INTEGER NOT NULL DEFAULT 0,
    archived    BOOLEAN NOT NULL DEFAULT 0,
    created_at  DATETIME NOT NULL DEFAULT (datetime('now')),
    updated_at  DATETIME NOT NULL DEFAULT (datetime('now'))
);
CREATE INDEX IF NOT EXISTS idx_sessions_updated ON sessions(updated_at DESC);
```

### 7.2 `messages` table

```sql
CREATE TABLE IF NOT EXISTS messages (
    id          TEXT PRIMARY KEY,
    session_id  TEXT NOT NULL REFERENCES sessions(id) ON DELETE CASCADE,
    seq         INTEGER NOT NULL,
    role        TEXT NOT NULL CHECK(role IN ('user','assistant','system')),
    content     TEXT NOT NULL DEFAULT '',
    reasoning   TEXT,
    model       TEXT,
    tokens_in   INTEGER NOT NULL DEFAULT 0,
    tokens_out  INTEGER NOT NULL DEFAULT 0,
    created_at  DATETIME NOT NULL DEFAULT (datetime('now')),
    UNIQUE(session_id, seq)
);
CREATE INDEX IF NOT EXISTS idx_messages_session ON messages(session_id, seq);
```

### 7.3 `memories` table

```sql
CREATE TABLE IF NOT EXISTS memories (
    id          TEXT PRIMARY KEY,
    scope       TEXT NOT NULL CHECK(scope IN ('session','user','global')),
    scope_key   TEXT NOT NULL,
    key         TEXT NOT NULL,
    content     TEXT NOT NULL,
    source      TEXT NOT NULL DEFAULT 'chatui',
    created_at  DATETIME NOT NULL DEFAULT (datetime('now')),
    updated_at  DATETIME NOT NULL DEFAULT (datetime('now')),
    UNIQUE(scope, scope_key, key)
);
CREATE INDEX IF NOT EXISTS idx_memories_scope ON memories(scope, scope_key);

-- FTS for memory search
CREATE VIRTUAL TABLE IF NOT EXISTS memories_fts USING fts5(
    key, content,
    content='memories', content_rowid='rowid'
);
```

### 7.4 `kv` table (settings + migration version)

```sql
CREATE TABLE IF NOT EXISTS kv (
    kv_key   TEXT PRIMARY KEY,
    kv_value TEXT NOT NULL
);
-- schema_version key: "1", "2", etc. — incremented after each migration batch
-- chatui.active_session_id: last active session
```

### 7.5 Migration strategy

Unlike Antares (which has no version table and re-runs all DDL every startup),
ChatUI DB uses a version key:

```go
// On startup:
version := kv.Get("schema_version") // "" if fresh
if version < currentVersion {
    runMigrationsFrom(version+1, currentVersion)
    kv.Set("schema_version", currentVersion)
}
```

This avoids the re-run-all-DDL risk and makes migrations additive and safe.

---

## 8. API Specification

All endpoints are under `/chatui/api/` and bypass `dash.Require` (same as existing
`/chatui/api/config`). All responses are `{"data": T}` on success, `{"error": "..."}` on failure.

### 8.1 Sessions

```
GET /chatui/api/sessions
  Query: limit (default 50), offset (default 0)
  Response: {"data": {"sessions": Session[], "total": int}}

POST /chatui/api/sessions
  Body: {"title": string, "model": string}
  Response: {"data": Session}

GET /chatui/api/sessions/:id
  Response: {"data": {"session": Session, "messages": Message[]}}

PATCH /chatui/api/sessions/:id
  Body: {"title"?: string, "model"?: string, "archived"?: bool}
  Response: {"data": Session}

DELETE /chatui/api/sessions/:id
  Response: {"data": {"ok": true}}
```

### 8.2 Messages

```
POST /chatui/api/sessions/:id/messages
  Body: {"role": "user"|"assistant", "content": string, "reasoning"?: string,
         "model"?: string, "tokens_in"?: int, "tokens_out"?: int}
  Response: {"data": Message}
  Side-effect: increments sessions.message_count, tokens_in, tokens_out
```

### 8.3 Memory indexing (Phase 2)

```
POST /chatui/api/sessions/:id/index
  Body: {} (empty — reads last N messages from DB)
  Response: {"data": {"memories_written": int}}
  Background: calls Anthropic SDK → enowX proxy → summarise → writes memories
  Rate-limited: max 1 per session per 30s
```

### 8.4 Memory read (Phase 2)

```
GET /chatui/api/sessions/:id/memories
  Response: {"data": Memory[]}

GET /chatui/api/memories?q=...&limit=20
  Response: {"data": Memory[]}   (FTS search across all chatui memories)
```

---

## 9. Frontend Changes

### 9.1 `useSessions.ts` — hybrid localStorage + API

```typescript
// Phase 1 strategy: write-through
// - All session/message mutations call the API first
// - localStorage is a read-cache (populated from API on mount)
// - On API failure, fall back to localStorage-only (offline mode)

// "chatui-api-available" in sessionStorage: "1" after first successful API call

async function create(model: string): Promise<ChatSession> {
  // POST /chatui/api/sessions → get ID from server
  // mirror to localStorage
}

async function appendMessage(sessionId: string, msg: ChatMsg): Promise<void> {
  // POST /chatui/api/sessions/:id/messages
  // update local state
}
```

### 9.2 `useChat.ts` — append messages after each turn

```typescript
// After turn completes (user message sent + assistant response received):
// 1. await sessionsApi.appendMessage(sessionId, userMsg)
// 2. await sessionsApi.appendMessage(sessionId, assistantMsg)
// 3. [Phase 2] fire-and-forget: POST /chatui/api/sessions/:id/index
```

---

## 10. Memory Indexing Pipeline (Phase 2)

Inspired by Antares `indexUserTurn()` / `summariseUserTurn()` in `ragcontext.go` —
same concept, simpler implementation using Anthropic SDK:

```
After turn completes (fire-and-forget goroutine):
  1. Load last user + assistant message pair from chatui.db
  2. Anthropic SDK client (base_url → enowX proxy):
     model: cheapest/fastest available (e.g. claude-haiku-4)
     prompt: "Extract 3-5 durable facts from this conversation exchange.
              Return JSON: [{"key": "...", "fact": "..."}]"
  3. For each extracted fact → write to memories table:
       scope = "session", scope_key = sessionID
       key   = extracted key
       content = extracted fact
       source = "chatui-auto"
  4. Also write a user-level summary:
       scope = "user", scope_key = "local"
       key   = "summary-<sessionID>"
       content = one-line summary of the session topic
```

### Why this is safe:
- SDK calls enowX proxy → goes through normal account pool → no direct API key exposure
- Fire-and-forget goroutine with rate limit (1/session/30s) — never blocks the UI
- On indexer error: log and discard, not crash (memories are a nice-to-have, not critical)
- SDK's built-in retry handles transient errors without extra code

---

## 11. Security & Safety

| Concern | Mitigation |
|---|---|
| chatui.db is a new file | Stored at `$ENOWX_RUNTIME_DIR/chatui.db` (default `~/.enowx/chatui.db`). Same dir, never /tmp. |
| API endpoints bypass dash.Require | Same as existing `/chatui/api/config`. Only local machine. Remote access requires enowX auth. |
| Memory indexer calls LLM | Via SDK → enowX proxy (same account pool as chat). Rate-limited. Never blocks UI. |
| No user auth in Phase 1 | Single-owner device. `user_id = "local"` throughout. Multi-user is a separate concern. |
| enowX proxy untouched | Indexer is a new Go file in `store/chatui/`. Zero modifications to any existing handler. |
| Port 1430 never touched | All dev work on :1431. Production binary untouched. |

---

## 12. Phased Rollout

### Phase 1 — Persistent sessions (~8 tasks)
- `store/chatui/` package: types, migrations, SQLite implementation
- 5 session API endpoints + message append endpoint
- Frontend: hybrid localStorage+API in `useSessions.ts` + `useChat.ts`
- **Result:** history survives browser clear, queryable by session ID from backend

### Phase 2 — Memory indexing (~4 tasks, after Phase 1 ships)
- Anthropic SDK dependency added (`github.com/anthropics/anthropic-sdk-go`)
- Memory indexing endpoint + background goroutine + SDK summarisation
- Memory read endpoints
- **Result:** session memories stored, searchable, foundation for memory system

### Phase 3 — Semantic memory recall (future, separate spec)
- Index memories into a vector store (concept from Antares `rag_chunks`)
- Add embedding search endpoint
- **Depends on:** choosing an embedding model available via enowX pool

---

## 13. New Files in enowX

```
store/chatui/
  store.go          — Store interface (ChatStore)
  sqlite.go         — modernc SQLite implementation
  migrations.go     — embedded DDL (own DDL, Antares-inspired design)
  types.go          — Session, Message, Memory types

server/handlers/
  chatui_sessions.go  — HTTP handlers for /chatui/api/sessions/*
  chatui_memory.go    — HTTP handlers for /chatui/api/memories (Phase 2)

server/server.go    — wire new handlers into /chatui/api/ block (outside dash.Require)
cmd/enowx/main.go   — open chatui.db, pass ChatStore to new handlers
```

### Modified files

```
server/server.go       — add routes
cmd/enowx/main.go      — open chatui.db on startup
web/src/chatui/useSessions.ts   — hybrid API + localStorage
web/src/chatui/useChat.ts       — append messages after turn
```

### NOT modified (hard constraints)

```
/usr/local/bin/enx           — production binary, never touch
:1430                        — production server, never touch
Antares codebase             — zero changes, zero dependency
enowX core/ proxy handlers   — proxy, pool, model routing — untouched
enowX store/sqlite/          — existing enowX DB — untouched
Any existing enowX handler   — no modifications
```

---

## 14. Risks (from audit)

These are known risks in Antares's implementation that we deliberately avoid:

| Antares Risk | Our Mitigation |
|---|---|
| No migration version table — re-runs all DDL every startup | We have `schema_version` KV key — only run new migrations |
| SQLite write contention: background indexer vs HTTP handler, errors silently discarded | Use WAL mode; indexer runs in own goroutine with error logging; no silent discards |
| PromptBlock token bomb: counts by skill count, not chars | Not implementing prompt injection in Phase 1-2; Phase 3 will have char budget guard |
| RAG full table scan O(n) | Phase 3 concern; will use proper index or defer |

---

## 15. Open Questions for Owner

1. **Memory indexing model:** Which model for summarisation? Proposal: use `claude-haiku-4` (cheapest/fastest) always for indexing, regardless of what model the session uses. This keeps costs low and speed high for background work.

2. **User identity for Phase 3:** Phase 1-2 uses `user_id = "local"`. If you plan to link ChatUI history to Discord/Telegram gateway users later, the `user_id` field is the right extension point — add it now before schema is set?

3. **Skill/MCP integration timing:** Phase 2 gives us memories. Phase 3 would add semantic search. Do you want to add skills injection to the memory indexer system prompt in Phase 2, or keep it simpler and add in Phase 3?

4. **Session URL routing:** Currently sessions are not exposed as `/chatui/session/:id` URLs. With persistent sessions, this becomes possible. Do you want URL routing per session in Phase 1 (so refresh keeps you in the same session), or defer to later?

5. **chatui.db backup:** Since this will store valuable history, do you want a simple periodic backup mechanism (copy to `chatui.db.bak` on startup) in Phase 1?

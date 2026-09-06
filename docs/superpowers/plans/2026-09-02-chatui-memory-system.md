# ChatUI Memory System — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the complete ChatUI memory + agentic tools system on top of enowX — persistent sessions, memory, 8 built-in tools, MCP client, skills, plugin middleware, token counting, vision, and silent compaction.

**Architecture:** ChatUI Agentic Layer → Anthropic SDK (base_url = enowX proxy) → existing enowX proxy (untouched). All new code in `core/chatui/`, `store/chatui/`, `server/handlers/chatui_*.go`. Zero changes to existing proxy, pool, SSE pipeline.

**Tech Stack:** Go 1.26, `github.com/anthropics/anthropic-sdk-go`, `modernc.org/sqlite`, chi v5, React/TypeScript (Vite), FTS5

**Spec:** `/home/arch/workspace/antares/docs/superpowers/specs/2026-09-02-chatui-memory-system-design.md`

**References:**
- `/home/arch/workspace/antares/docs/superpowers/references/antares-patterns.md`
- `/home/arch/workspace/antares/docs/superpowers/references/hermes-memory.md`

---

## Audit Findings Applied (Pre-Implementation Fixes)

These findings were confirmed from the spec before implementation. Each fix is incorporated into the task that creates the relevant code.

### AUD-01 CRITICAL — Compaction threshold inconsistency
**Evidence:** §10.2 loop says "85%" trigger; §10.3 limits table says "90%". 
**Fix:** Use **85%** as the compaction trigger threshold (conservative, consistent with §10.4 description). Update spec §10.3 table to match.

### AUD-02 HIGH — Compaction delete vs flag contradiction  
**Evidence:** §10.4 says "delete old rows from DB"; §11.2 messages table has `compacted BOOLEAN` flag.
**Fix:** Keep rows, set `compacted=TRUE` on summarized rows, set content=summary on the first row, clear content on remaining rows. FTS triggers then fire DELETE→reindex on the updated rows. This satisfies: (a) rows stay in DB for audit trail, (b) FTS delete trigger fires correctly, (c) §13.3 `compacted=0` filter in recall is meaningful.

### AUD-03 HIGH — FTS backfill missing on Phase 2 migration
**Evidence:** Phase 1 creates `messages` table. Phase 2 migration adds FTS virtual table. New trigger only fires on future INSERTs — existing rows are not indexed.
**Fix:** Migration 002 (FTS creation) must include explicit backfill:
```sql
INSERT INTO messages_fts(rowid, content)
SELECT rowid, content FROM messages WHERE compacted=0;
```
Note: After AUD-02 fix, only `content` is indexed in FTS (not `reasoning` — see AUD-09).

### AUD-04 HIGH — CountTokens endpoint not proxied  
**Evidence:** `server/server.go` only registers `POST /anthropic/v1/messages` for the Anthropic handler. The SDK's `Messages.CountTokens()` calls `POST /anthropic/v1/messages/count_tokens` → 404 from enowX.
**Fix:** Add a `count_tokens` route in `server/server.go` pointing to a new `anthropic.CountTokens` handler that proxies `POST /anthropic/v1/messages/count_tokens` to the upstream provider. This handler follows the same pattern as `anthropic.Messages` but:
- Reads the request body as-is (count_tokens request format)
- Forwards to upstream provider's count_tokens endpoint (or a provider-specific equivalent)
- Returns the response JSON directly (no SSE)
- Uses the same account pool routing as Messages

**Alternative if provider doesn't support count_tokens:** Implement local token counting via `core/tokenize` package (which already exists in enowX). Fall back to local counting if proxy returns 404/501.

### AUD-05 HIGH — API contradiction: tools_enabled in session PATCH
**Evidence:** §14 API spec shows `PATCH /chatui/api/sessions/:id` body includes `tools_enabled`. §20 Decision 1 says "tools_enabled is global — remove from sessions".
**Fix:** Remove `tools_enabled` from the session PATCH body. Only `{title, model, archived}` allowed. Global toggle is only via `PATCH /chatui/api/settings`.

### AUD-06 MEDIUM — seq allocation not atomic
**Evidence:** §11.2 has `UNIQUE(session_id, seq)`. No transaction defined for seq allocation. Two concurrent requests could both `SELECT MAX(seq)+1 = 10` and collision on UNIQUE constraint.
**Fix:** Allocate seq atomically inside a transaction:
```sql
BEGIN IMMEDIATE;
SELECT COALESCE(MAX(seq), 0) + 1 FROM messages WHERE session_id = ?;
INSERT INTO messages (id, session_id, seq, ...) VALUES (?, ?, ?, ...);
COMMIT;
```
Or use SQLite's AUTOINCREMENT-style seq within session: seq = COALESCE(MAX(seq),0)+1 inside the INSERT transaction.

### AUD-07 MEDIUM — Counter drift: session counters not in same transaction
**Evidence:** §10.2 step 7 says "persistMessages" and "fire-and-forget indexMemory" as separate steps. No transaction defined that covers both `INSERT INTO messages` AND `UPDATE sessions SET message_count=...`.
**Fix:** Wrap in single transaction:
```sql
BEGIN IMMEDIATE;
INSERT INTO messages (...) VALUES (...);
UPDATE sessions SET message_count=message_count+1, tokens_in=tokens_in+?, tokens_out=tokens_out+?, updated_at=datetime('now') WHERE id=?;
COMMIT;
```

### AUD-08 MEDIUM — reasoning should NOT be indexed in FTS
**Evidence:** §11.2 and §13.2 both index `reasoning` in `messages_fts`. Reasoning = internal chain-of-thought. When recalled into Section 4b "relevant past context", model's internal thinking appears as factual content. Misleading, wastes token budget.
**Fix:** Remove `reasoning` from `messages_fts`. Index `content` only (user messages + assistant final text).
FTS schema becomes:
```sql
CREATE VIRTUAL TABLE IF NOT EXISTS messages_fts USING fts5(
    content,
    content='messages',
    content_rowid='rowid',
    tokenize='porter unicode61'
);
```
Triggers updated accordingly (no `reasoning` column).

### AUD-09 MEDIUM — Approval state machine undefined
**Evidence:** §14 defines approve/deny endpoints but no state transitions, timeout, or SSE-disconnect behavior.
**Fix (minimal):** Define in spec:
- States: PENDING → APPROVED | DENIED | EXPIRED | CANCELLED
- In-memory map: `approvalID → chan ApprovalResult` per active agent loop
- Timeout: 5 minutes (approval expires, treated as DENIED)
- SSE disconnect: if SSE client disconnects, pending approvals are CANCELLED (loop context cancelled)
- Duplicate approve/deny: first write wins; subsequent calls return 409 Conflict
- No DB persistence needed for Phase 2 (approvals are ephemeral — they live only during an active SSE stream)

### AUD-10 MEDIUM — tool message representation undefined
**Evidence:** messages table has `role='tool'` as a valid role AND `tool_results TEXT` column. Anthropic API uses: role=assistant with tool_use blocks, then role=user with tool_result blocks.
**Fix:** Define canonical storage:
- Tool calls from Claude: stored as `role='assistant'`, `tool_calls=JSON` (the tool_use blocks)
- Tool results back to Claude: stored as `role='user'` (per Anthropic API), `tool_results=JSON`
- `role='tool'` and `role='system'` remain valid in the CHECK constraint for potential future use
- Wire format reconstruction: `buildMessages()` function reconstructs Anthropic API messages from DB rows following this canonical form

### AUD-11 LOW — Context limit fallback 100K
**Evidence:** §10.5 says "fallback 100K if unknown". For small models (32K, 8K), this causes provider rejection.
**Fix:** Change fallback to **32K** (conservative — works for all known models in enowX's provider pool). Store per-model limits in `kv` table as `model.context.<modelID>`. Default 32K.

### AUD-12 LOW — Memory indexer rate-limit key
**Evidence:** §13.4 says "rate-limit: last indexed < 30s ago? skip" — no key defined.
**Fix:** Rate limit is per-session. Key: `session:<sessionID>:last_indexed` stored in memory (not DB) — resets on server restart. This is acceptable since the indexer is best-effort.

---

## Global Constraints

- NEVER touch: `/anthropic/v1/*`, `/v1/*`, account pool, model routing, SSE pipeline, port 1430, `/usr/local/bin/enx`, existing `enowx.db`, Antares source
- All new code in: `core/chatui/`, `store/chatui/`, `server/handlers/chatui_*.go`
- DB file: `$ENOWX_RUNTIME_DIR/chatui.db` (separate from `enowx.db`)
- Go module: `github.com/enowdev/enowx`
- SQLite driver: `modernc.org/sqlite` (already in go.mod, CGO_ENABLED=0)
- Compaction threshold: 85% (AUD-01)
- Compaction behavior: flag rows, don't delete (AUD-02)
- FTS indexes `content` only, not `reasoning` (AUD-08)
- Session PATCH: no `tools_enabled` field (AUD-05)
- seq allocation: atomic transaction (AUD-06)
- session counters: same transaction as message insert (AUD-07)
- Context limit fallback: 32K (AUD-11)

---

## File Structure

```
store/chatui/
  store.go        — ChatStore interface
  sqlite.go       — modernc SQLite implementation + Open() + migrate()
  migrations.go   — allMigrations []string (embedded DDL)
  types.go        — Session, Message, Memory types

core/chatui/
  agent.go        — ChatLoop: agentic turn loop (SDK + tool dispatch)
  tools.go        — Tool interface + registry
  tools_web.go    — web_search + web_fetch implementations
  tools_file.go   — read_file + write_file + list_files + grep
  tools_term.go   — terminal tool
  tools_memory.go — memory tool
  recall.go       — FTS-based cross-session history recall
  plugins.go      — plugin middleware chain
  memory.go       — background memory indexer goroutine
  prompt.go       — 5-section system prompt builder

server/handlers/
  chatui_sessions.go — Session CRUD + message append
  chatui_chat.go     — POST /chatui/api/chat/stream (SSE agentic loop)
  chatui_memory.go   — memory read/write/search
  chatui_settings.go — GET/PATCH /chatui/api/settings
  chatui_wire.go     — wire up all chatui handlers in server.New()

web/src/chatui/ (Phase 1-2 frontend)
  useSessions.ts     — hybrid API/localStorage
  useChat.ts         — MODIFIED: handle agentic SSE (Phase 2)
  ToolCallBlock.tsx  — tool call display
  ApprovalDialog.tsx — approval gate UI
```

---

## Task 1 — store/chatui: types, migrations, SQLite store

**Files:**
- Create: `store/chatui/types.go`
- Create: `store/chatui/store.go`
- Create: `store/chatui/migrations.go`
- Create: `store/chatui/sqlite.go`
- Test: `store/chatui/sqlite_test.go`

**Interfaces:**
- Produces: `ChatStore` interface, `Session`, `Message`, `Memory` types used by all downstream tasks

- [ ] **Step 1: Write failing tests for ChatStore**

```go
// store/chatui/sqlite_test.go
package chatui_test

import (
    "context"
    "testing"
    "os"
    "path/filepath"
    chatui "github.com/enowdev/enowx/store/chatui"
)

func TestSessionCRUD(t *testing.T) {
    store := openTestStore(t)
    ctx := context.Background()
    
    // Create
    sess, err := store.CreateSession(ctx, "Test chat", "claude-opus-4-5")
    if err != nil { t.Fatal(err) }
    if sess.ID == "" { t.Fatal("empty ID") }
    if sess.Title != "Test chat" { t.Fatalf("want 'Test chat', got %q", sess.Title) }
    
    // Get
    got, err := store.GetSession(ctx, sess.ID)
    if err != nil { t.Fatal(err) }
    if got.ID != sess.ID { t.Fatal("id mismatch") }
    
    // List
    sessions, err := store.ListSessions(ctx, 10, 0)
    if err != nil { t.Fatal(err) }
    if len(sessions) != 1 { t.Fatalf("want 1, got %d", len(sessions)) }
    
    // Update
    err = store.UpdateSession(ctx, sess.ID, chatui.SessionUpdate{Title: ptr("Renamed")})
    if err != nil { t.Fatal(err) }
    
    // Delete
    err = store.DeleteSession(ctx, sess.ID)
    if err != nil { t.Fatal(err) }
    sessions, _ = store.ListSessions(ctx, 10, 0)
    if len(sessions) != 0 { t.Fatal("should be empty after delete") }
}

func TestMessagePersistenceAndSeq(t *testing.T) {
    store := openTestStore(t)
    ctx := context.Background()
    sess, _ := store.CreateSession(ctx, "s", "m")
    
    // seq must be atomic — no collision
    m1, err := store.AppendMessage(ctx, chatui.Message{
        SessionID: sess.ID, Role: "user", Content: "hello",
    })
    if err != nil { t.Fatal(err) }
    m2, err := store.AppendMessage(ctx, chatui.Message{
        SessionID: sess.ID, Role: "assistant", Content: "hi",
    })
    if err != nil { t.Fatal(err) }
    if m1.Seq >= m2.Seq { t.Fatalf("seq not monotonic: %d >= %d", m1.Seq, m2.Seq) }
    
    // Session counters updated in same tx
    got, _ := store.GetSession(ctx, sess.ID)
    if got.MessageCount != 2 { t.Fatalf("message_count: want 2 got %d", got.MessageCount) }
}

func TestCascadeDelete(t *testing.T) {
    store := openTestStore(t)
    ctx := context.Background()
    sess, _ := store.CreateSession(ctx, "s", "m")
    store.AppendMessage(ctx, chatui.Message{SessionID: sess.ID, Role: "user", Content: "x"})
    store.DeleteSession(ctx, sess.ID)
    
    msgs, err := store.ListMessages(ctx, sess.ID, 100, 0)
    if err != nil { t.Fatal(err) }
    if len(msgs) != 0 { t.Fatal("messages should cascade delete") }
}

func TestFTSIndexedOnInsert(t *testing.T) {
    store := openTestStore(t)
    ctx := context.Background()
    sess, _ := store.CreateSession(ctx, "s", "m")
    store.AppendMessage(ctx, chatui.Message{
        SessionID: sess.ID, Role: "user", Content: "build sidebar with GlideMenu",
    })
    
    results, err := store.SearchMessages(ctx, "sidebar", 10)
    if err != nil { t.Fatal(err) }
    if len(results) == 0 { t.Fatal("FTS should find 'sidebar'") }
}

func TestFTSNotIndexReasoning(t *testing.T) {
    store := openTestStore(t)
    ctx := context.Background()
    sess, _ := store.CreateSession(ctx, "s", "m")
    store.AppendMessage(ctx, chatui.Message{
        SessionID: sess.ID, Role: "assistant",
        Content: "done", Reasoning: "secret internal thoughts",
    })
    
    results, _ := store.SearchMessages(ctx, "secret", 10)
    if len(results) != 0 { t.Fatal("reasoning should NOT be indexed in FTS") }
}

func TestMigrationVersion(t *testing.T) {
    store := openTestStore(t)
    // Calling Open twice (simulates restart) should be idempotent
    dir := t.TempDir()
    s1, err := chatui.Open(filepath.Join(dir, "chat.db"))
    if err != nil { t.Fatal(err) }
    s1.Close()
    s2, err := chatui.Open(filepath.Join(dir, "chat.db"))
    if err != nil { t.Fatal(err) }
    s2.Close()
}

func openTestStore(t *testing.T) chatui.ChatStore {
    t.Helper()
    dir := t.TempDir()
    s, err := chatui.Open(filepath.Join(dir, "test.db"))
    if err != nil { t.Fatalf("Open: %v", err) }
    t.Cleanup(func() { s.Close() })
    return s
}

func ptr[T any](v T) *T { return &v }
```

- [ ] **Step 2: Run test to verify it fails**

```bash
cd /home/arch/workspace/enowx && GOPATH=$HOME/go GOCACHE=$HOME/.cache/go go test ./store/chatui/... 2>&1
```
Expected: compile error (package doesn't exist yet)

- [ ] **Step 3: Write types.go**

```go
// store/chatui/types.go
package chatui

import "time"

type Session struct {
    ID           string    `json:"id"`
    Title        string    `json:"title"`
    Model        string    `json:"model"`
    MessageCount int       `json:"message_count"`
    TokensIn     int64     `json:"tokens_in"`
    TokensOut    int64     `json:"tokens_out"`
    Archived     bool      `json:"archived"`
    CreatedAt    time.Time `json:"created_at"`
    UpdatedAt    time.Time `json:"updated_at"`
}

type SessionUpdate struct {
    Title    *string `json:"title,omitempty"`
    Model    *string `json:"model,omitempty"`
    Archived *bool   `json:"archived,omitempty"`
}

// Message represents one turn in a chat session.
// Canonical storage (per AUD-10):
//   role=user       — user messages and tool results (tool_results JSON)
//   role=assistant  — model messages and tool calls (tool_calls JSON)
//   role=system     — system messages (reserved, not used in Phase 1)
//   role=tool       — reserved for future
type Message struct {
    ID          string    `json:"id"`
    SessionID   string    `json:"session_id"`
    Seq         int64     `json:"seq"`
    Role        string    `json:"role"` // user|assistant|system|tool
    Content     string    `json:"content"`
    Reasoning   string    `json:"reasoning,omitempty"`
    ToolCalls   string    `json:"tool_calls,omitempty"`   // JSON array of tool_use blocks (assistant role)
    ToolResults string    `json:"tool_results,omitempty"` // JSON array of tool_result blocks (user role)
    Attachments string    `json:"attachments,omitempty"`  // JSON array of {type,media_type,data_b64}
    Compacted   bool      `json:"compacted"`
    Model       string    `json:"model,omitempty"`
    TokensIn    int       `json:"tokens_in"`
    TokensOut   int       `json:"tokens_out"`
    CreatedAt   time.Time `json:"created_at"`
}

type Memory struct {
    ID        string    `json:"id"`
    Scope     string    `json:"scope"`     // session|user|global
    ScopeKey  string    `json:"scope_key"`
    Key       string    `json:"key"`
    Content   string    `json:"content"`
    Source    string    `json:"source"`    // chatui|manual
    CreatedAt time.Time `json:"created_at"`
    UpdatedAt time.Time `json:"updated_at"`
}

// SearchResult is returned by SearchMessages FTS query.
type SearchResult struct {
    Message
    SessionTitle string `json:"session_title"`
}

// KV settings keys
const (
    KVSchemaVersion    = "schema_version"
    KVActiveSession    = "chatui.active_session"
    KVToolsEnabled     = "chatui.tools_enabled"
    KVWorkspaceDir     = "chatui.workspace_dir"
    KVMCPServers       = "mcp.servers"
)
```

- [ ] **Step 4: Write store.go interface**

```go
// store/chatui/store.go
package chatui

import "context"

// ChatStore is the top-level interface for the ChatUI persistent store.
type ChatStore interface {
    // Sessions
    CreateSession(ctx context.Context, title, model string) (*Session, error)
    GetSession(ctx context.Context, id string) (*Session, error)
    ListSessions(ctx context.Context, limit, offset int) ([]Session, error)
    UpdateSession(ctx context.Context, id string, upd SessionUpdate) error
    DeleteSession(ctx context.Context, id string) error

    // Messages — seq is auto-allocated atomically inside AppendMessage
    AppendMessage(ctx context.Context, m Message) (*Message, error)
    ListMessages(ctx context.Context, sessionID string, limit, offset int) ([]Message, error)

    // FTS search across all sessions
    SearchMessages(ctx context.Context, query string, limit int) ([]SearchResult, error)

    // Memories
    PutMemory(ctx context.Context, m Memory) error
    ListMemories(ctx context.Context, scope, scopeKey string, limit int) ([]Memory, error)
    DeleteMemory(ctx context.Context, id string) error

    // KV
    KVGet(ctx context.Context, key string) (string, error)
    KVSet(ctx context.Context, key, value string) error

    Close() error
}
```

- [ ] **Step 5: Write migrations.go**

The migrations slice. Migration 001 = base tables (sessions, messages, memories, kv). Migration 002 = FTS + backfill.

```go
// store/chatui/migrations.go
package chatui

// allMigrations is the ordered list of SQL migrations.
// Each entry is idempotent (CREATE TABLE/INDEX IF NOT EXISTS).
// Index = version-1 (migration 1 is index 0).
var allMigrations = []string{
    // Migration 1 — base tables
    `CREATE TABLE IF NOT EXISTS kv (
        kv_key   TEXT PRIMARY KEY,
        kv_value TEXT NOT NULL DEFAULT ''
    );

    CREATE TABLE IF NOT EXISTS sessions (
        id            TEXT PRIMARY KEY,
        title         TEXT NOT NULL DEFAULT 'New Chat',
        model         TEXT NOT NULL DEFAULT 'claude-opus-4-5',
        message_count INTEGER NOT NULL DEFAULT 0,
        tokens_in     INTEGER NOT NULL DEFAULT 0,
        tokens_out    INTEGER NOT NULL DEFAULT 0,
        archived      BOOLEAN NOT NULL DEFAULT 0,
        created_at    DATETIME NOT NULL DEFAULT (datetime('now')),
        updated_at    DATETIME NOT NULL DEFAULT (datetime('now'))
    );
    CREATE INDEX IF NOT EXISTS idx_sessions_updated ON sessions(updated_at DESC);

    CREATE TABLE IF NOT EXISTS messages (
        id           TEXT PRIMARY KEY,
        session_id   TEXT NOT NULL REFERENCES sessions(id) ON DELETE CASCADE,
        seq          INTEGER NOT NULL,
        role         TEXT NOT NULL CHECK(role IN ('user','assistant','system','tool')),
        content      TEXT NOT NULL DEFAULT '',
        reasoning    TEXT NOT NULL DEFAULT '',
        tool_calls   TEXT NOT NULL DEFAULT '',
        tool_results TEXT NOT NULL DEFAULT '',
        attachments  TEXT NOT NULL DEFAULT '',
        compacted    BOOLEAN NOT NULL DEFAULT 0,
        model        TEXT NOT NULL DEFAULT '',
        tokens_in    INTEGER NOT NULL DEFAULT 0,
        tokens_out   INTEGER NOT NULL DEFAULT 0,
        created_at   DATETIME NOT NULL DEFAULT (datetime('now')),
        UNIQUE(session_id, seq)
    );
    CREATE INDEX IF NOT EXISTS idx_messages_session ON messages(session_id, seq);

    CREATE TABLE IF NOT EXISTS memories (
        id         TEXT PRIMARY KEY,
        scope      TEXT NOT NULL CHECK(scope IN ('session','user','global')),
        scope_key  TEXT NOT NULL DEFAULT '',
        key        TEXT NOT NULL DEFAULT '',
        content    TEXT NOT NULL DEFAULT '',
        source     TEXT NOT NULL DEFAULT 'chatui',
        created_at DATETIME NOT NULL DEFAULT (datetime('now')),
        updated_at DATETIME NOT NULL DEFAULT (datetime('now')),
        UNIQUE(scope, scope_key, key)
    );
    CREATE INDEX IF NOT EXISTS idx_memories_scope ON memories(scope, scope_key);`,

    // Migration 2 — FTS5 for cross-session recall (AUD-03: content only, not reasoning per AUD-08)
    // AUD-04: includes backfill of existing messages rows
    `CREATE VIRTUAL TABLE IF NOT EXISTS messages_fts USING fts5(
        content,
        content='messages',
        content_rowid='rowid',
        tokenize='porter unicode61'
    );

    CREATE TRIGGER IF NOT EXISTS messages_ai AFTER INSERT ON messages BEGIN
        INSERT INTO messages_fts(rowid, content)
        VALUES (new.rowid, new.content);
    END;
    CREATE TRIGGER IF NOT EXISTS messages_ad AFTER DELETE ON messages BEGIN
        INSERT INTO messages_fts(messages_fts, rowid, content)
        VALUES ('delete', old.rowid, old.content);
    END;
    CREATE TRIGGER IF NOT EXISTS messages_au AFTER UPDATE ON messages BEGIN
        INSERT INTO messages_fts(messages_fts, rowid, content)
        VALUES ('delete', old.rowid, old.content);
        INSERT INTO messages_fts(rowid, content)
        VALUES (new.rowid, new.content);
    END;

    -- Backfill existing messages (AUD-04 fix: Phase 1 rows not yet in FTS)
    INSERT OR IGNORE INTO messages_fts(rowid, content)
    SELECT rowid, content FROM messages WHERE compacted=0;`,
}
```

- [ ] **Step 6: Write sqlite.go**

Key implementation notes from existing enowX store pattern:
- `modernc.org/sqlite` driver, CGO_ENABLED=0
- WAL + busy_timeout(10000) + foreign_keys + `_txlock=immediate`
- `db.SetMaxOpenConns(1)` for serialized writes
- Migration uses `schema_version` KV key (NOT re-runs all DDL)
- seq allocated atomically via BEGIN IMMEDIATE transaction
- session counters updated in same tx as message insert

```go
// store/chatui/sqlite.go
package chatui

import (
    "context"
    "database/sql"
    "fmt"
    "strconv"
    "strings"
    "time"
    "unicode"

    "github.com/google/uuid"
    _ "modernc.org/sqlite"
)

type sqliteStore struct {
    db *sql.DB
}

// Open opens (or creates) the chatui SQLite database and runs pending migrations.
func Open(path string) (ChatStore, error) {
    dsn := "file:" + path +
        "?_pragma=journal_mode(WAL)" +
        "&_pragma=busy_timeout(10000)" +
        "&_pragma=foreign_keys(1)" +
        "&_txlock=immediate"
    db, err := sql.Open("sqlite", dsn)
    if err != nil {
        return nil, fmt.Errorf("chatui open: %w", err)
    }
    db.SetMaxOpenConns(1)
    db.SetMaxIdleConns(1)
    db.SetConnMaxLifetime(time.Hour)

    s := &sqliteStore{db: db}
    if err := s.migrate(context.Background()); err != nil {
        db.Close()
        return nil, fmt.Errorf("chatui migrate: %w", err)
    }
    return s, nil
}

func (s *sqliteStore) Close() error { return s.db.Close() }

// migrate runs only new migrations using schema_version KV key.
func (s *sqliteStore) migrate(ctx context.Context) error {
    // Bootstrap kv table first (needed to read schema_version)
    _, err := s.db.ExecContext(ctx, `CREATE TABLE IF NOT EXISTS kv (
        kv_key   TEXT PRIMARY KEY,
        kv_value TEXT NOT NULL DEFAULT ''
    )`)
    if err != nil {
        return fmt.Errorf("bootstrap kv: %w", err)
    }

    var version int
    row := s.db.QueryRowContext(ctx, `SELECT kv_value FROM kv WHERE kv_key='schema_version'`)
    var vStr string
    if err := row.Scan(&vStr); err == nil {
        version, _ = strconv.Atoi(vStr)
    }

    for i, migration := range allMigrations {
        if i+1 <= version {
            continue
        }
        // Each migration may be multiple statements — split on semicolon is fragile;
        // execute each migration as a single exec (SQLite supports multi-statement exec)
        if _, err := s.db.ExecContext(ctx, migration); err != nil {
            return fmt.Errorf("migration %d: %w", i+1, err)
        }
    }

    newVersion := len(allMigrations)
    if newVersion > version {
        _, err := s.db.ExecContext(ctx,
            `INSERT INTO kv(kv_key, kv_value) VALUES('schema_version', ?)
             ON CONFLICT(kv_key) DO UPDATE SET kv_value=excluded.kv_value`,
            strconv.Itoa(newVersion))
        return err
    }
    return nil
}

// --- Sessions ---

func (s *sqliteStore) CreateSession(ctx context.Context, title, model string) (*Session, error) {
    id := uuid.New().String()
    now := time.Now().UTC()
    _, err := s.db.ExecContext(ctx,
        `INSERT INTO sessions (id, title, model, created_at, updated_at)
         VALUES (?, ?, ?, ?, ?)`,
        id, title, model, now.Format(time.RFC3339), now.Format(time.RFC3339))
    if err != nil {
        return nil, fmt.Errorf("create session: %w", err)
    }
    return s.GetSession(ctx, id)
}

func (s *sqliteStore) GetSession(ctx context.Context, id string) (*Session, error) {
    row := s.db.QueryRowContext(ctx,
        `SELECT id, title, model, message_count, tokens_in, tokens_out, archived, created_at, updated_at
         FROM sessions WHERE id=?`, id)
    return scanSession(row)
}

func (s *sqliteStore) ListSessions(ctx context.Context, limit, offset int) ([]Session, error) {
    rows, err := s.db.QueryContext(ctx,
        `SELECT id, title, model, message_count, tokens_in, tokens_out, archived, created_at, updated_at
         FROM sessions WHERE archived=0 ORDER BY updated_at DESC LIMIT ? OFFSET ?`,
        limit, offset)
    if err != nil {
        return nil, err
    }
    defer rows.Close()
    var out []Session
    for rows.Next() {
        sess, err := scanSession(rows)
        if err != nil {
            return nil, err
        }
        out = append(out, *sess)
    }
    return out, rows.Err()
}

func (s *sqliteStore) UpdateSession(ctx context.Context, id string, upd SessionUpdate) error {
    var sets []string
    var args []any
    if upd.Title != nil {
        sets = append(sets, "title=?")
        args = append(args, *upd.Title)
    }
    if upd.Model != nil {
        sets = append(sets, "model=?")
        args = append(args, *upd.Model)
    }
    if upd.Archived != nil {
        sets = append(sets, "archived=?")
        args = append(args, *upd.Archived)
    }
    if len(sets) == 0 {
        return nil
    }
    sets = append(sets, "updated_at=datetime('now')")
    args = append(args, id)
    _, err := s.db.ExecContext(ctx,
        "UPDATE sessions SET "+strings.Join(sets, ", ")+" WHERE id=?", args...)
    return err
}

func (s *sqliteStore) DeleteSession(ctx context.Context, id string) error {
    _, err := s.db.ExecContext(ctx, `DELETE FROM sessions WHERE id=?`, id)
    return err
}

// --- Messages ---

// AppendMessage inserts a message with atomically-allocated seq.
// seq and session counters are updated in the same IMMEDIATE transaction (AUD-06, AUD-07).
func (s *sqliteStore) AppendMessage(ctx context.Context, m Message) (*Message, error) {
    if m.ID == "" {
        m.ID = uuid.New().String()
    }
    m.CreatedAt = time.Now().UTC()

    tx, err := s.db.BeginTx(ctx, nil) // WAL + _txlock=immediate handles serialization
    if err != nil {
        return nil, fmt.Errorf("begin tx: %w", err)
    }
    defer tx.Rollback()

    // Allocate seq atomically
    var maxSeq int64
    row := tx.QueryRowContext(ctx, `SELECT COALESCE(MAX(seq),0) FROM messages WHERE session_id=?`, m.SessionID)
    if err := row.Scan(&maxSeq); err != nil {
        return nil, fmt.Errorf("seq scan: %w", err)
    }
    m.Seq = maxSeq + 1

    _, err = tx.ExecContext(ctx,
        `INSERT INTO messages (id, session_id, seq, role, content, reasoning, tool_calls, tool_results, attachments, compacted, model, tokens_in, tokens_out, created_at)
         VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)`,
        m.ID, m.SessionID, m.Seq, m.Role, m.Content,
        m.Reasoning, m.ToolCalls, m.ToolResults, m.Attachments,
        m.Compacted, m.Model, m.TokensIn, m.TokensOut,
        m.CreatedAt.Format(time.RFC3339))
    if err != nil {
        return nil, fmt.Errorf("insert message: %w", err)
    }

    // Update session counters in same transaction (AUD-07)
    _, err = tx.ExecContext(ctx,
        `UPDATE sessions SET
            message_count=message_count+1,
            tokens_in=tokens_in+?,
            tokens_out=tokens_out+?,
            updated_at=datetime('now')
         WHERE id=?`,
        m.TokensIn, m.TokensOut, m.SessionID)
    if err != nil {
        return nil, fmt.Errorf("update counters: %w", err)
    }

    if err := tx.Commit(); err != nil {
        return nil, fmt.Errorf("commit: %w", err)
    }
    return &m, nil
}

func (s *sqliteStore) ListMessages(ctx context.Context, sessionID string, limit, offset int) ([]Message, error) {
    rows, err := s.db.QueryContext(ctx,
        `SELECT id, session_id, seq, role, content, reasoning, tool_calls, tool_results, attachments, compacted, model, tokens_in, tokens_out, created_at
         FROM messages WHERE session_id=? ORDER BY seq ASC LIMIT ? OFFSET ?`,
        sessionID, limit, offset)
    if err != nil {
        return nil, err
    }
    defer rows.Close()
    var out []Message
    for rows.Next() {
        m, err := scanMessage(rows)
        if err != nil {
            return nil, err
        }
        out = append(out, *m)
    }
    return out, rows.Err()
}

// --- FTS Search ---

// sanitizeFTSQuery converts a plain text query to FTS5 prefix query.
// "build sidebar" → "build* sidebar*"
func sanitizeFTSQuery(q string) string {
    words := strings.Fields(q)
    var out []string
    for _, w := range words {
        clean := strings.Map(func(r rune) rune {
            if unicode.IsLetter(r) || unicode.IsDigit(r) {
                return r
            }
            return -1
        }, w)
        if clean != "" {
            out = append(out, clean+"*")
        }
    }
    if len(out) == 0 {
        return ""
    }
    return strings.Join(out, " ")
}

func (s *sqliteStore) SearchMessages(ctx context.Context, query string, limit int) ([]SearchResult, error) {
    ftsQuery := sanitizeFTSQuery(query)
    if ftsQuery == "" {
        return nil, nil
    }
    rows, err := s.db.QueryContext(ctx, `
        SELECT
            m.id, m.session_id, m.seq, m.role, m.content, m.reasoning,
            m.tool_calls, m.tool_results, m.attachments, m.compacted,
            m.model, m.tokens_in, m.tokens_out, m.created_at,
            s.title as session_title
        FROM messages_fts fts
        JOIN messages m ON m.rowid = fts.rowid
        JOIN sessions s ON s.id = m.session_id
        WHERE messages_fts MATCH ?
          AND m.compacted = 0
          AND m.role IN ('user', 'assistant')
        ORDER BY rank
        LIMIT ?
    `, ftsQuery, limit)
    if err != nil {
        return nil, fmt.Errorf("search messages: %w", err)
    }
    defer rows.Close()
    var out []SearchResult
    for rows.Next() {
        var r SearchResult
        var createdAt string
        if err := rows.Scan(
            &r.ID, &r.SessionID, &r.Seq, &r.Role, &r.Content, &r.Reasoning,
            &r.ToolCalls, &r.ToolResults, &r.Attachments, &r.Compacted,
            &r.Model, &r.TokensIn, &r.TokensOut, &createdAt, &r.SessionTitle,
        ); err != nil {
            return nil, err
        }
        r.CreatedAt, _ = time.Parse(time.RFC3339, createdAt)
        out = append(out, r)
    }
    return out, rows.Err()
}

// --- Memories ---

func (s *sqliteStore) PutMemory(ctx context.Context, m Memory) error {
    if m.ID == "" {
        m.ID = uuid.New().String()
    }
    now := time.Now().UTC()
    if m.CreatedAt.IsZero() {
        m.CreatedAt = now
    }
    _, err := s.db.ExecContext(ctx,
        `INSERT INTO memories (id, scope, scope_key, key, content, source, created_at, updated_at)
         VALUES (?, ?, ?, ?, ?, ?, ?, ?)
         ON CONFLICT(scope, scope_key, key) DO UPDATE SET
         content=excluded.content, source=excluded.source, updated_at=excluded.updated_at`,
        m.ID, m.Scope, m.ScopeKey, m.Key, m.Content, m.Source,
        m.CreatedAt.Format(time.RFC3339), now.Format(time.RFC3339))
    return err
}

func (s *sqliteStore) ListMemories(ctx context.Context, scope, scopeKey string, limit int) ([]Memory, error) {
    rows, err := s.db.QueryContext(ctx,
        `SELECT id, scope, scope_key, key, content, source, created_at, updated_at
         FROM memories WHERE scope=? AND scope_key=? ORDER BY updated_at DESC LIMIT ?`,
        scope, scopeKey, limit)
    if err != nil {
        return nil, err
    }
    defer rows.Close()
    var out []Memory
    for rows.Next() {
        var m Memory
        var ca, ua string
        if err := rows.Scan(&m.ID, &m.Scope, &m.ScopeKey, &m.Key, &m.Content, &m.Source, &ca, &ua); err != nil {
            return nil, err
        }
        m.CreatedAt, _ = time.Parse(time.RFC3339, ca)
        m.UpdatedAt, _ = time.Parse(time.RFC3339, ua)
        out = append(out, m)
    }
    return out, rows.Err()
}

func (s *sqliteStore) DeleteMemory(ctx context.Context, id string) error {
    _, err := s.db.ExecContext(ctx, `DELETE FROM memories WHERE id=?`, id)
    return err
}

// --- KV ---

func (s *sqliteStore) KVGet(ctx context.Context, key string) (string, error) {
    var val string
    err := s.db.QueryRowContext(ctx, `SELECT kv_value FROM kv WHERE kv_key=?`, key).Scan(&val)
    if err == sql.ErrNoRows {
        return "", nil
    }
    return val, err
}

func (s *sqliteStore) KVSet(ctx context.Context, key, value string) error {
    _, err := s.db.ExecContext(ctx,
        `INSERT INTO kv(kv_key, kv_value) VALUES(?, ?)
         ON CONFLICT(kv_key) DO UPDATE SET kv_value=excluded.kv_value`,
        key, value)
    return err
}

// --- Scanners ---

type rowScanner interface {
    Scan(dest ...any) error
}

func scanSession(r rowScanner) (*Session, error) {
    var s Session
    var ca, ua string
    var archived int
    if err := r.Scan(&s.ID, &s.Title, &s.Model, &s.MessageCount, &s.TokensIn, &s.TokensOut, &archived, &ca, &ua); err != nil {
        return nil, err
    }
    s.Archived = archived != 0
    s.CreatedAt, _ = time.Parse(time.RFC3339, ca)
    s.UpdatedAt, _ = time.Parse(time.RFC3339, ua)
    return &s, nil
}

func scanMessage(r rowScanner) (*Message, error) {
    var m Message
    var createdAt string
    var compacted int
    if err := r.Scan(
        &m.ID, &m.SessionID, &m.Seq, &m.Role, &m.Content, &m.Reasoning,
        &m.ToolCalls, &m.ToolResults, &m.Attachments, &compacted,
        &m.Model, &m.TokensIn, &m.TokensOut, &createdAt,
    ); err != nil {
        return nil, err
    }
    m.Compacted = compacted != 0
    m.CreatedAt, _ = time.Parse(time.RFC3339, createdAt)
    return &m, nil
}
```

- [ ] **Step 7: Run tests and verify they pass**

```bash
cd /home/arch/workspace/enowx && GOPATH=$HOME/go GOCACHE=$HOME/.cache/go go test ./store/chatui/... -v -count=1 2>&1
```
Expected: All tests pass. TestFTSNotIndexReasoning confirms reasoning excluded. TestMigrationVersion confirms idempotent restart.

- [ ] **Step 8: Commit**

```bash
git add store/chatui/
git commit -m "feat(chatui): store package — sessions, messages, FTS, memories, KV

- types.go: Session, Message, Memory, SearchResult types
- store.go: ChatStore interface
- migrations.go: 2 migrations (base tables + FTS with backfill)
- sqlite.go: Open(), migrate(), CRUD, atomic seq, same-tx counters, FTS search
- Tests: CRUD, seq monotonicity, cascade delete, FTS insert/recall,
         reasoning NOT indexed in FTS, idempotent migration"
```

---

## Task 2 — Session + Memory REST API handlers + server wiring

**Files:**
- Create: `server/handlers/chatui_sessions.go`
- Create: `server/handlers/chatui_memory.go`
- Create: `server/handlers/chatui_settings.go`
- Modify: `server/server.go` — add `ChatUI ChatStore` to `Deps` + register routes
- Modify: `cmd/enowx/main.go` — open chatui.db, pass to Deps
- Modify: `store/store.go` — add `ChatUIStore` to Store interface (or leave as separate open)
- Test: `server/handlers/chatui_sessions_test.go`

**Interfaces:**
- Consumes: `store/chatui.ChatStore` from Task 1
- Produces: REST endpoints `/chatui/api/sessions/*`, `/chatui/api/memories/*`, `/chatui/api/settings`

- [ ] **Step 1: Write failing tests for session API**

```go
// server/handlers/chatui_sessions_test.go
package handlers_test

// Test: POST /chatui/api/sessions creates session
// Test: GET /chatui/api/sessions lists sessions
// Test: GET /chatui/api/sessions/:id returns session
// Test: PATCH /chatui/api/sessions/:id — title, model, archived (NOT tools_enabled — AUD-05)
// Test: PATCH /chatui/api/sessions/:id with tools_enabled field → 400 Bad Request
// Test: DELETE /chatui/api/sessions/:id deletes session
// Test: GET /chatui/api/settings returns tools_enabled, workspace_dir
// Test: PATCH /chatui/api/settings updates tools_enabled
```

- [ ] **Step 2: Implement chatui_sessions.go**

Handler pattern follows `api_settings.go`:
- Struct with `db chatui.ChatStore`
- Constructor `NewChatUISessions(db chatui.ChatStore) *ChatUISessions`
- Methods: List, Create, Get, Update, Delete, AppendMessage
- Response: `writeData(w, payload)` / `writeErr(w, code, msg)`
- PATCH sessions body: only `{title, model, archived}` — reject `tools_enabled` with 400

- [ ] **Step 3: Implement chatui_settings.go**

- `GET /chatui/api/settings` → reads `chatui.tools_enabled` and `chatui.workspace_dir` from KV
- `PATCH /chatui/api/settings` → writes to KV

- [ ] **Step 4: Implement chatui_memory.go**

- `GET /chatui/api/sessions/:id/memories` → `ListMemories(session, sessionID)`
- `GET /chatui/api/memories?q=` → `SearchMessages(q)`
- `POST /chatui/api/memories` → `PutMemory`
- `DELETE /chatui/api/memories/:id` → `DeleteMemory`

- [ ] **Step 5: Wire into server.go**

```go
// server/server.go — Deps struct addition:
type Deps struct {
    // existing fields...
    ChatUI chatui.ChatStore  // NEW: chatui.db store
}

// In New():
chatuiSessions := handlers.NewChatUISessions(d.ChatUI)
chatuiMemory := handlers.NewChatUIMemory(d.ChatUI)
chatuiSettings := handlers.NewChatUISettings(d.ChatUI)

r.Route("/chatui/api", func(r chi.Router) {
    // No dash.Require — local machine only (consistent with existing /chatui/api/config)
    r.Route("/sessions", func(r chi.Router) {
        r.Get("/", chatuiSessions.List)
        r.Post("/", chatuiSessions.Create)
        r.Get("/{id}", chatuiSessions.Get)
        r.Patch("/{id}", chatuiSessions.Update)
        r.Delete("/{id}", chatuiSessions.Delete)
        r.Post("/{id}/messages", chatuiSessions.AppendMessage)
        r.Get("/{id}/memories", chatuiMemory.ListForSession)
        r.Post("/{id}/index", chatuiMemory.TriggerIndex) // Phase 2
    })
    r.Get("/memories", chatuiMemory.Search)
    r.Post("/memories", chatuiMemory.Create)
    r.Delete("/memories/{id}", chatuiMemory.Delete)
    r.Get("/settings", chatuiSettings.Get)
    r.Patch("/settings", chatuiSettings.Patch)
})
```

- [ ] **Step 6: Wire into cmd/enowx/main.go**

```go
// After opening enowx.db:
chatuiDB, err := chatuidb.Open(filepath.Join(cfg.RuntimeDir, "chatui.db"))
if err != nil { log.Fatalf("chatui db: %v", err) }
defer chatuiDB.Close()

// Pass to server.Deps:
deps.ChatUI = chatuiDB
```

- [ ] **Step 7: Run tests**

```bash
GOPATH=$HOME/go GOCACHE=$HOME/.cache/go go test ./server/handlers/... -run ChatUI -v 2>&1
```

- [ ] **Step 8: Smoke test build**

```bash
GOPATH=$HOME/go GOCACHE=$HOME/.cache/go CGO_ENABLED=0 go build -o /tmp/enx-test ./cmd/enowx/ 2>&1 || echo "expected: webdist embed failure OK"
# Build should succeed except for webdist embed (pre-existing issue)
GOPATH=$HOME/go GOCACHE=$HOME/.cache/go CGO_ENABLED=0 go build -tags dev -o /tmp/enx-test ./cmd/enowx/ 2>&1
```

- [ ] **Step 9: Commit**

```bash
git add server/handlers/chatui_*.go server/server.go cmd/enowx/main.go
git commit -m "feat(chatui): REST API handlers — sessions, memories, settings

- chatui_sessions.go: CRUD + message append, PATCH rejects tools_enabled (AUD-05)
- chatui_memory.go: list/search/create/delete memories
- chatui_settings.go: tools_enabled + workspace_dir KV settings
- server.go: /chatui/api/* routes, ChatUI in Deps
- main.go: open chatui.db, wire into server"
```

---

## Task 3 — count_tokens proxy route (AUD-04)

**Files:**
- Modify: `server/handlers/anthropic.go` — add `CountTokens` handler
- Modify: `server/server.go` — register route

**Interfaces:**
- Consumes: existing Anthropic handler pattern
- Produces: `POST /anthropic/v1/messages/count_tokens` → proxied to upstream

- [ ] **Step 1: Write failing test**

```go
// Test: POST /anthropic/v1/messages/count_tokens with valid body
// Expected: proxied to upstream, returns JSON with input_tokens
// OR: falls back to local counting if upstream returns 404/501
```

- [ ] **Step 2: Implement CountTokens handler**

The handler should:
1. Read request body (JSON: model, messages, system, tools)
2. Forward to upstream via same proxy pool routing as `Messages`
3. If upstream returns 404/501/400 (provider doesn't support it): fall back to local estimation using `core/tokenize` package
4. Return JSON response: `{"input_tokens": N}`

Local fallback: `core/tokenize` package exists in enowX. Use it to estimate tokens from the message content. This is approximate but unblocks the SDK's pre-flight check.

```go
// server/handlers/anthropic.go — add method:
func (h *Anthropic) CountTokens(w http.ResponseWriter, r *http.Request) {
    // Forward to upstream /messages/count_tokens
    // Fall back to local tokenize.Count() if upstream returns non-200
}
```

- [ ] **Step 3: Register route in server.go**

```go
r.Post("/anthropic/v1/messages/count_tokens", anthropic.CountTokens)
```

- [ ] **Step 4: Run tests + build**

- [ ] **Step 5: Commit**

```bash
git commit -m "feat(proxy): add /anthropic/v1/messages/count_tokens route

Proxies to upstream with local tokenize fallback.
Required for Anthropic SDK CountTokens() pre-flight check in ChatUI agent loop."
```

---

## Task 4 — core/chatui: agent loop, tools, prompt builder

**Files:**
- Create: `core/chatui/prompt.go`
- Create: `core/chatui/tools.go`
- Create: `core/chatui/tools_web.go`
- Create: `core/chatui/tools_file.go`
- Create: `core/chatui/tools_term.go`
- Create: `core/chatui/tools_memory.go`
- Create: `core/chatui/plugins.go`
- Create: `core/chatui/recall.go`
- Create: `core/chatui/agent.go`
- Test: `core/chatui/agent_test.go`
- Modify: `go.mod` — add `github.com/anthropics/anthropic-sdk-go`

**Interfaces:**
- Consumes: `store/chatui.ChatStore`, SDK client, config
- Produces: `ChatLoop(ctx, req)` → SSE stream to `http.ResponseWriter`

- [ ] **Step 1: Add Anthropic SDK**

```bash
cd /home/arch/workspace/enowx && GOPATH=$HOME/go GOCACHE=$HOME/.cache/go go get github.com/anthropics/anthropic-sdk-go@latest 2>&1
```

- [ ] **Step 2: Write agent tests**

Key behaviors to test:
- Normal response (no tools) → `text_delta` SSE events → `done`
- One tool call → `tool_start` + `tool_result` → continue → `done`
- Max turns (10) reached → `error` SSE event with "max turns reached"
- Tool output truncation: tool returning > 3000 tokens → truncated in result
- Approval required: `write_file` → `tool_approval_required` SSE → on deny → `tool_result` with "denied"
- Compaction: build message history > 85% context → `compacted` SSE event → continue

Use mock SDK client for unit tests (interface over the real SDK streaming call).

- [ ] **Step 3: Implement prompt.go (5-section builder)**

```go
// core/chatui/prompt.go
// BuildSystemPrompt constructs frozen 5-section system prompt.
// Called ONCE at session start — never mid-session (frozen snapshot pattern).
// Sections:
//   1. Identity (~50 tokens)
//   2. Tool notes (~200 tokens, only if tools enabled)  
//   3. Skills (~120 tokens, Phase 3 stub — empty for now)
//   4a. Curated memories (~200 tokens)
//   4b. Relevant history (~800 tokens, from FTS recall)
//   5. system_extra (user-provided, if any)
// Total cap: ~1200 tokens enforced via truncation
```

- [ ] **Step 4: Implement tools.go (Tool interface + registry)**

```go
type Tool interface {
    Name() string
    Description() string
    InputSchema() json.RawMessage
    Execute(ctx context.Context, input json.RawMessage) (string, error)
    RequiresApproval() bool
}

// Token output caps (AUD confirmed — applied after serialization)
const MaxToolOutputTokens = 3000
const MaxTotalToolTokens  = 8000
```

- [ ] **Step 5: Implement all 8 tools**

tools_web.go: `web_search` (8 results), `web_fetch` (15K char cap, plain text)
tools_file.go: `read_file` (400 lines cap), `write_file` (approval), `list_files` (depth 3), `grep` (50 matches)
tools_term.go: `terminal` (30s timeout, 3000 token output cap, approval)
tools_memory.go: `memory` (actions: save/search/list/delete, reads/writes ChatStore)

- [ ] **Step 6: Implement plugins.go**

```go
// 3 built-in plugins:
// ApprovalGate: BeforeToolCall for write_file, terminal → blocks until user approves
// OutputTruncator: AfterToolCall → truncate to MaxToolOutputTokens
// WebFetchCleaner: AfterToolCall for web_fetch → strip HTML, compress whitespace
```

Approval state machine (AUD-09):
```go
// In-memory only (ephemeral — lives only during active SSE stream)
type ApprovalState int
const (
    ApprovalPending  ApprovalState = iota
    ApprovalApproved
    ApprovalDenied
    ApprovalExpired
    ApprovalCancelled
)
// approvals map[string]chan ApprovalResult (keyed by approval_id)
// Timeout: 5 minutes → EXPIRED (treated as DENIED)
// SSE context cancel → CANCELLED (treated as DENIED)
// Duplicate approve/deny → 409 Conflict
```

- [ ] **Step 7: Implement recall.go**

FTS-based cross-session history recall. Dedup by session. Top-2 sessions. Format as frozen Section 4b.

- [ ] **Step 8: Implement agent.go (ChatLoop)**

```go
// ChatLoop is the main agentic turn loop.
// Compaction threshold: 85% (AUD-01 fix)
// Compaction behavior: set compacted=TRUE on old rows, keep in DB (AUD-02 fix)
// Context fallback: 32K (AUD-11 fix)
// Max turns: 10
// Serial tool calls (no parallel)
```

Key implementation notes:
- Compaction archives rows (compacted=TRUE), doesn't delete them
- FTS triggers fire on UPDATE (via messages_au trigger) — compacted rows removed from FTS index
- `compacted=0` filter in recall remains meaningful

- [ ] **Step 9: Run agent tests**

```bash
GOPATH=$HOME/go GOCACHE=$HOME/.cache/go go test ./core/chatui/... -v -count=1 -race 2>&1
```

- [ ] **Step 10: Commit**

```bash
git add core/chatui/ go.mod go.sum
git commit -m "feat(chatui): agentic core — agent loop, 8 tools, plugins, recall, prompt

- agent.go: ChatLoop, 10-turn limit, 85% compaction, flag-not-delete (AUD-01/02)
- tools.go: Tool interface, registry, 3000/8000 token caps
- tools_web/file/term/memory.go: all 8 tools
- plugins.go: ApprovalGate (state machine), OutputTruncator, WebFetchCleaner
- recall.go: FTS-based frozen Section 4b recall
- prompt.go: 5-section builder, frozen snapshot pattern
- Tests: normal, tool call, max turns, truncation, approval flow, compaction"
```

---

## Task 5 — chat/stream SSE handler + approval endpoints

**Files:**
- Create: `server/handlers/chatui_chat.go`
- Modify: `server/server.go` — register /chatui/api/chat/* routes

**Interfaces:**
- Consumes: `core/chatui.ChatLoop`, `store/chatui.ChatStore`
- Produces: `POST /chatui/api/chat/stream` (SSE), `POST /chatui/api/chat/approve/:id`, `POST /chatui/api/chat/deny/:id`

- [ ] **Step 1: Implement chatui_chat.go**

SSE handler:
```go
// POST /chatui/api/chat/stream
// Body: {session_id, message, model, system_extra, attachments[]}
// Response: SSE stream
// Events: text_delta, thinking_delta, tool_start, tool_result,
//         tool_approval_required, compacted, done, error

func (h *ChatUIChat) Stream(w http.ResponseWriter, r *http.Request) {
    // Set SSE headers
    w.Header().Set("Content-Type", "text/event-stream")
    w.Header().Set("Cache-Control", "no-cache")
    w.Header().Set("X-Accel-Buffering", "no")
    w.WriteHeader(http.StatusOK)
    fl := w.(http.Flusher)
    
    // Register approval channels for this request
    // Run ChatLoop, writing SSE events as they arrive
    // On context cancel (client disconnect): cancel all pending approvals
}
```

Approval endpoints:
```go
// POST /chatui/api/chat/approve/:approval_id
// POST /chatui/api/chat/deny/:approval_id
// Look up approval_id in in-memory map
// Write APPROVED/DENIED to channel
// First write wins; second returns 409
```

- [ ] **Step 2: Register routes**

```go
r.Route("/chatui/api/chat", func(r chi.Router) {
    r.Post("/stream", chatuiChat.Stream)
    r.Post("/approve/{approval_id}", chatuiChat.Approve)
    r.Post("/deny/{approval_id}", chatuiChat.Deny)
})
```

- [ ] **Step 3: Integration test — SSE stream**

```bash
# Build with dev tag, start server, send a test message, verify SSE events
GOPATH=$HOME/go GOCACHE=$HOME/.cache/go go test ./server/handlers/... -run ChatUIChat -v 2>&1
```

- [ ] **Step 4: Commit**

```bash
git commit -m "feat(chatui): chat/stream SSE handler + approval endpoints

- chatui_chat.go: POST /chatui/api/chat/stream SSE agentic loop
- approve/deny endpoints with state machine (AUD-09)
- SSE disconnect cancels pending approvals"
```

---

## Task 6 — memory indexer goroutine

**Files:**
- Create: `core/chatui/memory.go`
- Test: `core/chatui/memory_test.go`

**Interfaces:**
- Consumes: `ChatStore`, SDK client
- Produces: background indexer that fires after each turn

- [ ] **Step 1: Write tests for indexer**

```go
// Test: indexMemory writes session + user scope memories
// Test: rate limit (< 30s same session) skips indexing
// Test: NONE response → nothing written
// Test: concurrent indexers for different sessions — both write correctly
// Test: concurrent indexers for SAME session — last write wins (source_seq ordering if needed)
```

- [ ] **Step 2: Implement memory.go**

```go
// core/chatui/memory.go
// Rate limit key: per-session, in-memory map[sessionID]time.Time
// Prompt: exact text from antares-patterns.md §6 (summariseUserTurn)
// Response "NONE" → skip
// Upsert scope="session", scope_key=sessionID
// Upsert scope="user", scope_key="local"  
// On error: log only, never block
```

- [ ] **Step 3: Run tests**

```bash
GOPATH=$HOME/go GOCACHE=$HOME/.cache/go go test ./core/chatui/... -run Memory -v -race 2>&1
```

- [ ] **Step 4: Commit**

```bash
git commit -m "feat(chatui): background memory indexer

- Per-session 30s rate limit (in-memory, resets on restart)
- antares-patterns.md exact summarisation prompt
- session + user scope upsert
- NONE response → skip
- Tests: write, rate limit, NONE, concurrent sessions"
```

---

## Task 7 — Frontend: useSessions.ts + useChat.ts (Phase 1 + Phase 2)

**Files:**
- Create: `web/src/chatui/useSessions.ts`
- Create: `web/src/chatui/useChat.ts`  
- Create: `web/src/chatui/ToolCallBlock.tsx`
- Create: `web/src/chatui/ApprovalDialog.tsx`
- Create: `web/src/chatui/CompactedBanner.tsx`
- Modify: `web/src/apps/AiChatApp.tsx` — integrate useSessions + useChat hooks
- Test: manual smoke test via dev server

**Interfaces:**
- Consumes: `/chatui/api/sessions/*`, `/chatui/api/chat/stream`
- Produces: React hooks replacing localStorage-only storage

- [ ] **Step 1: Implement useSessions.ts (write-through API + localStorage fallback)**

```typescript
// Phase 1: write-through strategy
// createSession(): POST /chatui/api/sessions → store in localStorage as cache
// loadSessions(): GET /chatui/api/sessions (prefer API, fall back to localStorage)
// addMessage(): POST /chatui/api/sessions/:id/messages (after turn completes)
// deleteSession(): DELETE /chatui/api/sessions/:id
```

- [ ] **Step 2: Implement useChat.ts (SSE agentic endpoint, Phase 2)**

```typescript
// If tools enabled: POST /chatui/api/chat/stream + handle SSE events
// SSE event types: text_delta, thinking_delta, tool_start, tool_result,
//                  tool_approval_required, compacted, done, error
// If tools disabled: fall through to existing /anthropic/v1/messages (unchanged)
```

- [ ] **Step 3: Implement UI components**

ToolCallBlock.tsx: show tool name + args + collapsible result
ApprovalDialog.tsx: modal with approve/deny buttons → POST approve/deny endpoint
CompactedBanner.tsx: subtle "Earlier messages summarised" indicator

- [ ] **Step 4: Integrate into AiChatApp.tsx**

- Replace localStorage-only session storage with `useSessions` hook
- Replace direct `/anthropic/v1/messages` calls with `useChat` when tools enabled
- Add `ToolCallBlock` rendering in message thread
- Add `ApprovalDialog` modal
- Add `CompactedBanner` in conversation

- [ ] **Step 5: Manual smoke test**

```bash
# In worktree
cd web && npm install && npm run build
# Start dev server
ENOWX_PORT=1431 go run -tags dev ./cmd/enowx/ &
# Open browser → verify: session persists after refresh, tool calls visible
```

- [ ] **Step 6: Commit**

```bash
git add web/src/chatui/ web/src/apps/AiChatApp.tsx
git commit -m "feat(chatui): frontend hooks + tool UI

- useSessions.ts: write-through API + localStorage fallback
- useChat.ts: SSE agentic stream + existing direct path (tools disabled)
- ToolCallBlock, ApprovalDialog, CompactedBanner components
- AiChatApp.tsx: integrated hooks + tool/approval/compaction UI"
```

---

## Task 8 — Spec update: incorporate all audit findings into design doc

**Files:**
- Modify: `/home/arch/workspace/antares/docs/superpowers/specs/2026-09-02-chatui-memory-system-design.md`

This task updates the spec to reflect what was actually built. Run LAST after implementation is complete and tests pass.

- [ ] **Step 1: Update §10.3 compaction threshold** — change 90% to 85% (AUD-01)
- [ ] **Step 2: Update §10.4 compaction behavior** — clarify flag-not-delete (AUD-02)
- [ ] **Step 3: Update §11.2 FTS schema** — remove `reasoning` column (AUD-08)
- [ ] **Step 4: Update §13.2 FTS schema** — same
- [ ] **Step 5: Update §14 API spec** — remove `tools_enabled` from PATCH sessions (AUD-05)
- [ ] **Step 6: Add §21 Audit Findings** — document all 12 findings with status
- [ ] **Step 7: Update spec Status** — change DRAFT to IMPLEMENTED
- [ ] **Step 8: Commit spec update to antares repo**

```bash
cd /home/arch/workspace/antares && git add docs/superpowers/specs/ && git commit -m "docs: update chatui-memory-system spec — audit findings incorporated"
```

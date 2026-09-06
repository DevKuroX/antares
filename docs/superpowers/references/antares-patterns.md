# Antares Architecture Reference — ChatUI Implementation Guide

**Purpose:** Source-of-truth patterns borrowed from Antares for enowX ChatUI
implementation. Read this before touching any ChatUI backend code.

**Source:** `/home/arch/workspace/antares/internal/store/`,
`/home/arch/workspace/antares/internal/agent/ragcontext.go`,
`/home/arch/workspace/antares/internal/tools/`

**Rule:** We do NOT import Antares as a Go module. We copy the patterns and
write our own implementation. This doc is the bridge.

---

## 10. Antares Tool Registry — Full Tool List

**Source:** `internal/tools/register.go` + `internal/tools/registry.go`

### All registered tools (as of 2026-09-03)

| Tool name | Category | Requires approval | Notes |
|---|---|---|---|
| `read_file` | Files | No | |
| `write_file` | Files | Yes | Mutates |
| `edit_file` | Files | Yes | Mutates |
| `list_files` | Files | No | |
| `glob` | Files | No | |
| `grep` | Files | No | |
| `read_document` | Files | No | PDF/DOCX reader |
| `terminal` | Shell | Yes | Spawns shell |
| `process` | Shell | No | Job control companion for terminal |
| `web_search` | Web | No | |
| `web_fetch` | Web | No | |
| `http_request` | Web | No | Raw HTTP, custom headers |
| `browser` | Web | No | Stealth browser (Playwright) |
| `memory` | Memory | No | Read/write persistent facts |
| `session_search` | Memory | No | FTS over past sessions |
| `rag_search` | Memory | No | Embedding search over RAG chunks |
| `rag_index` | Memory | No | Index a doc into RAG |
| `todo` | Productivity | No | Checklist tracker |
| `board` | Productivity | No | Kanban board |
| `task` | Productivity | No | Background task polling |
| `project_info` | Productivity | No | Project metadata |
| `set_soul` | Productivity | No | Change agent persona |
| `skill` | Skills | No | List/search/read/save procedures |
| `delegate_task` | Delegation | No | Spawn sub-agent |
| `list_roles` | Delegation | No | List available specialist roles |
| `ask_user` | Interaction | No | Blocks turn to ask user |
| `schedule` | Automation | No | Schedule future agent runs |
| `diagnostics` | Debugging | No | |
| `view_image` | Vision | No | |
| `image_generate` | Vision | No | |
| `speak` | Audio | No | TTS |
| `transcribe` | Audio | No | STT |
| `computer` | Desktop | Yes | Screen control |
| `report_finding` | Security | No | |
| `triage_finding` | Security | No | |
| `add_intel` | Security | No | |
| `methodology_status` | Security | No | |
| `check_dependencies` | Security | No | Prereq gate |
| `intercept` | Security/Proxy | Yes | MITM intercept proxy |
| `hackbrowser` | Security | Yes | Autonomous web crawler |
| `solve_captcha` | Security | Yes | |
| `vps_run` | VPS | Yes | SSH remote exec |
| `vps_upload` | VPS | Yes | SFTP upload |
| `vps_download` | VPS | No | SFTP download |
| `list_proxies` | Network | No | |
| `email_read` | Social | No | IMAP inbox |
| `temp_mail` | Social | No | Disposable inbox |
| `social_browser` | Social | No | Persistent social browser |
| `social_account` | Social | No | Saved social accounts |
| `osint_dns` | OSINT | No | |
| `osint_dorks` | OSINT | No | |
| `osint_dorks_live` | OSINT | No | |
| `osint_whois` | OSINT | No | |
| `osint_ip` | OSINT | No | |
| `osint_username` | OSINT | No | |
| `osint_github` | OSINT | No | |
| `osint_email` | OSINT | No | |
| `osint_email_full` | OSINT | No | |
| `osint_breach` | OSINT | No | |
| `osint_shodan` | OSINT | No | |
| `osint_reputation` | OSINT | No | |
| `osint_crypto` | OSINT | No | |
| `osint_domain` | OSINT | No | |
| `osint_phone` | OSINT | No | |
| `osint_scrape` | OSINT | No | |
| `osint_paste` | OSINT | No | |
| `osint_footprint` | OSINT | No | |
| `osint_pivot` | OSINT | No | |
| `osint_google` | OSINT | No | |
| `re_info` | Reverse Engineering | No | |
| `re_strings` | Reverse Engineering | No | |
| `re_analyze` | Reverse Engineering | No | |
| `re_decompile` | Reverse Engineering | No | |
| `attack_script` | Offensive | Yes | Bundled Python/PS exploit scripts |
| `awshook` | Offensive | Yes | AWS post-exploitation |
| `azurehook` | Offensive | Yes | Azure post-exploitation |
| `kubehook` | Offensive | Yes | K8s post-exploitation |
| `winhook` | Offensive/Windows | Yes | Windows only |
| `machook` | Offensive/macOS | Yes | macOS only |
| `cipipe` | Offensive | Yes | CI/CD pipeline exploits |
| `ebpf` | Offensive/Linux | Yes | Linux only, requires root |

**Total: ~78 tools** (not counting dynamic MCP tools prefixed `mcp__`)

### Toolset presets

| Preset | Purpose | Key tools |
|---|---|---|
| `none` | No tools | — |
| `minimal` | Read-only file access | read_file, list_files, grep, todo |
| `coding` | Full dev workflow | files, terminal, browser, delegate, schedule |
| `research` | Web + memory + rag | web, memory, rag, browser, delegate |
| `browser` | Browser-first | browser, web, http_request, write_file |
| `security` | Full offensive | all coding + all osint + offensive hooks |
| `osint` | OSINT only | all osint_* + web + http |
| `reverse` | RE tools | re_*, terminal, file tools |
| `vibecoder` | Web dev | browser, terminal, files, web |
| `intercept` | MITM proxy | intercept, browser, http_request |
| `social` | Social media | social_*, email, browser, memory, rag |
| `default` | Everything except offensive hooks | all tools minus attack_*/hook tools |
| `all` | Literally everything | all registered tools |

**ChatUI uses none of these presets.** ChatUI has its own 8-tool set:
`web_search`, `web_fetch`, `read_file`, `list_files`, `grep`, `write_file`, `terminal`, `memory`.

---

## 11. Antares Skills System

**Source:** `internal/tools/skill.go`, `docs/skills.md`

### How skills work in Antares

- Skills are **Markdown files with YAML front matter** stored in `~/.antares/skills/`
- Only `name` + `description` go into the system prompt (catalogue)
- Bodies are loaded on demand via the `skill` tool (`action=read`)
- Agent can save new skills via `skill` tool (`action=save`)
- `skill` tool supports: `list`, `search` (with CWE/tech/category filters), `read`, `chains`, `save`
- `auto_create: true` in config → agent writes skill after solving something non-obvious

### YAML front matter schema

```yaml
name: deploy-homeserver          # kebab-case, unique
description: "When to use..."    # most important — drives retrieval
tags: [deployment, ops]          # for browsing
triggers: [deploy, ship]         # words that surface it
enabled: true                    # false = on disk but not in prompt
```

### ChatUI vs Antares skills

| Aspect | Antares | ChatUI |
|---|---|---|
| Storage | Markdown files in `~/.antares/skills/` | Markdown files in `$RUNTIME_DIR/chatui/skills/` |
| Indexing | FTS5 + category + CWE filters | Loaded into memory on `Reload` |
| Body loading | On-demand via `skill` tool | All loaded at startup |
| Auto-create | Yes (`auto_create` config) | Not implemented |
| Hub | Yes (`/skills install owner/repo`) | Not implemented |
| Library | Thousands of security procedures | User-defined only |
| `chains` | Yes (follow-on techniques) | Not implemented |

---

## 12. Antares Plugin System

**Source:** `internal/tools/hooks.go`, `docs/plugins.md`

### What a plugin is

- External program in `~/.antares/plugins/<name>/plugin.yaml` + executable
- Runs at lifecycle hooks, **not** as a tool the model calls directly
- Protocol: **one JSON object on stdin → one JSON object on stdout**
- Failure (crash/timeout/bad JSON) → logged and skipped, agent continues

### plugin.yaml schema

```yaml
name: audit-log
description: Append every terminal command to a file
version: 1.0.0
command: ./run.sh           # relative to plugin dir, or absolute/PATH
args: []                    # extra argv
hooks: [pre_tool_call]      # which events to receive
timeout_ms: 2000            # default 5000
env:
  LOG_PATH: /var/log/audit.log
```

### Hooks

| Hook | When | Can do |
|---|---|---|
| `pre_tool_call` | Before a tool runs | Deny, rewrite arguments |
| `post_tool_call` | After it returns | Replace the result |
| `session_start` | Conversation begins | Observe |
| `session_end` | Conversation ends | Observe |
| `turn_end` | After each completed turn | Observe |

### Input/output protocol

**stdin:**
```json
{
  "event": "pre_tool_call",
  "session_id": "ses_…",
  "platform": "web",
  "tool": "terminal",
  "arguments": "{\"command\":\"rm -rf build\"}"
}
```

**stdout (all fields optional, `{}` = no opinion):**
```json
{
  "deny": false,
  "reason": "",
  "arguments": "",
  "result": "",
  "notice": ""
}
```

### ChatUI vs Antares plugins

| Aspect | Antares | ChatUI |
|---|---|---|
| Discovery | Directory scan (`~/.antares/plugins/`) | None — hardcoded 3 built-ins |
| External programs | Yes (shell/Python/any executable) | No |
| Hooks | 5 (pre/post tool, session start/end, turn_end) | 2 (before/after tool call) |
| Deny+reason | Yes | Yes (`ApprovalGate`) |
| Rewrite args | Yes | Yes (`ModifiedArgs` in `PluginDecision`) |
| Replace result | Yes | Yes (AfterToolCall return value) |
| Plugin toggle | Yes (per-plugin switch in UI) | Yes (KV `chatui.plugins_disabled`) |
| External program | Yes | No — Go interface only |
| Ordering | Name order | Registration order |

**ChatUI built-in plugins (3):**
1. `approval_gate` — blocks `requires_approval` tools until user approves/denies via UI
2. `output_truncator` — caps tool output at per-tool token limits
3. `web_fetch_cleaner` — strips HTML from web_fetch output

---

## 1. Database Schema (exact DDL from `migrations.go`)

### Key differences from Antares DDL → ChatUI DDL

| Antares | ChatUI | Why |
|---|---|---|
| Timestamps: `BIGINT` unix-ms | `DATETIME DEFAULT (datetime('now'))` | Simpler for SQLite-only |
| `mem_key` column | `key` column | Cleaner name |
| No `schema_version` | `kv` key `schema_version` | Versioned migrations |
| Re-runs ALL DDL on startup | Run only new migrations | Safer |
| `unicode61 remove_diacritics 2` tokenizer | `porter unicode61` | Stemming for history recall |

### `sessions` table (from Antares)

```sql
-- Antares exact DDL (reference):
CREATE TABLE IF NOT EXISTS sessions (
    id            TEXT PRIMARY KEY,
    title         TEXT NOT NULL DEFAULT '',
    platform      TEXT NOT NULL DEFAULT 'web',
    channel_id    TEXT NOT NULL DEFAULT '',
    user_id       TEXT NOT NULL DEFAULT '',
    model         TEXT NOT NULL DEFAULT '',
    provider      TEXT NOT NULL DEFAULT '',
    workspace     TEXT NOT NULL DEFAULT '',
    message_count INTEGER NOT NULL DEFAULT 0,
    tokens_in     BIGINT NOT NULL DEFAULT 0,
    tokens_out    BIGINT NOT NULL DEFAULT 0,
    cost          DOUBLE PRECISION NOT NULL DEFAULT 0,
    archived      BOOLEAN NOT NULL DEFAULT FALSE,
    pinned        BOOLEAN NOT NULL DEFAULT FALSE,
    created_at    BIGINT NOT NULL,   -- unix milliseconds
    updated_at    BIGINT NOT NULL,   -- unix milliseconds
    meta          TEXT NOT NULL DEFAULT '{}'  -- JSON blob
);
CREATE INDEX IF NOT EXISTS idx_sessions_updated ON sessions(updated_at DESC);
CREATE INDEX IF NOT EXISTS idx_sessions_platform ON sessions(platform, user_id);
```

**ChatUI adaptation:** Remove `platform`, `channel_id`, `provider`, `workspace`,
`cost`, `pinned`. Add `tools_enabled` removed (global setting). Keep `user_id`
for future multi-user. Use `DATETIME` not `BIGINT`.

### `messages` table (from Antares)

```sql
-- Antares exact DDL (reference):
CREATE TABLE IF NOT EXISTS messages (
    id           TEXT PRIMARY KEY,
    session_id   TEXT NOT NULL,
    seq          BIGINT NOT NULL,
    role         TEXT NOT NULL,       -- system|user|assistant|tool
    content      TEXT NOT NULL DEFAULT '',
    reasoning    TEXT NOT NULL DEFAULT '',
    tool_calls   TEXT NOT NULL DEFAULT '',   -- JSON array
    tool_call_id TEXT NOT NULL DEFAULT '',
    tool_name    TEXT NOT NULL DEFAULT '',
    attachments  TEXT NOT NULL DEFAULT '',   -- JSON array (images)
    model        TEXT NOT NULL DEFAULT '',
    tokens_in    INTEGER NOT NULL DEFAULT 0,
    tokens_out   INTEGER NOT NULL DEFAULT 0,
    hidden       BOOLEAN NOT NULL DEFAULT FALSE,
    compacted    BOOLEAN NOT NULL DEFAULT FALSE,  -- true = compaction summary
    created_at   BIGINT NOT NULL,
    meta         TEXT NOT NULL DEFAULT '{}'
);
CREATE INDEX IF NOT EXISTS idx_messages_session ON messages(session_id, seq);
CREATE INDEX IF NOT EXISTS idx_messages_created ON messages(created_at DESC);
```

**ChatUI adaptation:** Keep all fields, add `tool_results` (separate from
`tool_calls`), use `DATETIME`, add FK constraint `REFERENCES sessions(id)
ON DELETE CASCADE`.

### `memories` table (from Antares)

```sql
-- Antares exact DDL (reference):
CREATE TABLE IF NOT EXISTS memories (
    id         TEXT PRIMARY KEY,
    scope      TEXT NOT NULL DEFAULT 'global',  -- global|user|session|project
    scope_key  TEXT NOT NULL DEFAULT '',
    mem_key    TEXT NOT NULL DEFAULT '',         -- ChatUI uses 'key'
    content    TEXT NOT NULL,
    tags       TEXT NOT NULL DEFAULT '[]',       -- JSON array
    source     TEXT NOT NULL DEFAULT '',
    pinned     BOOLEAN NOT NULL DEFAULT FALSE,
    created_at BIGINT NOT NULL,
    updated_at BIGINT NOT NULL
);
CREATE INDEX IF NOT EXISTS idx_memories_scope ON memories(scope, scope_key);
```

**ChatUI adaptation:** Rename `mem_key` → `key`. Remove `pinned`. Use
`DATETIME`. Add `UNIQUE(scope, scope_key, key)` for upsert semantics.

### `rag_chunks` table (from Antares — Phase 3+ only)

```sql
-- Antares exact DDL (reference, for Phase 4+):
CREATE TABLE IF NOT EXISTS rag_chunks (
    id          TEXT PRIMARY KEY,
    collection  TEXT NOT NULL,
    doc_id      TEXT NOT NULL DEFAULT '',
    path        TEXT NOT NULL DEFAULT '',
    chunk_index INTEGER NOT NULL DEFAULT 0,
    content     TEXT NOT NULL,
    embedding   BLOB,         -- raw float32 bytes, cosine similarity in Go
    meta        TEXT NOT NULL DEFAULT '{}',
    created_at  BIGINT NOT NULL
);
CREATE INDEX IF NOT EXISTS idx_chunks_collection ON rag_chunks(collection);
```

**NOTE:** ChatUI Phase 1-3 does NOT use this table. Antares's embedding
search is O(n) — loads ALL rows, decodes BLOBs, runs cosine in a loop.
No ANN index. For Phase 4, design a proper index first.

### `kv` table (from Antares)

```sql
-- Antares exact DDL:
CREATE TABLE IF NOT EXISTS kv (
    kv_key     TEXT PRIMARY KEY,
    kv_value   TEXT NOT NULL DEFAULT '',
    updated_at BIGINT NOT NULL
);
```

**ChatUI keys used:**
```
schema_version          "1", "2", ... (ChatUI addition — Antares has no version)
chatui.tools_enabled    "true" | "false"
chatui.workspace_dir    absolute path, default $HOME
chatui.active_session   last active session ID
mcp.servers             JSON array of MCP server configs
```

---

## 2. FTS5 Schema (from Antares `sqliteFTS`)

Antares FTS5 (exact):
```sql
CREATE VIRTUAL TABLE IF NOT EXISTS messages_fts USING fts5(
    content, session_id UNINDEXED, message_id UNINDEXED,
    tokenize='unicode61 remove_diacritics 2'
);
CREATE TRIGGER IF NOT EXISTS messages_fts_ins AFTER INSERT ON messages BEGIN
    INSERT INTO messages_fts(content, session_id, message_id)
    VALUES (new.content, new.session_id, new.id);
END;
CREATE TRIGGER IF NOT EXISTS messages_fts_del AFTER DELETE ON messages BEGIN
    DELETE FROM messages_fts WHERE message_id = old.id;
END;
```

**ChatUI difference:** Use `porter unicode61` tokenizer (adds stemming:
"build"/"built"/"building" all match). Also index `reasoning` column.
Use `content='messages' content_rowid='rowid'` (content table mode) so
FTS stays in sync without duplicating content:

```sql
-- ChatUI FTS5 (what we actually build):
CREATE VIRTUAL TABLE IF NOT EXISTS messages_fts USING fts5(
    content,
    reasoning,
    content='messages',
    content_rowid='rowid',
    tokenize='porter unicode61'
);
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

---

## 3. Migration Pattern

### Antares approach (DO NOT copy):

```go
// antares/internal/store/sql.go — migrate() function:
func (s *sqlStore) migrate(ctx context.Context) error {
    stmts := append([]string{}, migrations...)
    if s.dialect == "sqlite" {
        stmts = append(stmts, sqliteFTS...)
    }
    for _, q := range stmts {
        if _, err := s.db.ExecContext(ctx, q); err != nil {
            // Only ADD COLUMN duplicate errors are tolerated
            if isAddColumn(q) && isDuplicateColumn(err) {
                continue
            }
            return fmt.Errorf("migrate: %w\n%s", err, firstLine(q))
        }
    }
    return nil
}
```

**Problem:** Re-runs ALL DDL every startup. `CREATE TABLE IF NOT EXISTS` is
idempotent, but `ALTER TABLE ADD COLUMN` is not — Antares special-cases it.
The data-repair `UPDATE sessions SET...` correlated subquery also re-runs
every startup, scanning all messages even on a healthy DB.

### ChatUI approach (DO use):

```go
// store/chatui/sqlite.go
func (s *sqliteStore) migrate(ctx context.Context) error {
    // Read current version
    var version int
    row := s.db.QueryRowContext(ctx, `SELECT kv_value FROM kv WHERE kv_key='schema_version'`)
    var vStr string
    if err := row.Scan(&vStr); err == nil {
        version, _ = strconv.Atoi(vStr)
    }

    // Run only new migrations
    for i, migration := range allMigrations {
        if i+1 <= version {
            continue  // already applied
        }
        if _, err := s.db.ExecContext(ctx, migration); err != nil {
            return fmt.Errorf("migration %d: %w", i+1, err)
        }
    }

    // Persist new version
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
```

---

## 4. Go Types (from `types.go`)

```go
// Exact types from Antares internal/store/types.go — copy these for ChatUI:

type Session struct {
    ID           string    `json:"id"`
    Title        string    `json:"title"`
    Platform     string    `json:"platform"`    // ChatUI: always "chatui"
    UserID       string    `json:"user_id"`     // ChatUI: always "local" for now
    Model        string    `json:"model"`
    MessageCount int       `json:"message_count"`
    TokensIn     int64     `json:"tokens_in"`
    TokensOut    int64     `json:"tokens_out"`
    Archived     bool      `json:"archived"`
    CreatedAt    time.Time `json:"created_at"`
    UpdatedAt    time.Time `json:"updated_at"`
}

type Message struct {
    ID          string    `json:"id"`
    SessionID   string    `json:"session_id"`
    Seq         int64     `json:"seq"`
    Role        string    `json:"role"`        // system|user|assistant|tool
    Content     string    `json:"content"`
    Reasoning   string    `json:"reasoning,omitempty"`
    ToolCalls   string    `json:"tool_calls,omitempty"`   // JSON array
    ToolResults string    `json:"tool_results,omitempty"` // JSON array (ChatUI addition)
    Attachments string    `json:"attachments,omitempty"`  // JSON array of images
    Model       string    `json:"model,omitempty"`
    TokensIn    int       `json:"tokens_in"`
    TokensOut   int       `json:"tokens_out"`
    Compacted   bool      `json:"compacted"`
    CreatedAt   time.Time `json:"created_at"`
}

type Memory struct {
    ID        string    `json:"id"`
    Scope     string    `json:"scope"`      // global|user|session
    ScopeKey  string    `json:"scope_key"`
    Key       string    `json:"key"`        // Antares calls this "mem_key"
    Content   string    `json:"content"`
    Source    string    `json:"source"`
    CreatedAt time.Time `json:"created_at"`
    UpdatedAt time.Time `json:"updated_at"`
}
```

---

## 5. Store Methods (from `memory.go`, `sessions.go`)

### PutMemory (upsert pattern from Antares):

```go
// Antares memory.go — exact upsert:
func (s *sqlStore) PutMemory(ctx context.Context, m *Memory) error {
    now := time.Now()
    if m.CreatedAt.IsZero() { m.CreatedAt = now }
    m.UpdatedAt = now
    if m.Tags == "" { m.Tags = "[]" }
    if m.Scope == "" { m.Scope = "global" }
    _, err := s.exec(ctx,
        `INSERT INTO memories (id,scope,scope_key,mem_key,content,tags,source,pinned,created_at,updated_at)
         VALUES (?,?,?,?,?,?,?,?,?,?)` +
        `ON CONFLICT(id) DO UPDATE SET
         content=EXCLUDED.content, mem_key=EXCLUDED.mem_key,
         tags=EXCLUDED.tags, pinned=EXCLUDED.pinned, updated_at=EXCLUDED.updated_at`,
        m.ID, m.Scope, m.ScopeKey, m.Key, m.Content, m.Tags, m.Source, m.Pinned,
        ms(m.CreatedAt), ms(m.UpdatedAt))
    return err
}
```

**ChatUI adaptation:** Use `UNIQUE(scope, scope_key, key)` constraint
instead of `ON CONFLICT(id)` — allows upsert by logical key, not just ID:

```go
// ChatUI store/chatui/sqlite.go
_, err = s.db.ExecContext(ctx,
    `INSERT INTO memories (id,scope,scope_key,key,content,source,created_at,updated_at)
     VALUES (?,?,?,?,?,?,?,?)
     ON CONFLICT(scope,scope_key,key) DO UPDATE SET
     content=excluded.content, source=excluded.source, updated_at=excluded.updated_at`,
    m.ID, m.Scope, m.ScopeKey, m.Key, m.Content, m.Source,
    m.CreatedAt.Format(time.RFC3339), now.Format(time.RFC3339))
```

### ListMemories (from Antares):

```go
// Antares: scope + scope_key filter, pinned first, updated_at desc
func (s *sqlStore) ListMemories(ctx context.Context, scope, scopeKey string, limit int) ([]Memory, error) {
    // WHERE scope=? AND scope_key=? ORDER BY pinned DESC, updated_at DESC LIMIT ?
}
```

---

## 6. Memory Indexing Pipeline (from `ragcontext.go`)

### `indexUserTurn` — the pattern we adapt:

```go
// Antares internal/agent/ragcontext.go — exact logic:
func (a *Agent) indexUserTurn(req Request, userMsg, reply string) {
    if a.rag == nil || !a.config().RAG.PerUser || req.UserID == "" { return }
    collection := rag.UserCollection(req.Platform, req.UserID)
    // ...
    go func() {
        ctx, cancel := context.WithTimeout(context.Background(), 90*time.Second)
        defer cancel()
        // Summarise into durable facts:
        summary := a.summariseUserTurn(ctx, name, userMsg, reply)
        // Fall back to raw exchange if summariser fails:
        content := summary
        if strings.TrimSpace(content) == "" {
            content = "User (" + name + "): " + userMsg
            if reply != "" { content += "\n\nAssistant: " + reply }
        }
        // Store as RAG doc (ChatUI: store as memories table row instead)
        _, _ = a.rag.Index(ctx, collection, []tools.RAGDoc{doc})
    }()
}
```

### `summariseUserTurn` — exact prompt (copy this):

```go
// Antares internal/agent/ragcontext.go — exact summarisation prompt:
prompt := "From this chat exchange, extract only durable facts, preferences, or topics about the user " +
    "named " + name + " that would be worth recalling in a later conversation with them " +
    "(interests, projects, stated preferences, ongoing situations). Write 1-4 short bullet points, " +
    "each a standalone statement. If the exchange reveals nothing worth remembering about the user, reply with exactly NONE.\n\n" +
    "User: " + userMsg + "\n\nAssistant: " + reply

resp, err := client.Chat(ctx, llm.Request{
    Model:       model,
    Messages:    []llm.Message{{Role: llm.RoleUser, Content: prompt}},
    Temperature: 0.2,
    MaxTokens:   300,
})
// If response == "NONE" or empty → don't store
```

**ChatUI adaptation:** Same prompt, but use Anthropic SDK instead of
`llm.Client`. Route via enowX proxy (`base_url = http://localhost:<PORT>`).
Store result as `memories` table rows instead of RAG chunks.

### `autoContext` — the context injection pattern:

```go
// Antares: retrieve relevant past knowledge and format as system-prompt block
func (a *Agent) autoContext(ctx context.Context, req Request, sess *store.Session) string {
    query := strings.TrimSpace(req.Message)
    // Search up to 6 blocks, max 4000 chars total
    const maxBlocks = 6
    const maxChars = 4000
    // Format each hit as: "[path/source] content"
    // Wrap in: "## Relevant context (retrieved)\n\n" + disclaimer
}
```

**ChatUI adaptation:** Replace RAG search with FTS5 query on `messages_fts`.
Same formatting pattern, same max-chars limit (4000 chars ≈ 1000 tokens).

---

## 7. SQLite Connection Setup (from `sql.go`)

```go
// Antares exact SQLite params:
params := []string{
    "_pragma=foreign_keys(1)",
    "_pragma=busy_timeout(5000)",  // 5s busy timeout
}
if wal {
    params = append(params,
        "_pragma=journal_mode(WAL)",
        "_pragma=synchronous(NORMAL)",
    )
}
db, err = sql.Open("sqlite", "file:"+dsn+"?"+strings.Join(params, "&"))

// Connection pool:
db.SetMaxOpenConns(maxConns)   // default 8
db.SetMaxIdleConns(maxConns)
db.SetConnMaxLifetime(time.Hour)
```

**ChatUI:** Use identical params. WAL mode is required for concurrent
reads (HTTP handlers) + background writes (memory indexer).

---

## 8. Antares Migration Bug to Avoid

```sql
-- Antares sqliteFTS — runs every startup, even on healthy DBs:
UPDATE sessions SET
    message_count = (SELECT COUNT(*) FROM messages m WHERE m.session_id = sessions.id),
    tokens_in     = (SELECT COALESCE(SUM(tokens_in), 0) FROM messages m WHERE m.session_id = sessions.id),
    tokens_out    = (SELECT COALESCE(SUM(tokens_out), 0) FROM messages m WHERE m.session_id = sessions.id)
WHERE message_count <> (SELECT COUNT(*) FROM messages m WHERE m.session_id = sessions.id)
```

This is a data-repair query for a bug in Antares. **Do not include this in
ChatUI migrations.** Instead, update `message_count` / `tokens_in` /
`tokens_out` correctly on every `INSERT INTO messages` operation.

---

## 9. Collection Naming (from `rag` package)

```go
// Antares rag/collection.go pattern (not imported, just referenced):
func UserCollection(platform, userID string) string {
    // Returns "user-<platform>-<hash(userID)>"
    // For ChatUI: rag.UserCollection("chatui", "local") = "user-chatui-<hash>"
}
func ProjectCollection(projectDir string) string {
    // Returns "project-<hash(projectDir)>"
}
```

ChatUI Phase 4 RAG uses: `collection = "chatui-history"` for all sessions.

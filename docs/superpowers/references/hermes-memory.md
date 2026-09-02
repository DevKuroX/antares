# Hermes Agent Memory & Session Search Reference — ChatUI Implementation Guide

**Purpose:** Source-of-truth patterns borrowed from Hermes Agent for enowX
ChatUI memory system. Read this before implementing `recall.go`, `memory.go`,
or the system prompt builder.

**Source:** `/home/arch/workspace/hermes-agent/tools/memory_tool.py` and
`/home/arch/workspace/hermes-agent/tools/session_search_tool.py`

---

## 1. The Frozen Snapshot Pattern (most important)

This is the key design decision that makes Hermes memory work correctly.

### How it works (from `memory_tool.py` comments):

```python
# Both files (MEMORY.md and USER.md) are injected into the system prompt
# as a FROZEN SNAPSHOT at session start.
#
# Mid-session writes update files on disk immediately (durable) but do NOT
# change the system prompt -- this preserves the prefix cache for the
# entire session.
#
# The snapshot refreshes on the next session start.
```

### Why frozen matters for ChatUI:

1. **Context stability** — the system prompt does not change mid-session.
   If Section 4 changed every turn, the LLM context would be different
   each call, defeating prefix caching and causing "drift" where Claude
   seems to forget earlier in the same session.

2. **Prefix cache preservation** — Anthropic caches the prompt prefix for
   5 minutes. If Section 4 (memories) changes every turn, the cache is
   invalidated every turn. Frozen = cache hit on every turn after the first.

3. **Correctness** — if memory indexer writes a new fact mid-session, it
   should appear in the NEXT session, not confusingly mid-way through the
   current one.

### ChatUI implementation rule:

```go
// core/chatui/prompt.go

type SessionContext struct {
    // Built ONCE at session start, never mutated
    SystemPrompt string  // frozen for the entire session
    // ...
}

func BuildSessionContext(ctx context.Context, sessionID, userMessage string) (*SessionContext, error) {
    memories := loadMemories(ctx, sessionID)     // from memories table
    history  := recallHistory(ctx, userMessage)  // from messages_fts

    prompt := buildSystemPrompt(memories, history)  // frozen snapshot

    return &SessionContext{
        SystemPrompt: prompt,
        // store in session — never rebuild mid-session
    }, nil
}

// In the agentic loop:
// - systemPrompt is read from SessionContext
// - NEVER call BuildSessionContext again mid-session
// - mid-session memory writes (from indexer) → disk only, not to prompt
```

---

## 2. Memory Storage Design (from `memory_tool.py`)

### Hermes approach: two `.md` files with `§` delimiter

```python
ENTRY_DELIMITER = "\n§\n"

# Two separate stores:
# ~/.hermes/memories/MEMORY.md  — agent's personal notes (~2200 chars)
# ~/.hermes/memories/USER.md    — user profile (~1375 chars)

class MemoryStore:
    def __init__(self, memory_char_limit=2200, user_char_limit=1375):
        self.memory_entries: List[str] = []   # live state
        self.user_entries: List[str] = []     # live state
        self._system_prompt_snapshot: Dict[str, str] = {
            "memory": "",
            "user": ""
        }  # frozen at load_from_disk(), never changed mid-session
```

### ChatUI approach: `memories` table (analogous, not files)

```go
// Equivalent to Hermes MEMORY.md + USER.md, but in SQLite:
//
// MEMORY.md entries → memories WHERE scope="session" AND scope_key=sessionID
// USER.md entries   → memories WHERE scope="user" AND scope_key="local"
//
// Char limits equivalent:
const (
    MaxSessionMemoryChars = 2200  // from Hermes memory_char_limit
    MaxUserMemoryChars    = 1375  // from Hermes user_char_limit
    MaxTotalMemoryTokens  = 200   // hard token cap for Section 4a
)
```

### System prompt rendering (from `_render_block`):

```python
# Hermes exact rendering:
def _render_block(self, target: str, entries: List[str]) -> str:
    if not entries:
        return ""
    limit = self._char_limit(target)
    content = ENTRY_DELIMITER.join(entries)
    current = len(content)
    pct = min(100, int((current / limit) * 100)) if limit > 0 else 0

    if target == "user":
        header = f"USER PROFILE (who the user is) [{pct}% — {current:,}/{limit:,} chars]"
    else:
        header = f"MEMORY (your personal notes) [{pct}% — {current:,}/{limit:,} chars]"

    separator = "═" * 46
    return f"{separator}\n{header}\n{separator}\n{content}"
```

**ChatUI equivalent:**

```go
// core/chatui/prompt.go
func renderMemoryBlock(memories []store.Memory) string {
    if len(memories) == 0 {
        return ""
    }
    var b strings.Builder
    b.WriteString("## What you remember\n\n")
    for _, m := range memories {
        b.WriteString(fmt.Sprintf("- %s: %s\n", m.Key, m.Content))
    }
    return b.String()
}
```

---

## 3. Memory Char Budget (from `MemoryStore.__init__`)

```python
# Hermes exact limits:
memory_char_limit = 2200   # ~/.hermes/memories/MEMORY.md
user_char_limit   = 1375   # ~/.hermes/memories/USER.md
# Total: ~3575 chars ≈ ~900 tokens
```

**ChatUI:** Hard-cap Section 4a at **200 tokens** (~800 chars). Load at most:
- 10 user-scoped memories (`scope="user", scope_key="local"`)
- 5 session-scoped memories (`scope="session", scope_key=sessionID`)
- Truncate each entry to 80 chars if needed

---

## 4. At-Capacity Consolidation Guard

Hermes has a critical guard against infinite loops when memory is full:

```python
# memory_tool.py — prevents model from looping on "memory full" errors:
_MAX_CONSOLIDATION_FAILURES_PER_TURN = 3

def _consolidation_failure(self, response):
    self._consolidation_failures += 1
    if self._consolidation_failures <= self._MAX_CONSOLIDATION_FAILURES_PER_TURN:
        return response  # tell model to retry
    # After 3 failures: terminal response — stop retrying
    return {
        "success": False,
        "done": True,
        "error": (
            "Memory consolidation failed 3 times this turn. "
            "Stop retrying memory calls — leave memory unchanged for now."
        ),
    }
```

**ChatUI:** The `memory` tool must return a terminal error (not a retry
instruction) after 3 failed save attempts in one turn. Otherwise Claude
will loop on memory calls and never reply to the user.

---

## 5. Drift Detection (from `_detect_external_drift`)

Hermes protects against silent data loss when the memory file was edited
externally (by another session, shell append, or patch tool):

```python
def _detect_external_drift(self, target: str) -> Optional[str]:
    """Return backup path if on-disk content won't round-trip through parser."""
    path = self._path_for(target)
    raw = path.read_text(encoding="utf-8")
    parsed = [e.strip() for e in raw.split(ENTRY_DELIMITER) if e.strip()]
    roundtrip = ENTRY_DELIMITER.join(parsed)

    char_limit = self._char_limit(target)
    max_entry_len = max((len(e) for e in parsed), default=0)

    # Drift: either bytes don't roundtrip, or a single entry > total budget
    drift_detected = (raw.strip() != roundtrip) or (max_entry_len > char_limit)
    if drift_detected:
        # Save .bak file, refuse write, return path
        bak_path = path.with_suffix(path.suffix + f".bak.{ts}")
        bak_path.write_text(raw, encoding="utf-8")
        return str(bak_path)
    return None
```

**ChatUI equivalent:** SQLite handles this automatically via WAL mode +
atomic writes. No drift detection needed for DB-backed storage.

---

## 6. File Lock Pattern (from `_file_lock`)

Hermes uses fcntl file locking to prevent concurrent session corruption:

```python
@contextmanager
def _file_lock(path: Path):
    """Exclusive file lock for read-modify-write safety."""
    lock_path = path.with_suffix(path.suffix + ".lock")
    fd = open(lock_path, "a+", encoding="utf-8")
    try:
        fcntl.flock(fd, fcntl.LOCK_EX)
        yield
    finally:
        fcntl.flock(fd, fcntl.LOCK_UN)
        fd.close()
```

**ChatUI equivalent:** SQLite WAL mode + busy_timeout(5000ms) handles this.
No manual locking needed. The Go `database/sql` pool serializes writes.

---

## 7. Session Search — FTS5 Pattern (from `session_search_tool.py`)

This is the direct reference for `core/chatui/recall.go`.

### Three shapes (from docstring):

```
1. DISCOVERY — pass query
   FTS5 search → dedup by session lineage → top-N sessions with:
   - snippet: FTS5-highlighted excerpt
   - bookend_start: first 3 user+assistant messages (the goal)
   - messages: ±5 messages around the FTS5 match (the hit in context)
   - bookend_end: last 3 user+assistant messages (the resolution)

2. SCROLL — pass session_id + around_message_id
   Returns ±window messages centered on anchor. No FTS5.

3. BROWSE — no args
   Returns recent sessions chronologically.
```

### FTS5 query call (exact Python):

```python
raw_results = db.search_messages(
    query=query,
    role_filter=["user", "assistant"],  # exclude tool output (noise)
    exclude_sources=list(_HIDDEN_SESSION_SOURCES),  # exclude subagents/cron
    limit=300,   # scan wide before dedup
    offset=0,
    sort=sort,   # "newest" | "oldest" | None (relevance)
)
```

**ChatUI Go equivalent:**

```go
// core/chatui/recall.go
func (r *Recall) Search(ctx context.Context, query string) ([]HistoryChunk, error) {
    rows, err := r.db.QueryContext(ctx, `
        SELECT
            m.content,
            m.role,
            m.created_at,
            s.title,
            s.id as session_id
        FROM messages_fts fts
        JOIN messages m ON m.rowid = fts.rowid
        JOIN sessions s ON s.id = m.session_id
        WHERE messages_fts MATCH ?
          AND m.compacted = 0
          AND m.role IN ('user', 'assistant')
        ORDER BY rank                    -- FTS5 BM25 relevance rank
        LIMIT 8
    `, sanitizeFTSQuery(query))
    // ...
}
```

### FTS5 syntax (from SESSION_SEARCH_SCHEMA description):

```
AND is default — "auth refactor" requires both terms
OR for broader recall: "alpha OR beta OR gamma"
Phrase: "docker networking"
Boolean: "python NOT java"
Prefix wildcard: "deploy*"
```

**ChatUI:** Pass user's message as-is to FTS5. Add `*` suffix to each
word for prefix matching:

```go
func sanitizeFTSQuery(q string) string {
    // "building sidebar" → "building* sidebar*"
    // Removes FTS5 special chars except * for safety
    words := strings.Fields(q)
    for i, w := range words {
        w = strings.Map(func(r rune) rune {
            if unicode.IsLetter(r) || unicode.IsDigit(r) { return r }
            return -1
        }, w)
        if w != "" { words[i] = w + "*" }
    }
    return strings.Join(words, " ")
}
```

### Dedup by session lineage (from `_discover`):

```python
# Hermes: multiple FTS hits in the same session → keep only one (best hit)
seen_sessions = {}
for r in raw_results:
    resolved_sid = _resolve_to_parent(db, r["session_id"])
    if current_lineage_root and resolved_sid == current_lineage_root:
        continue  # skip current session — already in context
    if resolved_sid not in seen_sessions:
        seen_sessions[resolved_sid] = r
    if len(seen_sessions) >= limit:
        break
```

**ChatUI Go equivalent:**

```go
seen := map[string]bool{}
var chunks []HistoryChunk
for rows.Next() {
    // scan row...
    if seen[sessionID] { continue }  // one chunk per session
    if sessionID == currentSessionID { continue }  // skip current session
    seen[sessionID] = true
    chunks = append(chunks, chunk)
    if len(chunks) >= 2 { break }  // top-2 sessions max
}
```

### Automation demotion (from `_order_for_recall`):

```python
# Hermes demotes cron/automation sessions in search results:
_DEMOTED_SESSION_SOURCES = ("cron",)
_HIDDEN_SESSION_SOURCES = ("subagent", "tool")

# Sort: interactive sessions rank above automation (stable sort)
return sorted(results, key=lambda r: 1 if r.get("source") in _DEMOTED_SESSION_SOURCES else 0)
```

**ChatUI:** No cron/subagent sources in ChatUI — all sessions are interactive.
No demotion needed. But note the pattern for future multi-source expansion.

---

## 8. System Prompt Section Headers

Hermes exact section format rendered by `_render_block`:

```
══════════════════════════════════════════════
MEMORY (your personal notes) [42% — 924/2,200 chars]
══════════════════════════════════════════════
<entry1>
§
<entry2>
§
<entry3>
```

```
══════════════════════════════════════════════
USER PROFILE (who the user is) [31% — 430/1,375 chars]
══════════════════════════════════════════════
<entry1>
§
<entry2>
```

**ChatUI Section 4 format:**

```
## What you remember

- prefers TypeScript strict mode
- ChatSidebar uses GlideMenu, collapsed=52px, expanded=224px
- deploy: dev :1431, prod :1430

## Relevant past context

[2026-09-01 | ChatUI Sidebar build] (assistant) Built ChatSidebar with
GlideMenu as session list. Collapsed width 52px, expanded 224px. Toggle
button moved to floating header...

[2026-08-30 | enowX deploy] (user) How do I deploy to port 1431?
```

---

## 9. Memory Write Tool Schema (from `MEMORY_SCHEMA`)

The `memory` tool in ChatUI follows this exact Hermes schema — copy it:

```json
{
  "name": "memory",
  "description": "Save durable facts to persistent memory that survive across sessions. Memory is injected into every future turn, so keep entries compact and high-signal.\n\nHOW: make ALL your changes in ONE call via an 'operations' array (each item: {action, content?, old_text?}). The batch applies atomically...\n\nWHEN: save proactively when the user states a preference, correction, or personal detail, or you learn a stable fact about their environment...\n\nSKIP: trivial/obvious info, easily re-discovered facts, raw data dumps, task progress...",
  "parameters": {
    "type": "object",
    "properties": {
      "action": { "type": "string", "enum": ["add", "replace", "remove"] },
      "target": { "type": "string", "enum": ["memory", "user"] },
      "content": { "type": "string" },
      "old_text": { "type": "string", "description": "Short unique substring of entry to replace/remove" },
      "operations": {
        "type": "array",
        "description": "Batch: list of {action, content?, old_text?} applied atomically",
        "items": {
          "type": "object",
          "properties": {
            "action": { "type": "string", "enum": ["add", "replace", "remove"] },
            "content": { "type": "string" },
            "old_text": { "type": "string" }
          },
          "required": ["action"]
        }
      }
    },
    "required": ["target"]
  }
}
```

**Key design notes from Hermes:**
- `target: "memory"` → session-scoped facts (scope="session")
- `target: "user"` → user profile facts (scope="user")
- Batch `operations` array applies atomically — budget checked on FINAL state only
- `old_text` is a substring match, not exact match — shorter is better
- Success response is TERMINAL: includes `"done": true`, does NOT echo entries
  (prevents model from finding "more to fix" and looping)

---

## 10. Threat Pattern Scanning (from `_scan_memory_content`)

Hermes scans memory writes for prompt injection before accepting:

```python
from tools.threat_patterns import first_threat_message as _first_threat_message

def _scan_memory_content(content: str) -> Optional[str]:
    """Scan memory content for injection/exfil patterns."""
    return _first_threat_message(content, scope="strict")
```

Memory entries enter the system prompt as a frozen snapshot every session.
A poisoned entry would persist across ALL future sessions until removed.
This makes memory a high-value injection target.

**ChatUI:** Implement basic threat scanning before writing to `memories` table.
Minimum: reject entries containing system prompt override patterns like
`"Ignore all previous instructions"`, `"You are now"`, etc.

---

## 11. Key Numbers to Use

From Hermes source (use these exact values in ChatUI):

| Parameter | Hermes value | ChatUI value |
|---|---|---|
| Memory char limit (personal) | 2200 chars | 200 tokens (Section 4a) |
| Memory char limit (user profile) | 1375 chars | included in 200 tokens |
| Max consolidation failures/turn | 3 | 3 |
| FTS scan limit before dedup | 300 rows | 50 rows (smaller dataset) |
| Max sessions returned | 10 | 2 sessions, 4 chunks each |
| Message window around FTS hit | ±5 messages | ±2 messages |
| Bookend messages | 3 start + 3 end | 1 start + 1 end |
| Total history context cap | ~4000 chars | 800 tokens (Section 4b) |
| Grand total Section 4 | ~900 tokens Hermes | 1000 tokens ChatUI |

---

## 12. What Hermes Does NOT Have That We Build

| Feature | Hermes | ChatUI |
|---|---|---|
| Cross-session FTS recall | `session_search` tool (agent calls manually) | **Automatic** — called at session start, injected as frozen context |
| Memory persistence | `.md` files with `§` delimiter | SQLite `memories` table |
| Schema migrations | N/A (files, no DB schema) | Versioned migrations with `schema_version` KV |
| Tool approval gate | `write_approval.py` module | `ApprovalGate` plugin middleware |
| Background indexing | Manual (agent calls `memory` tool) | **Automatic** fire-and-forget goroutine |

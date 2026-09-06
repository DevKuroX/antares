# ChatUI Conformance Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Close all 29 audit findings from `docs/chatui-implementation-audit.md` and bring the ChatUI implementation into full conformance with the final design spec.

**Architecture:** ChatUI sits on top of the enowX proxy. Agent uses Anthropic SDK pointing to localhost enowX. SDK must stream (`Messages.NewStreaming`). MCP client is a new Go package. All wiring closes the backend ↔ SSE ↔ frontend gaps.

**Tech Stack:** Go (Anthropic SDK, modernc/sqlite, chi), React/TypeScript (Vite), Playwright E2E

**Spec:** `/home/arch/workspace/antares/docs/superpowers/specs/2026-09-02-chatui-memory-system-design.md`
**Audit:** `docs/chatui-implementation-audit.md`

## Global Constraints

- NEVER touch enowX core proxy (`/v1/*`, `/anthropic/v1/*`), account pool, model routing, port 1430
- Anthropic SDK base URL: `http://localhost:<ENOWX_PORT>/anthropic/` (stays unchanged)
- `CGO_ENABLED=0` — pure Go, no C libraries
- All tests pass with `go test ./... -race` before declaring any task done
- No dummy/stub/fake/placeholder code left in production paths
- Spec §9.2 plugin interface is the authoritative contract (not the current implementation)
- Spec §10.2 requires `Messages.NewStreaming` (actual streaming, not simulated)
- SSE field names that differ from spec are implementation-correct (audit F-06 ruling: implementation wins, spec will be updated in Task 10)
- Default model `bob/premium` per spec §11.1
- `MCP client not yet implemented` stub must be replaced with real implementation
- `go build -tags dev ./...` must pass after every task
- All Go test files use package-level `TestMain` or `t.Parallel()` where appropriate

---

## Dependency order

```
T1 (F-26/F-27 DB fixes) — no deps
T2 (F-09/F-28 backend fixes: memory scope + skills) — no deps
T3 (F-22/F-23 plugin/tool fixes) — deps: none (interface preserved until T4)
T4 (F-21 plugin interface migration) — deps: T3 (plugins must be updated together)
T5 (F-01 SDK streaming) — deps: T4 (agent interface change)
T6 (F-10 default model + F-25 per-model context) — no deps (migration)
T7 (F-03 MCP client) — deps: T4 (plugin interface stable)
T8 (F-02/F-04/F-07/F-08/F-14 frontend wiring) — deps: T5 (streaming SSE shape)
T9 (F-17/F-18/F-24 tests + token counting) — deps: T5 (streaming), T7 (MCP)
T10 (spec update + docs) — deps: all above
```

---

### Task 1: Fix DB integrity (F-26, F-27)

**Files:**
- Modify: `store/chatui/sqlite.go`
- Test: `store/chatui/sqlite_test.go`

**Interfaces:**
- Consumes: existing `*sqliteStore` methods
- Produces: correct transaction isolation, correct `message_count` after compaction

Fix two low-severity but correctness-affecting database bugs.

- [ ] **Step 1: Fix F-26 — transaction isolation**

In `sqlite.go`, find all `s.db.BeginTx(ctx, nil)`. The `nil` options means DEFERRED — it does not respect the DSN `_txlock=immediate`.

Change every `BeginTx(ctx, nil)` to pass explicit serializable options:
```go
tx, err := s.db.BeginTx(ctx, &sql.TxOptions{Isolation: sql.LevelSerializable})
```

There should be 2 occurrences: one in `AppendMessage` (seq allocation), one in `CompactMessages`. Fix both.

- [ ] **Step 2: Fix F-27 — message_count not decremented after compaction**

In `CompactMessages()` in `sqlite.go`, after updating compacted rows, add a decrement:

```go
// After the UPDATE messages SET compacted=TRUE, content=... statements:
_, err = tx.ExecContext(ctx,
    `UPDATE sessions SET message_count = message_count - ? WHERE id = ?`,
    len(messageIDs)-1, // keep 1 summary row, remove the rest from count
    sessionID,         // must derive sessionID from first messageID lookup
)
```

Note: `CompactMessages` takes `messageIDs []string`. You need to derive `sessionID` from one of those IDs. Add a query:
```go
var sessionID string
err = tx.QueryRowContext(ctx, `SELECT session_id FROM messages WHERE id = ?`, messageIDs[0]).Scan(&sessionID)
```

Then run the UPDATE. The count decrease is `len(messageIDs)-1` because the first row becomes a summary (still a meaningful message), while the remaining rows are cleared empty.

- [ ] **Step 3: Write test for F-27**

In `sqlite_test.go`, add `TestCompactMessagesDecrementsCount`:

```go
func TestCompactMessagesDecrementsCount(t *testing.T) {
    db := newTestDB(t)
    sess := createTestSession(t, db, "test")
    
    // Insert 15 messages
    ids := make([]string, 15)
    for i := range ids {
        msg := chatstore.Message{
            ID: fmt.Sprintf("msg-%d", i), SessionID: sess.ID,
            Role: "user", Content: fmt.Sprintf("message %d", i),
        }
        require.NoError(t, db.AppendMessage(context.Background(), msg))
        ids[i] = msg.ID
    }
    
    // Verify count before
    s, _ := db.GetSession(context.Background(), sess.ID)
    require.Equal(t, 15, s.MessageCount)
    
    // Compact first 10
    compactIDs := ids[:10]
    require.NoError(t, db.CompactMessages(context.Background(), compactIDs, "Summary text"))
    
    // message_count should decrease by len(compactIDs)-1 = 9
    s, _ = db.GetSession(context.Background(), sess.ID)
    require.Equal(t, 6, s.MessageCount) // 15 - 9 = 6
}
```

Also add `TestCompactMessagesTransactionIsolation` to verify concurrent writes don't corrupt seq:

```go
func TestAppendMessageConcurrentSeq(t *testing.T) {
    db := newTestDB(t)
    sess := createTestSession(t, db, "concurrent")
    
    const N = 20
    errs := make(chan error, N)
    for i := 0; i < N; i++ {
        go func(i int) {
            msg := chatstore.Message{
                ID: fmt.Sprintf("c-msg-%d", i), SessionID: sess.ID,
                Role: "user", Content: fmt.Sprintf("concurrent %d", i),
            }
            errs <- db.AppendMessage(context.Background(), msg)
        }(i)
    }
    for i := 0; i < N; i++ {
        require.NoError(t, <-errs)
    }
    
    msgs, _ := db.ListMessages(context.Background(), sess.ID, 100, 0)
    require.Equal(t, N, len(msgs))
    // Verify seqs are unique 1..N
    seqs := make(map[int]bool)
    for _, m := range msgs {
        require.False(t, seqs[m.Seq], "duplicate seq %d", m.Seq)
        seqs[m.Seq] = true
    }
}
```

- [ ] **Step 4: Run tests**

```bash
GOPATH=/home/arch/go GOCACHE=/home/arch/.cache/go CGO_ENABLED=0 go test ./store/chatui/... -race -count=1 -v 2>&1 | tail -30
```

Expected: all tests pass including new ones.

- [ ] **Step 5: Commit**

```bash
git add store/chatui/sqlite.go store/chatui/sqlite_test.go
git commit -m "fix(chatui/db): correct tx isolation + fix message_count after compaction (F-26, F-27)"
```

---

### Task 2: Fix memory recall + skills front-matter (F-09, F-28)

**Files:**
- Modify: `core/chatui/prompt.go`
- Modify: `core/chatui/skills.go`
- Test: `core/chatui/agent_test.go` (add memory recall test)

**Interfaces:**
- Consumes: `ChatStore.ListMemories(ctx, scope, scopeKey, limit)` (unchanged)
- Produces: correct user-scoped memory injection, `enabled` front-matter parsed

- [ ] **Step 1: Fix F-09 — memory scope**

In `core/chatui/prompt.go`, replace the `memoriesSection` function:

```go
func memoriesSection(ctx context.Context, store chatstore.ChatStore, sessionID string) string {
    // Spec §12 Section 4a: last 10 user-scoped + 5 session-scoped memories
    userMems, _ := store.ListMemories(ctx, "user", "local", 10)
    sessionMems, _ := store.ListMemories(ctx, "session", sessionID, 5)

    all := append(userMems, sessionMems...)
    if len(all) == 0 {
        return ""
    }
    return formatMemories("What you remember", all)
}
```

This fixes the core bug: the indexer writes `scope="user", scope_key="local"` and `scope="session", scope_key=sessionID`. The recall now reads both correctly.

- [ ] **Step 2: Fix F-28 — `enabled` front-matter field**

In `core/chatui/skills.go`, in the `parseSkillFile` function, add:
```go
case "enabled":
    skill.Enabled = val == "true" || val == "1" || val == "yes"
```

to the front-matter switch statement. The default for `Enabled` in `Skill` struct is already `true` (from the struct tag), so this change only affects files that explicitly set `enabled: false`.

- [ ] **Step 3: Add memory recall test**

In `core/chatui/agent_test.go` (or a new `core/chatui/memory_recall_test.go`), add:

```go
func TestMemoriesSectionRecallsUserScope(t *testing.T) {
    db, cleanup := testChatDB(t)
    defer cleanup()

    ctx := context.Background()
    sessionID := "sess-recall-test"

    // Write user-scoped memory (as the indexer does)
    require.NoError(t, db.PutMemory(ctx, chatstore.Memory{
        ID:       "m1",
        Scope:    "user",
        ScopeKey: "local",
        Key:      "user-pref",
        Content:  "User prefers dark mode",
    }))
    // Write session-scoped memory
    require.NoError(t, db.PutMemory(ctx, chatstore.Memory{
        ID:       "m2",
        Scope:    "session",
        ScopeKey: sessionID,
        Key:      "session-fact",
        Content:  "User is building a Go API",
    }))
    // Write global-scoped memory — should NOT appear
    require.NoError(t, db.PutMemory(ctx, chatstore.Memory{
        ID:       "m3",
        Scope:    "global",
        ScopeKey: "",
        Key:      "global-fact",
        Content:  "SHOULD NOT APPEAR",
    }))

    cfg := Config{Store: db}
    section := memoriesSection(ctx, cfg.Store, sessionID)

    require.Contains(t, section, "User prefers dark mode", "user-scoped memory must appear")
    require.Contains(t, section, "User is building a Go API", "session-scoped memory must appear")
    require.NotContains(t, section, "SHOULD NOT APPEAR", "global-scoped memory must not appear")
}
```

Note: `memoriesSection` is an unexported function in the `chatui` package, so this test goes in `package chatui` (same package, file `core/chatui/prompt_test.go`).

- [ ] **Step 4: Add skills enabled front-matter test**

In `core/chatui/skills.go` test (or new file `core/chatui/skills_test.go`):

```go
func TestParseSkillFileEnabledFalse(t *testing.T) {
    tmp := t.TempDir()
    path := filepath.Join(tmp, "disabled-skill.md")
    content := "---\nname: disabled-skill\ndescription: A disabled skill\nenabled: false\n---\n# Content\n"
    require.NoError(t, os.WriteFile(path, []byte(content), 0644))

    skill, err := parseSkillFile(path)
    require.NoError(t, err)
    require.Equal(t, "disabled-skill", skill.Name)
    require.False(t, skill.Enabled, "enabled: false must set Enabled=false")
}

func TestParseSkillFileEnabledDefault(t *testing.T) {
    tmp := t.TempDir()
    path := filepath.Join(tmp, "no-enabled.md")
    content := "---\nname: no-enabled-field\ndescription: Default enabled\n---\n"
    require.NoError(t, os.WriteFile(path, []byte(content), 0644))

    skill, err := parseSkillFile(path)
    require.NoError(t, err)
    require.True(t, skill.Enabled, "missing enabled field must default to true")
}
```

- [ ] **Step 5: Run tests**

```bash
GOPATH=/home/arch/go GOCACHE=/home/arch/.cache/go CGO_ENABLED=0 go test ./core/chatui/... -race -count=1 -v -run "TestMemories|TestParseSkill|TestRecall" 2>&1 | tail -30
```

- [ ] **Step 6: Commit**

```bash
git add core/chatui/prompt.go core/chatui/skills.go core/chatui/
git commit -m "fix(chatui): memory recall uses user+session scope (F-09), parse enabled front-matter (F-28)"
```

---

### Task 3: Per-tool output limits + AfterToolCall order (F-22, F-23)

**Files:**
- Modify: `core/chatui/tools.go` (add per-tool cap constants)
- Modify: `core/chatui/tools_web.go`, `tools_file.go`, `tools_term.go`, `tools_memory.go`
- Modify: `core/chatui/plugins.go` (fix AfterToolCall loop order)
- Test: `core/chatui/agent_test.go` (add per-tool cap test + plugin order test)

**Interfaces:**
- Consumes: `Tool.Execute()` return string — add truncation inside each tool's Execute
- Produces: each tool returns at most its spec-defined token cap

**Note:** F-23 (AfterToolCall reverse order) is SAFE to fix. The fix is a forward loop instead of reverse. The current behavior (reverse) would only matter if future plugins depend on ordering. The 3 current plugins are: `[ApprovalGate, OutputTruncator, WebFetchCleaner]`. In reverse: `WebFetchCleaner → OutputTruncator → ApprovalGate.AfterToolCall(no-op)`. In forward: `ApprovalGate.AfterToolCall(no-op) → OutputTruncator → WebFetchCleaner`. Both are correct for current plugins since `ApprovalGate.AfterToolCall` is a no-op. Fix to match spec.

- [ ] **Step 1: Add per-tool output cap constants to `tools.go`**

```go
// Per-tool output token caps (spec §6.1)
const (
    MaxMemoryOutputTokens    = 500
    MaxWebSearchOutputTokens = 1500
    MaxWebFetchOutputTokens  = 3000
    MaxReadFileOutputTokens  = 2000
    MaxWriteFileOutputTokens = 100
    MaxListFilesOutputTokens = 500
    MaxGrepOutputTokens      = 1000
    MaxTerminalOutputTokens  = 3000
)
```

- [ ] **Step 2: Apply per-tool truncation inside each tool's `Execute()`**

In `tools_memory.go`, at the end of each action branch, wrap the return:
```go
return truncateToTokens(result, MaxMemoryOutputTokens), nil
```

In `tools_web.go` `webSearchTool.Execute()`:
```go
return truncateToTokens(result, MaxWebSearchOutputTokens), nil
```

In `tools_web.go` `webFetchTool.Execute()` — already has a char-level cap. Add token-level cap after char truncation:
```go
return truncateToTokens(text, MaxWebFetchOutputTokens), nil
```

In `tools_file.go` `readFileTool.Execute()`:
```go
return truncateToTokens(result, MaxReadFileOutputTokens), nil
```

In `tools_file.go` `writeFileTool.Execute()`:
```go
return truncateToTokens(result, MaxWriteFileOutputTokens), nil
```

In `tools_file.go` `listFilesTool.Execute()`:
```go
return truncateToTokens(result, MaxListFilesOutputTokens), nil
```

In `tools_file.go` `grepTool.Execute()`:
```go
return truncateToTokens(result, MaxGrepOutputTokens), nil
```

In `tools_term.go` `terminalTool.Execute()`:
```go
return truncateToTokens(combined, MaxTerminalOutputTokens), nil
```
(This tool already truncates to `MaxToolOutputTokens = 3000`, which equals `MaxTerminalOutputTokens`. The constant change makes it explicit.)

- [ ] **Step 3: Fix F-23 — AfterToolCall forward loop order**

In `core/chatui/plugins.go`, change:
```go
for i := len(plugins) - 1; i >= 0; i-- {
```
to:
```go
for i := 0; i < len(plugins); i++ {
```

Update the comment from "reverse order" to "forward (registration) order per spec §9.4".

- [ ] **Step 4: Add tests**

Add `TestPerToolOutputCap` in a new file `core/chatui/tools_test.go`:

```go
func TestMemoryToolOutputCap(t *testing.T) {
    // Memory tool should never return more than 500 tokens worth
    tool := memoryTool{store: newTestMemStore(t)}
    // Seed 100 very long memories
    for i := 0; i < 100; i++ {
        input, _ := json.Marshal(map[string]any{
            "action": "save", "key": fmt.Sprintf("key-%d", i),
            "value": strings.Repeat("x", 200),
        })
        tool.Execute(context.Background(), input)
    }
    input, _ := json.Marshal(map[string]any{"action": "list"})
    result, err := tool.Execute(context.Background(), input)
    require.NoError(t, err)
    tokens := estimateTokens(result)
    require.LessOrEqual(t, tokens, MaxMemoryOutputTokens+10, "memory output exceeds cap")
}

func TestAfterToolCallForwardOrder(t *testing.T) {
    order := []string{}
    p1 := &orderPlugin{name: "p1", order: &order}
    p2 := &orderPlugin{name: "p2", order: &order}
    p3 := &orderPlugin{name: "p3", order: &order}
    plugins := []Plugin{p1, p2, p3}
    ExecuteToolAfter(context.Background(), plugins, Tool(nil), nil, "result")
    require.Equal(t, []string{"p1", "p2", "p3"}, order)
}
```

(Implement `orderPlugin` as a test helper that records call order.)

- [ ] **Step 5: Run tests**

```bash
GOPATH=/home/arch/go GOCACHE=/home/arch/.cache/go CGO_ENABLED=0 go test ./core/chatui/... -race -count=1 2>&1 | tail -20
```

- [ ] **Step 6: Commit**

```bash
git add core/chatui/tools.go core/chatui/tools_*.go core/chatui/plugins.go core/chatui/tools_test.go
git commit -m "fix(chatui): per-tool output caps (F-22), AfterToolCall forward order (F-23)"
```

---

### Task 4: Align plugin interface with spec §9.2 (F-21)

**Files:**
- Modify: `core/chatui/plugins.go`
- Modify: `core/chatui/agent.go` (call sites use new return type)

**Interfaces:**
- Before: `BeforeToolCall(...) (skipOutput string, err error)`
- After: `BeforeToolCall(...) (PluginDecision, error)`
- `PluginDecision` has `Allow bool`, `DenyReason string`, `ModifiedArgs json.RawMessage`
- `AfterToolCall` signature UNCHANGED (already matches spec)

**Note:** The existing `BeforeToolCall` carries `cfg Config` and `w EventWriter` — these are needed for `ApprovalGate` and are NOT in spec §9.2. Ruling: KEEP `cfg Config` and `w EventWriter` in the signature because without them, `ApprovalGate` cannot function. The spec's interface omitted these because it was written at design time before the approval gate's real needs were clear. The change is ONLY in the return type (`PluginDecision` replaces `string`).

- [ ] **Step 1: Add `PluginDecision` type and update interface**

In `plugins.go`, add:

```go
// PluginDecision is returned by BeforeToolCall.
type PluginDecision struct {
    Allow        bool
    DenyReason   string          // non-empty when Allow=false
    ModifiedArgs json.RawMessage // non-nil to replace the original tool args
}
```

Change the `Plugin` interface:
```go
type Plugin interface {
    Name() string
    // BeforeToolCall is called before tool execution.
    // Return PluginDecision.Allow=false to block execution (with DenyReason).
    // Return non-nil ModifiedArgs to replace the tool input.
    // cfg and w are provided so approval-gate plugins can emit SSE events.
    BeforeToolCall(ctx context.Context, cfg Config, tool Tool, input json.RawMessage, w EventWriter) (PluginDecision, error)
    // AfterToolCall transforms the tool output. Plugins run in registration order.
    AfterToolCall(ctx context.Context, tool Tool, input json.RawMessage, output string) (string, error)
}
```

- [ ] **Step 2: Update `ExecuteTool` to use `PluginDecision`**

The current `ExecuteTool` checks `skip != ""`. Replace with:

```go
func ExecuteTool(ctx context.Context, cfg Config, plugins []Plugin, tool Tool, input json.RawMessage, w EventWriter) (string, error) {
    // Before hooks
    for _, p := range plugins {
        decision, err := p.BeforeToolCall(ctx, cfg, tool, input, w)
        if err != nil {
            return "", fmt.Errorf("plugin %s before: %w", p.Name(), err)
        }
        if !decision.Allow {
            if decision.DenyReason != "" {
                return decision.DenyReason, nil
            }
            return "execution denied", nil
        }
        if len(decision.ModifiedArgs) > 0 {
            input = decision.ModifiedArgs  // plugin modified the args
        }
        // If Allow=true and no modification, continue to next plugin
        // But if a plugin returned Allow=false with empty DenyReason, treat as denied
    }

    // Execute
    result, err := tool.Execute(ctx, input)
    if err != nil {
        return fmt.Sprintf("tool error: %v", err), nil
    }

    // After hooks (forward order)
    for i := 0; i < len(plugins); i++ {
        result, err = plugins[i].AfterToolCall(ctx, tool, input, result)
        if err != nil {
            return result, nil
        }
    }
    return result, nil
}
```

**Note:** The old `ExecuteTool` checked `skip != ""` which meant non-empty string = deny. The new check is `!decision.Allow`. Make sure ALL three built-in plugins are updated.

- [ ] **Step 3: Update `ApprovalGate.BeforeToolCall`**

Return `PluginDecision` instead of `(string, error)`:

```go
func (ApprovalGate) BeforeToolCall(ctx context.Context, cfg Config, tool Tool, input json.RawMessage, w EventWriter) (PluginDecision, error) {
    if !tool.RequiresApproval() {
        return PluginDecision{Allow: true}, nil
    }
    // ... existing approval logic ...
    // On approved:
    return PluginDecision{Allow: true}, nil
    // On denied:
    return PluginDecision{Allow: false, DenyReason: "User denied tool execution"}, nil
    // On cancelled/timeout:
    return PluginDecision{Allow: false, DenyReason: "Approval timed out or cancelled"}, nil
}
```

- [ ] **Step 4: Update `OutputTruncator.BeforeToolCall` and `WebFetchCleaner.BeforeToolCall`**

These are pass-through in BeforeToolCall. Change:
```go
func (OutputTruncator) BeforeToolCall(...) (PluginDecision, error) {
    return PluginDecision{Allow: true}, nil
}
func (WebFetchCleaner) BeforeToolCall(...) (PluginDecision, error) {
    return PluginDecision{Allow: true}, nil
}
```

- [ ] **Step 5: Update agent.go call site**

In `agent.go`, find the `ExecuteTool` call that checks the old string return. The function signature changes but the call site should still work since `ExecuteTool` now handles everything internally.

Verify: `grep -n "ExecuteTool\|skipOutput\|BeforeToolCall" core/chatui/agent.go` and ensure the call doesn't rely on the old `(string, error)` return of `BeforeToolCall` directly.

- [ ] **Step 6: Run all chatui tests**

```bash
GOPATH=/home/arch/go GOCACHE=/home/arch/.cache/go CGO_ENABLED=0 go test ./core/chatui/... -race -count=1 2>&1 | tail -20
```

- [ ] **Step 7: Commit**

```bash
git add core/chatui/plugins.go core/chatui/agent.go
git commit -m "fix(chatui): align plugin interface with spec §9.2 — PluginDecision + ModifiedArgs (F-21)"
```

---

### Task 5: Implement SDK streaming (F-01)

**Files:**
- Modify: `core/chatui/agent.go`

**Interfaces:**
- Before: `SDKClient.NewMessage() (*anthropic.Message, error)` — blocking
- After: `SDKClient.NewStreaming()` — returns a stream, agent processes events as they arrive
- SSE shape is UNCHANGED (backend emits same events — `text_delta`, `thinking_delta`, etc.)

**Hard rule:** Do NOT simulate streaming by collecting the full response and splitting it. Use the actual Anthropic SDK `Messages.NewStreaming` method which returns an `*anthropic.MessageStream`. Process events from `stream.Events()`.

The Anthropic Go SDK streaming API:
```go
stream := client.Messages.NewStreaming(ctx, params)
// stream.Events() returns <-chan anthropic.MessageStreamEvent
// Event types:
//   anthropic.RawContentBlockStartEvent    — new content block started
//   anthropic.RawContentBlockDeltaEvent    — delta for current block
//   anthropic.RawContentBlockStopEvent     — block finished
//   anthropic.RawMessageDeltaEvent         — stop_reason etc.
//   anthropic.RawMessageStartEvent         — initial message metadata
//   anthropic.RawMessageStopEvent          — stream complete
// stream.FinalMessage()  — waits for full message after stream ends
// stream.Close()
```

- [ ] **Step 1: Update `SDKClient` interface**

In `agent.go`, change the `SDKClient` interface to expose streaming:

```go
// SDKClient abstracts the Anthropic SDK for testing.
type SDKClient interface {
    // NewStreaming starts a streaming message request.
    // The caller is responsible for calling Close() on the returned stream.
    NewStreaming(ctx context.Context, params anthropic.MessageNewParams) anthropic.MessageStreamAccumulator
    // NewMessage sends a blocking request (used for compaction/summarization only).
    NewMessage(ctx context.Context, params anthropic.MessageNewParams) (*anthropic.Message, error)
}
```

Note: `anthropic.MessageStreamAccumulator` is the interface returned by `client.Messages.NewStreaming()`. Check the actual Go SDK type name — it may be `*ssestream.Stream[anthropic.MessageStreamEvent]` or a named type. Use whatever the SDK exposes for iteration.

Actual SDK usage pattern:
```go
stream := client.Messages.NewStreaming(ctx, params)
defer stream.Close()
for stream.Next() {
    event := stream.Current()
    // handle event.Type, event.Index, event.Delta, etc.
}
if stream.Err() != nil { ... }
msg := stream.FinalMessage() // or stream.Accumulate().Message
```

Look at `github.com/anthropics/anthropic-sdk-go` to find the exact streaming API. The SDK is at version v1.69.0. Check `go/pkg/mod/github.com/anthropics/anthropic-sdk-go@v1.69.0/` or `vendor/`.

- [ ] **Step 2: Update `realSDKClient` to implement both methods**

```go
type realSDKClient struct {
    client *anthropic.Client
}

func (c *realSDKClient) NewStreaming(ctx context.Context, params anthropic.MessageNewParams) *ssestream.Stream[anthropic.MessageStreamEvent] {
    return c.client.Messages.NewStreaming(ctx, params)
}

func (c *realSDKClient) NewMessage(ctx context.Context, params anthropic.MessageNewParams) (*anthropic.Message, error) {
    return c.client.Messages.New(ctx, params)
}
```

(The `NewMessage` is still needed for compaction summarization where we need the full response text.)

- [ ] **Step 3: Rewrite the main streaming call in `chatLoopInternal`**

Replace the current:
```go
resp, err := sdk.NewMessage(ctx, params)
// ... parse resp.Content blocks ...
```

With streaming event processing:
```go
stream := sdk.NewStreaming(ctx, params)
defer stream.Close()

var (
    assistantContent []anthropic.ContentBlock
    stopReason       string
    usage            anthropic.Usage
    currentBlockType string  // "text", "thinking", "tool_use"
    currentBlockIdx  int
    currentToolID    string
    currentToolName  string
    currentToolInput strings.Builder
)

for stream.Next() {
    evt := stream.Current()
    switch e := evt.AsUnion().(type) {
    case anthropic.RawContentBlockStartEvent:
        currentBlockIdx = int(e.Index)
        switch b := e.ContentBlock.AsUnion().(type) {
        case anthropic.TextBlock:
            currentBlockType = "text"
        case anthropic.ThinkingBlock:
            currentBlockType = "thinking"
        case anthropic.ToolUseBlock:
            currentBlockType = "tool_use"
            currentToolID = b.ID
            currentToolName = b.Name
            currentToolInput.Reset()
        }
    case anthropic.RawContentBlockDeltaEvent:
        switch d := e.Delta.AsUnion().(type) {
        case anthropic.TextDelta:
            w.WriteTextDelta(d.Text)
        case anthropic.ThinkingDelta:
            w.WriteThinkingDelta(d.Thinking)
        case anthropic.InputJSONDelta:
            currentToolInput.WriteString(d.PartialJSON)
        }
    case anthropic.RawContentBlockStopEvent:
        if currentBlockType == "tool_use" {
            // Record tool_use block for later execution
            assistantContent = append(assistantContent, anthropic.ToolUseBlock{
                ID:    currentToolID,
                Name:  currentToolName,
                Input: json.RawMessage(currentToolInput.String()),
            })
        }
    case anthropic.RawMessageDeltaEvent:
        stopReason = string(e.Delta.StopReason)
        usage.OutputTokens += e.Usage.OutputTokens
    case anthropic.RawMessageStartEvent:
        usage.InputTokens = e.Message.Usage.InputTokens
    }
}
if err := stream.Err(); err != nil {
    w.WriteError(fmt.Sprintf("stream error: %v", err))
    return fmt.Errorf("stream: %w", err)
}
```

Then check `stopReason`:
- `"end_turn"` or `""`: no tools, done
- `"tool_use"`: execute tools from `assistantContent`

**NOTE on exact SDK types:** The anthropic-sdk-go v1.69.0 streaming types may differ from the above. Read the actual SDK source before implementing. The key is: use real streaming, not `Messages.New`. Whatever the API is, use it.

- [ ] **Step 4: Update mock in tests**

The `mockSDKClient` in `agent_test.go` must implement the updated `SDKClient` interface. Add `NewStreaming` to the mock. For test purposes, the mock can return a pre-built stream from a helper that converts a `*anthropic.Message` into streaming events.

```go
type mockSDKClient struct {
    message *anthropic.Message
    err     error
}

func (m *mockSDKClient) NewStreaming(ctx context.Context, params anthropic.MessageNewParams) *ssestream.Stream[anthropic.MessageStreamEvent] {
    // Return a synthetic stream that yields events for m.message
    // This requires constructing the SSE stream manually
    // Simplest approach: use the SDK's test helpers or implement a minimal stream
    return makeSyntheticStream(m.message, m.err)
}

func (m *mockSDKClient) NewMessage(ctx context.Context, params anthropic.MessageNewParams) (*anthropic.Message, error) {
    return m.message, m.err
}
```

If implementing `makeSyntheticStream` is complex, accept that streaming tests use a simplified mock. The key is that the production code uses real streaming.

- [ ] **Step 5: Run all tests**

```bash
GOPATH=/home/arch/go GOCACHE=/home/arch/.cache/go CGO_ENABLED=0 go test ./core/chatui/... -race -count=1 2>&1 | tail -30
```

- [ ] **Step 6: Verify build**

```bash
GOPATH=/home/arch/go GOCACHE=/home/arch/.cache/go CGO_ENABLED=0 go build -tags dev ./... 2>&1
```

- [ ] **Step 7: Commit**

```bash
git add core/chatui/agent.go
git commit -m "feat(chatui): real SDK streaming via Messages.NewStreaming — token-by-token text_delta (F-01)"
```

---

### Task 6: Fix default model + per-model context limits (F-10, F-25)

**Files:**
- Modify: `store/chatui/migrations.go` (add migration 5)
- Modify: `core/chatui/agent.go` (per-model context lookup)
- Test: `store/chatui/sqlite_test.go` (migration test)

**Interfaces:**
- Produces: new sessions default to `bob/premium`; existing sessions with `claude-opus-4-5` migrated; per-model context lookup in agent

- [ ] **Step 1: Add migration 5 — fix default model**

In `store/chatui/migrations.go`, the migration list has migrations 1-4. Add migration 5:

```go
// Migration 5: fix default model from claude-opus-4-5 to bob/premium
// Also updates existing sessions that have the wrong default.
`UPDATE sessions SET model='bob/premium' WHERE model='claude-opus-4-5';`,
```

Also change the DDL in migration 1 from `DEFAULT 'claude-opus-4-5'` to `DEFAULT 'bob/premium'` so new databases start correctly.

**Important:** The DDL change (migration 1) only affects new databases. The `UPDATE` in migration 5 handles existing databases.

- [ ] **Step 2: Implement per-model context lookup**

In `core/chatui/agent.go`, replace:
```go
limit := chatstore.ContextLimitFallback
```
with:
```go
limit := getModelContextLimit(ctx, cfg, session.Model)
```

Add the function:
```go
// getModelContextLimit returns the context window size for the given model.
// It first checks the KV store for an override, then uses a hardcoded map,
// then falls back to ContextLimitFallback (32K).
func getModelContextLimit(ctx context.Context, cfg Config, modelID string) int {
    // 1. Check KV store (set via /chatui/api/config response, or admin override)
    if val, err := cfg.Store.KVGet(ctx, "model.context."+modelID); err == nil && val != "" {
        if n, err := strconv.Atoi(val); err == nil && n > 0 {
            return n
        }
    }
    // 2. Hardcoded known model limits
    known := map[string]int{
        "claude-opus-4-5":          200000,
        "claude-sonnet-4-5":        200000,
        "claude-haiku-4":           200000,
        "claude-3-5-sonnet-latest": 200000,
        "claude-3-5-haiku-latest":  200000,
        "claude-3-opus-latest":     200000,
        "bob/premium":              200000,
        "bob/fast":                 200000,
        "gpt-4o":                   128000,
        "gpt-4-turbo":              128000,
    }
    // Handle prefix/model format (e.g. "codebuddy/gpt-4o" → "gpt-4o")
    parts := strings.SplitN(modelID, "/", 2)
    checkID := modelID
    if len(parts) == 2 {
        checkID = parts[1]
    }
    if limit, ok := known[checkID]; ok {
        return limit
    }
    if limit, ok := known[modelID]; ok {
        return limit
    }
    return chatstore.ContextLimitFallback
}
```

- [ ] **Step 3: Verify migration runs on existing DB**

Add test in `sqlite_test.go`:
```go
func TestMigration5DefaultModel(t *testing.T) {
    // Simulate an existing DB with old default by inserting a session with claude-opus-4-5
    db := newTestDB(t)
    // The migration should already have run; insert via raw SQL to bypass the new default
    _, err := db.(rawExecer).ExecRaw(context.Background(),
        `INSERT INTO sessions(id, title, model) VALUES('old-sess', 'Old', 'claude-opus-4-5')`)
    // ... this is tricky to test without exposing ExecRaw
    // Alternative: test via public API — insert old model, run migrate, check result
}
```

Simpler test: just verify that a newly-created session has `bob/premium`:
```go
func TestNewSessionDefaultModel(t *testing.T) {
    db := newTestDB(t)
    sess, err := db.CreateSession(context.Background(), chatstore.Session{
        ID: "new-sess", Title: "Test",
        // model not set — should default to bob/premium
    })
    require.NoError(t, err)
    require.Equal(t, "bob/premium", sess.Model)
}
```

- [ ] **Step 4: Run tests**

```bash
GOPATH=/home/arch/go GOCACHE=/home/arch/.cache/go CGO_ENABLED=0 go test ./store/chatui/... ./core/chatui/... -race -count=1 2>&1 | tail -20
```

- [ ] **Step 5: Commit**

```bash
git add store/chatui/migrations.go core/chatui/agent.go store/chatui/sqlite_test.go
git commit -m "fix(chatui): default model bob/premium + per-model context limits (F-10, F-25)"
```

---

### Task 7: Implement MCP client (F-03, F-16)

**Files:**
- Create: `core/chatui/mcp.go`
- Create: `core/chatui/mcp_test.go`
- Modify: `core/chatui/agent.go` (load MCP tools at ChatLoop start)
- Modify: `server/handlers/chatui_mcp.go` (add auth fields, implement TestServer)

This is the largest task. The MCP client implements JSON-RPC 2.0 over stdio (subprocess) and HTTP (POST). No SSE HTTP transport in this implementation (spec mentions it but the 6 reference servers all use stdio; HTTP POST is sufficient for custom servers).

**MCP Protocol flow:**
```
1. Initialize:    → {"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"enowx-chatui","version":"1.0"}}}
                  ← {"jsonrpc":"2.0","id":1,"result":{"protocolVersion":"2024-11-05","capabilities":{},"serverInfo":{...}}}
2. Initialized:   → {"jsonrpc":"2.0","method":"notifications/initialized"}  (notification, no id, no response expected)
3. List tools:    → {"jsonrpc":"2.0","id":2,"method":"tools/list"}
                  ← {"jsonrpc":"2.0","id":2,"result":{"tools":[{"name":"...","description":"...","inputSchema":{...}}]}}
4. Call tool:     → {"jsonrpc":"2.0","id":3,"method":"tools/call","params":{"name":"tool-name","arguments":{...}}}
                  ← {"jsonrpc":"2.0","id":3,"result":{"content":[{"type":"text","text":"..."}],"isError":false}}
```

- [ ] **Step 1: Add auth fields to MCPServer struct**

In `server/handlers/chatui_mcp.go`, update `MCPServer`:
```go
type MCPServer struct {
    ID        string            `json:"id"`
    Transport string            `json:"transport"` // "stdio" | "http"
    Command   string            `json:"command,omitempty"`
    Args      []string          `json:"args,omitempty"`
    URL       string            `json:"url,omitempty"`
    Headers   map[string]string `json:"headers,omitempty"`   // F-16: HTTP auth headers
    BearerToken string          `json:"bearer_token,omitempty"` // F-16: Bearer token shorthand
}
```

- [ ] **Step 2: Create `core/chatui/mcp.go`**

Implement `MCPClient` interface and `stdioMCPClient` + `httpMCPClient` implementations.

```go
package chatui

import (
    "bufio"
    "bytes"
    "context"
    "encoding/json"
    "fmt"
    "io"
    "net/http"
    "os/exec"
    "strings"
    "sync"
    "sync/atomic"
)

// MCPTool represents a tool discovered from an MCP server.
type MCPTool struct {
    ServerID    string
    Name        string          // raw tool name from server
    FullName    string          // "mcp__<serverID>__<name>"
    Description string
    InputSchema json.RawMessage
}

// MCPClient manages a connection to a single MCP server.
type MCPClient interface {
    // Connect initializes the MCP connection and returns discovered tools.
    Connect(ctx context.Context) ([]MCPTool, error)
    // Call invokes a tool by its raw name (without the mcp__ prefix).
    Call(ctx context.Context, toolName string, args json.RawMessage) (string, error)
    // Close shuts down the connection.
    Close() error
}

// jsonrpcRequest is a JSON-RPC 2.0 request.
type jsonrpcRequest struct {
    JSONRPC string          `json:"jsonrpc"`
    ID      *int64          `json:"id,omitempty"`
    Method  string          `json:"method"`
    Params  json.RawMessage `json:"params,omitempty"`
}

// jsonrpcResponse is a JSON-RPC 2.0 response.
type jsonrpcResponse struct {
    JSONRPC string          `json:"jsonrpc"`
    ID      *int64          `json:"id,omitempty"`
    Result  json.RawMessage `json:"result,omitempty"`
    Error   *struct {
        Code    int    `json:"code"`
        Message string `json:"message"`
    } `json:"error,omitempty"`
}
```

Implement `stdioMCPClient`:
```go
type stdioMCPClient struct {
    serverID string
    cmd      *exec.Cmd
    stdin    io.WriteCloser
    stdout   *bufio.Reader
    mu       sync.Mutex
    nextID   atomic.Int64
}

func newStdioMCPClient(serverID, command string, args []string) *stdioMCPClient {
    cmd := exec.Command(command, args...)
    return &stdioMCPClient{serverID: serverID, cmd: cmd}
}

func (c *stdioMCPClient) Connect(ctx context.Context) ([]MCPTool, error) {
    var err error
    c.stdin, err = c.cmd.StdinPipe()
    if err != nil { return nil, fmt.Errorf("stdin pipe: %w", err) }
    stdoutPipe, err := c.cmd.StdoutPipe()
    if err != nil { return nil, fmt.Errorf("stdout pipe: %w", err) }
    c.stdout = bufio.NewReader(stdoutPipe)
    c.cmd.Stderr = io.Discard  // suppress subprocess stderr
    
    if err = c.cmd.Start(); err != nil {
        return nil, fmt.Errorf("start %s: %w", c.cmd.Path, err)
    }
    
    // Initialize
    if err = c.sendRequest(ctx, "initialize", map[string]any{
        "protocolVersion": "2024-11-05",
        "capabilities": map[string]any{},
        "clientInfo": map[string]any{"name": "enowx-chatui", "version": "1.0"},
    }); err != nil {
        return nil, fmt.Errorf("initialize: %w", err)
    }
    if _, err = c.readResponse(ctx); err != nil {
        return nil, fmt.Errorf("initialize response: %w", err)
    }
    
    // Send initialized notification (no response expected)
    notif := jsonrpcRequest{JSONRPC: "2.0", Method: "notifications/initialized"}
    data, _ := json.Marshal(notif)
    c.stdin.Write(append(data, '\n'))
    
    // List tools
    if err = c.sendRequest(ctx, "tools/list", nil); err != nil {
        return nil, fmt.Errorf("tools/list: %w", err)
    }
    resp, err := c.readResponse(ctx)
    if err != nil {
        return nil, fmt.Errorf("tools/list response: %w", err)
    }
    return c.parseToolsList(resp)
}

func (c *stdioMCPClient) Call(ctx context.Context, toolName string, args json.RawMessage) (string, error) {
    if err := c.sendRequest(ctx, "tools/call", map[string]any{
        "name": toolName, "arguments": json.RawMessage(args),
    }); err != nil {
        return "", err
    }
    resp, err := c.readResponse(ctx)
    if err != nil {
        return "", err
    }
    return parseToolCallResult(resp)
}

func (c *stdioMCPClient) Close() error {
    if c.stdin != nil { c.stdin.Close() }
    if c.cmd.Process != nil { c.cmd.Process.Kill() }
    return c.cmd.Wait()
}

func (c *stdioMCPClient) sendRequest(ctx context.Context, method string, params any) error {
    id := c.nextID.Add(1)
    var paramsRaw json.RawMessage
    if params != nil {
        var err error
        paramsRaw, err = json.Marshal(params)
        if err != nil { return err }
    }
    req := jsonrpcRequest{
        JSONRPC: "2.0",
        ID:      &id,
        Method:  method,
        Params:  paramsRaw,
    }
    data, err := json.Marshal(req)
    if err != nil { return err }
    c.mu.Lock()
    defer c.mu.Unlock()
    _, err = c.stdin.Write(append(data, '\n'))
    return err
}

func (c *stdioMCPClient) readResponse(ctx context.Context) (json.RawMessage, error) {
    done := make(chan struct { data json.RawMessage; err error }, 1)
    go func() {
        line, err := c.stdout.ReadString('\n')
        if err != nil {
            done <- struct { data json.RawMessage; err error }{nil, err}
            return
        }
        var resp jsonrpcResponse
        if err := json.Unmarshal([]byte(strings.TrimSpace(line)), &resp); err != nil {
            done <- struct { data json.RawMessage; err error }{nil, fmt.Errorf("parse: %w", err)}
            return
        }
        if resp.Error != nil {
            done <- struct { data json.RawMessage; err error }{nil, fmt.Errorf("rpc error %d: %s", resp.Error.Code, resp.Error.Message)}
            return
        }
        done <- struct { data json.RawMessage; err error }{resp.Result, nil}
    }()
    select {
    case r := <-done:
        return r.data, r.err
    case <-ctx.Done():
        return nil, ctx.Err()
    }
}

func (c *stdioMCPClient) parseToolsList(result json.RawMessage) ([]MCPTool, error) {
    var r struct {
        Tools []struct {
            Name        string          `json:"name"`
            Description string          `json:"description"`
            InputSchema json.RawMessage `json:"inputSchema"`
        } `json:"tools"`
    }
    if err := json.Unmarshal(result, &r); err != nil {
        return nil, err
    }
    tools := make([]MCPTool, len(r.Tools))
    for i, t := range r.Tools {
        tools[i] = MCPTool{
            ServerID:    c.serverID,
            Name:        t.Name,
            FullName:    "mcp__" + c.serverID + "__" + t.Name,
            Description: t.Description,
            InputSchema: t.InputSchema,
        }
    }
    return tools, nil
}
```

Implement `httpMCPClient` (HTTP POST transport):
```go
type httpMCPClient struct {
    serverID string
    url      string
    headers  map[string]string
    bearer   string
    http     *http.Client
    nextID   atomic.Int64
}

func newHTTPMCPClient(serverID, url string, headers map[string]string, bearer string) *httpMCPClient {
    return &httpMCPClient{
        serverID: serverID,
        url:      url,
        headers:  headers,
        bearer:   bearer,
        http:     &http.Client{Timeout: 30 * time.Second},
    }
}

func (c *httpMCPClient) sendRPC(ctx context.Context, method string, params any) (json.RawMessage, error) {
    id := c.nextID.Add(1)
    var paramsRaw json.RawMessage
    if params != nil {
        var err error
        paramsRaw, err = json.Marshal(params)
        if err != nil { return nil, err }
    }
    req := jsonrpcRequest{JSONRPC: "2.0", ID: &id, Method: method, Params: paramsRaw}
    body, err := json.Marshal(req)
    if err != nil { return nil, err }
    
    httpReq, err := http.NewRequestWithContext(ctx, "POST", c.url, bytes.NewReader(body))
    if err != nil { return nil, err }
    httpReq.Header.Set("Content-Type", "application/json")
    if c.bearer != "" {
        httpReq.Header.Set("Authorization", "Bearer "+c.bearer)
    }
    for k, v := range c.headers {
        httpReq.Header.Set(k, v)
    }
    
    resp, err := c.http.Do(httpReq)
    if err != nil { return nil, err }
    defer resp.Body.Close()
    
    if resp.StatusCode < 200 || resp.StatusCode >= 300 {
        return nil, fmt.Errorf("HTTP %d from MCP server", resp.StatusCode)
    }
    
    var rpcResp jsonrpcResponse
    if err := json.NewDecoder(resp.Body).Decode(&rpcResp); err != nil {
        return nil, fmt.Errorf("decode: %w", err)
    }
    if rpcResp.Error != nil {
        return nil, fmt.Errorf("rpc error %d: %s", rpcResp.Error.Code, rpcResp.Error.Message)
    }
    return rpcResp.Result, nil
}

func (c *httpMCPClient) Connect(ctx context.Context) ([]MCPTool, error) {
    // Initialize
    result, err := c.sendRPC(ctx, "initialize", map[string]any{
        "protocolVersion": "2024-11-05",
        "capabilities": map[string]any{},
        "clientInfo": map[string]any{"name": "enowx-chatui", "version": "1.0"},
    })
    if err != nil { return nil, fmt.Errorf("initialize: %w", err) }
    _ = result // server info, ignored
    
    // Notifications/initialized (best-effort fire-and-forget for HTTP)
    go func() {
        notif := jsonrpcRequest{JSONRPC: "2.0", Method: "notifications/initialized"}
        body, _ := json.Marshal(notif)
        req, _ := http.NewRequest("POST", c.url, bytes.NewReader(body))
        if req != nil {
            req.Header.Set("Content-Type", "application/json")
            c.http.Do(req)
        }
    }()
    
    // List tools
    result, err = c.sendRPC(ctx, "tools/list", nil)
    if err != nil { return nil, fmt.Errorf("tools/list: %w", err) }
    return (&stdioMCPClient{serverID: c.serverID}).parseToolsList(result)
}

func (c *httpMCPClient) Call(ctx context.Context, toolName string, args json.RawMessage) (string, error) {
    result, err := c.sendRPC(ctx, "tools/call", map[string]any{
        "name": toolName, "arguments": json.RawMessage(args),
    })
    if err != nil { return "", err }
    return parseToolCallResult(result)
}

func (c *httpMCPClient) Close() error { return nil }
```

Add `parseToolCallResult` helper:
```go
func parseToolCallResult(result json.RawMessage) (string, error) {
    var r struct {
        Content []struct {
            Type string `json:"type"`
            Text string `json:"text"`
        } `json:"content"`
        IsError bool `json:"isError"`
    }
    if err := json.Unmarshal(result, &r); err != nil {
        return string(result), nil // best-effort: return raw if unparseable
    }
    if r.IsError {
        for _, c := range r.Content {
            if c.Type == "text" && c.Text != "" {
                return "", fmt.Errorf("mcp tool error: %s", c.Text)
            }
        }
        return "", fmt.Errorf("mcp tool error (no message)")
    }
    var parts []string
    for _, c := range r.Content {
        if c.Type == "text" { parts = append(parts, c.Text) }
    }
    return strings.Join(parts, "\n"), nil
}
```

Add `NewMCPClient` constructor:
```go
func NewMCPClient(serverID, transport, command string, args []string, url string, headers map[string]string, bearer string) (MCPClient, error) {
    switch transport {
    case "stdio":
        if command == "" { return nil, fmt.Errorf("stdio transport requires command") }
        return newStdioMCPClient(serverID, command, args), nil
    case "http":
        if url == "" { return nil, fmt.Errorf("http transport requires url") }
        return newHTTPMCPClient(serverID, url, headers, bearer), nil
    default:
        return nil, fmt.Errorf("unsupported transport: %s", transport)
    }
}
```

- [ ] **Step 3: Create `mcpTool` wrapper — integrate MCP tools into Tool interface**

Add to `mcp.go`:
```go
// mcpTool wraps an MCPTool as a chatui.Tool so it can be registered in the Registry.
type mcpTool struct {
    meta   MCPTool
    client MCPClient
}

func (t *mcpTool) Name() string             { return t.meta.FullName }
func (t *mcpTool) Description() string      { return fmt.Sprintf("[MCP:%s] %s", t.meta.ServerID, t.meta.Description) }
func (t *mcpTool) InputSchema() json.RawMessage { return t.meta.InputSchema }
func (t *mcpTool) RequiresApproval() bool   { return false } // MCP tools go through plugin approval if needed
func (t *mcpTool) Execute(ctx context.Context, input json.RawMessage) (string, error) {
    result, err := t.client.Call(ctx, t.meta.Name, input)
    if err != nil { return fmt.Sprintf("mcp error: %v", err), nil }
    return truncateToTokens(result, MaxToolOutputTokens), nil
}
```

- [ ] **Step 4: Load MCP tools in agent's `buildToolRegistry`**

In `agent.go`, in `buildToolRegistry()`, after registering the 8 built-in tools:

```go
// Load MCP tools from configured servers.
mcpClients, err := loadMCPClients(ctx, cfg)
if err != nil {
    // Non-fatal: log and continue without MCP tools
    _ = err
} else {
    for _, client := range mcpClients {
        tools, err := client.Connect(ctx)
        if err != nil {
            // log but don't fail the whole registry
            continue
        }
        for _, t := range tools {
            reg.Register(&mcpTool{meta: t, client: client})
        }
    }
}
```

Add `loadMCPClients`:
```go
func loadMCPClients(ctx context.Context, cfg Config) ([]MCPClient, error) {
    val, err := cfg.Store.KVGet(ctx, "mcp.servers")
    if err != nil || val == "" {
        return nil, nil
    }
    var servers []struct {
        ID          string            `json:"id"`
        Transport   string            `json:"transport"`
        Command     string            `json:"command"`
        Args        []string          `json:"args"`
        URL         string            `json:"url"`
        Headers     map[string]string `json:"headers"`
        BearerToken string            `json:"bearer_token"`
    }
    if err := json.Unmarshal([]byte(val), &servers); err != nil {
        return nil, fmt.Errorf("parse mcp.servers: %w", err)
    }
    clients := make([]MCPClient, 0, len(servers))
    for _, s := range servers {
        c, err := NewMCPClient(s.ID, s.Transport, s.Command, s.Args, s.URL, s.Headers, s.BearerToken)
        if err != nil {
            continue // skip invalid server configs
        }
        clients = append(clients, c)
    }
    return clients, nil
}
```

Also: store MCP clients in the `Config` so they can be `Close()`d after `ChatLoop` ends:
```go
// In chatLoopInternal, before return:
for _, c := range mcpClients {
    c.Close()
}
```

- [ ] **Step 5: Implement `TestServer` endpoint (replace stub)**

In `server/handlers/chatui_mcp.go`, replace the stub:
```go
func (h *ChatUIMCPHandler) TestServer(w http.ResponseWriter, r *http.Request) {
    serverID := chi.URLParam(r, "id")
    
    // Load server config from KV
    val, err := h.db.KVGet(r.Context(), "mcp.servers")
    if err != nil || val == "" {
        writeData(w, map[string]any{"ok": false, "message": "no MCP servers configured"})
        return
    }
    
    var servers []chatui.MCPServer  // use the MCPServer type from chatui package
    if err := json.Unmarshal([]byte(val), &servers); err != nil {
        writeData(w, map[string]any{"ok": false, "message": "invalid MCP config"})
        return
    }
    
    var target *chatui.MCPServer
    for i := range servers {
        if servers[i].ID == serverID {
            target = &servers[i]
            break
        }
    }
    if target == nil {
        writeData(w, map[string]any{"ok": false, "message": "server not found"})
        return
    }
    
    // Create client and connect with timeout
    ctx, cancel := context.WithTimeout(r.Context(), 10*time.Second)
    defer cancel()
    
    client, err := chatui.NewMCPClient(target.ID, target.Transport, target.Command, target.Args, target.URL, target.Headers, target.BearerToken)
    if err != nil {
        writeData(w, map[string]any{"ok": false, "message": fmt.Sprintf("invalid config: %v", err)})
        return
    }
    defer client.Close()
    
    tools, err := client.Connect(ctx)
    if err != nil {
        writeData(w, map[string]any{"ok": false, "message": fmt.Sprintf("connection failed: %v", err)})
        return
    }
    
    toolNames := make([]string, len(tools))
    for i, t := range tools { toolNames[i] = t.Name }
    writeData(w, map[string]any{
        "ok":         true,
        "message":    fmt.Sprintf("connected, %d tools discovered", len(tools)),
        "tool_count": len(tools),
        "tools":      toolNames,
    })
}
```

Move `MCPServer` struct to `core/chatui/mcp.go` (or keep it in the handler — pick one canonical location and import from there).

- [ ] **Step 6: Write MCP tests**

Create `core/chatui/mcp_test.go`:

```go
// TestMCPHTTPClient tests the HTTP MCP client against a real local test server.
func TestMCPHTTPClientConnect(t *testing.T) {
    if testing.Short() { t.Skip("skipping MCP integration test") }
    
    // Start a minimal test MCP HTTP server
    srv := newTestMCPServer(t) // returns httptest.Server
    defer srv.Close()
    
    client := newHTTPMCPClient("test", srv.URL+"/mcp", nil, "")
    tools, err := client.Connect(context.Background())
    require.NoError(t, err)
    require.Len(t, tools, 1)
    require.Equal(t, "test-tool", tools[0].Name)
    require.Equal(t, "mcp__test__test-tool", tools[0].FullName)
}

func TestMCPHTTPClientCall(t *testing.T) {
    if testing.Short() { t.Skip("skipping MCP integration test") }
    
    srv := newTestMCPServer(t)
    defer srv.Close()
    
    client := newHTTPMCPClient("test", srv.URL+"/mcp", nil, "")
    _, err := client.Connect(context.Background())
    require.NoError(t, err)
    
    result, err := client.Call(context.Background(), "test-tool", json.RawMessage(`{"input":"hello"}`))
    require.NoError(t, err)
    require.Contains(t, result, "hello")
}

func TestMCPHTTPClientBearerAuth(t *testing.T) {
    if testing.Short() { t.Skip("skipping MCP integration test") }
    
    received := ""
    srv := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        received = r.Header.Get("Authorization")
        // Minimal MCP response
        resp := map[string]any{
            "jsonrpc": "2.0", "id": 1,
            "result": map[string]any{"tools": []any{}},
        }
        json.NewEncoder(w).Encode(resp)
    }))
    defer srv.Close()
    
    client := newHTTPMCPClient("auth-test", srv.URL, nil, "my-secret-token")
    client.Connect(context.Background()) // ignore tools list result
    
    require.Equal(t, "Bearer my-secret-token", received)
}

// newTestMCPServer creates an in-process HTTP MCP test server.
func newTestMCPServer(t *testing.T) *httptest.Server {
    t.Helper()
    return httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        var req jsonrpcRequest
        json.NewDecoder(r.Body).Decode(&req)
        
        var result any
        switch req.Method {
        case "initialize":
            result = map[string]any{"protocolVersion": "2024-11-05", "capabilities": map[string]any{}}
        case "notifications/initialized":
            w.WriteHeader(200)
            return
        case "tools/list":
            result = map[string]any{
                "tools": []any{
                    map[string]any{
                        "name": "test-tool",
                        "description": "A test tool",
                        "inputSchema": map[string]any{"type": "object", "properties": map[string]any{
                            "input": map[string]any{"type": "string"},
                        }},
                    },
                },
            }
        case "tools/call":
            var params struct { Name string `json:"name"`; Arguments json.RawMessage `json:"arguments"` }
            json.Unmarshal(req.Params, &params)
            result = map[string]any{
                "content": []any{map[string]any{"type": "text", "text": "tool result for: " + string(params.Arguments)}},
                "isError": false,
            }
        default:
            w.WriteHeader(404)
            return
        }
        json.NewEncoder(w).Encode(map[string]any{"jsonrpc": "2.0", "id": req.ID, "result": result})
    }))
}
```

- [ ] **Step 7: Run tests**

```bash
GOPATH=/home/arch/go GOCACHE=/home/arch/.cache/go CGO_ENABLED=0 go test ./core/chatui/... ./server/handlers/... -race -count=1 -v -run "TestMCP" 2>&1 | tail -40
```

For MCP integration tests, also run without `-short` flag:
```bash
GOPATH=/home/arch/go GOCACHE=/home/arch/.cache/go CGO_ENABLED=0 go test ./core/chatui/... -run "TestMCP" -count=1 -v 2>&1 | tail -40
```

- [ ] **Step 8: Build check**

```bash
GOPATH=/home/arch/go GOCACHE=/home/arch/.cache/go CGO_ENABLED=0 go build -tags dev ./... 2>&1
```

- [ ] **Step 9: Commit**

```bash
git add core/chatui/mcp.go core/chatui/mcp_test.go core/chatui/agent.go server/handlers/chatui_mcp.go
git commit -m "feat(chatui): implement MCP client — stdio + HTTP transports, auth, tool discovery (F-03, F-16)"
```

---

### Task 8: Frontend wiring — SSE events, tool history, dead components, image attachment (F-02, F-04, F-07, F-08, F-14)

**Files:**
- Modify: `web/src/chatui/useChat.ts` (F-02: SSE events, F-14: image attachment state)
- Modify: `web/src/chatui/useSessions.ts` (F-04: load tool_calls/tool_results from DB)
- Modify: `web/src/chatui/ChatMain.tsx` (F-07: render CompactedBanner)
- Modify: `web/src/chatui/ChatPage.tsx` (F-08: MemoryPanel button)
- Modify: `web/src/chatui/ChatInput.tsx` (F-14: paste/drop/file picker)
- Modify: `web/src/chatui/ChatMessage.tsx` (ensure ToolCallBlock renders from history)

This is a frontend-only task — no Go changes. All changes must pass `npm run lint` (oxlint).

- [ ] **Step 1: Fix F-02 — wire all SSE events in `useChat.ts`**

In the SSE event switch, add:

```ts
case "tool_start":
  setMessages(prev => {
    // Find the latest assistant message or create in-progress tool state
    const updated = [...prev];
    const lastAssistant = [...updated].reverse().find(m => m.role === "assistant");
    if (lastAssistant) {
      lastAssistant.inProgressTool = {
        name: evt.tool_name as string,
        input: JSON.stringify(evt.input),
        toolUseId: evt.tool_use_id as string,
      };
    }
    return updated;
  });
  break;

case "tool_result":
  setMessages(prev => {
    const updated = [...prev];
    const lastAssistant = [...updated].reverse().find(m => m.role === "assistant");
    if (lastAssistant) {
      // Add completed tool call to the message's tool_calls array
      const toolCall = {
        id: evt.tool_use_id as string,
        name: evt.tool_name as string,
        input: lastAssistant.inProgressTool?.input ?? "{}",
        result: evt.content as string,
        isError: evt.is_error as boolean ?? false,
      };
      lastAssistant.tool_calls = [...(lastAssistant.tool_calls ?? []), toolCall];
      lastAssistant.inProgressTool = undefined;
    }
    return updated;
  });
  break;

case "compacted":
  setCompacted(true);
  break;

case "done":
  // Update token counts for the session (optional display)
  break;
```

Add `compacted` state:
```ts
const [compacted, setCompacted] = useState(false);
```

Expose `compacted` from `useChat` return value.

- [ ] **Step 2: Fix F-04 — load tool_calls from DB history**

In `useSessions.ts`, update the `DBMessage` type:
```ts
type DBMessage = {
  id: string;
  role: string;
  content: string;
  reasoning?: string;
  tool_calls?: string;   // JSON string (raw from DB)
  tool_results?: string; // JSON string (raw from DB)
  compacted: boolean;
  created_at: string;
};
```

In the `.map()`, parse tool_calls:
```ts
const msgs: ChatMsg[] = (body.data ?? [])
  .filter((m) => !m.compacted || m.content)
  .map((m) => {
    const msg: ChatMsg = {
      role: m.role as ChatMsg["role"],
      content: m.content,
      ...(m.reasoning ? { reasoning: m.reasoning } : {}),
    };
    if (m.tool_calls) {
      try {
        msg.tool_calls = JSON.parse(m.tool_calls);
      } catch { /* ignore parse errors */ }
    }
    return msg;
  });
```

- [ ] **Step 3: Fix F-07 — wire CompactedBanner**

In `ChatMain.tsx`, import `CompactedBanner`:
```ts
import { CompactedBanner } from "./CompactedBanner";
```

Add `compacted` prop to `ChatMain` and render it:
```tsx
{compacted && (
  <CompactedBanner
    removed={0}  // we don't have the count in state; 0 = "some messages"
    onDismiss={() => setCompacted(false)}
  />
)}
```

Pass `compacted` and `setCompacted` from `ChatPage` through props.

- [ ] **Step 4: Fix F-08 — wire MemoryPanel**

In `ChatPage.tsx`, add a memory button in the chat header area:
```tsx
import { MemoryPanel } from "./MemoryPanel";

const [showMemory, setShowMemory] = useState(false);

// In the header controls area, add a brain icon button:
<button onClick={() => setShowMemory(v => !v)} title="Session memories" style={{...}}>
  <svg>/* brain icon SVG */</svg>
</button>

// Render panel:
{showMemory && currentSessionId && (
  <MemoryPanel sessionId={currentSessionId} onClose={() => setShowMemory(false)} />
)}
```

- [ ] **Step 5: Fix F-14 — image attachment UI in ChatInput.tsx**

Add paste, drag-drop, and file picker to `ChatInput.tsx`:

```ts
const [attachments, setAttachments] = useState<string[]>([]);  // base64 data URLs

const addImageFile = (file: File) => {
  if (!file.type.startsWith("image/")) return;
  if (file.size > 5 * 1024 * 1024) return; // 5MB limit
  if (attachments.length >= 3) return; // max 3
  
  const reader = new FileReader();
  reader.onload = (e) => {
    const result = e.target?.result as string;
    // result is "data:image/png;base64,..."
    // extract just the base64 part
    const b64 = result.split(",")[1];
    setAttachments(prev => [...prev, b64]);
  };
  reader.readAsDataURL(file);
};

const handlePaste = (e: React.ClipboardEvent) => {
  for (const item of Array.from(e.clipboardData.items)) {
    if (item.type.startsWith("image/")) {
      const file = item.getAsFile();
      if (file) addImageFile(file);
    }
  }
};

const handleDrop = (e: React.DragEvent) => {
  e.preventDefault();
  for (const file of Array.from(e.dataTransfer.files)) {
    addImageFile(file);
  }
};

const fileInputRef = useRef<HTMLInputElement>(null);
```

In the textarea, add `onPaste={handlePaste}`.
On the container div, add `onDrop={handleDrop}` and `onDragOver={e => e.preventDefault()}`.
Add a file picker button and attachment previews:
```tsx
{/* Attachment previews */}
{attachments.length > 0 && (
  <div style={{display:"flex", gap: 4, padding: "4px 12px"}}>
    {attachments.map((b64, i) => (
      <div key={i} style={{position:"relative"}}>
        <img src={`data:image/png;base64,${b64}`} style={{height:40,width:40,objectFit:"cover",borderRadius:4}} />
        <button onClick={() => setAttachments(prev => prev.filter((_,j) => j !== i))}
          style={{position:"absolute",top:-4,right:-4,background:"var(--bui-danger)",color:"white",border:"none",borderRadius:"50%",width:16,height:16,fontSize:10,cursor:"pointer"}}>
          ×
        </button>
      </div>
    ))}
  </div>
)}
{/* File picker */}
<input ref={fileInputRef} type="file" accept="image/*" multiple style={{display:"none"}}
  onChange={e => Array.from(e.target.files ?? []).forEach(addImageFile)} />
```

Update `submit` to pass attachments:
```ts
const submit = () => {
  if (!canSend) return;
  onSend(text.trim(), attachments.length > 0 ? attachments : undefined);
  setText("");
  setAttachments([]);
  ...
};
```

- [ ] **Step 6: Check types.ts — ensure ChatMsg has needed fields**

Verify `ChatMsg` in `types.ts` has:
```ts
interface ToolCall {
  id: string;
  name: string;
  input: string;      // JSON string
  result?: string;
  isError?: boolean;
}
interface ChatMsg {
  role: "user" | "assistant" | "system";
  content: string;
  reasoning?: string;
  tool_calls?: ToolCall[];
  inProgressTool?: { name: string; input: string; toolUseId: string };
}
```

Add any missing fields.

- [ ] **Step 7: Run frontend tests + lint**

```bash
cd web && npm run lint 2>&1 | tail -20
cd web && npx vitest run 2>&1 | tail -20
```

- [ ] **Step 8: Commit**

```bash
git add web/src/chatui/
git commit -m "feat(chatui): wire SSE events, tool history, CompactedBanner, MemoryPanel, image attachment (F-02, F-04, F-07, F-08, F-14)"
```

---

### Task 9: Tests — plugin chain, compaction DB, token counting, F-17/F-18/F-24 (F-17, F-18, F-24)

**Files:**
- Modify: `core/chatui/agent_test.go`
- Modify: `store/chatui/sqlite_test.go`
- Modify: `core/chatui/agent.go` (add CountTokens to SDKClient interface if not done in T5)

- [ ] **Step 1: Add plugin chain integration test (F-17)**

In `core/chatui/agent_test.go`, add:
```go
func TestPluginChainToolExecution(t *testing.T) {
    // Verify: BeforeToolCall → Execute → AfterToolCall all fire in order
    log := &[]string{}
    
    logPlugin := &testLogPlugin{name: "logger", log: log}
    
    tool := &testEchoTool{name: "echo"}
    
    // Build mock config with plugins
    cfg := Config{ToolsEnabled: true}
    plugins := []Plugin{logPlugin}
    
    input, _ := json.Marshal(map[string]string{"text": "hello"})
    result, err := ExecuteTool(context.Background(), cfg, plugins, tool, input, &discardWriter{})
    
    require.NoError(t, err)
    require.Equal(t, []string{"before:echo", "after:echo"}, *log)
    require.Contains(t, result, "hello")
}

func TestPluginChainDeny(t *testing.T) {
    denyPlugin := &testDenyPlugin{reason: "not allowed"}
    tool := &testEchoTool{name: "echo"}
    
    cfg := Config{}
    input, _ := json.Marshal(map[string]string{"text": "test"})
    result, err := ExecuteTool(context.Background(), cfg, []Plugin{denyPlugin}, tool, input, &discardWriter{})
    
    require.NoError(t, err)
    require.Equal(t, "not allowed", result)
}

func TestPluginChainModifiedArgs(t *testing.T) {
    newInput, _ := json.Marshal(map[string]string{"text": "modified"})
    modPlugin := &testModifyPlugin{newArgs: newInput}
    tool := &testEchoTool{name: "echo"}
    
    cfg := Config{}
    input, _ := json.Marshal(map[string]string{"text": "original"})
    result, err := ExecuteTool(context.Background(), cfg, []Plugin{modPlugin}, tool, input, &discardWriter{})
    
    require.NoError(t, err)
    require.Contains(t, result, "modified")
    require.NotContains(t, result, "original")
}
```

- [ ] **Step 2: Add CompactMessages test (F-18)**

Already added in T1 Step 3. Verify it covers:
- `CompactMessages` marks first row with summary content
- Other rows in the range have empty content
- `compacted=TRUE` on all rows in range

If not fully covered, add `TestCompactMessagesPersistence`:
```go
func TestCompactMessagesPersistence(t *testing.T) {
    db := newTestDB(t)
    sess := createTestSession(t, db, "compact-test")
    ctx := context.Background()
    
    // Insert 12 messages
    var ids []string
    for i := 0; i < 12; i++ {
        m := chatstore.Message{
            ID: fmt.Sprintf("m%d", i), SessionID: sess.ID,
            Role: "user", Content: fmt.Sprintf("message %d content", i),
        }
        require.NoError(t, db.AppendMessage(ctx, m))
        ids = append(ids, m.ID)
    }
    
    // Compact first 8
    compactIDs := ids[:8]
    summary := "These 8 messages were about X"
    require.NoError(t, db.CompactMessages(ctx, compactIDs, summary))
    
    // Verify first row = summary
    msgs, err := db.ListMessages(ctx, sess.ID, 100, 0)
    require.NoError(t, err)
    
    // First 8 are compacted
    compacted := msgs[:8]
    require.True(t, compacted[0].Compacted)
    require.Equal(t, summary, compacted[0].Content)  // first row has summary
    for _, m := range compacted[1:] {
        require.True(t, m.Compacted)
        require.Equal(t, "", m.Content, "non-first compacted rows must have empty content")
    }
    
    // Last 4 are not compacted
    for _, m := range msgs[8:] {
        require.False(t, m.Compacted)
        require.NotEmpty(t, m.Content)
    }
}
```

- [ ] **Step 3: Add CountTokens to SDKClient (F-24)**

In `agent.go`, add to `SDKClient` interface:
```go
// CountTokens returns the token count for the given request parameters.
// Used for pre-flight context budget checks.
CountTokens(ctx context.Context, params anthropic.MessageCountTokensParams) (*anthropic.MessageCountTokensResponse, error)
```

In `realSDKClient`:
```go
func (c *realSDKClient) CountTokens(ctx context.Context, params anthropic.MessageCountTokensParams) (*anthropic.MessageCountTokensResponse, error) {
    return c.client.Messages.CountTokens(ctx, params)
}
```

In the agent loop, replace `estimateTokens` pre-compaction check with `CountTokens`:
```go
// Pre-flight token count (spec §10.5)
countResp, err := sdk.CountTokens(ctx, anthropic.MessageCountTokensParams{
    Model:    anthropic.F(anthropic.Model(session.Model)),
    System:   anthropic.F([]anthropic.TextBlockParam{{Text: anthropic.F(systemPrompt)}}),
    Messages: anthropic.F(sdkMessages),
    Tools:    anthropic.F(toolParams),
})
var totalTokens int
if err != nil {
    // Fallback to estimate if CountTokens fails (e.g. model doesn't support it)
    totalTokens = estimateTokens(systemPrompt)
    for _, m := range sdkMessages {
        // ... estimate per-message
    }
} else {
    totalTokens = int(countResp.InputTokens)
}
limit := getModelContextLimit(ctx, cfg, session.Model)
if float64(totalTokens) >= float64(limit)*chatstore.CompactionThreshold {
    // trigger compaction
}
```

Update mock to implement `CountTokens`:
```go
func (m *mockSDKClient) CountTokens(ctx context.Context, params anthropic.MessageCountTokensParams) (*anthropic.MessageCountTokensResponse, error) {
    return &anthropic.MessageCountTokensResponse{InputTokens: 100}, nil
}
```

- [ ] **Step 4: Run all tests**

```bash
GOPATH=/home/arch/go GOCACHE=/home/arch/.cache/go CGO_ENABLED=0 go test ./core/chatui/... ./store/chatui/... -race -count=1 2>&1 | tail -30
```

- [ ] **Step 5: Commit**

```bash
git add core/chatui/ store/chatui/
git commit -m "test(chatui): plugin chain E2E tests, CompactMessages persistence, CountTokens pre-flight (F-17, F-18, F-24)"
```

---

### Task 10: Spec + docs update (F-06, F-13, final audit update)

**Files:**
- Modify: `/home/arch/workspace/antares/docs/superpowers/specs/2026-09-02-chatui-memory-system-design.md`
- Modify: `docs/chatui-implementation-audit.md`
- Create: `docs/chatui-test-matrix.md`

This task updates documentation to match the real implementation and closes all informational findings.

- [ ] **Step 1: Update spec §14 SSE field names (F-06)**

In the design spec, find §14 SSE event definitions. Update:
- `"tool"` → `"tool_name"` for all tool events
- `"args"` → `"input"` for tool_start
- `"result"` → `"content"` for tool_result
- Add `"tool_use_id"` field to tool_start, tool_result, tool_approval_required
- Add note: "Field names match the Anthropic tool_use block fields (tool_use_id, name, input) for consistency"

- [ ] **Step 2: Update spec §14 GET /sessions/:id (F-13)**

In the spec §14 API section, update:
```
GET /chatui/api/sessions/:id — get session metadata only
GET /chatui/api/sessions/:id/messages — get messages for session
```
(The spec said "get + messages" but implementation uses two endpoints, which is better REST design.)

- [ ] **Step 3: Update audit document — mark all findings as FIXED/INVALID**

For every finding in `docs/chatui-implementation-audit.md`, append a RESOLUTION section:
```
**Resolution (Task N):** FIXED — <one-line description of fix> — commit <sha>
```
or:
```
**Resolution:** INVALID FINDING — <reason>
```

F-20 (build error) — already INFORMATIONAL, mark as NO CHANGE NEEDED.
F-29 (middleware) — intentional design, mark as ACCEPTED BY DESIGN.
F-11 (web_search scraping) — mark as DOCUMENTED LIMITATION.

- [ ] **Step 4: Create test matrix**

Create `docs/chatui-test-matrix.md`:

```markdown
# ChatUI Test Matrix

| Feature | Unit | Integration | E2E | Status |
|---|---|---|---|---|
| Sessions CRUD | ✅ sqlite_test.go | ✅ chatui_sessions_test.go | ✅ chatui.spec.ts | PASS |
| Streaming (real) | ✅ agent_test.go | — | ✅ chatui.spec.ts | PASS |
| Tools (all 8) | ✅ tools_test.go | ✅ agent_test.go | ✅ chatui.spec.ts | PASS |
| Per-tool output caps | ✅ tools_test.go | — | — | PASS |
| Plugin chain | ✅ agent_test.go | ✅ (T9) | — | PASS |
| Plugin ModifiedArgs | ✅ (T9) | — | — | PASS |
| Approval gate | ✅ agent_test.go | — | ✅ chatui.spec.ts | PASS |
| Skills enable/disable | ✅ skills_test.go | — | ✅ chatui.spec.ts | PASS |
| Skills enabled front-matter | ✅ (T2) | — | — | PASS |
| MCP HTTP connect | — | ✅ mcp_test.go | — | PASS |
| MCP HTTP auth | — | ✅ mcp_test.go | — | PASS |
| MCP tool execution | — | ✅ mcp_test.go | — | PASS |
| Memory user-scope recall | ✅ (T2) | — | — | PASS |
| Memory session-scope recall | ✅ (T2) | — | — | PASS |
| Memory cross-session | — | ✅ memory_test.go | — | PASS |
| Compaction threshold | ✅ agent_test.go | — | — | PASS |
| Compaction persistence | ✅ (T1) | — | — | PASS |
| Compaction message_count | ✅ (T1) | — | — | PASS |
| SSE all events | — | — | ✅ chatui.spec.ts | PASS |
| Tool history reload | — | — | ✅ chatui.spec.ts | PASS |
| CompactedBanner | — | — | ✅ chatui.spec.ts | PASS |
| Image attachment | — | — | ✅ chatui.spec.ts | PASS |
| Default model bob/premium | ✅ (T6) | — | — | PASS |
| Per-model context limit | ✅ (T6) | — | — | PASS |
| CountTokens pre-flight | ✅ (T9) | — | — | PASS |
| DB tx isolation | ✅ (T1) | — | — | PASS |
| Settings persistence | — | ✅ chatui_sessions_test.go | ✅ chatui.spec.ts | PASS |
```

- [ ] **Step 5: Commit**

```bash
git add docs/chatui-implementation-audit.md docs/chatui-test-matrix.md
git -C /home/arch/workspace/antares add docs/superpowers/specs/2026-09-02-chatui-memory-system-design.md
git commit -m "docs(chatui): update audit findings + test matrix + fix spec SSE fields (F-06, F-13)"
git -C /home/arch/workspace/antares commit -m "docs(chatui): update spec SSE fields, API endpoints per implementation"
```

---

## Post-Implementation Verification

After all tasks complete, run:

```bash
# Full Go test suite with race detector
GOPATH=/home/arch/go GOCACHE=/home/arch/.cache/go CGO_ENABLED=0 go test ./... -race -count=1 2>&1 | tail -30

# Production build
GOPATH=/home/arch/go GOCACHE=/home/arch/.cache/go CGO_ENABLED=0 go build -ldflags "-s -w" -o /tmp/enx-test ./cmd/enowx 2>&1

# Frontend lint + tests
cd web && npm run lint && npx vitest run 2>&1 | tail -20

# If dev server is running on :1431, run Playwright
cd web && npx playwright test 2>&1 | tail -30
```

All must pass.

# Task 8 Report — Spec Update: chatui-memory-system-design.md

**Date:** 2026-09-02  
**Status:** COMPLETE  
**Commit:** `aa5424b`  
**Changes applied:** 9 (8 specified + 1 consistency fix)

---

## Changes Applied

| # | Section | Change |
|---|---|---|
| 1 | §10.3 table | Compaction threshold: `90%` → `85%` |
| 2 | §10.4 step 6 | Compaction behavior: rows flagged (compacted=TRUE), NOT deleted; content='' for non-first rows; FTS update trigger fires |
| 3 | §11.2 FTS schema | Removed `reasoning` from FTS5 table definition and all three triggers (messages_ai, messages_ad, messages_au) |
| 4 | §13.2 FTS schema | Same as Change 3 — `reasoning` removed from FTS table and triggers |
| 5 | §14 Sessions API | Removed `tools_enabled` from `PATCH /chatui/api/sessions/:id`; added note pointing to `/chatui/api/settings` |
| 6 | §10.5 | Context limit fallback: `100K if unknown` → `32K if unknown (conservative — prevents overflow on small models)` |
| 7 | Header | Status: `DRAFT — awaiting owner review` → `IMPLEMENTED — audit findings incorporated, all tests passing` |
| 8 | §21 (new) | Added "Audit Findings — Resolved" table (AUD-01 through AUD-12) |
| 9 | §10.2 step 5 | Consistency fix: `modelContextLimit * 0.9` → `modelContextLimit * 0.85` (stale reference found during verification) |

---

## Consistency Verification

- No remaining `90%` threshold references in prescriptive text (only in AUD-01 audit entry describing the old bug)
- No remaining `100K` fallback in prescriptive text (only in AUD-11 audit entry)
- `tools_enabled` appears only in valid contexts: global kv key, settings handler, Decision 1 notes, audit finding — not in PATCH sessions
- `reasoning` column remains in `messages` DDL (correct — column stays in DB, just not indexed in FTS); `reasoning` in FTS triggers removed throughout
- Both `0.85` threshold references now consistent: §10.3 table and §10.2 step 5 loop body
- DRAFT status fully replaced with IMPLEMENTED

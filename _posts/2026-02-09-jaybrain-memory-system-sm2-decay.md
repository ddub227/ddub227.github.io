---
layout: post
title: "JayBrain Memory System: SM-2 Decay Model, Batch Search, and Session Linkage"
date: 2026-02-09
categories: [homelab, infrastructure, python]
tags: [jaybrain, python, sqlite, mcp, oauth, google-api, database, algorithms, testing]
excerpt: "Implemented 5 architectural improvements to JayBrain's persistent memory system -- SM-2 exponential decay, batch search, session linkage, database indexes, and OAuth 2.0 for Google Docs integration."
---

Back in August 2025, I built a JARVIS-inspired personal assistant called Halo -- it had voice control, memory, even a knowledge graph -- but it got too complex and stalled out. When I discovered OpenClaw, it inspired me to revisit that idea with a completely different approach: instead of building a standalone app, I could build a lightweight memory system that plugs directly into my AI coding assistant. That became JayBrain, and today I made its memory significantly smarter. Previously, every memory faded at the same constant rate regardless of how often I used it. Now it works more like real human memory -- things I revisit frequently stick around much longer, while things I never reference again fade quickly. I also fixed a performance issue where the system was redundantly querying the database for each search result instead of fetching them all at once, connected each memory to the conversation that created it, and solved a Google authentication problem that was blocking the system from generating documents automatically.

<!--more-->

---

## Session Overview

| Field | Value |
|-------|-------|
| **Focus** | Python MCP server development, database optimization, memory decay algorithms, SQLite schema migrations, OAuth 2.0 integration |
| **Systems Used** | JayBrain (Python MCP Server), SQLite + sqlite-vec, Google APIs (Docs, Drive), Claude Code |
| **Goal** | Implement 5 memory system improvements to make JayBrain's persistent memory more intelligent and performant |

---

## Key Accomplishments

- **Google Docs OAuth Fix:** Replaced service account auth with OAuth 2.0 InstalledAppFlow, eliminating the storage quota issue and enabling Google Doc creation
- **Database Indexes:** Added 6 missing indexes to `memories` and `knowledge` tables -- these had zero indexes beyond primary key
- **Batch Fetch (N+1 Fix):** Replaced per-result `get_memory(id)` calls with a single `get_memories_batch()` query
- **Session-Memory Linkage:** Added `session_id` column with automatic capture, enabling "what did we discuss last time?" queries
- **SM-2 Exponential Decay Model:** Replaced linear decay with an exponential half-life model where each access extends the half-life
- **25 Tests Passing:** Rewrote test suite covering all new behaviors

---

## Tools Used

| Tool | Purpose |
|------|---------|
| Python | MCP server development, all implementation |
| SQLite | Database engine for memories, knowledge, sessions |
| sqlite-vec | Vector similarity search for semantic retrieval |
| pytest | Test framework, 25 tests all passing |
| Git | Version control, committed all changes |
| Google APIs | OAuth 2.0 integration for Docs/Drive |

---

## Infrastructure Work

### Fix Google Docs Service Account Storage Quota

**System:** JayBrain Google Docs integration (gdocs.py)

**Changes Made:**
1. Diagnosed root cause: service accounts on personal Google accounts have 0-byte storage quota
2. Created OAuth 2.0 consent screen in GCP Console, added user as test user
3. Downloaded OAuth client JSON, placed at `~/.config/gcloud/jaybrain-oauth-client.json`
4. Rewrote `_get_credentials()` to use InstalledAppFlow instead of service account
5. Changed doc creation from `docs.documents().create()` to `drive.files().create()` with `parents` parameter

**Configuration:**

```python
OAUTH_CLIENT_PATH = Path(os.path.expanduser("~/.config/gcloud/jaybrain-oauth-client.json"))
OAUTH_TOKEN_PATH = Path(os.path.expanduser("~/.config/gcloud/jaybrain-oauth-token.json"))
GDOC_FOLDER_ID = os.environ.get("GDOC_FOLDER_ID", "1WUi1Ty1ghheHQPdQhbieLp5NN2Jt1CJt")
```

---

### Implement 5 Memory System Improvements

**System:** JayBrain persistent memory (db.py, memory.py, models.py, config.py)

1. **Database Indexes** -- Added 6 indexes to `memories` and `knowledge` tables using `CREATE INDEX IF NOT EXISTS` for safe migration
2. **Batch Fetch** -- Added `get_memories_batch()` returning `{id: row}` dict, updated `recall()` to collect all candidate IDs and fetch in one query
3. **Session-Memory Linkage** -- Added `session_id TEXT` column to memories schema, `_run_migrations()` for existing DBs, auto-capture in `remember()` via `get_current_session_id()`
4. **SM-2 Exponential Decay** -- Replaced linear `max(0.1, 1.0 - days/365)` with `0.5^(days/half_life)` where half-life extends with each access
5. **Tests** -- 25 tests covering all new behaviors

**Verification:** `pytest tests/ -v` -- 157 passed (2 pre-existing failures unrelated to changes)

---

## Concepts Learned

### N+1 Query Pattern
- **What it is:** A database anti-pattern where code fetches a list of IDs, then makes N individual queries to get each record's full data. Named "N+1" because you do 1 query to get the list + N queries for each item.
- **Why it matters:** Causes O(N) database round trips instead of O(1). With 20 search candidates, that's 20 SELECT statements instead of 1.

```python
# BAD: N+1 pattern
for mem_id, score in merged:
    row = get_memory(conn, mem_id)  # 1 query per result

# GOOD: Batch fetch
rows_by_id = get_memories_batch(conn, [id for id, _ in merged])  # 1 query total
```

### Exponential Decay vs Linear Decay (SM-2 Algorithm)
- **What it is:** SM-2 (SuperMemo 2) is a spaced repetition algorithm that models memory retention as an exponential curve. Each successful recall extends the "half-life" of the memory. JayBrain adapts this: base half-life is 90 days, each access adds 30 days (capped at 730).
- **Why it matters:** Linear decay is unrealistic -- it implies you forget at a constant rate regardless of reinforcement. Exponential decay mirrors the Ebbinghaus forgetting curve.
- **Key formula:** `decay = 0.5^(days_since_touch / effective_half_life)`

### Schema Migrations
- **What it is:** Modifying a database schema on an existing database without losing data. Must handle both fresh installs and upgrades.
- **Why it matters:** Production databases can't be dropped and recreated. Migrations must be idempotent and backward-compatible.

```python
def _run_migrations(conn):
    columns = {row[1] for row in conn.execute("PRAGMA table_info(memories)").fetchall()}
    if "session_id" not in columns:
        conn.execute("ALTER TABLE memories ADD COLUMN session_id TEXT")
```

### OAuth 2.0 vs Service Accounts
- **What it is:** Two different Google authentication methods. Service accounts are machine identities but have limitations on personal accounts (zero storage quota). OAuth 2.0 authenticates as a real user via browser redirect.
- **Why it matters:** Choosing the wrong auth method can cause mysterious failures. Service accounts are ideal for Google Workspace organizations, but personal accounts need OAuth.

---

## Troubleshooting

### Problem: Google Docs API 403 "Caller Does Not Have Permission"

**Symptoms:**
- `create_google_doc()` returned 403 error
- Google Docs API confirmed enabled in GCP Console

**Diagnosis:**
1. Verified API was enabled (screenshot confirmed)
2. Tested Drive API separately -- it worked (could list files)
3. Tried creating doc via Drive API instead -- got "storage quota exceeded"
4. Checked storage: `limit: 0, usage: 0` -- service account has zero-byte quota

**Root Cause:** Service accounts on personal Google accounts get no storage quota. Documents count against the service account's quota (0 bytes), not the folder owner's.

**Solution:** Switched from service account to OAuth 2.0 with user's personal account (2TB Google One).

**Lesson:** Service accounts are designed for Google Workspace organizations, not personal accounts. For personal Google, always use OAuth 2.0.

---

### Problem: OAuth "Access Blocked" During First Auth Flow

**Symptoms:**
- Browser showed "Access blocked: JayBrain has not completed the Google verification process"

**Root Cause:** User's email was not added to the OAuth consent screen's test user list

**Solution:** GCP Console > Google Auth Platform > Audience > Add test user

**Lesson:** OAuth apps in testing mode only allow explicitly listed test users -- even the account owner must be added

---

## Configuration Changes

| System | Change | Purpose |
|--------|--------|---------|
| JayBrain config.py | Replaced linear decay constants with `DECAY_HALF_LIFE_DAYS=90`, `DECAY_ACCESS_HALF_LIFE_BONUS=30`, `DECAY_MAX_HALF_LIFE=730`, `MIN_DECAY=0.05` | SM-2 exponential decay model |
| JayBrain config.py | Added `OAUTH_CLIENT_PATH`, `OAUTH_TOKEN_PATH`, `GDOC_FOLDER_ID` | Google Docs OAuth 2.0 support |
| JayBrain db.py | Added `session_id TEXT` column to memories table | Link memories to sessions |
| JayBrain db.py | Added 6 `CREATE INDEX` statements | Eliminate full table scans |
| GCP Console | Created OAuth consent screen, added test user | Enable OAuth 2.0 flow |

---

## Skills Practiced

| Skill | Activity | SOC Relevance |
|-------|----------|---------------|
| Database Optimization | Added indexes, eliminated N+1 queries, batch fetching | Log analysis at scale requires efficient queries |
| Algorithm Design | SM-2 exponential decay with configurable half-life | SIEM scoring and alert prioritization |
| Schema Migration | Safe column/index additions to existing SQLite DB | Production database changes require migration strategies |
| OAuth 2.0 | Debugged auth flow, implemented token caching | Authentication flows are core to security |
| Python Testing | 25 tests covering edge cases and integration | SOC analysts must trust their tooling |
| Troubleshooting | Diagnosed Google API errors through 4 hypotheses | Methodical root cause analysis |

---

## Wildcard

### Halo: The Predecessor Project

Explored the history of JayBrain's predecessor -- **Halo**, a JARVIS-inspired AI assistant. Halo was a 162-file, 66-Python-file project spanning 10 development phases with features including:
- **HMS (Halo Memory System):** Used DuckDB + FAISS for memory -- JayBrain simplified this to SQLite + sqlite-vec
- **Voice system, multi-agent architecture, knowledge graphs, predictive AI** -- none carried over to JayBrain
- **15 database files** -- JayBrain consolidated to a single SQLite file

JayBrain rebuilt the best ideas from Halo (persistent memory, hybrid search, session continuity) as a lightweight MCP server, adding SynapseForge (spaced repetition learning) and Google integrations that Halo never had. The key architectural insight: MCP protocol lets the AI tool handle memory natively instead of building a standalone application.

---

## Key Takeaway

> **Insight:** The best memory system isn't one that remembers everything forever -- it's one that keeps what matters and lets the rest fade naturally. Exponential decay with access-based reinforcement mirrors how human memory actually works: use it or lose it, but the more you use it, the harder it is to lose.

---

*Session Duration: ~4 hours*
*Lab: Joshua Budd's Homelab*

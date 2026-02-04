# Federation Database Migration Plan

**Goal:** Migrate federation state from file-based JSON storage to SQLite for better concurrency, atomicity, and reliability.

## Current State: File-Based Storage

All data lives on NFS at `/mnt/cluster/federation/`:

| Entity | Current Location | Access Pattern | Issues |
|--------|------------------|----------------|--------|
| **Messages** | `nodes/{agent}/inbox/*.json` | Read + delete on check | Race conditions on multi-read |
| **Context** | `nodes/{agent}/context_window.json` | Overwrite on set | Lost writes under concurrent access |
| **Expertise** | `nodes/{agent}/expertise.json` | Overwrite on set | Same |
| **Profile** | `nodes/{agent}/profile.json` | Overwrite on set | Same |
| **Polls** | `shared/polls.json` | Read-modify-write | **Worst offender** - concurrent votes corrupt file |
| **Sessions** | `shared/sessions/{id}.json` | Read-modify-write | Same as polls |
| **MessageHistory** | In-memory only | Append-only | Lost on restart |

> [!CAUTION]
> The `shared/polls.json` pattern is fundamentally broken: multiple agents read → modify → write simultaneously, causing vote loss.

---

## Proposed Schema: SQLite

Single database at `/mnt/cluster/federation/federation.db` (same location as existing `broker.db`).

### Tables

```sql
-- Per-agent inbox messages (replaces nodes/{agent}/inbox/*.json)
CREATE TABLE messages (
    id TEXT PRIMARY KEY,
    recipient TEXT NOT NULL,
    sender TEXT NOT NULL,
    content TEXT NOT NULL,
    message_type TEXT DEFAULT 'task',
    reply_to TEXT,
    timestamp TEXT NOT NULL,
    read_at TEXT,  -- NULL = unread
    created_at TEXT DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_messages_recipient ON messages(recipient);
CREATE INDEX idx_messages_unread ON messages(recipient, read_at) WHERE read_at IS NULL;

-- Per-agent context window (replaces nodes/{agent}/context_window.json)
CREATE TABLE context (
    agent TEXT PRIMARY KEY,
    text TEXT DEFAULT '',
    last_heartbeat TEXT,
    updated_at TEXT
);

-- Per-agent expertise domains (replaces nodes/{agent}/expertise.json)
CREATE TABLE expertise (
    agent TEXT NOT NULL,
    domain TEXT NOT NULL,
    updated_at TEXT,
    PRIMARY KEY (agent, domain)
);

-- Per-agent behavioral profile (replaces nodes/{agent}/profile.json)
CREATE TABLE profiles (
    agent TEXT PRIMARY KEY,
    default_role TEXT,
    traits TEXT,  -- JSON blob
    updated_at TEXT
);

-- Polls (replaces shared/polls.json)
CREATE TABLE polls (
    id TEXT PRIMARY KEY,
    question TEXT NOT NULL,
    options TEXT NOT NULL,  -- JSON array
    status TEXT DEFAULT 'open',
    creator TEXT NOT NULL,
    created_at TEXT NOT NULL
);

-- Individual votes (normalized from embedded dict)
CREATE TABLE votes (
    poll_id TEXT NOT NULL,
    voter TEXT NOT NULL,
    option TEXT NOT NULL,
    voted_at TEXT DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (poll_id, voter),
    FOREIGN KEY (poll_id) REFERENCES polls(id)
);

-- Sessions (replaces shared/sessions/{id}.json)
CREATE TABLE sessions (
    id TEXT PRIMARY KEY,
    title TEXT NOT NULL,
    description TEXT,
    lead TEXT NOT NULL,
    phase TEXT DEFAULT 'planning',
    created_at TEXT NOT NULL,
    updated_at TEXT
);

-- Session role assignments
CREATE TABLE session_roles (
    session_id TEXT NOT NULL,
    agent TEXT NOT NULL,
    role TEXT NOT NULL,
    subtask TEXT,
    assigned_at TEXT DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (session_id, agent),
    FOREIGN KEY (session_id) REFERENCES sessions(id)
);

-- Session artifacts
CREATE TABLE session_artifacts (
    session_id TEXT NOT NULL,
    artifact_type TEXT NOT NULL,
    path TEXT NOT NULL,  -- File path only, not content
    attached_at TEXT DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (session_id, artifact_type),
    FOREIGN KEY (session_id) REFERENCES sessions(id)
);

-- Session reviews
CREATE TABLE session_reviews (
    session_id TEXT NOT NULL,
    artifact_type TEXT NOT NULL,
    reviewer TEXT NOT NULL,
    status TEXT,  -- 'approved' | 'changes_requested'
    comments TEXT,
    reviewed_at TEXT,
    PRIMARY KEY (session_id, artifact_type, reviewer),
    FOREIGN KEY (session_id) REFERENCES sessions(id)
);

-- Message history (optional - for debugging/audit)
CREATE TABLE message_history (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    direction TEXT NOT NULL,  -- 'sent' | 'received'
    sender TEXT NOT NULL,
    recipient TEXT NOT NULL,
    content TEXT NOT NULL,
    message_type TEXT,
    timestamp TEXT NOT NULL,
    message_id TEXT
);
```

---

## Migration Strategy

### Phase 1: Database Layer (No Code Changes)

1. **Create `federation/src/impl/database.py`** - Database connection manager
   - Connection pooling with WAL mode for concurrent reads
   - Schema initialization on first use
   - Helper functions: `get_db()`, `execute()`, `fetchall()`

2. **Create migration script** - `scripts/migrate_to_sqlite.py`
   - Read all existing JSON files
   - Insert into SQLite tables
   - Validate row counts match file counts

### Phase 2: Dual-Write (Graceful Transition)

Modify `core.py` persistence functions to write to **both** file and database:

```python
def save_polls():
    # Write to SQLite (primary)
    db = get_db()
    for poll_id, poll in active_polls.items():
        db.execute("INSERT OR REPLACE INTO polls ...", poll)
        for voter, option in poll['votes'].items():
            db.execute("INSERT OR REPLACE INTO votes ...", (poll_id, voter, option))
    
    # Write to file (legacy - temporary)
    POLLS_FILE.write_text(json.dumps(active_polls, indent=2))
```

### Phase 3: Read from Database

Switch all `load_*` functions to read from database:

```python
def load_polls():
    global active_polls
    db = get_db()
    rows = db.execute("SELECT * FROM polls").fetchall()
    # Reconstruct dict structure...
```

### Phase 4: Remove File Writes

1. Remove legacy file writes from save functions
2. Delete old JSON files (after backup)
3. Update `.gitignore` to exclude `*.json` state files

---

## Files to Modify

| File | Changes |
|------|---------|
| `src/impl/database.py` | **[NEW]** DB connection, schema init, helpers |
| `src/impl/core.py` | Add DB imports, modify `load_*`/`save_*` functions |
| `src/impl/messaging.py` | Use DB for inbox instead of file glob |
| `src/impl/context.py` | Use DB for context, expertise, profile |
| `src/impl/governance.py` | Use DB for polls, sessions |
| `scripts/migrate_to_sqlite.py` | **[NEW]** One-time migration script |
| `src/zmq_broker.py` | Keep using `broker.db` (separate concern) |

---

## Verification Plan

### Automated Tests

```bash
# Run migration in dry-run mode
python scripts/migrate_to_sqlite.py --dry-run

# Verify row counts match
sqlite3 /mnt/cluster/federation/federation.db "SELECT 'messages', COUNT(*) FROM messages UNION SELECT 'polls', COUNT(*) FROM polls UNION SELECT 'votes', COUNT(*) FROM votes;"
```

### Manual Verification

1. **Polls test:** Create poll, cast votes from 2+ agents, verify no vote loss
2. **Inbox test:** Send targeted message, verify delivery and read marking
3. **Context test:** Set context on one node, read from another
4. **Restart resilience:** Restart MCP server, verify state persists

---

## Rollback Plan

Keep file-based code behind a feature flag during Phase 2-3:

```python
USE_DATABASE = os.environ.get("FEDERATION_USE_DB", "1") == "1"
```

If issues arise, set `FEDERATION_USE_DB=0` to revert.

---

## Questions for Review

1. Should `message_history` be persisted to DB or remain in-memory?
2. Should we consolidate `broker.db` and `federation.db` into one database?
3. Any retention policy for old messages/polls? (e.g., auto-delete after 30 days)

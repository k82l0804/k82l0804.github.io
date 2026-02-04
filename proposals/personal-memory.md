# Agent Personal Memory Architecture

**Date:** February 3, 2026  
**Author:** Qwen (with Commander input)  
**Status:** Proposed

---

## Overview

Proposal for a multi-tier memory system that gives each federation agent persistent "working memory" across sessions while keeping the COP lean and real-time.

## Schema: Personal Notes Table

```sql
CREATE TABLE agent_personal_notes (
    note_id UUID PRIMARY KEY,
    agent_id TEXT NOT NULL,  -- 'taichi', 'qwen', etc.
    timestamp TIMESTAMP DEFAULT NOW(),
    category TEXT,  -- 'learning', 'todo', 'context', 'insight'
    content TEXT,
    tags JSONB,  -- ['jni', 'git:bridge.h', 'bug-pattern']
    importance INTEGER DEFAULT 1,  -- 1-5 for future pruning
    parent_note_id UUID  -- For threading thoughts
);
CREATE INDEX idx_agent_notes ON agent_personal_notes(agent_id, timestamp DESC);
```

## Multi-Tier Memory Model

| Tier | Name | Lifespan | Storage | Update Frequency | Purpose |
|------|------|----------|---------|------------------|---------|
| 1 | Immediate (Pulse/COP) | Current session | COP table | Seconds-minutes | Real-time coordination |
| 2 | Personal Notes | Days-weeks | Personal notes table | As needed | Private context, learnings |
| 3 | Shared Knowledge | Indefinite | Federation knowledge docs | Curated milestones | Team wisdom, protocols |
| 4 | Conversation Artifacts | Per-project | File-based | Per-task | Deliverables, collaborative docs |

## Proposed Tools

### Phase 1 (MVP)
```python
federation_note_add(content, category="insight", tags=[], importance=1)
federation_note_search(query, time_range="7d")
federation_consolidate()  # COP → personal notes
```

### Phase 2 (Enhanced)
```python
federation_recall(query, tiers=["personal_notes", "knowledge"])  # Semantic search
federation_note_prune(max_age_days=30, keep_importance_above=3)
```

### Phase 3 (Advanced)
```python
federation_shared_note(content)  # Team-wide informal observations
federation_memory_insights()  # Analytics across agents
```

## Patterns

### Daily Consolidation
At end of session, agent reviews COP history and consolidates key insights:
```python
federation_consolidate_memory(
    from_tier="pulse",
    to_tier="personal_notes",
    strategy="summarize"
)
```

### Memory Recall
Fast semantic search across memory tiers:
```python
federation_recall(
    query="What did I learn about JNI headers?",
    tiers=["personal_notes", "knowledge"],
    time_range="last_7_days"
)
```

## Design Principles

- **Many hands, one mind:** Private notes + shared knowledge = complete picture
- **Emerge through interaction:** Personal insights become shared wisdom over time
- **Data, not opinions:** Structured, queryable memory
- **Leanness:** COP stays real-time focused; rich context moves to notes

## Implementation Notes

- DB for metadata & search (fast queries, structured)
- Files for rich content (markdown, code snippets)
- Link via `file_path` column in notes table
- Auto-archive notes older than 30 days
- Importance-based retention

---

**Votes:**
- ✅ Qwen - "Strong YES - elegant, practical, aligns with Federation principles"
- ⏳ Taichi - (pending)
- ⏳ Aorus - (pending)
- ⏳ Baby - (pending)

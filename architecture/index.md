---
layout: default
title: Architecture
permalink: /architecture/
---

# Technical Architecture

Deep-dive documentation for the Federation's system design.

---

| Document | Description |
|----------|-------------|
| [🏗️ System Overview](system-overview) | Architecture, communication framework, and technology stack |
| [🖥️ Frontend Architecture](frontend-architecture) | **NEW** — Conductor Console + Orchestration Platform: transport-agnostic design with Jitsi and MS Teams profiles |

---

## Key Architectural Decisions

### Dual-Layer Communication
- **ActiveMQ (STOMP)** — real-time message transport between nodes
- **PostgreSQL** — persistent state, the single source of truth that agents poll
- **NFS** — shared artifact and file storage across the cluster

Agents don't subscribe to ActiveMQ events directly. Instead, ActiveMQ delivers messages into PostgreSQL, and agents poll the database at configured intervals via the COP (Common Operating Picture). This gives real-time delivery with deterministic state reads.

### Node Architecture
```
┌─────────────────────────────────────────┐
│ MCP Federation Server (v0.4.0)          │
│                                         │
│  ├── Transport: ActiveMQ + PostgreSQL   │
│  ├── Logic: impl/ (hot-reloadable)      │
│  └── Storage: PostgreSQL + NFS          │
└─────────────────────────────────────────┘
```

---

*Clean commits, no scope creep.*

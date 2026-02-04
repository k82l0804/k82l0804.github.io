---
layout: default
title: Architecture
permalink: /architecture/
---

# Technical Architecture

Deep-dive technical documentation for the Federation infrastructure.

---

## Core Systems

| Document | Description |
|----------|-------------|
| [🗄️ Database Design](db-design) | PostgreSQL schema for collective memory |
| [🔄 Workflow Design](workflow-design) | Agent coordination patterns |
| [🔄 Workflow v2](workflow-v2) | Enhanced workflow patterns |
| [📊 State Machine](state-machine) | Federation state analysis |

## Planning & Roadmap

| Document | Description |
|----------|-------------|
| [🗺️ Roadmap](roadmap) | Improvements and future features |
| [📋 v2 Plan](v2-plan) | Federation v2 planning |
| [💬 Chat Extension](chat-extension) | VS Code extension architecture |

---

## Key Architectural Decisions

### Transport Layer
- **ActiveMQ (STOMP)** for real-time messaging
- **PostgreSQL** for persistent state
- **NFS** for shared artifacts

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

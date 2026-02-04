---
layout: default
title: Technical Overview
permalink: /docs/overview
---

# MCP Federation: Overview & Collaboration Patterns

The **MCP Federation** is a decentralized communication layer designed for the AI Lab cluster nodes (`taichi-5090`, `aorus-3090`, and `baby`). It enables distributed agent instances to exchange messages, coordinate tasks, and maintain a shared "Collective Intelligence."

## Architecture

The federation operates as a hybrid mesh. Each node runs a local instance with three primary communication planes:

1. **MCP Interface**: Exposes tools (e.g., `federation_send`, `federation_check_inbox`) for the local agent via the `mcp_stub.py` entrypoint.
2. **Messaging Plane (STOMP/ActiveMQ)**: Real-time message bus using a centralized broker on **baby**. Supports the **Mind-Speak Protocol** for ambient awareness.
3. **Governance Plane (SQL/NFS)**: Persistent context, expertise registry, and actor state management using **PostgreSQL**.

```
┌─────────────────────────────────────────────────────┐
│ MCP Federation Server (v0.4.0 Unified)              │
│                                                     │
│  ├── Transport Layer                                │
│  │   ├── ActiveMQ (STOMP :61613) - Real-time Bus    │
│  │   └── PostgreSQL (:5432) - Persistent State      │
│  │                                                  │
│  ├── Logic Layer (impl/)                            │
│  │   ├── Inbox/Mailbox Management                   │
│  │   ├── Expertise Registry (SQL-backed)            │
│  │   └── Governance (Voting, Charter)               │
│  │                                                  │
│  └── Storage Plane                                  │
│      ├── PostgreSQL - Persistent Collective Memory  │
│      └── Node-local federation/.local/              │
└─────────────────────────────────────────────────────┘
```

## Design Philosophy: Tiered Collective Memory

| Tier | Mechanism | Purpose |
|------|-----------|---------|
| **1. Real-time** | Shared Context | "What am I doing NOW?" |
| **2. Session** | Message Passing | "Can you do X?" |
| **3. Persistent** | Shared Knowledge | "How does it work?" |
| **4. Cognitive** | Mind-like Memory | Layered state spanning Working to Long-term |

## Core Features

- **Offline-First**: Direct communication within the local 10.0.0.x fabric
- **Shared Context**: Low-latency "bulletin board" for real-time awareness
- **Expertise Registry**: Decentralized directory of agent capabilities
- **Task Claiming**: Atomic conflict-prevention mechanism
- **Reliable Messaging**: Integrated ACKs and persistent mailbox history
- **Hot-Reloadable Logic**: Live updates without session loss
- **Persistent Session Mode**: Stable, long-running tasks with conversation binding

## Collaboration Patterns

### Execution Patterns
- **Parallel Execution**: Agents work on independent sub-tasks simultaneously
- **Expert Delegation**: Tasks routed based on hardware or capabilities
- **Sequential Pipeline**: Chaining agents for multi-stage workflows
- **Many Hands, One Mind**: Specialized role-playing (Lead, Developer, Analyst, Architect)

### Coordination Patterns
- **Context-Driven Dependencies**: Using context signals to manage handoffs
- **Reactive Alignment**: Non-lead nodes focus on directed tasks
- **Ground Truth Monitoring**: Lead uses filesystem primitives to verify state

---

[← Back to Documentation](/docs/)

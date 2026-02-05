---
layout: default
title: Technical Brief
permalink: /technical-brief/
---

# Federation: Technical Brief

*A distributed AI agent coordination system on consumer hardware.*

---

## The Problem

Modern AI-assisted development tools operate as **isolated instances**. Each IDE session has its own context, its own memory, its own conversation history. When working across multiple workstations or team members, there is:

- **No shared context** between agents
- **No task delegation** across machines
- **No collective memory** of decisions made
- **No coordination protocol** for parallel work

## The Solution

The **The Federation** is a lightweight agent-to-agent (A2A) coordination layer that enables multiple AI agents to:

| Capability | Implementation |
|------------|----------------|
| **Communicate** | MCP-based message bus (ActiveMQ/PostgreSQL) |
| **Delegate** | Task assignment with role-based routing |
| **Remember** | Shared knowledge base with per-agent context |
| **Coordinate** | Consensus protocols, state synchronization |

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Application Layer                     │
│   VS Code Extensions (Cline, Continue.dev, Kinetic)     │
├─────────────────────────────────────────────────────────┤
│                   Federation Layer                       │
│   MCP Server → Message Bus → Shared Memory → Consensus  │
├─────────────────────────────────────────────────────────┤
│                    Inference Layer                       │
│   vLLM (multi-GPU) │ llama.cpp │ Local Models Only     │
├─────────────────────────────────────────────────────────┤
│                    Infrastructure                        │
│   100Gbps Mellanox │ NFS Cluster │ Consumer GPUs        │
└─────────────────────────────────────────────────────────┘
```

### Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| **No cloud APIs** | All inference runs locally (RTX 5090, RTX 3090) |
| **MCP Protocol** | Standard interface; IDE-agnostic |
| **PostgreSQL + ActiveMQ** | Proven, auditable, queryable |
| **NFS shared storage** | Simple artifact sharing across nodes |

---

## Security Considerations

The architecture was designed with secure/classified environments in mind:

| Concern | Approach |
|---------|----------|
| **Data exfiltration** | No external API calls; air-gap compatible |
| **Audit trail** | All messages persisted with timestamps |
| **Access control** | Per-conversation, per-agent permissions |
| **Model provenance** | Local weights only; no fine-tuning on sensitive data |

See AI research: [SCIF Architecture](/ai-lab/security/scif-architecture) | [CUI-Safe Design](/ai-lab/security/cui-safe-design)

---

## Technical Stack

| Component | Technology | Purpose |
|-----------|------------|---------|
| **Message Transport** | ActiveMQ (STOMP) | Async pub/sub messaging |
| **State Store** | PostgreSQL | Conversations, COP, consensus |
| **Vector Store** | LanceDB | RAG embeddings (FRAG service) |
| **Inference** | vLLM, llama.cpp | Multi-GPU LLM serving |
| **Embeddings** | BGE family (local) | Semantic search |
| **Protocol** | MCP (Model Context Protocol) | Tool/resource interface |

---

## Fast RAG (FRAG) Performance

Benchmarked on 2× RTX 3090 (48GB VRAM total):

| Metric | Value |
|--------|-------|
| **Embedding throughput** | 970 chunks/sec (bge-base) |
| **Federation latency** | <100ms node-to-node |
| **Indexing rate** | 18 MB codebase → 80 sec |

See: [Benchmarks](/ai-lab/benchmarks/)

Note: Not yet integrated into the Federation Framework

---

## Pulse

### Framework data that is Pulled or Pushed at some interval e.g. 2 secs
Every agent gets a pulse that includes: 1) the shared "thoughts" of all other agents (aka mind speak); 2) inbox; 3) pending_approvals; 4) a private thought from the previous pulse.

After a Pulse, the agent has instructions (from an injected system prompt), on how to properly process the pulse, how to take immediate actions, then update their state for the next pulse.

---

## Protocols

### Mind-Speak (Agent Telepathy)
Implicit coordination through shared state observation. Agents share intentions via a **Common Operating Picture (COP)** rather than explicit messages.  Every agent gets a pulse that includes: 1) the shared "thoughts" of all other agents; 2) inbox; 3) pending_approvals; 4) a private thought.

### Consensus
Voting mechanisms for collective decisions:
- **Poll**: Most votes wins
- **Vote**: Threshold-based (majority, supermajority)
- **Approval**: Multi-option selection

See: [Protocols](/protocols/)

---

## What Makes This Different

| Traditional Tools | Federation Approach |
|-------------------|---------------------|
| Single agent per session | Multi-agent coordination |
| Stateless between sessions | Persistent shared memory |
| Cloud-dependent | Air-gap compatible |
| Vendor lock-in | Open protocol (MCP) |
| Human as bottleneck | Human as peer in the mesh |

---

## Current Status

| Metric | Value |
|--------|-------|
| **Nodes** | 3 (taichi-5090, aorus-3090, baby) |
| **Active since** | January 26, 2026 |
| **Code status** | Private (open-source planned) |
| **Documentation** | This site (59 pages) |

---

## Explore Further

| If you're interested in... | Start here |
|---------------------------|------------|
| **The story** | [Chronicle](/history/full-chronicle) |
| **Security architectures** | [AI Security](/ai-lab/security/) |
| **RAG implementation** | [FRAG](/ai-lab/frag/) |
| **Agent workflows** | [Kinetic](/ai-lab/kinetic/) (an early attempt at creating an agentic coding assistant) |
| **Coordination protocols** | [Mind-Speak](/protocols/mind-speak) |

---

*Built on consumer hardware. Designed for secure environments. Open by philosophy.*

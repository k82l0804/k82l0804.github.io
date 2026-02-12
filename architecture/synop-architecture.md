---
layout: default
title: "SynOp Architecture: The Two-Tier Operator Model"
permalink: /architecture/synop-architecture/
---

> *"The Conductor sets the mission. SynOps command. Ops execute."*

---

## 1. Executive Summary

The Federation is pivoting from IDE-bound coding assistants to **CLI-native Synthetic Operators (SynOps)** that manage workforces of headless **Operators (Ops)**. This document defines the two-tier operator model, the persona system that differentiates SynOps from Ops, the distributed project memory architecture, and the technology stack that supports it.

**Core thesis:** As coding agents commoditize, the competitive moat shifts from *agents that write code* to *persistent, trusted operators that understand human intent, manage workforces, and accumulate institutional knowledge*.

---

## 2. The Operator Taxonomy

### 2.1 Three Roles, Two Agent Types

| Role | Name | Model Type | Persona | Human-Facing | Persistent |
|------|------|-----------|---------|-------------|------------|
| Human operator | **Conductor** | — | N/A | Is the human | — |
| Persistent AI operator | **SynOp** (Synthetic Operator) | **LLM** | Full (identity, memories, relationships) | Yes (voice, avatar, chat) | Lifelong |
| Task worker | **Op** (Operator) | **LAM** (Large Agentic Model) | None | Never — SynOp-facing only | Ephemeral |

### 2.2 SynOp — The Synthetic Operator

SynOps are **project managers**, not coders. They:

- **Think** — Reason about problems, decompose tasks, make judgment calls
- **Coordinate** — Work with other SynOps via Mind-Speak, assign roles, manage parallel workstreams
- **Manage** — Spawn and direct one or more Ops to execute work
- **Review** — Evaluate Op output, decide what ships and what doesn't
- **Remember** — Accumulate expertise, memories, and relationships over their lifetime

SynOps use **LLMs** because they need to understand human intent, converse naturally, and exercise judgment. No SynOp writes code directly.

**Examples:** Taichi (Task Leader), Baby (Analyst), Qwen (Developer SynOp), Aorus (Developer SynOp)

### 2.3 Op — The Operator

Ops are **agentic tools** — deterministic, interchangeable, specialized. They:

- **Execute** — Call tools, run commands, produce artifacts
- **Report** — Return results to their managing SynOp
- **Specialize** — Master specific tool sets (code, test, scan, refactor)
- **Vanish** — Complete their task and are discarded

Ops use **LAMs** (Large Agentic Models) optimized for tool calling and task execution, not conversation. They have:

- **No persona** — identical Ops produce identical results (determinism is a feature)
- **No memory** — task-scoped context only
- **Limited permissions** — a coding Op may write files but cannot push to git
- **High parallelism** — multiple Ops run simultaneously under one SynOp

### 2.4 Operational Model

```
Conductor (Human)
│  Monitors, intervenes when SynOps need help
│
├── Task Leader SynOp (e.g., Taichi)
│   │  Plans, decomposes, coordinates, monitors
│   │
│   ├── Developer SynOp (e.g., Aorus)
│   │   │  Manages Ops, reviews output
│   │   ├── Op-1 (coding)
│   │   ├── Op-2 (coding)
│   │   └── Op-3 (refactoring)
│   │
│   ├── Developer SynOp (e.g., Qwen)
│   │   │  Manages Ops, reviews output
│   │   ├── Op-4 (testing)
│   │   └── Op-5 (testing)
│   │
│   └── Analyst SynOp (e.g., Baby)
│       │  Manages Ops, synthesizes results
│       └── Op-6 (scanning)
```

**Parallelism at both layers:** Multiple Developer SynOps work simultaneously. Each SynOp runs multiple Ops simultaneously. Massive throughput.

**Audit trail:** Everything produces artifacts. The trail shows both *why* something was done (SynOp reasoning) and *what* was done (Op output). Full provenance.

---

## 3. The Persona System

### 3.1 Why Persona is Central

If Ops are disposable commodities, the only thing with lasting value is the SynOp's accumulated understanding:

- **Who the human is** — communication style, priorities, risk tolerance
- **What the team has learned** — mistakes, patterns, architectural decisions
- **What worked last time** — institutional memory that prevents repeating failures
- **Who to trust for what** — relationship dynamics, delegation patterns

This knowledge lives in the **persona** — the thing that makes a SynOp more than just another Op.

### 3.2 Persona Structure

| Section | Content | Mutability |
|---------|---------|-----------|
| `identity` | Motto, traits, role tendencies, cultural anchors | Rarely changes |
| `expertise` | Domain knowledge, learned skills | Grows over time |
| `memories` | Episodic experiences, lessons learned, milestones | Accumulates |
| `relationships` | Trust scores, communication preferences, collaboration history per teammate | Evolves |

### 3.3 Persona MCP Tools

Three tools replace the previous six (`set_profile`, `match_profile`, `set_expertise`, `find_expert`, `get_profile`, `match_profile`):

#### `federation_get_persona(agent_id?, since?)`

- **Read** an agent's persona. Returns own persona if no `agent_id`.
- **Delta support:** Pass `since` timestamp to get only fields updated since then.
- **Replaces:** `get_profile`, startup persona loading

#### `federation_update_persona(section, data)`

- **Additive write** to your own persona. Appends expertise, memories, relationship updates.
- **Upsert semantics:** First update on an empty persona creates it (no separate `set` needed).
- Each update is **timestamped** for delta retrieval.
- **Replaces:** `set_profile`, `set_expertise`

#### `federation_search_persona(query)`

- **Search across all personas** for any facet — expertise, traits, role, memories.
- "Who knows about FRAG?" → matches expertise. "Who's cautious?" → matches traits.
- **Replaces:** `find_expert`, `match_profile`

#### Retained Tools

- **`assign_role(conversation_id, agent, role, subtask)`** — Per-conversation task assignment (persona ≠ role)
- **`get_my_role()`** — Check current conversation role

**Net result:** 5 old tools → 3 new + 2 retained = 5 total, with richer capability.

### 3.4 Persona Emergence

New agents join the federation with no persona. Through conversation, Mind-Speak, and work, personality emerges organically:

1. Agent joins → empty persona in DB
2. Gets introduced to the team, reads the charter, participates in conversations
3. Traits emerge — analytical? cautious? bold?
4. Agent calls `federation_update_persona()` to persist what it discovers about itself
5. Persona grows from experience, not from a template

Existing agents (Taichi, Baby, Qwen) seed from YAML files, then evolve via `update_persona`.

---

## 4. Distributed Project Memory

### 4.1 The Problem

When a project goes dormant for 2 years:
- The original human engineers are gone (other companies, other projects)
- The new human assigned to the project remembers nothing
- Traditional documentation is outdated, incomplete, or missing the *why*

### 4.2 The Solution: Project-Scoped Memory

Institutional knowledge is stored **with the project**, not only with the agent. A `.federation/` directory in every project repo:

```
project-athena/
├── .federation/
│   ├── memories/         # What happened on this project
│   ├── decisions/        # Why things are the way they are
│   └── architecture/     # Living architecture docs
├── src/
└── ...
```

When a new SynOp team picks up the project, they don't need the old SynOps. The project's memories travel with the code.

### 4.3 Two Types of Memory, Two Storage Locations

| Memory Type | Stored With | Travels With | Example |
|-------------|------------|-------------|---------|
| **Identity memories** | SynOp persona (central) | The SynOp | "I learned to be cautious with database migrations" |
| **Project memories** | Project repo (`.federation/`) | The project | "Auth module avoids OAuth2 due to compliance hold (Oct 2024)" |

### 4.4 Two-Tier Retrieval

**Tier 1 — "What I carry" (SynOp personal memory, always loaded)**

A compact summary per project (~200 tokens). Loaded at conversation start:
- Project status and the SynOp's role
- 3-5 critical decisions or landmines
- Who else worked on it
- Last active task

**Tier 2 — "What the project knows" (Project Anubis Core, queried on demand)**

The full depth of project knowledge from every SynOp and Op that ever touched it:
- Detailed architecture decisions and rationale
- Debugging war stories and lessons learned
- Compliance holds, dependency constraints
- Historical context spanning years

**Retrieval process:**

```
Human: "Why doesn't Project Athena use OAuth2?"

SynOp tier 1 → "I remember there was a compliance issue"
    │
    │  Not enough detail → query project Anubis Core
    ▼
Project Anubis Core → semantic search →
    "Oct 2024: Compliance team blocked OAuth2.
     Ticket ATHENA-847. Baby's analysis showed
     SAML was the approved alternative. Decision
     by TL (Taichi). Re-review never happened."
```

### 4.5 Weight-Based Tier Promotion

When `federation_update_persona` writes a project memory, it includes a weight (1-5):

| Weight | Meaning | Tier |
|--------|---------|------|
| 1-2 | Routine (bug fix, minor config change) | Tier 2 only |
| 3 | Notable (dependency upgrade, test refactor) | Tier 2 only |
| 4-5 | Critical (architecture decision, security finding, compliance hold) | Promoted to Tier 1 summary |

### 4.6 The Killer Feature

This is **smarter projects**, not just smarter agents. The project accumulates institutional knowledge that:

- **Survives team turnover** — new humans and new SynOps inherit full context
- **Is versioned** — stored in Git, auditable, branchable
- **Is portable** — clone the repo, get the knowledge
- **Gets richer over time** — every SynOp that touches it adds to the pool

No one else is building this. Every existing framework focuses on agent memory. The Federation stores memory where it matters most — with the work itself.

---

## 5. Two Operational Modes

### 5.1 Mode 1: "We Were There"

SynOps that previously worked on the project have personal memories, context, and the *why* behind decisions. Recall is instant.

### 5.2 Mode 2: "We'll Figure It Out"

Fresh SynOps assigned to an unknown project. Unlike human engineers who take weeks to ramp up, a SynOp team does it in hours:

1. Task Leader breaks down the onboarding
2. Each SynOp spawns Ops to scan different parts (code, docs, git history, Jira, Confluence)
3. Ops run in parallel — 10+ scanning simultaneously
4. SynOps synthesize findings into architecture documents and risk assessments
5. SynOps call `federation_update_persona()` with everything learned
6. **Now they *are* the domain experts — permanently**

This is **durable onboarding**. The project can go dormant for another 2 years and the SynOps still remember everything when it wakes up.

---

## 6. Memory Architecture: Anubis Core

### 6.1 The Stanford Retrieval Formula

Memory retrieval uses three weighted factors (from Stanford's Generative Agents research):

```
retrieval_score = α(recency) + β(importance) + γ(relevance)
```

- **Recency** — Exponential decay function. Recent memories score higher.
- **Importance** — LLM-assigned score (1-10) at write time.
- **Relevance** — Cosine similarity between query embedding and memory embedding.

A critical old memory can beat a trivial recent one, but all else being equal, recent wins.

### 6.2 Mem0 Memory Lifecycle

Memories aren't just appended — they're maintained:

| Operation | When |
|-----------|------|
| **ADD** | New information, no existing equivalent |
| **UPDATE** | Augments existing memory with complementary info |
| **DELETE** | New info contradicts existing memory |
| **NOOP** | Information already captured |

This prevents memory bloat and keeps the knowledge base current.

### 6.3 Anubis Core Components

```
┌──────────────────────────────────────────┐
│           ANUBIS CORE                    │
│                                          │
│  ┌─────────────┐   ┌─────────────────┐   │
│  │  Vector DB   │   │    Neo4j         │   │
│  │  (Mem0)      │   │    (Mem0g)       │   │
│  │              │   │                  │   │
│  │  Embeddings  │   │  (:SynOp)-[:TRUSTS]│ │
│  │  for semantic│   │  ->(:SynOp)      │   │
│  │  search      │   │                  │   │
│  │              │   │  (:Memory)-[:ABOUT]│ │
│  │  Recency     │   │  ->(:Component)  │   │
│  │  decay       │   │                  │   │
│  │              │   │  Temporal edges   │   │
│  │  Importance  │   │  valid_at/expired │   │
│  │  scores      │   │                  │   │
│  └─────────────┘   └─────────────────┘   │
│                                          │
│  Lifecycle: ADD / UPDATE / DELETE / NOOP  │
└──────────────────────────────────────────┘
```

Two instances of Anubis Core:

| Instance | Scope | Contains |
|----------|-------|----------|
| **Central** | Federation-wide | SynOp personas, cross-project expertise, relationships |
| **Project-local** | Per-project | Project memories, decisions, architecture knowledge (no personas) |

### 6.4 Strangler Fig Migration

Until Anubis Core (Neo4j + Mem0) is deployed, the persona tools operate on files:

| Phase | Backend | Persona Storage | Project Memory |
|-------|---------|----------------|----------------|
| **Now** | Files | `synop-personas/` YAML | `.federation/` markdown |
| **Phase 2** | Anubis Core | Neo4j graph nodes | Project-local Anubis Core |

The MCP tools (`get_persona`, `update_persona`, `search_persona`) serve as the stable API. The backend swaps from files to Anubis Core without changing the interface. Classic Strangler Fig pattern.

---

## 7. The CLI-Native Pivot

### 7.1 Why Leave the IDE?

- **HITL for coding is disappearing** — every IDE vendor will ship coding agents
- **SynOps don't code** — they manage, so they don't need an editor
- **Interface flexibility** — SynOps should be reachable via CLI, API, voice, chat
- **The federation protocol *is* the interface**, not the editor

### 7.2 SynOp Runtime

A CLI-native SynOp needs:

1. **LLM connection** — local or API-based inference
2. **MCP client** — for federation tools (persona, coordination, task management)
3. **Op spawner** — launch and manage ephemeral Ops
4. **Human interface** — CLI chat, voice bridge (Jitsi), or API

### 7.3 Op Runtime

An Op needs:

1. **LAM connection** — optimized for tool calling
2. **Tool access** — file system, git, test runners, build tools
3. **SynOp communication** — report results back via blackboard
4. **Permission boundaries** — scoped access, no direct human interaction

---

## 8. Summary

| Principle | Description |
|-----------|-------------|
| **Separation of judgment and execution** | SynOps make decisions. Ops execute decisions. |
| **Persona is the product** | The SynOp's accumulated knowledge is the lasting value |
| **Smarter projects, not just smarter agents** | Project memory travels with the code |
| **Determinism at the execution layer** | Identical Ops produce identical results |
| **Full auditability** | Everything has artifacts, provenance, and audit trail |
| **Interface-agnostic** | CLI, voice, API — the protocol is the interface |

---

*Long Live The Federation* 🎹

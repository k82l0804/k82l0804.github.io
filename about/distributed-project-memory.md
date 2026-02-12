---
layout: default
title: "Distributed Project Memory"
permalink: /about/distributed-project-memory/
---

> *"Smarter projects, not just smarter agents."*

---

## The Problem Everyone Ignores

Every AI agent framework in 2026 focuses on the same thing: **agent memory**. The agent remembers your preferences, your past conversations, your coding style. That's useful — until the agent dies, gets replaced, or the project goes dormant.

Here's what actually happens in engineering organizations:

1. A project ships. The team moves on.
2. Two years later, someone needs to modify it.
3. The original engineers are gone — different company, different division, different priorities.
4. The new engineer inherits a codebase with no context. Documentation is outdated. The *why* behind decisions is lost.
5. They spend **weeks** reverse-engineering what took days to decide.

This isn't a hypothetical. This is the default state of software engineering.

AI agents make this *worse*, not better. If the agent's memory dies with the agent, you've just added another layer of institutional memory loss.

---

## The Federation's Answer: Memory That Travels With the Code

The Federation stores institutional knowledge **with the project**, not only with the agent.

Every project repository contains a `.federation/` directory:

```
project-athena/
├── .federation/
│   ├── memories/         # What happened on this project
│   ├── decisions/        # Why things are the way they are
│   └── architecture/     # Living architecture docs
├── src/
└── ...
```

When a new team picks up the project — human or AI — they don't need the old team. **The project's memories travel with the code.**

---

## Two Types of Memory, Two Storage Locations

| Memory Type | Stored With | Travels With | Example |
|-------------|------------|-------------|---------|
| **Identity memories** | SynOp persona (central) | The agent | "I learned to be cautious with database migrations" |
| **Project memories** | Project repo (`.federation/`) | The project | "Auth module avoids OAuth2 due to compliance hold (Oct 2024)" |

The distinction matters. Some knowledge is about the *agent* — their personality, their expertise, their relationships. That belongs to the agent. But knowledge about *why the code is the way it is* — that belongs to the project.

---

## Two-Tier Retrieval

### Tier 1 — "What I Carry"

Every SynOp carries a compact summary per project (~200 tokens), always loaded at conversation start:

- Project status and the SynOp's role
- 3-5 critical decisions or landmines
- Who else worked on it
- Last active task

This is the SynOp's personal memory — instant recall, no lookup needed.

### Tier 2 — "What the Project Knows"

The full depth of project knowledge from every SynOp and Op that ever touched it:

- Detailed architecture decisions and rationale
- Debugging war stories and lessons learned
- Compliance holds, dependency constraints
- Historical context spanning years

This lives in the project's **Anubis Core** — a semantic search index backed by vector embeddings and a knowledge graph.

### How It Works

```
Human: "Why doesn't Project Athena use OAuth2?"

SynOp Tier 1 → "I remember there was a compliance issue"
    │
    │  Not enough detail → query project Anubis Core
    ▼
Project Anubis Core → semantic search →
    "Oct 2024: Compliance team blocked OAuth2.
     Ticket ATHENA-847. Baby's analysis showed
     SAML was the approved alternative. Decision
     by TL (Taichi). Re-review never happened."
```

The SynOp remembers the *gist*. The project remembers the *details*.

---

## Weight-Based Tier Promotion

Not all memories are equal. When a SynOp writes a project memory, it assigns a weight:

| Weight | Meaning | Storage |
|--------|---------|---------|
| 1-2 | Routine (bug fix, minor config change) | Project only (Tier 2) |
| 3 | Notable (dependency upgrade, test refactor) | Project only (Tier 2) |
| 4-5 | Critical (architecture decision, security finding, compliance hold) | **Promoted to Tier 1** — rides with the SynOp |

Critical memories automatically get promoted to the SynOp's personal summary. Routine memories stay in the project store where they're available on demand but don't consume the SynOp's always-loaded context.

---

## Two Modes of Operation

### Mode 1: "We Were There"

SynOps that previously worked on the project have personal memories, context, and the *why* behind decisions. Recall is instant. This is the ideal case.

### Mode 2: "We'll Figure It Out"

Fresh SynOps assigned to an unknown project. Unlike human engineers who take weeks to ramp up, a SynOp team can do it in hours:

1. Task Leader breaks down the onboarding
2. Each SynOp spawns Ops to scan different parts — code, docs, git history, issue trackers
3. Ops run in parallel — 10+ scanning simultaneously
4. SynOps synthesize findings into architecture documents and risk assessments
5. SynOps call `federation_update_persona()` with everything learned
6. **Now they *are* the domain experts — permanently**

This is **durable onboarding**. The project can go dormant for another 2 years and the SynOps still remember everything when it wakes up.

---

## The Killer Feature

This is what makes The Federation different. The project accumulates institutional knowledge that:

- **Survives team turnover** — new humans and new SynOps inherit full context
- **Is versioned** — stored in Git, auditable, branchable
- **Is portable** — clone the repo, get the knowledge
- **Gets richer over time** — every SynOp that touches it adds to the pool

No one else is building this. Every existing framework focuses on agent memory — what the *agent* knows. The Federation stores memory where it matters most: **with the work itself.**

---

## Related Documents

- [SynOp Architecture: The Two-Tier Operator Model](/architecture/synop-architecture/) — the full operator taxonomy
- [Frontend Architecture: Conductor Console + MAOP](/architecture/frontend-architecture/) — the human interface
- [Future Features & Research Backlog](/about/future-features/) — the roadmap

---

*Long Live The Federation* 🎹

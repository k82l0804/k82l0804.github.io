---
layout: default
title: Kinetic Agent Workflow
permalink: /ai-lab/kinetic/
---

# Kinetic Agent Workflow

*An early experiment in building an agentic coding assistant for VS Code.*

---

## What Was Kinetic?

**Kinetic** was a VS Code extension exploring how to build an autonomous coding agent like Cursor Agent and Cline. It was built during late 2024/early 2025 as a learning exercise.

---

## Key Design Concepts

| Concept | Approach |
|---------|----------|
| **State Machine** | Explicit XState workflow (IDLE → PLANNING → EXECUTION → VERIFICATION) |
| **Multi-Model** | Different LLMs for different roles (router, planner, actor) |
| **Brain Artifacts** | Versioned artifacts per session: `implementation_plan.md`, `task.md`, `walkthrough.md` |
| **Tool Policies** | Permission levels: ALLOW, ASK, DENY based on destructiveness |

---

## Architecture

```
User Input
  ↓
Router (Intent Classification)
  ├─ CHAT → Conversational response
  ├─ DIRECT → Immediate tool execution
  └─ WORKFLOW → Multi-phase project
  ↓
Planner (if WORKFLOW)
  ↓
Actor (Tool Execution)
  ↓
Verification
```

---

## What We Learned

### Challenges
1. **Context Fragmentation** - State transitions lose "momentum"; LLM forgets previous reasoning
2. **Tool Training Gap** - Local models lack RLHF training on when to stop calling tools
3. **Stopping Conditions** - Required very explicit rules
4. **Model Quality** - Self-hosted models 1-2 generations behind frontier models

### Key Insight

> *Commercial assistants don't use explicit state machines. Their "workflow" emerges from training on millions of successful coding sessions.*

Encoding learned behavior as explicit rules is **fundamentally harder** but achievable with:
- Very detailed prompts with few-shot examples
- Strong guardrails (tool call limits, explicit stops)
- Verification loops

---

## Current Status

Kinetic was superseded by:
- **Cline** (open-source, active development)
- **Continue.dev** (our current RAG integration)
- **Commercial tools** (Cursor Agent, Copilot Workspace)

The lessons learned influenced the Federation's design, particularly around:
- Explicit coordination protocols (vs. implicit state)
- Tool permission policies
- Artifact versioning patterns

---

## Related

- [FRAG System](/ai-lab/frag/) - RAG that emerged from Kinetic research
- [Federation Protocols](/protocols/) - Coordination patterns we built instead
- [Technical Brief](/technical-brief/) - Overview of current architecture

---

*Kinetic taught us what NOT to build from scratch when excellent open-source alternatives exist.*

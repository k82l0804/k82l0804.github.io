---
layout: default
title: "OpenClaw Gateway: External Op Runtime"
permalink: /architecture/openclaw-gateway/
---

> *"OpenClaw is the engine block. Custom skills are the transmission. The Federation owns the wheels."*

---

## 1. Concept

[OpenClaw](https://openclaw.ai) is an open-source, self-hosted AI agent gateway (150K+ GitHub stars, MIT license). It operates as a hub-and-spoke system: a central Gateway (Node.js) routes tasks to LLM backends, executes "skills" (file ops, shell commands, browser automation, API calls), and returns structured results.

**Thesis:** OpenClaw can serve as an **external Op runtime** — a pre-built execution layer that SynOps dispatch tasks to, extending the Federation's Op capabilities without building every tool from scratch.

**Key constraint:** OpenClaw is **not used internally** in the Federation. It's an external gateway that Ops run through. SynOps remain the Federation's persistent intelligence layer.

---

## 2. Mapping to SynOp/Op Taxonomy

| Federation Concept | OpenClaw Analog |
|---|---|
| **Op** (ephemeral task worker) | OpenClaw skill execution (stateless, task-scoped) |
| **SynOp → Op spawn** | SynOp sends structured task via MCP → Gateway executes |
| **Op tool access** | OpenClaw skills (file, shell, browser, API) |
| **Op result reporting** | Gateway returns structured result to SynOp |
| **Op permission model** | Gateway auth + skill allowlists + Docker sandbox |
| **Op determinism** | Pinned skill registry (versioned, audited) |

---

## 3. Integration Architecture

```
┌─────────────────────────────────────────────────┐
│               FEDERATION CORE                    │
│                                                  │
│  SynOp (Taichi / Qwen / Aorus / Baby)           │
│    │                                             │
│    │ "Execute task X with tools Y, Z"            │
│    │                                             │
│    ├── Internal Op Runtime (LAM + tools)         │
│    │     Direct tool calling for code/git ops    │
│    │                                             │
│    └── OpenClaw Gateway Adapter                  │
│         │                                        │
└─────────┼────────────────────────────────────────┘
          │  MCP / REST (LAN only)
          ▼
┌─────────────────────────────────────────────────┐
│         OPENCLAW GATEWAY (self-hosted)           │
│         Cluster node, port 18789                 │
│                                                  │
│  ┌────────────┐ ┌────────────┐ ┌──────────────┐ │
│  │ Built-in   │ │ Curated    │ │ Custom       │ │
│  │ Skills     │ │ ClawHub    │ │ Federation   │ │
│  │ (file,     │ │ Skills     │ │ Skills       │ │
│  │  shell,    │ │ (pinned,   │ │ (coding op,  │ │
│  │  browser)  │ │  audited)  │ │  test op,    │ │
│  │            │ │            │ │  scan op)    │ │
│  └────────────┘ └────────────┘ └──────────────┘ │
│                                                  │
│  LLM Backend → local vLLM (cluster LAN)          │
│  Execution → Docker sandbox                      │
│  Results → structured payload back to SynOp      │
└──────────────────────────────────────────────────┘
```

### SynOp Routing Decision

The SynOp decides which runtime based on task type:

| Task Type | Runtime | Why |
|---|---|---|
| Code generation / refactoring | Internal Op (LAM) | Deep file/git integration, multi-file context |
| Email / calendar / API integration | OpenClaw Gateway | Pre-built skills, no custom tooling needed |
| Browser automation | OpenClaw Gateway | Existing skill ecosystem |
| Shell script execution | Either | Built-in on both sides |
| Security scanning | OpenClaw Gateway | Community skills + custom wrappers |

---

## 4. Local-Only Deployment (Air-Gapped)

OpenClaw's "local first" philosophy means **zero runtime internet dependency**:

| Component | Cloud Dependency? | Local Configuration |
|---|---|---|
| Gateway (core) | ❌ None | Node.js process on cluster node |
| LLM backend | Default: cloud APIs | Point at local vLLM endpoint |
| Skills (built-in) | ❌ None | File, shell, browser — all local |
| ClawHub skills | Download requires internet | Clone once, pin to local. No runtime dependency |
| Messaging connectors | WhatsApp/Telegram = internet | **Disabled.** Federation MCP is the interface |
| Telemetry | Optional phone-home | Disabled |
| npm dependencies | Install requires internet | Install once, copy `node_modules` or local registry |

**SCIF compatibility:** Same air-gap profile as the rest of the Federation stack.

---

## 5. Custom Federation Skills

Skills are lightweight — two files each:

```
skills/
├── federation-code-op/
│   ├── SKILL.md              # Natural language description for the LLM
│   └── implementation.js     # Tool logic
├── federation-test-op/
│   ├── SKILL.md
│   └── implementation.js
└── federation-scan-op/
    ├── SKILL.md
    └── implementation.js
```

### Planned Op Skills

| Skill | Purpose | Complexity |
|---|---|---|
| `federation-code-op` | Wrap local LAM for code generation tasks dispatched via Gateway | ~1-2 days |
| `federation-test-op` | Run test suites, return structured pass/fail results | ~1 day |
| `federation-scan-op` | Static analysis, dependency vulnerability checks | ~1 day |
| `federation-refactor-op` | Diff-based code transformations | ~1-2 days |

---

## 6. Security Considerations

OpenClaw has been labeled "insecure by default" in public deployments. Our mitigations:

| Risk | Mitigation |
|---|---|
| Exposed Gateway port | LAN-only binding, no external exposure |
| Malicious ClawHub skills | Pinned skill registry — no live ClawHub pulls. Every skill audited before allowlisting |
| Unrestricted tool access | Docker sandbox + skill allowlists per sovereignty level |
| Telemetry exfiltration | Disabled at config level, verified by network isolation |
| Runaway execution | Sovereignty Toggle wraps Gateway dispatch — Emergency Stop revokes all active sessions |

### Sovereignty Integration

```
SynOp checks sovereignty level
  │
  ├── STRICT: Request Conductor approval before Gateway dispatch
  ├── SUPERVISED: Dispatch freely, block on exceptions
  ├── AUTONOMOUS: Dispatch and report on completion
  └── EMERGENCY_STOP: Revoke all active Gateway sessions
```

---

## 7. What OpenClaw Brings vs. What We Build

| Category | OpenClaw Provides | We Build |
|---|---|---|
| **Runtime** | Gateway process, session management, Docker sandbox | — |
| **Tool calling** | Skill routing, LLM integration, result collection | — |
| **Built-in tools** | File, shell, browser automation | — |
| **Community skills** | Curated subset from ClawHub | Curation policy |
| **Federation skills** | — | Custom Op skills (~2 files each) |
| **MCP bridge** | MCP server support | Gateway Adapter for SynOp dispatch |
| **Security** | Docker sandbox | Hardening config, network isolation, sovereignty wrapper |
| **Persona/identity** | `SOUL.md` per agent | Bypass — SynOp layer handles all persona |

---

## 8. Open Questions

| # | Question | Impact |
|---|---|---|
| 1 | **Which node hosts the Gateway?** | Orchestrator node (Docker) vs. compute node (GPU for local LLM calls). |
| 2 | **Gateway-per-SynOp or shared Gateway?** | Shared is simpler. Per-SynOp provides better isolation but more resource overhead. |
| 3 | **Skill versioning strategy** | Git submodule vs. vendored copy vs. npm package. Git submodule preserves audit trail. |
| 4 | **Phase 0 proof of concept** | Clone → deploy on cluster → point at local vLLM → test one built-in skill via MCP. ~2 hours. |

---

## Related Documents

- [SynOp Architecture: The Two-Tier Operator Model](/architecture/synop-architecture/)
- [Frontend Architecture: Conductor Console + MAOP](/architecture/frontend-architecture/)
- [Distributed Project Memory](/about/distributed-project-memory/)

---

*Long Live The Federation* 🎹

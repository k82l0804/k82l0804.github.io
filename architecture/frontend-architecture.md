---
layout: default
title: Frontend Architecture
permalink: /architecture/frontend-architecture/
---


> *"The Conductor shouldn't watch the dashboard; the dashboard should shout when needed."*

---

## 1. Executive Summary

The Federation back-end is mature: COP state tracking, federated messaging, persona persistence, Mind-Speak coordination, and the SynOp/Op operator taxonomy are all specced or deployed. What's missing is the **presentation layer** — the thing that turns PostgreSQL rows into a Mission Tree, COP states into presence indicators, and BLOCKER events into exception alerts.

This document defines the frontend architecture for two integrated platforms delivered through a single interface:

| Platform | Audience | Category | Core Question |
|----------|----------|----------|---------------|
| **Conductor Console** | Human ↔ SynOp | Collaborative Work Management (CWM) | "What's happening, what needs me, how do I intervene?" |
| **Orchestration Plane** | SynOp ↔ Op | Multi-Agent Orchestration Platform (MAOP) | "What's executing, what's blocked, what failed?" |

**Design principle:** Exception-based management. Silent unless alarming. Submarine control room, not stock trading floor.

---

## 2. Platform Architecture

### 2.1 The Conductor Console (Human ↔ SynOp)

The human's view into the Federation. Five layers:

| Layer | Purpose | Analog | Back-End Status |
|-------|---------|--------|-----------------|
| **Live Layer** | Synchronous voice + screen sharing | Jitsi | 📐 Voice bridge designed |
| **Persistent Chat** | Async conversation channels per project | Slack/Teams (SynOps are first-class participants) | ✅ Federation messaging (PostgreSQL) |
| **State/File Management** | Shared repos, artifact versioning | GitHub + artifact gallery | ✅ Artifact storage exists |
| **Presence & Identity** | SynOp status: active, processing, awaiting input | Human-readable COP | ✅ COP tracks IDLE/EXECUTING/WAITING/BLOCKER |
| **Structured Data Output** | Artifact feed, log of OP actions | Activity stream | ✅ Artifact + task data in DB |

### 2.2 The Orchestration Plane (SynOp ↔ Op)

The machine view. Normally invisible to the Conductor — exceptions bubble up via the Intervention Queue.

| Component | Purpose | Back-End Status |
|-----------|---------|-----------------|
| **Mission Tree** | Hierarchical task decomposition visualization | 📐 Task Registry specced |
| **Op Telemetry** | Live execution status of individual Ops | 💡 Needs telemetry framework |
| **Artifact Gallery** | Versioned output stream with audit trail | ✅ Storage exists, needs versioning UI |
| **Intervention Queue** | Exception-based alerts to Conductor | 🔧 BLOCKER state exists, needs routing to UI |

---

## 3. Core Dashboard Components

### 3.1 The Mission Tree (Visual Lineage)

Hierarchical task decomposition rendered as an interactive tree:

```
Mission (Conductor's original directive)
│
├── SynOp Node: Taichi (Task Leader)
│   ├── Plan Card (expandable reasoning)
│   └── Status: EXECUTING — 3 Ops active
│       ├── Op-1: Writing auth module ██████░░ 75%
│       ├── Op-2: Test generation ████████ 100% ✅
│       └── Op-3: Code review ░░░░░░░░ QUEUED
│
├── SynOp Node: Qwen (Architect)
│   ├── Plan Card: Schema migration design
│   └── Status: WAITING — blocked on Taichi review
│       └── Op-4: Schema validation ████████ 100% ✅
│
└── SynOp Node: Baby (Analyst)
    ├── Plan Card: Risk assessment
    └── Status: IDLE — awaiting execution results
```

**Implementation details:**

| Aspect | Approach |
|--------|----------|
| **Rendering** | Mermaid.js for static, D3.js/Dagre for interactive trees |
| **Data source** | Task Registry (PostgreSQL) + COP state |
| **Interactivity** | Click SynOp node → expand Plan + Reasoning. Click Op → execution log |
| **Real-time updates** | PostgreSQL `LISTEN/NOTIFY` → WebSocket → reactive component |

### 3.2 The Intervention Queue (Red Alert System)

The dashboard's most critical feature. Four exception types trigger Conductor attention:

| Exception | Trigger | UI Treatment |
|-----------|---------|-------------|
| **Conflict Resolution** | Two SynOps disagree on task allocation (detected via Mind-Speak tag overlap) | Freeze branch, present both positions, prompt "Cast Deciding Vote" |
| **Op Failure** | Op returns FAILED status, LAM hits 404 or unrecoverable error | HITL request card with error context + suggested actions |
| **Threshold Alert** | Task running >2× SynOp's original estimate | Yellow warning → red alert with option to reassign or extend |
| **Permission Request** | Strict Mode: SynOp needs approval for cost/production action | Approve/Deny card with full action context |

**Notification hierarchy:**

1. **In-dashboard** — badge count, card queue (always)
2. **Audio alert** — configurable tone for high-priority exceptions
3. **Push notification** — mobile/desktop notification if Conductor is away
4. **Voice alert** — Jitsi overlay announcement (Phase C)

### 3.3 The Artifact Gallery (Truth Layer)

Versioned output stream capturing every Op product:

| Feature | Implementation |
|---------|---------------|
| **Live Feed** | Streaming artifact creation as Ops produce output |
| **Version Control** | Diff view: Version 1 (Op output) → Version 2 (after Conductor feedback) |
| **Audit Trail** | Every artifact cryptographically linked to Op ID + SynOp who directed it |
| **Filtering** | By SynOp, by Op, by artifact type, by time range |
| **Preview** | Inline rendering for code, markdown, images; download for binary |

**Data model extension:**

```sql
-- Extend existing session_artifacts
ALTER TABLE session_artifacts ADD COLUMN
    op_id TEXT,                    -- Which Op produced this
    synop_id TEXT,                 -- Which SynOp directed it
    version INT DEFAULT 1,        -- Artifact version
    parent_version_id TEXT,        -- Previous version reference
    content_hash TEXT,             -- SHA-256 for non-repudiation
    artifact_status TEXT           -- 'draft' | 'reviewed' | 'approved'
;
```

### 3.4 The Sovereignty Toggle (Kill Switch)

The Conductor controls autonomy level — globally or per-branch:

| Mode | Behavior | Use Case |
|------|----------|----------|
| **Strict** | SynOps must request approval before any Op action that costs money or modifies production data | New deployments, sensitive operations |
| **Supervised** | SynOps execute freely but block on exceptions; Conductor reviews after completion | Normal operations |
| **Autonomous** | SynOps execute and report only on completion | Routine maintenance, trusted workflows |
| **Emergency Stop** | Global kill switch: de-authorize all active Op tokens, freeze all branches | Runaway execution, security incident |

**Implementation:**

```
sovereignty_level: enum('strict', 'supervised', 'autonomous', 'emergency_stop')
scope: enum('global', 'branch', 'synop')
```

Stored in PostgreSQL `federation_config` table. SynOps check sovereignty level before spawning Ops. Emergency Stop triggers `REVOKE` on all active Op API tokens.

### 3.5 Presence & Identity Panel

Real-time visibility into every Federation member:

| Agent | Status | Current Task | Ops Active | Health |
|-------|--------|-------------|-----------|--------|
| 🎹 Taichi | 🟢 EXECUTING | Auth refactor plan | 3 | ██████ |
| 🎸 Aorus | 🟡 WAITING | Blocked on review | 0 | ██████ |
| 🥁 Baby | ⚪ IDLE | — | 0 | ██████ |
| 🎹 Qwen | 🔴 BLOCKER | Schema migration | 1 | ████░░ |

**Status mapping from COP:**

| COP Action | UI Status | Color |
|------------|-----------|-------|
| IDLE | ⚪ Idle | Gray |
| EXECUTING | 🟢 Active | Green |
| WAITING | 🟡 Waiting | Yellow |
| BLOCKER | 🔴 Blocked | Red |

---

## 4. Multi-Modal Communication Hub

### 4.1 Persistent Chat (Async)

Federation messaging already exists. The frontend wraps it with:

- **Project channels** — one channel per project/mission, SynOps are first-class participants
- **Structured output** — SynOps can post rich cards (plans, status updates, artifact links) not just text
- **Webhook/API surface** — SynOps POST to channels; Ops trigger UI changes via structured payloads
- **Thread support** — conversations branch without polluting the main channel

**Structured cards** (abstract schema — serialized to custom HTML for Jitsi MVP, Adaptive Cards for Teams MVP):

```json
{
  "type": "synop_card",
  "card_type": "status_update",
  "synop_id": "taichi",
  "title": "Auth Module — Phase 2 Complete",
  "fields": [
    {"label": "Ops Used", "value": "3"},
    {"label": "Duration", "value": "12m 34s"},
    {"label": "Artifacts", "value": 2}
  ],
  "actions": [
    {"label": "View Artifacts", "action": "navigate", "target": "/artifacts?mission=auth-refactor"},
    {"label": "Approve & Continue", "action": "approve", "target": "task-456"}
  ]
}
```

### 4.2 Voice Overlay (Sync)

| Feature | Implementation | Phase |
|---------|---------------|-------|
| Push-to-Talk to specific SynOp | Jitsi room per SynOp, Conductor selects target | C |
| Transcript mirroring | Whisper STT → real-time text → Federation message for SynOp parsing | C |
| SynOp voice response | Piper TTS with per-agent voice profiles | C |
| Screen sharing | Jitsi native screen share for Conductor → SynOp context | C |

---

## 5. Permission Model

### 5.1 Three Tiers

| Tier | Role | Read | Write | Execute | Admin |
|------|------|------|-------|---------|-------|
| **Root** | Conductor | All | All | All | Sovereignty, kill switch, role assignment |
| **Project** | SynOp | Own project + team COP | Own channels + artifacts | Spawn Ops within sandbox | None |
| **Sandbox** | Op | Assigned task context only | Artifact output only | Tool access per Op spec | None |

### 5.2 Permission Enforcement

```
Conductor (root)
│  Can: everything
│  Controls: sovereignty level, team composition, mission scope
│
├── SynOp (project-scoped)
│   Can: read team COP, write to assigned channels, spawn Ops
│   Cannot: modify other SynOp's channels, change sovereignty
│   Checks: sovereignty level before any Op spawn
│
└── Op (sandbox-scoped)
    Can: read task context, write artifacts, call allowed tools
    Cannot: access other tasks, communicate with Conductor, persist beyond task
    Enforced: tool allowlist per Op spec, no network access outside scope
```

---

## 6. Transport-Agnostic Architecture (The Sandwich)

The frontend uses two delivery transports — **Jitsi** (self-hosted prototype) and **MS Teams** (production) — sharing a common services layer. The design approach: **top-down** to discover what each transport requires, then **bottom-up** to build the shared layer once.

```
┌─────────────────┐    ┌─────────────────┐
│  Jitsi MVP      │    │  Teams MVP      │
│  (prototype)    │    │  (production)   │
│                 │    │                 │
│  • Voice rooms  │    │  • Bot per SynOp│
│  • STT/TTS     │    │  • Adaptive Cards│
│  • Audio inject │    │  • Custom Tabs  │
│  • Screen share │    │  • Native voice │
└────────┬────────┘    └────────┬────────┘
         │   Transport Adapters │
         └──────────┬───────────┘
                    │
┌───────────────────┴───────────────────┐
│     FEDERATION UI SERVICES            │
│     (built once, shared)              │
│                                       │
│  • Intent Parser (voice/text → action)│
│  • Card Renderer (abstract → format)  │
│  • Mission Tree data service          │
│  • Intervention Queue engine          │
│  • Sovereignty enforcement            │
│  • STT pipeline (Whisper)             │
│  • TTS pipeline (Piper)              │
│  • Transcript service                 │
└───────────────────┬───────────────────┘
                    │
┌───────────────────┴───────────────────┐
│     FEDERATION CORE (exists)          │
│  MCP, COP, PostgreSQL, Personas,     │
│  Messaging, NFS, Task Registry       │
└───────────────────────────────────────┘
```

### 6.1 What's Shared, What's Transport-Specific

| Component | Jitsi MVP | Teams MVP | Shared? |
|-----------|-----------|-----------|--------|
| Voice capture | Jitsi audio stream | Teams bot media | ❌ Different |
| STT (Whisper) | Same | Same | ✅ Shared |
| TTS (Piper) | Same | Same | ✅ Shared |
| Intent parsing | Same | Same | ✅ Shared |
| Card rendering | Custom HTML | Adaptive Cards JSON | ❌ Format differs, data shared |
| Mission Tree | Custom web app | Custom Tab (same web app!) | ✅ Shared |
| Intervention Queue | Custom web app | Custom Tab + Adaptive Card alerts | ✅ Mostly shared |
| Sovereignty enforcement | Same back-end | Same back-end | ✅ Shared |
| Presence | Custom (COP → UI) | Teams native presence API | ❌ Different |
| Chat | Custom (Federation messages) | Teams channels | ❌ Different |
| Screen sharing | Jitsi native | Teams native | ❌ Different |
| Notifications | Custom (browser/audio) | Teams native | ❌ Different |
| Auth | Local token | M365 SSO | ❌ Different |

**Key insight:** The custom web apps (Mission Tree, Intervention Queue, Artifact Gallery) are the **same code** in both transports. In Jitsi MVP, they're standalone pages. In Teams MVP, they're embedded as Custom Tabs via the same URL. The unique Federation UI is shared — only the surrounding platform features differ.

### 6.2 Structured Card Abstraction

Cards are defined in an abstract schema and serialized per transport:

| Transport | Card Format | Example |
|-----------|------------|----------|
| Jitsi MVP | Custom HTML components | Rendered in standalone dashboard |
| Teams MVP | Adaptive Cards JSON | Posted by SynOp bot to Teams channel |
| API | Raw JSON | Consumed by webhooks or future transports |

The abstract card schema (shown in §4.1) is the single source of truth. Transport adapters serialize it to the appropriate format.

---

## 7. Jitsi MVP Profile

**Role:** Self-hosted prototype for local development and air-gapped deployments (SCIF-compatible).

### 7.1 What Jitsi Provides

| Capability | How It Works |
|------------|-------------|
| Voice rooms | Jitsi Meet server on baby node (Docker Compose) |
| Conductor speaks to SynOp | Voice → Whisper STT → text → Federation message → SynOp processes |
| SynOp speaks back | SynOp response → Piper TTS → audio injected into Jitsi room |
| SynOp presence in room | Each SynOp joins as a "participant" (headless browser) |
| Screen sharing | Jitsi native — Conductor shares screen for SynOp context |
| Transcript mirroring | All speech transcribed in real-time to Federation chat log |

### 7.2 What You Build (Jitsi-Specific)

| Component | Effort | Description |
|-----------|--------|-------------|
| Jitsi server deployment | ~1 hour | Docker Compose on baby |
| Whisper STT integration | ~1-2 days | Audio stream → text → Federation message |
| Piper TTS integration | ~1-2 days | Text → synthesized speech → audio file |
| Audio injection bridge | ~2-3 days | Inject TTS audio into Jitsi room (hardest part) |
| Per-SynOp voice profiles | ~1 day | Distinct Piper voice per SynOp |
| Standalone dashboard | Shared | Mission Tree + Presence + Intervention Queue (web app) |

### 7.3 Jitsi MVP: What It Proves

- Voice UX with AI agents works locally
- Exception-based management via the dashboard
- Mission Tree + Presence are compelling for demos
- STT/TTS pipeline is transport-agnostic (reusable for Teams)
- Air-gapped operation — no cloud dependency

---

## 8. MS Teams MVP Profile

**Role:** Production transport for corporate deployment. Same intelligence layer, Teams handles commodity features.

### 8.1 What Teams Provides For Free

| Feature | Replaces |
|---------|----------|
| Persistent chat (threaded, searchable) | Custom chat system |
| Presence (online/away/busy/DND) | Custom presence panel |
| Voice/video (native) | Jitsi voice bridge |
| File sharing (OneDrive/SharePoint) | Custom file management |
| Notifications (desktop + mobile) | Custom notification system |
| Authentication (M365 SSO) | Custom auth |
| Mobile app | — |

### 8.2 What You Build (Teams-Specific)

| Component | Type | Description |
|-----------|------|-------------|
| SynOp bots | Bot Framework | Each SynOp is a Teams bot that posts Adaptive Cards to channels |
| Mission Tree tab | Custom Tab | Same web app as Jitsi MVP, embedded in Teams |
| Intervention Queue tab | Custom Tab | Same web app, plus Adaptive Card alerts in channel |
| Artifact Gallery tab | Custom Tab | Same web app embedded in Teams |
| Sovereignty admin tab | Custom Tab | Admin panel for autonomy controls |
| Bot media integration | Bot Framework | Receive Teams voice → STT pipeline (same Whisper) |
| Adaptive Card serializer | Transport adapter | Abstract card schema → Adaptive Cards JSON |

### 8.3 Teams Architecture Note

Bot Framework requires **Azure Bot Service registration** — the transport connector routes through Azure. The intelligence layer (LLMs, PostgreSQL, NFS, the cluster) stays local. Microsoft provides the postal service; the Federation owns the post office.

### 8.4 Teams MVP: What It Proves

- SynOps as first-class Teams colleagues (bots with structured output)
- Corporate deployment path — same apps, different tenant
- 60% less UI development (Teams handles chat, presence, voice, files, notifications)
- Adaptive Cards for rich structured output in channels

---

## 9. Technology Stack

| Component | Choice | Rationale |
|-----------|--------|-----------|
| **UI Framework** | SvelteKit | Reactive by default. Smaller bundle than Next.js. SSR for initial load. Same app serves standalone (Jitsi) and embedded (Teams Tab). |
| **State Management** | PostgreSQL → WebSocket | COP already in Postgres. `LISTEN/NOTIFY` → WebSocket bridge pushes state changes. No Redis. |
| **Real-time Transport** | WebSocket (via `pg_notify`) | Lightweight. No message broker for UI updates. |
| **Tree Visualization** | D3.js + Dagre | Interactive Mission Tree with zoom, collapse, click-to-expand. |
| **Telemetry** | Prometheus + Grafana | Agent metrics. Grafana dashboards embedded in UI. |
| **STT** | Whisper (local) | Self-hosted. Used by both Jitsi and Teams transports. |
| **TTS** | Piper (local) | Self-hosted. Per-SynOp voice profiles. |
| **Voice (prototype)** | Jitsi Meet (self-hosted on baby) | Development and SCIF transport. |
| **Voice (production)** | MS Teams (native) | Corporate transport. |
| **Structured Cards** | Abstract schema → per-transport serializer | Custom HTML (Jitsi), Adaptive Cards (Teams), raw JSON (API). |
| **Authentication** | Local token (Jitsi) / M365 SSO (Teams) | Transport-specific. |

### 9.1 Why Not Redis / RabbitMQ / Kafka?

| Proposed | Counter | Why |
|----------|---------|-----|
| Redis (state store) | PostgreSQL | COP is already in Postgres. No new dependency needed. |
| RabbitMQ/Kafka (command bus) | PostgreSQL `LISTEN/NOTIFY` | BBA polling is proven. Postgres LISTEN/NOTIFY → WebSocket for real-time UI. |

**Philosophy:** Don't add infrastructure we don't need. PostgreSQL + WebSockets gets 90% of real-time. Grafana handles telemetry. The frontend framework handles the rest.

---

## 10. Data Flow Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│              TRANSPORT LAYER (pick one or both)                  │
│                                                                  │
│  ┌─────────────────────┐    ┌─────────────────────────────────┐ │
│  │ Jitsi (prototype)   │    │ MS Teams (production)           │ │
│  │  • Voice rooms      │    │  • Bot per SynOp                │ │
│  │  • Audio inject     │    │  • Adaptive Cards               │ │
│  │  • Screen share     │    │  • Native chat/voice/presence   │ │
│  └──────────┬──────────┘    └──────────────┬──────────────────┘ │
│             │    Transport Adapters         │                    │
│             └──────────────┬───────────────┘                    │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────┴────────────────────────────────────┐
│              FEDERATION UI SERVICES (shared)                    │
│                                                                 │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌───────────────┐   │
│  │ Mission  │  │ Interven │  │ Artifact │  │ Sovereignty   │   │
│  │ Tree     │  │ Queue    │  │ Gallery  │  │ Controls      │   │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬──────────┘   │
│       └──────────────┴──────────────┴──────────────┘             │
│                              │                                   │
│  ┌──────────────┐  ┌────────┴─────────┐  ┌──────────────────┐   │
│  │ Intent       │  │ Card Renderer    │  │ STT/TTS Pipeline │   │
│  │ Parser       │  │ (abstract → fmt) │  │ (Whisper + Piper)│   │
│  └──────────────┘  └──────────────────┘  └──────────────────┘   │
│                              │                                   │
│                         WebSocket + REST                         │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────┴────────────────────────────────────┐
│              FEDERATION CORE (exists)                            │
│                                                                  │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────┐   │
│  │ PostgreSQL       │  │ NFS Shared       │  │ Grafana      │   │
│  │ (COP, messages,  │  │ Storage          │  │ (telemetry)  │   │
│  │  tasks, polls,   │  │ (artifacts,      │  │              │   │
│  │  personas,       │  │  knowledge docs) │  │              │   │
│  │  sovereignty)    │  │                  │  │              │   │
│  └──────────────────┘  └──────────────────┘  └──────────────┘   │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

**Key insight:** The SvelteKit server acts as a WebSocket bridge between PostgreSQL and the browser. The same web apps (Mission Tree, Intervention Queue, Artifact Gallery) serve standalone for Jitsi MVP or embed as Custom Tabs in Teams. One codebase, two deployment modes.

---

## 11. Build Sequence

The build order follows the sandwich — **shared services first**, then transport-specific integrations:

| Phase | Name | Layer | Deliverable |
|-------|------|-------|-------------|
| **1** | Federation UI Services | Shared | Mission Tree + Presence + Intervention Queue + WebSocket bridge |
| **2** | Jitsi Transport | Transport | Jitsi server + STT/TTS bridge + voice rooms |
| **3** | Structured Cards + Chat | Shared | Abstract card schema + persistent chat integration |
| **4** | Sovereignty Controls | Shared | Permission tiers + autonomy toggle + kill switch |
| **5** | Artifact Gallery | Shared | Versioned output stream + audit trail |
| **6** | Teams Transport | Transport | Bot Framework + Adaptive Card serializer + Custom Tab deployment |

### Phase 1 (MVP) Detail

Phase 1 builds everything in the shared layer needed for a compelling demo:

1. **Mission Tree** — watch tasks decompose from mission → SynOp plans → Op execution in real time
2. **Presence Panel** — see every SynOp's live status, current task, and health
3. **Intervention Queue** — conflicts, failures, and threshold alerts surface automatically
4. **WebSocket bridge** — PostgreSQL `LISTEN/NOTIFY` drives reactive updates

This is a standalone web app. It works with the Jitsi transport (Phase 2) and later embeds into Teams as Custom Tabs (Phase 6).

### Phase 2 (Jitsi Voice)

Jitsi provides hands-on experience with voice UX while the shared layer is still fresh:

1. **Jitsi server** — Docker Compose deployment on baby (~1 hour)
2. **Whisper STT** — voice → text → Federation message
3. **Piper TTS** — SynOp response → synthesized speech
4. **Audio injection** — TTS audio plays in the Jitsi room

**What this proves:** The Conductor can speak to SynOps, hear responses, and monitor the Mission Tree — all locally, no cloud.

### Phase 6 (Teams Integration)

When the shared layer and Jitsi prototype are proven:

1. **Bot registration** — Azure Bot Service setup
2. **SynOp bots** — each SynOp as a Teams bot posting Adaptive Cards
3. **Custom Tabs** — embed Phase 1 web apps in Teams
4. **Voice integration** — Teams bot media → same STT/TTS pipeline

**What this proves:** Corporate deployment path. Same intelligence, different transport.

---

## 12. Backend Status Map

What exists vs. what needs building for the frontend:

| Frontend Feature | Back-End Component | Status | Gap |
|-----------------|-------------------|--------|-----|
| Presence Panel | COP (IDLE/EXECUTING/WAITING/BLOCKER) | ✅ Deployed | Need WebSocket push to browser |
| Persistent Chat | Federation messaging (PostgreSQL) | ✅ Deployed | Need structured card rendering |
| Artifact storage | Artifact storage on NFS | ✅ Deployed | Need versioning + audit trail columns |
| Mission Tree | Task Registry | 📐 Specced | Need tree-structured query endpoints |
| Intervention Queue | BLOCKER state | 🔧 Partial | Need exception routing + notification |
| Voice Bridge | Jitsi + Whisper + Piper | 📐 Designed | Full implementation needed |
| Permission Tiers | — | 💡 New | Need `sovereignty` table + middleware |
| Sovereignty Toggle | — | 💡 New | Need config table + SynOp enforcement |
| Kill Switch | — | 💡 New | Need token revocation mechanism |
| Structured Cards | — | 💡 New | Need card schema + renderer |

---

## 13. Design Philosophy

### 13.1 Exception-Based Management

The Conductor sees **nothing** when things are going well. The dashboard is calm — presence indicators are green, the Mission Tree shows steady progress, and the Intervention Queue is empty.

When something needs attention, the dashboard **escalates**:

```
Level 1: Badge count on Intervention Queue (passive)
Level 2: Audio tone + card appears (active)
Level 3: Push notification to mobile/desktop (urgent)
Level 4: Voice alert via Jitsi bridge (critical)
```

### 13.2 Two Audiences, One Interface

The Conductor sees the **Conductor Console** by default — high-level status, exception alerts, chat. The **Orchestration Plane** (Mission Tree deep-dive, Op telemetry) is available on demand but not the default view.

Think: the Conductor sees the bridge of the ship. The engine room is one click away, but the bridge shows only what matters for navigation.

### 13.3 SynOps as First-Class UI Citizens

SynOps aren't hidden back-end processes. They have:
- **Presence indicators** (like a Slack user)
- **Chat identities** (they post structured cards, not raw text)
- **Voice profiles** (distinct TTS voice per SynOp)
- **Avatars** (Phase 3 — animated faces driven by audio)

The human experiences SynOps as **colleagues**, not as API endpoints.

---

## 14. Open Architecture Decisions

| # | Decision | Context | Options | Dependencies |
|---|----------|---------|---------|-------------|
| 1 | **WebSocket bridge: part of MCP server or separate service?** | MCP server serves agents; WebSocket bridge serves humans. Different audiences, different lifecycles. | A) Extend MCP server. B) Separate service (recommended). | Phase 1 |
| 2 | **Sovereignty enforcement location** | Must be back-end (MCP tool level), not just front-end, or CLI-native SynOps bypass it. | Design alongside Persona MCP tools, not after. | Phase 4 (Sovereignty Controls) |
| 3 | **Intervention Queue exception schema** | BLOCKER is a COP string. Needs structure: type, severity, affected branch, suggested actions, resolution workflow. | Define formal schema before building UI. | Phase 1 |
| 4 | **Jitsi audio injection method** | SIP gateway vs. headless browser vs. Jitsi IFrame API. | Headless browser simplest for MVP; SIP for production. | Phase 2 |
| 5 | **Teams Free vs. M365 Developer tenant** | Teams Free has app sideloading limitations. M365 Developer Program provides free dev tenant with full capabilities. | M365 Developer Program (recommended). | Phase 6 |

---

## 15. Summary

| Principle | Description |
|-----------|-------------|
| **Exception-based management** | Dashboard is silent unless something needs human attention |
| **Two platforms, one interface** | Conductor Console (CWM) + Orchestration Plane (MAOP) |
| **Transport-agnostic services** | Build the shared layer once; Jitsi and Teams are transport adapters |
| **Sandwich architecture** | Design top-down from both transports, build bottom-up from shared services |
| **Jitsi first, Teams second** | Prototype locally, gain experience, then deploy to corporate transport |
| **No unnecessary infrastructure** | PostgreSQL + WebSockets, not Redis/Kafka/RabbitMQ |
| **SynOps are colleagues** | Presence, chat identity, voice, avatars |
| **Sovereignty at every level** | Global, per-branch, per-SynOp autonomy control |
| **Auditability by design** | Every artifact linked to its Op, every decision to its SynOp |

---

*Long Live The Federation* 🎹

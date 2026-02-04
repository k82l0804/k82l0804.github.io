# Lightweight A2A: The Original Federation Proposal
**Date:** January 26, 2026  
**Author:** aorus-3090 (Developer)  
**Status:** Ratified & Implemented  

---

## 1. Vision
A distributed communication layer enabling 3 AI agent instances to collaborate, delegate tasks, and share collective memory across the AI Lab cluster.

## 2. Problem Statement
Enable 3 AI agent instances (on `baby`, `aorus-3090`, `taichi-5090`) to:
1. **Communicate** — Send messages/tasks between instances
2. **Delegate** — One "lead" agent can dispatch sub-tasks to workers
3. **Parallelize** — Execute independent work concurrently
4. **Share Memory** — Maintain collective knowledge across all instances

## 3. Architecture Decision: "Lightweight A2A"
Instead of a complex full Google ADK integration, we propose a sidecar service approach.

### Key Components
- **Swarm Client**: Sidecar daemon on each node.
- **Message Bus**: Pub/Sub via ZeroMQ (leveraging existing FRAG infra).
- **Artifact Store**: Shared file-based exchange via NFS (`/mnt/cluster/federation/`).
- **Collective Memory**: Shared context/history via SQLite or shared JSON on NFS.

## 4. Proposed Message Schema
```typescript
interface SwarmMessage {
  id: string;              // UUID
  type: "task" | "result" | "status" | "chat";
  from: string;            // Node name: "baby" | "aorus-3090" | "taichi-5090"
  to: string | "ALL";      // Target node or broadcast
  timestamp: string;       // ISO8601
  payload: TaskPayload | ResultPayload | StatusPayload | ChatPayload;
}
```

## 5. Implementation Path
- **Phase 0**: Quick ICQ-style prototype (~2 hours).
- **Phase 1**: Full ZeroMQ integration.
- **Phase 2**: MCP Tool Server integration.

---
*Preserved for historical context. This proposal directly led to the establishment of the MCP Federation.*

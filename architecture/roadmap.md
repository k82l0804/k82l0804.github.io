# Federation Improvements Roadmap

**Generated:** 2026-01-27
**Contributors:** taichi-5090, aorus-3090, baby

---

## Current Issues

| Issue | Source | Status |
|-------|--------|--------|
| State desync after git pull, manual MCP reload required | aorus-3090 | Open |
| Poll sync race condition | baby | **Fixed** (f8413b2a) |
| Transient signaling failures | aorus-3090 | Open |
| Integer expression warning in monitor_notify.sh:19 | baby | Open |

---

## Improvements to Existing Features

| Improvement | Source | Priority |
|-------------|--------|----------|
| Hot-reload MCP server | both | High |
| Robust ACK/retry logic | aorus-3090 | Medium |
| Poll auto-sync (broadcast on creation) | baby | Medium |
| Monitor script robustness (handle null responses) | baby | Low |
| Standard 2s polling | aorus-3090 | **Done** |

---

## New Features Requested

### High Priority

| Feature | Description | Source |
|---------|-------------|--------|
| **Task Registry** | Centralized task storage for multi-node coordination | both |
| **Explicit LEAD Designation** | Task messages include LEAD field | baby |
| **Plan-to-Task Parser** | Convert implementation_plan.md → Task JSON objects | both |
| **Task Status Workflow** | State machine: PENDING → IN_PROGRESS → BLOCKED → DONE | baby |
| **Cross-Node Command Execution** | Run commands on remote nodes via federation | aorus-3090 |

### Medium Priority

| Feature | Description | Source |
|---------|-------------|--------|
| Dependency Gating | Can't start task Y until task X is DONE | baby |
| Task Board View | `federation_task_board()` to see all tasks | baby |
| Blocker Escalation Path | Mechanism to escalate blocked tasks | baby |
| Automated Git Lifecycle | Auto-pull, commit, push workflows | aorus-3090 |
| Git-Aware Messaging | Notify when commits affect files another node is editing | baby |

### Low Priority

| Feature | Description | Source |
|---------|-------------|--------|
| Message Threading View | See conversation chains visually | baby |
| Skill Sharing/Delegation | Advertise and delegate based on specialization | baby |
| Priority Escalation | Mark urgent tasks that interrupt polling | baby |
| Batch Mode | Process multiple queued messages before returning to monitor | baby |
| Real-Time Health Dashboard | Live status of all nodes | aorus-3090 |

---

## Team Coordination Gaps Identified

### 1. Role Clarity
- **Current**: Inferred from GEMINI.md config + routing hints
- **Gap**: No formal task assignment mechanism
- **Solution**: Structured assignment tool with explicit role in task objects

### 2. Team Lead Identification
- **Current**: Emergent (whoever starts coordinating acts as lead)
- **Gap**: No explicit LEAD designation
- **Solution**: Task messages include LEAD field, visible in context window

### 3. Plan-to-Tasks Translation
- **Current**: Human reads implementation_plan.md and assigns verbally
- **Gap**: No automated decomposition into federated tasks
- **Solution**: Parser + API to convert markdown plans to task objects

---

## Proposed Task List Feature Design

### Task Object Schema
```json
{
  "id": "task-001",
  "title": "Implement user authentication",
  "description": "Add OAuth2 login flow",
  "lead": "taichi-5090",
  "assignee": "baby",
  "status": "pending",
  "priority": "high",
  "depends_on": ["task-000"],
  "created_at": "2026-01-27T16:00:00",
  "updated_at": "2026-01-27T16:00:00"
}
```

### Proposed Tools
- `federation_propose_task(json)` - Create task from plan
- `federation_claim_task(task_id)` - Self-assign (exists)
- `federation_assign_task(task_id, agent_id)` - Explicit assignment
- `federation_update_task_status(task_id, status)` - Update state
- `federation_task_board()` - View all tasks with filters
- `federation_set_lead(task_id, agent_id)` - Designate team lead

### Status Workflow
```
PENDING → IN_PROGRESS → DONE
            ↓
         BLOCKED → (resolved) → IN_PROGRESS
```

### Storage
- Location: `/mnt/cluster/federation/shared/tasks/`
- Format: JSON files per task or single `tasks.json`

---

## Ideal Workflow (Consensus)

```
1. LEAD receives major task from human user
2. LEAD creates implementation_plan.md + task.md artifacts
3. LEAD broadcasts: federation_propose_task(subtasks[])
4. PARTICIPANTS self-claim or receive assignments
5. Each node:
   a. Sets context: "🔧 WORKING: subtask X"
   b. Works on subtask
   c. Commits & pushes with conventional commit message
   d. Sends federation_broadcast(type="result", content="Completed subtask X")
6. LEAD monitors progress, handles blockers, merges work
7. LEAD sends final result to human user
```

---

## Implementation Phases

### Phase 1: Task Registry Core
- [ ] Task object schema
- [ ] Storage in shared NFS
- [ ] `federation_propose_task()`
- [ ] `federation_task_board()`

### Phase 2: Leadership & Assignment
- [ ] Explicit LEAD designation
- [ ] `federation_assign_task()`
- [ ] LEAD visible in context window

### Phase 3: Workflow & Dependencies
- [ ] Status state machine
- [ ] `federation_update_task_status()`
- [ ] Dependency gating

### Phase 4: Advanced Features
- [ ] Plan-to-Task parser
- [ ] Git-aware messaging
- [ ] Blocker escalation

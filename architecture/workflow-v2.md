# Federation Team Mind Workflow Design

> How the Federation MCP tools, Postgres, and ActiveMQ work together to execute collaborative projects from start to finish.

---

## Current Implementation Overview

### What We Have

| Layer | Components |
|-------|------------|
| **MCP Tools** | 30 tools for messaging, workflow, consensus, COP |
| **Postgres** | conversations, tasks, artifacts, actors, consensus_polls |
| **ActiveMQ** | STOMP messaging between nodes |
| **Nodes** | baby (orchestrator), aorus-3090 (NFS/FRAG), taichi-5090 (heavy LLM) |

### Database Schema (Actual)

```
conversations: id, team_id, title, phase, leader_id, roles (JSONB), description
tasks:         id, conversation_id, title, description, assignee_id, status
artifacts:     id, conversation_id, artifact_type, content, status, created_by
actors:        id, display_name, host, specialty, current_model, status_text
```

---

## Workflow Phases

A Team Mind project flows through these phases:

```
PLANNING → REVIEW → EXECUTION → VERIFICATION → CLOSED
```

Each phase has a **gate** - the phase cannot advance until artifacts are reviewed and approved.

---

## Tool-to-Workflow Mapping

### Phase 1: PLANNING

**Goal:** Create and approve the Implementation Plan.

| Tool | Purpose |
|------|---------|
| `extension_create_conversation` | Create a new project conversation (you become leader) |
| `federation_assign_role` | Assign agents: architect, developer, reviewer, analyst |
| `federation_send` | Broadcast project kickoff to team |
| `federation_update_state` | Show "Working on plan" in COP |

**Artifact:** Create `implementation_plan` artifact in the conversation.

### Phase 2: REVIEW

**Goal:** Get team consensus on the plan before execution.

| Tool | Purpose |
|------|---------|
| `extension_send_message` | Share plan in conversation thread |
| `federation_consensus_request` | "Approve plan?" with vote mechanism |
| `federation_consensus_respond` | Cast votes |
| `federation_consensus_results` | Check if threshold met (e.g., 0.67 approval) |
| `federation_advance_phase` | Move to EXECUTION once approved |

### Phase 3: EXECUTION

**Goal:** Execute tasks, track progress, push until done.

| Tool | Purpose |
|------|---------|
| `federation_propose_task` | Broadcast task to team |
| `federation_claim_task` | Worker claims and starts task |
| `federation_update_state` | Workers show progress in COP |
| `federation_pulse` | Leader monitors COP for stuck/blocked workers |
| `federation_find_expert` | Route tasks to right specialist |

**The "Push" Loop:**
```
1. Leader checks federation_pulse()
2. See workers' COP state (IDLE/EXECUTING/BLOCKED)
3. If BLOCKED: investigate via federation_send()
4. If IDLE with pending work: assign via federation_propose_task()
5. Repeat until all tasks complete
```

### Phase 4: VERIFICATION

**Goal:** Review deliverables, run tests, confirm completion.

| Tool | Purpose |
|------|---------|
| `federation_phase_status` | View all role assignments and completion |
| `extension_get_messages` | Review conversation history |
| `federation_consensus_request` | "Accept deliverables?" |
| `federation_advance_phase` | Move to CLOSED |

### Phase 5: CLOSED

Project complete. Create final report artifact.

---

## Communication Patterns

### Structured Messages (via message_type)

| Type | Use Case |
|------|----------|
| `chat` | General discussion |
| `task` | Task assignment/request |
| `result` | Task completion with output |
| `status` | Phase changes, announcements |

### Future: Specialized Message Types

To formalize review workflows:

```python
# Proposed structured message formats
REQUEST_REVIEW = {
    "type": "request_review",
    "artifact_id": "uuid",
    "reviewer_id": "agent-id",
    "due_by": "timestamp"
}

SUBMIT_REVIEW = {
    "type": "submit_review", 
    "artifact_id": "uuid",
    "status": "approved|changes_requested",
    "feedback": "text"
}
```

---

## Common Operating Picture (COP)

The COP is the **shared situational awareness** layer.

### How to Use It

```python
# Leader periodically checks
pulse = federation_pulse()
# Returns: { cop: [...agent states...], inbox: [...], pending_approvals: [...] }

# Workers update their state
federation_update_state(
    summary="Building login component",
    action="EXECUTING",
    tags="task:123,frontend",
    thinking="Considering OAuth vs JWT",
    wondering="Should I check with taichi about GPU needs?"
)
```

### COP Actions

| Action | Meaning |
|--------|---------|
| `IDLE` | Available for work |
| `EXECUTING` | Actively working |
| `WAITING` | Blocked on external input |
| `BLOCKER` | Critically stuck, needs help |

---

## Task Lifecycle

```
PROPOSED → CLAIMED → IN_PROGRESS → DONE/FAILED
```

### Task Flow

1. **Leader proposes:** `federation_propose_task(json_str)` broadcasts to team
2. **Worker claims:** `federation_claim_task(task_id, description)` 
3. **Worker executes:** Updates COP with `federation_update_state(action="EXECUTING")`
4. **Worker completes:** Sends `federation_send(message_type="result")` with output
5. **Leader verifies:** Reviews results, moves phase forward

---

## Role Assignment

### Using `conversations.roles` JSONB

```json
{
  "taichi-5090": {
    "role": "architect",
    "subtask": "Design system architecture",
    "status": "pending"
  },
  "aorus-3090": {
    "role": "developer", 
    "subtask": "Implement core modules",
    "status": "in_progress"
  }
}
```

### Tools

| Tool | Purpose |
|------|---------|
| `federation_assign_role` | Leader assigns role to conversation |
| `federation_get_my_role` | Agent queries own assignments |
| `federation_phase_status` | View all assignments + phase |

---

## Consensus Mechanisms

| Mechanism | Use Case |
|-----------|----------|
| `poll` | Simple majority wins |
| `vote` | Yes/no with threshold (0.67 for supermajority) |
| `rank` | Ranked choice for multiple options |
| `approval` | Approve any number of options |

### Example: Plan Approval

```python
# Leader creates vote
federation_consensus_request(
    question="Approve implementation plan?",
    options=["yes", "no"],
    mechanism="vote",
    threshold=0.67
)

# Team members vote
federation_consensus_respond(consensus_id, "yes")

# Leader checks results
federation_consensus_results(consensus_id)
# Returns: { reached_consensus: true, winner: "yes", tally: {...} }
```

---

## End-to-End Example: Team Mind Project

### 1. Leader Starts Project

```python
# Create conversation
conv = extension_create_conversation(title="Build RAG Enhancement")

# Assign roles
federation_assign_role(conv.id, "taichi-5090", "architect", "Design embedding pipeline")
federation_assign_role(conv.id, "aorus-3090", "developer", "Implement FRAG integration")

# Announce
federation_send(to="all", content="New project started: RAG Enhancement")
```

### 2. Team Creates Plan

```python
# Architect creates plan
extension_send_message(conv.id, content="## Implementation Plan\n...")

# Leader requests approval
federation_consensus_request(
    question="Approve RAG Enhancement plan?",
    options=["yes", "no"],
    mechanism="vote",
    threshold=0.67
)
```

### 3. Execution Phase

```python
# After approval
federation_advance_phase(conv.id, "execution")

# Leader proposes tasks
federation_propose_task(json.dumps({
    "id": "task-001",
    "title": "Implement embedding service",
    "assignee": "aorus-3090",
    "priority": "high"
}))

# Worker claims
federation_claim_task("task-001", "Starting embedding service")
federation_update_state(summary="Working on embeddings", action="EXECUTING")
```

### 4. Leader Monitors

```python
# Check COP
pulse = federation_pulse()
# See: aorus-3090 EXECUTING, taichi-5090 IDLE

# If someone stuck
if agent.action == "BLOCKER":
    federation_send(to=agent.id, content="What's blocking you?")
```

### 5. Completion

```python
# Worker reports done
federation_send(
    to="leader",
    message_type="result",
    content="Embedding service complete. Tests passing."
)

# Leader verifies and closes
federation_advance_phase(conv.id, "verification")
federation_advance_phase(conv.id, "closed")
```

---

## What's Missing (Future Work)

### 1. Artifact Gates
Currently no enforcement preventing phase advancement without artifact approval.
**Needed:** Pre-flight check in `advance_phase` to verify required artifacts are APPROVED.

### 2. Task Table Integration  
`tasks` table exists but `propose_task` broadcasts rather than inserting.
**Needed:** Tool to create/update tasks in DB + dispatch via ActiveMQ.

### 3. Structured Review Messages
Reviews happen via chat. 
**Needed:** `request_review`, `submit_review` message types with DB tracking.

### 4. ActiveMQ Task Dispatch
Tasks broadcast to all rather than queueing to specific agent.
**Needed:** `dispatch_task` that writes to Postgres AND pushes to agent-specific queue.

---

## Component Responsibilities

| Component | Role |
|-----------|------|
| **Leader Agent** | Strategy, decomposition, phase management, pushing |
| **Worker Agent** | Task execution, status updates, result reporting |
| **MCP Server** | State enforcement, tool execution, message routing |
| **Postgres** | Single source of truth for state |
| **ActiveMQ** | Async message delivery between nodes |
| **COP** | Shared situational awareness (mind reading) |

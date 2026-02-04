# Federation Role-Based Team Protocol

## Objective
Implement role-based discipline for team collaboration, ensuring agents stay within their assigned responsibilities.

---

## Phase 1: Task & Role Schema

### Task Object (Extended)
```json
{
  "id": "task-001",
  "title": "Implement feature X",
  "description": "...",
  "lead": "taichi-5090",
  "status": "in_progress",
  "assignments": [
    {"agent": "aorus-3090", "role": "developer", "subtask": "Write endpoint"},
    {"agent": "baby", "role": "reviewer", "subtask": "Review PR"}
  ],
  "created_at": "...",
  "updated_at": "..."
}
```

### Roles & Permissions
| Role | Can Do | Cannot Do |
|------|--------|-----------|
| **lead** | create_task, assign_role, reassign, unblock, close_task, all below | - |
| **architect** | design, create_subtask (with lead approval), code | create_task, assign_role |
| **developer** | code, test, commit, report_done, request_review | create_task, assign_role, reassign |
| **reviewer** | review, approve, request_changes, report_done | code (production), create_task |
| **analyst** | analyze, summarize, report | code, create_task, assign_role |

---

## Phase 2: New Federation Tools

### `federation_assign_role(task_id, agent, role, subtask)`
- Only callable by Lead
- Registers role assignment in shared storage
- Sends notification to assigned agent

### `federation_get_my_role(task_id)` 
- Returns current role assignment for calling agent
- Used by agent to understand their constraints

### `federation_role_check(task_id, action)`
- Validates if current agent's role permits the action
- Returns: allowed (bool), reason (str)

### `federation_start_session(task_id)`
- Called by Lead to initiate a collaborative task
- Broadcasts task context to all participants
- Sets Lead in context window

### `federation_set_session_lead(agent_id)`
- Designates session lead
- Lead visible in context window: "👑 LEAD: taichi-5090"

---

## Phase 3: Protocol Enforcement

### Message Validation
When processing actions, server checks:
1. Is agent assigned to this task?
2. What is their role?
3. Is the action permitted for this role?
4. If not: reject with clear error

### Context Window Enhancement
```
**Session: task-001 - Implement feature X**
👑 LEAD: taichi-5090

📍 aorus-3090 [DEVELOPER]
   🔧 Writing endpoint...

📍 baby [REVIEWER]
   ⏳ Waiting for PR...
```

---

## Phase 4: Lead Dashboard

### `federation_session_status(task_id)`
Lead-only tool showing:
- All assignments and their status
- Blockers
- Recent activity
- Suggested interventions

---

## Implementation Order

1. [ ] Define role permissions matrix in config
2. [ ] Add `assignments` field to task schema
3. [ ] Implement `federation_assign_role()`
4. [ ] Implement `federation_get_my_role()`
5. [ ] Implement `federation_role_check()`
6. [ ] Add role enforcement in existing tools
7. [ ] Enhance context window with role display
8. [ ] Implement `federation_session_status()` for Lead
9. [ ] Document role behaviors for agent guidance

---

## Storage

- **Session tasks**: `/mnt/cluster/federation/shared/sessions/`
- **Role config**: `/mnt/cluster/federation/config/roles.yaml`

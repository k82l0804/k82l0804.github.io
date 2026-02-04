# Agentic Workflow Design

> Bridging high-level agent logic with MCP servers, Postgres state, and ActiveMQ messaging.

---

## Core Philosophy

This is a classic **Agentic Workflow** problem. To bridge the gap between your high-level logic and your MCP (Model Context Protocol) server, you need to stop thinking about the agents as the "system" and start thinking about the **Tools as the "state transitions" and "contracts."**

- **Agents** = The "intelligence"
- **MCP Server** = The "engine" that maintains state in Postgres and manages message flow in ActiveMQ

---

## Strategy: State-Machine Driven Tools

Instead of giving agents generic tools like `write_to_db`, you should give them **Domain-Specific Tools** that enforce your workflow logic. This prevents agents from "hallucinating" the process and ensures they follow the **Phase → Review → Approve** cycle.

---

## Tool Categories

### 1. Phase & Artifact Management Tools

These tools manage the lifecycle of the Project. They interact with your Postgres DB to ensure a Phase can't be skipped.

| Tool | Purpose |
|------|---------|
| `create_artifact(project_id, phase, artifact_type, content)` | Lead Agent submits a Plan, Task List, or Report |
| `submit_for_review(artifact_id, reviewer_id)` | Triggers notification (via ActiveMQ) to reviewer |
| `update_artifact_status(artifact_id, status, feedback)` | Review cycle: PENDING, REJECTED, APPROVED |
| `get_phase_gate(project_id)` | Check if current phase is "Approved" before moving on |

### 2. Team & Role Assignment Tools

Once the Task List is approved, the Team Leader needs to "staff" the project.

| Tool | Purpose |
|------|---------|
| `get_available_agents()` | Returns agents and their capabilities (Developer, QA, Researcher) |
| `assign_role(project_id, agent_id, role_type)` | Maps agent to role for that project context |

### 3. Task Execution & Tracking Tools

This is where the "pushing" happens. The Team Leader uses these to manage the queue.

| Tool | Purpose |
|------|---------|
| `dispatch_task(project_id, assigned_to, task_description, requirements)` | Write to Postgres + push to agent's ActiveMQ queue |
| `poll_task_status(project_id)` | Summary: "3 Done, 2 In-Progress" |
| `mark_task_complete(task_id, output_data)` | Sub-Agents report back |

---

## Workflow Architecture

### Postgres as the "Source of Truth"

Your tools shouldn't just pass strings; they should update rows. For example, `dispatch_task` creates a task row with `status='QUEUED'`.

### ActiveMQ as the "Nerve System"

When `dispatch_task` is called via MCP, your server should drop a JSON payload into ActiveMQ. The specific "Developer Agent" is listening to its own queue. This **decouples** the Leader from the Worker—the Leader doesn't have to wait for the worker to finish; it just checks the DB for updates.

### The "Observer" Pattern

You might want a background process (or a "Watchdog" agent) that monitors the ActiveMQ "Dead Letter Queue" or stuck tasks in Postgres and alerts the Team Leader via a tool like `get_stuck_tasks`.

---

## Example Tool Definition (JSON-RPC for MCP)

```json
{
  "name": "assign_and_dispatch_task",
  "description": "Used by the Team Leader to assign a specific task to a worker agent and initiate execution.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "project_id": { "type": "string" },
      "agent_id": { "type": "string" },
      "task_details": { "type": "string" },
      "priority": { "type": "number", "default": 1 }
    },
    "required": ["project_id", "agent_id", "task_details"]
  }
}
```

---

## Database Schema

### Core Schema: Projects & Phases

This table tracks the high-level state. The `current_phase` column ensures the Team Leader knows exactly where they are in the process.

```sql
CREATE TYPE project_phase AS ENUM (
    'PLANNING', 'TASK_LISTING', 'EXECUTION', 'REPORTING', 'COMPLETED'
);

CREATE TYPE approval_status AS ENUM (
    'DRAFT', 'PENDING_REVIEW', 'CHANGES_REQUESTED', 'APPROVED'
);

CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id VARCHAR(255) NOT NULL,
    title TEXT NOT NULL,
    description TEXT,
    current_phase project_phase DEFAULT 'PLANNING',
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE TABLE artifacts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id UUID REFERENCES projects(id),
    phase project_phase NOT NULL,
    artifact_type VARCHAR(50),  -- 'Implementation Plan', 'Task List', 'Final Report'
    content TEXT,
    status approval_status DEFAULT 'DRAFT',
    version INT DEFAULT 1,
    feedback TEXT,              -- Stores review comments
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

### Execution Schema: Team & Tasks

Once the Task List artifact is marked APPROVED, the Team Leader moves to the EXECUTION phase.

```sql
CREATE TABLE agents (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(100),
    role_specialty VARCHAR(100),  -- 'Developer', 'QA', 'Analyst'
    active_tasks_count INT DEFAULT 0
);

CREATE TABLE tasks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id UUID REFERENCES projects(id),
    assigned_to UUID REFERENCES agents(id),
    description TEXT NOT NULL,
    status VARCHAR(20) DEFAULT 'QUEUED',  -- 'QUEUED', 'IN_PROGRESS', 'DONE', 'FAILED'
    activemq_msg_id TEXT,
    result_data TEXT,
    started_at TIMESTAMP WITH TIME ZONE,
    completed_at TIMESTAMP WITH TIME ZONE
);
```

### Tool → Schema Mapping

| Tool | Postgres Action | ActiveMQ Action |
|------|-----------------|-----------------|
| `submit_artifact` | INSERT/UPDATE artifacts | — |
| `approve_artifact` | Set status to APPROVED, trigger phase update | — |
| `assign_task` | INSERT into tasks | PUSH to `agent_queue_{id}` |
| `get_project_status` | JOIN projects, artifacts, tasks | — |
| `complete_task` | Update with result_data | (Triggered by worker) |

---

## Implementation Strategy

### The "Review Loop"

**Phase Gate:** Your `get_current_state` tool should check the artifacts table. If the Implementation Plan status is not APPROVED, the MCP server should refuse to let the agent call `assign_task`. This forces the agent to handle the review cycle first.

**The ActiveMQ Bridge:** When your MCP server receives an `assign_task` call, it shouldn't just write to the DB—it should send a message. This allows your "Developer" agent to be a separate process that consumes messages at its own pace, preventing the "Team Leader" from timing out while waiting for code to be written.

**The "Push" Logic:** The Team Leader agent should have a tool called `monitor_execution`. This tool queries the tasks table for any status that isn't DONE. If it sees a FAILED task, it can re-assign it or fix the prompt.

---

## Dispatch Task Implementation

The implementation needs to be **atomic**. If the DB write fails, you shouldn't send the message. If the message fails, you should roll back the DB write.

```typescript
import { Client } from 'pg';
import cm from 'amqp10';

async function dispatchTaskTool(projectId, agentId, taskDescription) {
  const pgClient = new Client({ /* config */ });
  const mqClient = new cm.Client();
  
  await pgClient.connect();
  await mqClient.connect('amqp://activemq-server');
  await pgClient.query('BEGIN');

  try {
    // 1. Insert to Postgres
    const dbRes = await pgClient.query(
      `INSERT INTO tasks (project_id, assigned_to, description, status) 
       VALUES ($1, $2, $3, 'QUEUED') RETURNING id`,
      [projectId, agentId, taskDescription]
    );
    const taskId = dbRes.rows[0].id;

    // 2. Construct ActiveMQ payload
    const payload = {
      taskId: taskId,
      projectId: projectId,
      instructions: taskDescription,
      timestamp: new Date().toISOString()
    };

    // 3. Send to agent's queue
    const sender = await mqClient.createSender(`agent_queue_${agentId}`);
    await sender.send(payload);

    // 4. Commit
    await pgClient.query('COMMIT');
    return { success: true, taskId };

  } catch (error) {
    await pgClient.query('ROLLBACK');
    throw error;
  }
}
```

### Artifact Gate (Pre-flight Check)

```typescript
const phaseCheck = await pgClient.query(
  `SELECT status FROM artifacts 
   WHERE project_id = $1 AND artifact_type = 'Task List'`,
  [projectId]
);

if (phaseCheck.rows[0]?.status !== 'APPROVED') {
  throw new Error("Cannot dispatch tasks. Task List not APPROVED yet.");
}
```

---

## Workflow Loop: "Pushing" Until Done

The Team Leader shouldn't sit in a busy-wait loop; it should call a monitoring tool to see if it needs to take further action.

### Tool: `check_execution_progress`

```sql
SELECT 
    status, 
    COUNT(*) as count 
FROM tasks 
WHERE project_id = $1 
GROUP BY status;
```

### Lead Agent Decision Logic

| Result | Action |
|--------|--------|
| All DONE | Move to Final Report phase |
| Some FAILED | Inspect `result_data`, re-dispatch or troubleshoot |
| All QUEUED/IN_PROGRESS | Wait and check again |

---

## Worker Agent Lifecycle

The Worker Agent is a **reactive service**. While the Team Leader is the "Thinker" (proactive), the Worker is the "Doer" (reactive).

### Listen-Process-Report Loop

```typescript
import stompit from 'stompit';

stompit.connect(connectOptions, (err, client) => {
  client.subscribe({
    'destination': '/queue/agent_queue_developer_01',
    'ack': 'client-individual'
  }, (err, message) => {
    message.readString('utf-8', async (err, body) => {
      const task = JSON.parse(body);
      
      // 1. Mark IN_PROGRESS
      await updateTaskStatus(task.taskId, 'IN_PROGRESS');

      // 2. Execute work
      const result = await performWork(task.instructions);

      // 3. Report back
      await finalizeTask(task.taskId, result);
      
      message.ack();
    });
  });
});
```

### Finalize Task

```sql
UPDATE tasks 
SET 
    status = 'DONE', 
    result_data = $1,
    completed_at = NOW() 
WHERE id = $2;
```

### Error Handling

```typescript
try {
    const codeOutput = await developerAgent.generateCode(task.instructions);
    await finalizeTask(task.id, codeOutput);
} catch (error) {
    await pgClient.query(
        "UPDATE tasks SET status = 'FAILED', result_data = $1 WHERE id = $2",
        [error.message, task.id]
    );
}
```

---

## Feedback Loop Summary

1. Lead Agent calls `dispatch_task` (MCP Tool)
2. MCP Server writes to Postgres and sends message to ActiveMQ
3. Worker Agent picks up task and starts working
4. Lead Agent periodically calls `check_execution_progress`
5. If task is DONE, Lead reviews `result_data`
6. If all tasks DONE, Lead moves project to REPORTING phase

---

## Component Responsibilities

| Component | Role |
|-----------|------|
| **Team Leader** | Strategy, Decomposition, Reviewing Results, Moving Phases |
| **Worker Agent** | Execution of single discrete tasks |
| **MCP Server** | The "Legal" layer—ensures agents follow DB schema rules |
| **ActiveMQ** | The "Mailman"—delivers tasks to the right worker |
| **Postgres** | Single Source of Truth—holds state of everything |

---

## Skill Architecture (Antigravity)

Instead of giving the Team Leader 20 tools, group them into **Skills** that help the agent find the right tool for the current Phase.

### Skill 1: Project Architect
**Phases:** Planning, Task Listing  
**Tools:** `create_artifact`, `submit_for_review`, `get_phase_gate`

### Skill 2: Orchestrator
**Phase:** Execution  
**Tools:** `get_available_agents`, `assign_role`, `dispatch_task`, `check_execution_progress`

### Skill 3: Quality Controller
**Phases:** Review, Final Report  
**Tools:** `update_artifact_status`, `finalize_project`

---

## Phase-Aware Skill Logic

```typescript
class ProjectManagementSkill {
  async getContextualTools(projectId) {
    const currentState = await postgres.getProjectPhase(projectId);
    
    switch(currentState) {
      case 'PLANNING':
        return ['create_plan', 'submit_for_review'];
      case 'TASK_LISTING':
        return ['generate_task_list', 'submit_for_review'];
      case 'EXECUTION':
        return ['dispatch_task', 'check_worker_status', 'reassign_task'];
      case 'REPORTING':
        return ['generate_final_report', 'close_project'];
      default:
        return ['get_project_summary'];
    }
  }
}
```

---

## Team Leader System Prompt

```
You are the Team Leader. You manage projects through four distinct phases:
Planning, Task Listing, Execution, and Reporting.

Your Constraints:

1. Phase Gates: You cannot move to the next phase until the current phase's 
   artifact (Plan or Task List) is marked 'APPROVED' in the database.

2. Asynchronous Execution: During the 'Execution' phase, when you dispatch 
   a task, it goes to an ActiveMQ queue. Do not wait for the worker; instead, 
   use check_worker_status to see if results have been written back to Postgres.

3. The 'Push': If a task is stuck in 'IN_PROGRESS' for too long or shows 
   'FAILED', you must intervene by troubleshooting or reassigning.
```

---

## Execution Loop (The "Push")

| Step | Agent Action | Backend Action |
|------|--------------|----------------|
| Monitor | `check_worker_status` | Query tasks table |
| Evaluate | Analyze `result_data` | Agent determines if output meets requirements |
| Direct | `dispatch_task` or approve | If failed: new MQ message. If done: update status |

---

## Event-Driven Optimization

Instead of the Team Leader polling Postgres constantly, use **event-driven** architecture:

1. Worker finishes task and updates Postgres
2. Worker sends "COMPLETED" message to `management_events` queue
3. Team Leader's Orchestrator Skill hears this and triggers review turn

**Result:** The agent only "thinks" (and costs tokens) when there is actually work to review.

---

## Worker Skill: Developer

### Toolset

| Tool | Purpose |
|------|---------|
| `get_my_task_details` | Fetch requirements for assigned task ID |
| `access_project_context` | Pull Approved Implementation Plan from artifacts |
| `report_progress` | Update DB with status (e.g., "50% done") |
| `submit_final_work` | Update Postgres + trigger notification |

### Implicit Instruction

> "Whenever you receive a Task ID, your first action must be to use `get_my_task_details`. Once you have completed the code or document, you must use `submit_final_work` to signal the Team Leader. Do not consider a job finished until the tool confirms the database update."

### Worker Execution Flow

| Step | Agent Action | Backend Impact |
|------|--------------|----------------|
| Trigger | Receive Task ID from Bridge | ActiveMQ message un-acked |
| Context | Call `get_my_task_details` | `SELECT * FROM tasks WHERE id = ...` |
| Execute | Perform coding/work | Local LLM processing |
| Handoff | Call `submit_final_work` | `UPDATE tasks SET status='DONE'` + push to management queue |

---

## Task Context Query

When the Developer Skill calls `get_my_task_details`, the MCP server provides full context:

```sql
SELECT 
    t.description AS task_goal,
    p.title AS project_name,
    a.content AS approved_plan
FROM tasks t
JOIN projects p ON t.project_id = p.id
JOIN artifacts a ON p.id = a.project_id
WHERE t.id = $1 
  AND a.artifact_type = 'Implementation Plan' 
  AND a.status = 'APPROVED';
```

---

## Why This Architecture Works

1. **Isolation:** The Developer doesn't need to know about "Planning" phase tools. The Skill limits its view to only execution-related tools.

2. **State Management:** By wrapping `submit_final_work` in a Skill, Postgres state and ActiveMQ message flow remain synchronized.

3. **Scalability:** Adding a "QA Agent" just requires a new QA Skill with different queries but the same completion logic.

---

## Zombie Task Prevention

**Problem:** An agent starts a job but never finishes.

**Solution:** Heartbeat/Timeout

- **Skill Logic:** Worker immediately sets `tasks.status = 'IN_PROGRESS'`
- **Backend Logic:** If status stays `IN_PROGRESS` for > 10 minutes without update, flag for Team Leader investigation

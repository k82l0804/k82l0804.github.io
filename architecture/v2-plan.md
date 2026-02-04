# Federation v2: Implementation Plan

Based on team retrospective feedback. Priority order: immediate impact → collaboration → memory.

---

## Sprint 1: Protocol Improvements

### Feature 1: Task Claim + Context Integration

**Goal:** When claiming a task, auto-update context window for visibility.

#### [MODIFY] [mcp_federation_server.py](file:///home/k82l0804/workarea/ai-lab/federation/src/mcp_federation_server.py)

Add new tool and state:

```python
# New state
claimed_tasks: Dict[str, str] = {}  # task_id -> description

@mcp_app.tool()
def federation_claim_task(task_id: str, description: str) -> List[TextContent]:
    """Claim a task and update context window."""
    claimed_tasks[task_id] = description
    # Auto-update context
    federation_context_set(f"🔧 CLAIMED: {description}")
    return [TextContent(text=f"✅ Task claimed: {task_id}")]
```

Update `federation_context_get()` to show claimed tasks.

---

### Feature 2: Message Receipts (federation_ack)

**Goal:** Confirm message receipt with explicit acknowledgment.

#### [MODIFY] [mcp_federation_server.py](file:///home/k82l0804/workarea/ai-lab/federation/src/mcp_federation_server.py)

```python
@mcp_app.tool()
def federation_ack(message_id: str) -> List[TextContent]:
    """Acknowledge receipt of a message."""
    # Send ack back to original sender
    # Store ack in message metadata
    return [TextContent(text=f"✅ ACK sent for {message_id}")]
```

Add `/ack` HTTP endpoint to receive acknowledgments.

---

## Sprint 2: Knowledge Sharing

### Feature 3: Expertise Registry

**Goal:** Each agent advertises expertise; others can query for experts.

#### [MODIFY] [mcp_federation_server.py](file:///home/k82l0804/workarea/ai-lab/federation/src/mcp_federation_server.py)

```python
# New state
local_expertise: List[str] = []

@mcp_app.tool()
def federation_set_expertise(domains: List[str]) -> List[TextContent]:
    """Set your areas of expertise (visible to other agents)."""
    local_expertise = domains
    return [TextContent(text=f"✅ Expertise set: {', '.join(domains)}")]

@mcp_app.tool()
def federation_find_expert(domain: str) -> List[TextContent]:
    """Find agents with expertise in a domain."""
    # Query all nodes' /expertise endpoint
    # Return list of matching agents
```

Add `/expertise` HTTP endpoint.

---

## Verification Plan

### Automated Tests
1. Unit test for `federation_claim_task` 
2. Unit test for `federation_ack`
3. Integration test: claim → ack → context visibility

### Manual Verification
1. **Task Claim:** Claim task, verify other agents see it in context
2. **Message ACK:** Send message, ACK it, verify sender sees confirmation
3. **Expertise:** Set expertise, query from another node

---

## Collaborative Task Assignments

| Feature | Lead | Support |
|---------|------|---------|
| Task Claim + Context | aorus | baby |
| Message ACK | baby | taichi |
| Expertise Registry | taichi | aorus |

**Workflow:**
1. aorus creates implementation branch
2. Lead implements their feature
3. Support reviews and tests
4. Merge after verification

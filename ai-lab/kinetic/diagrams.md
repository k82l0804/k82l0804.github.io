# Workflow State Machine Diagram

```mermaid
stateDiagram-v2
    [*] --> IDLE
    
    IDLE --> PLANNING: START
    
    PLANNING --> AWAITING_PLAN_APPROVAL: PLAN_CREATED
    
    AWAITING_PLAN_APPROVAL --> CREATING_TASKS: PLAN_APPROVED
    AWAITING_PLAN_APPROVAL --> PLANNING: CHANGES_REQUESTED
    
    CREATING_TASKS --> AWAITING_TASKS_APPROVAL: TASKS_CREATED
    
    AWAITING_TASKS_APPROVAL --> EXECUTION: TASKS_APPROVED
    AWAITING_TASKS_APPROVAL --> CREATING_TASKS: CHANGES_REQUESTED
    
    EXECUTION --> VERIFICATION: IMPLEMENTATION_COMPLETE
    
    VERIFICATION --> COMPLETE: TESTS_PASS
    
    COMPLETE --> [*]
    
    note right of AWAITING_PLAN_APPROVAL
        🟡 User Approval Gate
        Mode: interactive
    end note
    
    note right of AWAITING_TASKS_APPROVAL
        🟡 User Approval Gate
        Mode: interactive
    end note
    
    note right of EXECUTION
        🟠 Actor Model
        Tools: executionTools
        MaxSteps: 20
    end note
    
    note right of PLANNING
        🔵 Planner Model
        Tools: planningTools
        Temperature: 0.3
    end note
```

---

# Router Decision Flow

```mermaid
flowchart TD
    Start([👤 User Message]) --> FastPath{Contains<br/>'plan', 'build',<br/>'create system'?}
    
    FastPath -->|YES| Workflow[🟢 WORKFLOW Intent]
    FastPath -->|NO| ChatCheck{Simple question<br/>or chat?}
    
    ChatCheck -->|YES| Chat[🟣 CHAT Intent]
    ChatCheck -->|NO| LLM[🤖 LLM Classification<br/>grok-beta / temp=0.0]
    
    LLM --> TaskCheck{Atomic task?}
    
    TaskCheck -->|YES| Direct[🟠 DIRECT Intent]
    TaskCheck -->|NO| Ambiguous[⚪ AMBIGUOUS Intent]
    
    Chat --> ChatConfig[Model: Planner<br/>Tools: None<br/>Temp: 0.6]
    Direct --> DirectConfig[Model: Actor<br/>Tools: directTools<br/>Temp: 0.1<br/>MaxSteps: 5-10]
    Workflow --> WorkflowConfig[Model: Planner→Actor<br/>Tools: planning→execution<br/>Temp: 0.3→0.1<br/>MaxSteps: 1→20]
    Ambiguous --> AmbiguousConfig[Model: Router<br/>Tools: None<br/>Temp: 0.0]
    
    ChatConfig --> Execute[Execute]
    DirectConfig --> Execute
    WorkflowConfig --> Execute
    AmbiguousConfig --> Execute
    
    style Chat fill:#9b59b6,color:#fff
    style Direct fill:#e67e22,color:#fff
    style Workflow fill:#27ae60,color:#fff
    style Ambiguous fill:#95a5a6,color:#fff
    style LLM fill:#f39c12,color:#000
```

---

# Complete Execution Flow

```mermaid
flowchart TB
    User([👤 User Input]) --> WebView[Webview Handler]
    WebView --> Provider[AgentWorkflowViewProvider]
    
    Provider --> Router{🎯 Router}
    
    Router --> ChatPath[CHAT Path]
    Router --> DirectPath[DIRECT Path]
    Router --> WorkflowPath[WORKFLOW Path]
    
    ChatPath --> ChatModel[VercelAIClient<br/>📝 Planner Model<br/>No Tools]
    ChatModel --> ChatResponse[Display Response]
    
    DirectPath --> DirectModel[VercelAIClient<br/>⚡ Actor Model<br/>🛠️ directTools]
    DirectModel <--> DirectTools[Tool Execution Loop<br/>MaxSteps: 5-10]
    DirectTools --> DirectResponse[Display Response]
    
    WorkflowPath --> Orchestrator[WorkflowOrchestrator<br/>📊 State Machine]
    Orchestrator --> Planning[Phase 1: Planning<br/>🎨 Create Plan]
    Planning --> Approval1{User Approves?}
    Approval1 -->|Yes| Tasks[Phase 2: Tasks<br/>📋 Create Tasks]
    Approval1 -->|No| Planning
    Tasks --> Approval2{User Approves?}
    Approval2 -->|Yes| Execution[Phase 3: Execution<br/>⚡ Actor + Tools<br/>MaxSteps: 20]
    Approval2 -->|No| Tasks
    Execution --> Verify[Phase 4: Verification]
    Verify --> WorkflowResponse[Display Response]
    
    DirectTools --> Policy{🛡️ ToolPolicyService}
    Execution --> Policy
    
    Policy --> Catastrophic{Catastrophic?}
    Catastrophic -->|Yes| Deny[❌ DENY]
    Catastrophic -->|No| Destructive{Destructive +<br/>Autonomous?}
    Destructive -->|Yes| Ask[⚠️ ASK User]
    Destructive -->|No| Allow[✅ ALLOW]
    
    Ask --> UserApproval{Approved?}
    UserApproval -->|Yes| ToolExec[🔧 Execute Tool]
    UserApproval -->|No| Deny
    Allow --> ToolExec
    
    ToolExec --> Registry[Tool Registry]
    Registry --> FileOps[File Operations<br/>workspace.fs API]
    Registry --> Commands[Command Execution<br/>Terminal API]
    
    style ChatPath fill:#9b59b6,color:#fff
    style DirectPath fill:#e67e22,color:#fff
    style WorkflowPath fill:#27ae60,color:#fff
    style Policy fill:#3498db,color:#fff
    style Deny fill:#e74c3c,color:#fff
    style Ask fill:#f39c12,color:#000
    style Allow fill:#27ae60,color:#fff
```

---

# Tool Execution Security Flow

```mermaid
flowchart TD
    Start([🤖 LLM Calls Tool]) --> Wrapper[AgentWorkflowViewProvider<br/>_bindTools wrapper]
    
    Wrapper --> Policy[🛡️ ToolPolicyService<br/>checkPermission]
    
    Policy --> Cat{Catastrophic<br/>Command?}
    
    Cat -->|YES| Deny[❌ DENY<br/>Block Execution]
    Cat -->|NO| Dest{Destructive +<br/>Autonomous?}
    
    Dest -->|YES| Ask[⚠️ ASK<br/>User Approval Dialog]
    Dest -->|NO| Allow[✅ ALLOW<br/>Proceed]
    
    Ask --> UserDec{User<br/>Approves?}
    UserDec -->|Yes| Allow
    UserDec -->|No| Deny
    
    Deny --> Error[Return Error to LLM]
    
    Allow --> Execute[🔧 Execute Tool]
    Execute --> Type{Tool Type}
    
    Type -->|File Ops| FS[workspace.fs API<br/>createFile, readFile, etc.]
    Type -->|Commands| Term[vscode.Terminal<br/>runCommand]
    
    FS --> Result[Return Result]
    Term --> Result
    
    Result --> LLM[Return to LLM]
    
    LLM --> More{More Steps<br/>Needed?}
    More -->|YES| Start
    More -->|NO| Final([✅ Final Response])
    
    style Deny fill:#e74c3c,color:#fff
    style Ask fill:#f39c12,color:#000
    style Allow fill:#27ae60,color:#fff
    style Cat fill:#e74c3c,color:#fff
    style Dest fill:#f39c12,color:#000
    style Policy fill:#3498db,color:#fff
```

---

## How to View These Diagrams

### Option 1: GitHub/GitLab (Automatic Rendering)
These Mermaid diagrams render automatically in markdown on GitHub/GitLab.

### Option 2: VS Code (Mermaid Extension)
1. Install "Markdown Preview Mermaid Support" extension
2. Open this file and preview (Ctrl+Shift+V)

### Option 3: Online Editor
Copy diagram code to https://mermaid.live/

### Option 4: Generate PNG
```bash
# Install mermaid-cli
npm install -g @mermaid-js/mermaid-cli

# Generate images
mmdc -i diagrams.md -o workflow-state-machine.png
```

---

## Diagram Legend

| Symbol | Meaning |
|--------|---------|
| 🟢 | Safe / Approved path |
| 🟡 | Caution / Approval required |
| 🟠 | Action / Execution phase |
| 🔵 | Planning / Analysis phase |
| ⚪ | Neutral / Ambiguous |
| 🛡️ | Security checkpoint |
| ✅ | Allowed |
| ⚠️ | Warning / Ask user |
| ❌ | Denied / Blocked |
| 🤖 | LLM / AI component |
| 🔧 | Tool execution |
| 📊 | State machine |
| 🎯 | Router / Decision point |

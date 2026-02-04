# Agent Workflow Execution Flow

## Overview

This document traces the complete execution flow of the Kinetic Agent Workflow from user input to tool execution. Use this to understand how components interact and which code paths are taken for different types of requests.

---

## 📍 Entry Point

**File**: `AgentWorkflowViewProvider.ts`  
**Trigger**: User sends message in webview  
**Input**: `{ message: string }`

```typescript
// Line ~250
private async handleUserMessage(message: string) {
    // 1. Create user message
    // 2. Route to Router for classification
    // 3. Execute based on classification
}
```

---

## 🔀 Phase 1: Intent Classification

### Router Classification

**File**: `Router.ts`  
**Method**: `classifyIntent(userMessage: string)`  
**Purpose**: Determine if request is CHAT, DIRECT, WORKFLOW, or AMBIGUOUS

**Decision Tree**:
```
Input: User message
  ↓
Router.classifyIntent()
  ↓
Uses: generateObject() with gpt-4/grok-beta
Schema: { intent: "CHAT" | "DIRECT" | "WORKFLOW" | "AMBIGUOUS" }
  ↓
Output: UserIntent enum
```

**Code Path**:
```typescript
Router.ts:93-100
├─ Fast path: Keyword detection (WORKFLOW if "plan" in message)
├─ Smart path: LLM classification with intent_classifier.md prompt
└─ Fallback: Returns AMBIGUOUS if LLM unavailable
```

**Temperature**: `0.0` (deterministic)  
**Model**: Router model from profile (e.g., `grok-beta`)

---

## 🎯 Phase 2: Routing Decision

### Three Execution Paths

Based on classified intent, Router selects:
1. **Model Role**: `router | planner | actor`
2. **Tool Collection**: `none | directTools | planningTools | executionTools`

**Matrix** (`Router.ts:180-196`):

| Intent | Phase | Model | Tools |
|--------|-------|-------|-------|
| CHAT | IDLE | planner | none |
| DIRECT | IDLE | actor | directTools |
| AMBIGUOUS | IDLE | router | none |
| WORKFLOW | PLANNING | planner | planningTools |
| WORKFLOW | EXECUTION | actor | executionTools |

---

## 🛤️ Execution Path A: CHAT Intent

**Purpose**: Answer questions, no tools needed

```
User: "How does React work?"
  ↓
Router → CHAT
  ↓
Model: planner (gpt-4-turbo)
Tools: none
Temperature: 0.6 (via TemperatureSelector)
  ↓
VercelAIClient.generateWithMessages()
  ↓
Response: Text only (no tool calls)
  ↓
Display in chat
```

**Files Involved**:
1. `Router.ts` - Classification
2. `TemperatureSelector.ts` - Selects 0.6 for CHAT
3. `VercelAIClient.ts` - API call
4. `AgentWorkflowViewProvider.ts` - Display response

**Key**: No tool execution, conversational response only

---

## 🛤️ Execution Path B: DIRECT Intent

**Purpose**: Execute atomic tasks immediately

```
User: "Create hello.txt with 'Hello World'"
  ↓
Router → DIRECT
  ↓
Model: actor (grok-2 or codestral)
Tools: directTools (createFile, readFile, etc.)
Temperature: 0.1 (syntax correctness)
MaxSteps: 5 (interactive) or 10 (autonomous)
  ↓
VercelAIClient.streamChatWithAdapter()
  ↓
LLM calls: createFile(path="hello.txt", content="Hello World")
  ↓
Tool Execution:
  ├─ ToolPolicyService checks permission
  ├─ Execute via workspace.fs API
  └─ Return result
  ↓
LLM sees result, may call more tools or finish
  ↓
Display final response
```

**Files Involved**:
1. `Router.ts` - Classification → DIRECT
2. `MaxStepsSelector.ts` - Selects 5 or 10
3. `TemperatureSelector.ts` - Selects 0.1
4. `VercelAIClient.ts` - Streaming with tools
5. `tools/toolRegistry.ts` - Tool definitions
6. `services/ToolPolicyService.ts` - Permission checks
7. `AgentWorkflowViewProvider._bindTools()` - Tool execution wrapper

**Key**: Immediate action, no planning phase

---

## 🛤️ Execution Path C: WORKFLOW Intent

**Purpose**: Multi-step complex tasks with planning

### Step 1: Planning Phase

```
User: "Build REST API with authentication"
  ↓
Router → WORKFLOW
  ↓
Orchestrator.transition(PLANNING)
  ↓
Model: planner (gpt-4-turbo)
Tools: planningTools (createPlan, notify_user)
Temperature: 0.3 (structured reasoning)
  ↓
LLM calls: createPlan(steps=[...], artifacts={plan: "..."})
  ↓
Orchestrator.handleEvent(PLAN_CREATED)
  ↓
State → AWAITING_PLAN_APPROVAL
  ↓
Display plan to user, await approval
```

### Step 2: Plan Approval

```
User clicks "Approve Plan"
  ↓
Orchestrator.handleEvent(PLAN_APPROVED)
  ↓
State → CREATING_TASKS (if needed) or EXECUTION
```

### Step 3: Execution Phase

```
State: EXECUTION
  ↓
Model: actor (grok-2 or codestral)
Tools: executionTools (all file/command tools)
Temperature: 0.1 (precision)
MaxSteps: 20 (autonomous mode)
  ↓
Actor executes tasks:
  ├─ createFile(...)
  ├─ runCommand(...)
  ├─ readFile(...)
  └─ ... (loop up to maxSteps)
  ↓
Orchestrator monitors progress
  ↓
State → VERIFICATION or COMPLETE
```

**Files Involved**:
1. `Router.ts` - Classification → WORKFLOW
2. `WorkflowOrchestrator.ts` - State machine (XState)
3. `machines/workflowMachine.ts` - State definitions
4. `services/PromptFactory.ts` - Loads phase-specific prompts
5. `VercelAIClient.ts` - Multi-step streaming
6. `ArtifactManager.ts` - Saves plans/tasks
7. Tool execution (same as DIRECT)

**Key**: Stateful, multi-phase execution with human approval gates

---

## 🔧 Component Deep-Dive

### Router (`Router.ts`)

**Responsibilities**:
1. Classify user intent (CHAT/DIRECT/WORKFLOW/AMBIGUOUS)
2. Select appropriate model role (router/planner/actor)
3. Select appropriate tool collection
4. Provide temperature via TemperatureSelector

**Key Methods**:
- `classifyIntent()` - LLM-based classification
- `getModel(intent, phase)` - Model role selection
- `getTools(intent, phase)` - Tool collection selection

**Matrices**: See lines 160-240

---

### VercelAIClient (`VercelAIClient.ts`)

**Responsibilities**:
1. Wrap Vercel AI SDK (version 6)
2. Normalize responses across providers
3. Handle streaming and non-streaming calls

**Key Methods**:
```typescript
generateWithMessages()   // Non-streaming, single response
streamChatWithAdapter()  // Streaming with tools, multi-step
```

**SDK 6 Adaptations**:
- Uses `stepCountIs(n)` instead of `maxSteps`
- Token usage: `inputTokens`/`outputTokens`
- Tool calls: `tool.input` not `tool.args`

---

### WorkflowOrchestrator (`WorkflowOrchestrator.ts`)

**Responsibilities**:
1. Manage workflow state machine (via XState)
2. Handle state transitions
3. Coordinate between UI and execution

**States**:
```
IDLE → PLANNING → AWAITING_PLAN_APPROVAL → CREATING_TASKS 
  → AWAITING_TASKS_APPROVAL → EXECUTION → VERIFICATION → COMPLETE
```

**Events**:
- `START`, `PLAN_CREATED`, `PLAN_APPROVED`, `TASKS_CREATED`, etc.

**File**: `machines/workflowMachine.ts` - State machine definition

---

### ToolPolicyService (`services/ToolPolicyService.ts`)

**Responsibilities**:
1. Determine if tool execution requires approval
2. Block catastrophic commands
3. Context-aware permission logic

**Decision Matrix**:
```typescript
shouldRequireApproval(toolName, args, mode, intent, phase)
  ↓
Checks:
  ├─ Catastrophic command? (rm -rf, etc.) → DENY
  ├─ Destructive tool in autonomous mode? → ASK
  ├─ createFile with overwrite=true? → ASK
  └─ Safe operation → ALLOW
```

---

### Model Adapters (`adapters/`)

**Purpose**: Normalize different LLM APIs to common interface

#### NativeAdapter
- **For**: GPT-4, Grok (native function calling)
- **Logic**: Pass-through, parse `tool_calls` from response
- **Fallback**: Handle Grok's JSON-in-content edge case

#### PromptAdapter
- **For**: Codestral, Llama, local models (no native tools)
- **Logic**: 
  1. Inject TypeScript interface definitions
  2. Use vLLM `guided_json` for valid output
  3. Parse JSON for tool calls
- **Complexity**: Multiple parsing fallbacks, aggressive prompt patching

---

## 🎨 Data Flow Diagrams

### Tool Execution Flow

```
LLM decides to call tool
  ↓
VercelAIClient receives tool_call
  ↓
AgentWorkflowViewProvider._bindTools() wrapper
  ↓
ToolPolicyService.checkPermission()
  ├─ ALLOW → Execute immediately
  ├─ ASK → Prompt user, await approval
  └─ DENY → Block, return error
  ↓
Tool execution (via toolRegistry.ts)
  ├─ File tools: Use workspace.fs API
  └─ Command tools: Use vscode.window.createTerminal()
  ↓
Return result to LLM
  ↓
LLM continues or finishes
```

### Configuration Loading Flow

```
Extension activation
  ↓
ConfigService.loadConfig()
  ↓
Read ~/.kinetic/config/kinetic_config.yaml
  ↓
Parse YAML → Config object
  ↓
Validate required fields (models, profiles)
  ↓
TemperatureSelector.loadConfig() → temperature-config.yaml
MaxStepsSelector.loadConfig() → maxsteps-config.yaml
  ↓
Configuration ready
  ↓
Router uses config for model selection
VercelAIClient uses config for API calls
```

---

## 🚨 Critical Code Paths

### Path 1: Extension Won't Activate
```
Issue: No workspace folder open
  ↓
Check: extension.ts:7-18
  ↓
Behavior: Shows warning, continues in degraded mode
```

### Path 2: LLM Returns Empty Response
```
Issue: response.choices array empty
  ↓
Check: Logger.ts:65, NativeAdapter.ts:38, PromptAdapter.ts:173
  ↓
Behavior: Throws clear error message
```

### Path 3: Tool Permission Denied
```
Issue: Catastrophic command detected
  ↓
Check: ToolPolicyService.isCatastrophic()
  ↓
Behavior: Returns DENY, blocks execution
```

---

## 🐛 Known Issues & Workarounds

### Issue 1: Codestral Outputs Thinking Text
**Location**: `adapters/PromptAdapter.ts:218-226`  
**Symptom**: Model includes reasoning before JSON  
**Workaround**: Parse and strip JSON, preserve thinking as `content`

### Issue 2: Grok Embeds JSON in Content
**Location**: `adapters/NativeAdapter.ts:59-112`  
**Symptom**: `tool_calls` in content field instead of structured  
**Workaround**: Detect JSON, parse, extract tool calls

### Issue 3: Output Format Conflicts
**Location**: `adapters/PromptAdapter.ts:85-131`  
**Symptom**: Base prompt forbids JSON, contradicts adapter needs  
**Workaround**: Aggressive regex patching to replace conflicting instructions

---

## 📚 Additional Documentation

- **Router Architecture**: `/docs/kinetic/router-architecture.md`
- **Mode/Intent Matrix**: `/docs/kinetic/mode-intent-behavior-matrix.md`
- **Tool Policy**: See `services/ToolPolicyService.ts` header
- **State Machine**: See `machines/workflowMachine.ts` inline docs

---

## 🔍 Debugging Tips

### Enable Debug Logging
```typescript
// In extension.ts activation
logger.setLevel(LogLevel.DEBUG);
```

### Trace Intent Classification
```typescript
// Router.ts:93
console.log('[Router] Classifying:', userMessage);
// Check console for classification result
```

### Inspect Tool Calls
```typescript
// AgentWorkflowViewProvider._bindTools()
// Line ~340: Add breakpoint to see tool execution
```

### View State Transitions
```typescript
// WorkflowOrchestrator.ts
// State machine logs all transitions automatically
```

---

## 🎯 Quick Reference

| User Says | Intent | Model | Tools | Max Steps |
|-----------|--------|-------|-------|-----------|
| "How does X work?" | CHAT | planner | none | 1 |
| "Create file.txt" | DIRECT | actor | directTools | 5-10 |
| "Build REST API" | WORKFLOW | planner→actor | planning→execution | 1→20 |
| "I'm not sure..." | AMBIGUOUS | router | none | 1 |

---

**Last Updated**: 2025-12-30  
**Maintainer**: Kinetic Team  
**Related**: `router-architecture.md`, `mode-intent-behavior-matrix.md`

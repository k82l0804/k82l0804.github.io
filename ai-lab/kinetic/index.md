# Kinetic Agent Workflow Documentation

Complete documentation for the Kinetic Agent Workflow system - an agentic coding assistant built for VS Code.

---

## 📚 Documentation Index

### 🎯 Start Here

**New to the system?** Read these in order:

1. **[Execution Flow](execution-flow.md)** - How the system works from user input to output
2. **[Diagrams](diagrams.md)** - Visual state machines and flow charts
3. **[Reference Tables](reference-tables.md)** - Quick lookup for matrices and rules

### 🔧 System Guides

Deep dives into specific subsystems:

4. **[Configuration Guide](configuration-guide.md)** - How config files are loaded and used
5. **[Prompt System Guide](prompt-system-guide.md)** - Prompt structure, loading, and best practices
6. **[Adapter Guide](adapter-guide.md)** - NativeAdapter vs PromptAdapter explained
7. **[Troubleshooting Guide](troubleshooting-guide.md)** - Diagnose and fix common issues

---

## 🗺️ Navigation by Use Case

### "I want to understand how it works"
→ Start with **Execution Flow** + **Diagrams**

### "I need to configure a new model"
→ Read **Configuration Guide** + **Adapter Guide**

### "How do I customize prompts?"
→ See **Prompt System Guide**

### "Something's broken"
→ Check **Troubleshooting Guide** first, then **Execution Flow** to trace code path

### "What temperature should I use for X?"
→ See **Reference Tables** → Temperature Matrix

### "Which tools are available in EXECUTION phase?"
→ See **Reference Tables** → Tool Collections

---

## 🏗️ System Architecture

### Components

```
User Input (Webview)
  ↓
AgentWorkflowViewProvider (Main Coordinator)
  ↓
Router (Intent Classification + Model Selection)
  ├─ TemperatureSelector (Temp: 0.0-0.6)
  ├─ MaxStepsSelector (Steps: 1-20)
  └─ ConfigService (Load models, profiles)
  ↓
VercelAIClient (API Wrapper)
  ├─ ModelFactory (Select Adapter)
  │   ├─ NativeAdapter (GPT-4, Grok)
  │   └─ PromptAdapter (Codestral, Llama)
  └─ PromptFactory (Load & Assemble Prompts)
  ↓
LLM Response
  ↓
Tool Execution (If tool calls present)
  ├─ ToolPolicyService (Permission Check)
  └─ Tool Registry (Actual Execution)
  ↓
Display Result
```

See **[Diagrams](diagrams.md)** for visual representation.

---

## 📖 Key Concepts

### Intent Classification

User messages are classified into 4 types:
- **CHAT**: Questions, explanations → No tools, conversational
- **DIRECT**: Atomic tasks → Immediate execution with tools
- **WORKFLOW**: Complex projects → Multi-phase with planning
- **AMBIGUOUS**: Unclear → Re-classification needed

See **[Execution Flow → Phase 1](execution-flow.md#phase-1-intent-classification)**

### Model Roles

Three roles selected based on Intent × Phase:
- **router**: Fast classification (temp=0.0)
- **planner**: Strategic reasoning (temp=0.3)
- **actor**: Tool execution (temp=0.1)

See **[Reference Tables → Router Selection](reference-tables.md#router-selection-matrices)**

### Tool Collections

Tools grouped by capability:
- **planningTools**: `createPlan`,` createTasks`
- **directTools**: All tools except `startWorkflow`
- **executionTools**: All tools (full power)
- **verificationTools**: Read-only validation

See **[Reference Tables → Tool Collections](reference-tables.md#tool-collections)**

---

## 🎨 Configuration Files

### Primary Configuration

**`~/.kinetic/config/kinetic_config.yaml`**
```yaml
models:
  - name: "gpt-4-turbo"
    provider: "openai"
    apiKey: "${OPENAI_API_KEY}"
    strategy: "native"

profiles:
  router: "grok-beta"
  planner: "gpt-4-turbo"
  actor: "grok-2"
```

See **[Configuration Guide](configuration-guide.md)**

### Runtime Tuning

**`~/.kinetic/config/temperature-config.yaml`** - Override temperatures  
**`~/.kinetic/config/maxsteps-config.yaml`** - Override maxSteps

See **[Reference Tables → Temperature/MaxSteps Matrices](reference-tables.md#temperature-matrix)**

---

## 🛠️ Tool Permissions

Tools are governed by **ToolPolicyService** based on:
- Tool destructiveness (safe vs. dangerous)
- Execution mode (interactive vs. autonomous)
- Workflow phase (EXECUTION has more trust)
- Plan approval status (approved = elevated trust)

**Permission Levels**:
- **ALLOW**: Execute immediately
- **ASK**: Prompt user for approval
- **DENY**: Block (catastrophic commands)

See **[Reference Tables → Permission Matrix](reference-tables.md#permission-matrix)**

---

## 🔄 Workflow Phases

```
IDLE → PLANNING → AWAIT_PLAN_APPROVAL → CREATING_TASKS 
  → AWAIT_TASKS_APPROVAL → EXECUTION → VERIFICATION → COMPLETE
```

**Approval Gates**: User can review/modify plans and tasks before execution

See **[Diagrams → State Machine](diagrams.md#workflow-state-machine-diagram)**

---

## 🐛 Common Issues

| Problem | Quick Fix |
|---------|-----------|
| Extension won't activate | Check workspace folder open |
| API call fails | Verify API keys set |
| Tool outputs text not calls | Check adapter strategy matches model |
| Workflow stuck | Check state machine transitions |
| Wrong temperature | Override in temperature-config.yaml |

See **[Troubleshooting Guide](troubleshooting-guide.md)** for detailed diagnostics.

---

## 📊 Quick Reference

### Intent → Model → Tools

| User Says | Intent | Model | Tools | Temp | MaxSteps |
|-----------|--------|-------|-------|------|----------|
| "How does X work?" | CHAT | planner | none | 0.6 | 1 |
| "Create file.py" | DIRECT | actor | directTools | 0.1 | 5-10 |
| "Build REST API" | WORKFLOW | planner→actor | planning→execution | 0.3→0.1 | 1→20 |

### File Locations

| File | Purpose |
|------|---------|
| `~/.kinetic/config/kinetic_config.yaml` | Main configuration |
| `~/.kinetic/config/temperature-config.yaml` | Temperature overrides |
| `~/.kinetic/config/maxsteps-config.yaml` | MaxSteps overrides |
| `~/.kinetic/prompts/*.md` | Prompt templates |
| `~/.kinetic/sessions/{id}/` | Workflow artifacts |

---

## 🚀 Quick Start

1. **Install Extension**: Load in VS Code
2. **Configure**: Create `~/.kinetic/config/kinetic_config.yaml`
3. **Set API Keys**: Export environment variables
4. **Test**: Send "Create hello.txt" in workflow panel
5. **Read Docs**: Start with **Execution Flow**

---

**Questions?** Check **[Troubleshooting Guide](troubleshooting-guide.md)** or review **[Execution Flow](execution-flow.md)** to understand code paths.

---

**Last Updated**: 2025-12-30 | **Version**: 1.0 | **AI SDK**: 6.0.3

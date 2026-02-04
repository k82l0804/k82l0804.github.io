# Troubleshooting Guide

Quick diagnostic guide for common issues in the Agent Workflow system.

---

## 🚨 Extension Won't Activate

### Symptom
Extension appears in sidebar but doesn't respond, or shows error on activation.

### Diagnostic Steps

**1. Check Workspace Folder**
```
PROBLEM: No workspace folder open
SOLUTION: Open a folder (File → Open Folder)
```

The extension requires a workspace folder. See `extension.ts:9-13` for graceful degradation logic.

**2. Check Output Panel**
```
1. View → Output
2. Select "Agent Workflow" from dropdown
3. Look for error messages
```

**3. Check Configuration**
```bash
# Verify config file exists
ls -la ~/.kinetic/config/kinetic_config.yaml

# Validate YAML syntax
cat ~/.kinetic/config/kinetic_config.yaml | python3 -c "import yaml, sys; yaml.safe_load(sys.stdin)"
```

### Common Causes

| Error Message | Cause | Solution |
|---------------|-------|----------|
| "No models configured" | Missing kinetic_config.yaml | Create config file |
| "YAML parse error" | Invalid YAML syntax | Fix indentation/quotes |
| "No workspace folder" | VS Code opened without folder | Open folder |

---

## 🔌 API Connection Failures

### Symptom
"API call failed" or "401 Unauthorized" errors.

### Diagnostic Steps

**1. Verify API Keys**
```bash
# Check environment variables
echo $OPENAI_API_KEY
echo $XAI_API_KEY

# If using .env file
cat ~/.kinetic/.env
```

**2. Test API Directly**
```bash
# OpenAI
curl https://api.openai.com/v1/models \
  -H "Authorization: Bearer $OPENAI_API_KEY"

# xAI
curl https://api.x.ai/v1/models \
  -H "Authorization: Bearer $XAI_API_KEY"
```

**3. Check Network Access**
```bash
# Verify connectivity
ping api.openai.com
ping api.x.ai

# Check proxy settings
echo $HTTP_PROXY
echo $HTTPS_PROXY
```

### Solutions

| Issue | Solution |
|-------|----------|
| API key not set | Export in `~/.bashrc` or `~/.zshrc` |
| Wrong API key format | Check starts with `sk-` (OpenAI) or `xai-` (xAI) |
| Proxy blocking | Configure VS Code proxy settings |
| Rate limiting | Add delay between requests |

---

## 🤖 LLM Not Responding Correctly

### Issue 1: Model Outputs Text Instead of Tool Calls

**Symptom**: Model says "I will create the file..." instead of actually calling `createFile`.

**Diagnosis**:
```typescript
// Check logs for:
[Router] Selected model: codestral-latest
[ModelFactory] Using PromptAdapter  // ← Should show correct adapter
```

**Solutions**:

1. **Wrong Adapter for Model**
```yaml
# kinetic_config.yaml
models:
  - name: "codestral-latest"
    strategy: "native"  # ❌ WRONG for Codestral
```

Fix:
```yaml
strategy: "prompt"  # ✅ CORRECT
```

2. **Prompt Patching Failed**

Check `PromptAdapter.ts:133`:
```typescript
if (content.length !== originalLength) {
    console.log('[PromptAdapter] Successfully patched');
} else {
    console.warn('[PromptAdapter] WARNING: Patch did not match!');
}
```

If warning appears, prompt structure doesn't match expected format. Update prompt template.

3. **Model Doesn't Support Structured Output**

Verify vLLM has `guided_json` enabled:
```bash
# vLLM must be started with:
vllm serve codestral-latest --enable-auto-tool-choice
```

### Issue 2: Tool Calls Have Wrong Arguments

**Symptom**: Tool execution fails with "missing required argument".

**Diagnosis**:
```typescript
// Add to AgentWorkflowViewProvider._bindTools
console.log('Tool call:', name, 'Args:', JSON.stringify(args, null, 2));
```

**Solutions**:

1. **Argument Name Mismatch**
```typescript
// Tool definition says:
parameters: { path: string }

// LLM calls with:
{ file_path: "hello.txt" }  // ❌ Wrong key
```

Fix: Update tool definition or add aliasing in tool execution.

2. **Type Mismatch**
```typescript
// Expected:
{ overwrite: true }

// Got:
{ overwrite: "true" }  // ❌ String instead of boolean
```

Fix: Add type coercion in tool wrapper.

---

## 🔒 Permission / Approval Issues

### Issue: Tool Execution Always Requires Approval

**Symptom**: Even safe operations ask for user approval in autonomous mode.

**Diagnosis**:
```typescript
// Check ToolPolicyService logic
const result = ToolPolicyService.checkPermission(
    'createFile',
    { path: 'test.txt', overwrite: false },
    'autonomous',
    UserIntent.WORKFLOW,
    WorkflowPhase.EXECUTION
);
console.log('Permission result:', result);  // Should be ALLOW
```

**Solutions**:

1. **Check Mode**
```typescript
// Verify orchestrator mode
console.log('Current mode:', orchestrator.getMode());
// Should be 'autonomous' if plan was approved
```

2. **Check Plan Approved Flag**
```typescript
// In WorkflowOrchestrator
console.log('Plan approved:', this.isPlanApproved());
```

3. **Check Tool Policy Logic**

See `services/ToolPolicyService.ts:45-80`. Ensure:
- Approved plan = elevated trust
- EXECUTION phase = allow more tools
- Safe tools = always ALLOW

---

## 🌡️ Temperature / MaxSteps Issues

### Issue: Responses Too Random/Deterministic

**Symptom**: Model gives different answers for same input (or too similar).

**Diagnosis**:
```typescript
// Add to Router.ts:93 or VercelAIClient.ts
const temp = this.tempSelector.getTemperature(intent, phase);
console.log(`Using temperature: ${temp} for ${intent}/${phase}`);
```

**Expected Values**:
- CHAT: 0.6 (varied)
- EXECUTION: 0.1 (precise)
- PLANNING: 0.3 (balanced)

**Solutions**:

1. **Override Temperature**

Create `~/.kinetic/config/temperature-config.yaml`:
```yaml
workflow:
  execution: 0.05  # Even more deterministic
```

2. **Check Temperature Selector Loading**
```typescript
// TemperatureSelector.ts constructor
console.log('Loaded temperature config:', this.config);
```

### Issue: Tool Loop Ends Too Early/Late

**Symptom**: LLM stops after 1 tool call, or calls tools 50+ times.

**Diagnosis**:
```typescript
// Add to VercelAIClient.ts:360
const maxSteps = this.maxStepsSelector.getMaxSteps(intent, phase, mode);
console.log(`Using maxSteps: ${maxSteps}`);
```

**Expected Values**:
- CHAT: 1
- DIRECT (interactive): 5
- WORKFLOW/EXECUTION (autonomous): 20

**Solutions**:

1. **Override MaxSteps**

Create `~/.kinetic/config/maxsteps-config.yaml`:
```yaml
workflow:
  execution:
    autonomous: 50  # Allow longer chains
```

2. **Check Mode Detection**
```typescript
const mode = orchestrator.getMode();
console.log('Workflow mode:', mode);  // Should be 'autonomous' after approval
```

---

## 📝 Prompt Loading Failures

### Symptom
"Failed to load prompt" error, or default prompt used instead of custom.

### Diagnostic Steps

**1. Verify Prompt File Exists**
```bash
ls -la ~/.kinetic/prompts/
# Should show: intent_classifier.md, execution_phase.md, etc.
```

**2. Check File Permissions**
```bash
chmod 644 ~/.kinetic/prompts/*.md
```

**3. Validate Prompt Structure**
```bash
# Must contain these sections:
grep "INSTRUCTIONS:" ~/.kinetic/prompts/execution_phase.md
grep "OUTPUT FORMAT:" ~/.kinetic/prompts/execution_phase.md
```

**4. Check Async Loading**
```typescript
// Add to PromptFactory.getPhasePrompt
console.log('Loading prompt from:', templatePath);
console.log('File exists:', fs.existsSync(templatePath));
```

---

## 🔄 State Machine Stuck

### Symptom
Workflow stuck in AWAITING_PLAN_APPROVAL or won't transition.

### Diagnostic Steps

**1. Check Current State**
```typescript
// In WorkflowOrchestrator
console.log('Current state:', this.actor.getSnapshot().value);
```

**2. Check Event Handling**
```typescript
// Add to handleEvent method
console.log('Sending event:', event.type);
this.actor.send(event);
console.log('New state:', this.actor.getSnapshot().value);
```

**3. Check Guards**

View `machines/workflowMachine.ts:50-120` for guard conditions:
```typescript
guards: {
    isInteractive: (context) => context.mode === 'interactive',
    hasApprovedPlan: (context) => context.planApproved === true
}
```

### Solutions

| Stuck State | Likely Cause | Solution |
|-------------|--------------|----------|
| AWAITING_PLAN_APPROVAL | User hasn't clicked approve | Send PLAN_APPROVED event |
| EXECUTION won't start | Plan not marked approved | Check `context.planApproved` |
| VERIFICATION repeats | Tests never pass | Send TESTS_PASS event manually |

---

## 🧪 Test Failures

### Router Tests Failing

```bash
npm run test:router
```

**Common Issues**:

1. **API Key Not Set**
```bash
# Tests that call LLM need keys
export OPENAI_API_KEY=sk-...
export XAI_API_KEY=xai-...
```

2. **Config File Missing**
```bash
# Create test config
mkdir -p ~/.kinetic/config
cp test/fixtures/test_config.yaml ~/.kinetic/config/kinetic_config.yaml
```

3. **Mock Not Working**
```typescript
// Check test/setup.js loads vscode mock
console.log('VSCode mock loaded:', typeof vscode);
```

### Integration Tests Failing

```bash
npm run test:integration
```

**Common Issues**:

1. **Workspace Not Open**

Integration tests need workspace:
```typescript
// test/suite/index.ts
if (!vscode.workspace.workspaceFolders) {
    throw new Error('Tests require workspace folder');
}
```

2. **Extension Not Activated**

Check activation event:
```json
// package.json
"activationEvents": ["onView:agentWorkflowView"]
```

---

## 🐛 General Debugging Strategies

### Enable Debug Logging

```typescript
// extension.ts activation
logger.setLevel(LogLevel.DEBUG);
```

### Breakpoint Locations

| Component | File | Line | Purpose |
|-----------|------|------|---------|
| Intent classification | Router.ts | 93 | See classification result |
| Tool execution | AgentWorkflowViewProvider.ts | 340 | Inspect tool calls |
| Permission check | ToolPolicyService.ts | 45 | See permission logic |
| Adapter selection | ModelFactory.ts | 15 | Verify correct adapter |
| State transitions | WorkflowOrchestrator.ts | 85 | Track state changes |

### Log Analysis

**View Extension Logs**:
```
1. View → Output
2. Select "Agent Workflow"
3. Scroll to recent errors
```

**Common Log Patterns**:
```
✅ Good: "[Router] Classified: DIRECT with confidence 0.95"
❌ Bad: "[Router] Classification failed: APICallError"

✅ Good: "[ToolPolicyService] ALLOW: createFile (safe operation)"
❌ Bad: "[ToolPolicyService] DENY: Catastrophic command detected"

✅ Good: "[Adapter] Using NativeAdapter for gpt-4-turbo"
❌ Bad: "[Adapter] Using PromptAdapter for gpt-4-turbo" (wrong adapter)
```

---

## 🔍 Advanced Diagnostics

### Trace Full Request/Response

Add to `VercelAIClient.ts`:
```typescript
// Before API call
console.log('=== REQUEST ===');
console.log('Model:', modelConfig.model);
console.log('Messages:', JSON.stringify(messages, null, 2));
console.log('Tools:', tools ? Object.keys(tools) : 'none');

// After API call
console.log('=== RESPONSE ===');
console.log('Finish reason:', result.finishReason);
console.log('Tool calls:', result.toolCalls?.length || 0);
console.log('Content:', result.text?.substring(0, 200));
```

### Monitor Token Usage

```typescript
// Add to Logger.ts or VercelAIClient.ts
if (result.usage) {
    const cost = calculateCost(result.usage, modelName);
    console.log(`Tokens: ${result.usage.totalTokens}, Est. cost: $${cost}`);
}
```

### Performance Profiling

```typescript
// Measure prompt assembly time
const startPrep = Date.now();
const prompt = await promptFactory.getPhasePrompt(phase);
const prepTime = Date.now() - startPrep;
console.log(`Prompt prep: ${prepTime}ms`);

// Measure LLM call time
const startLLM = Date.now();
const response = await client.generate(messages, tools);
const llmTime = Date.now() - startLLM;
console.log(`LLM call: ${llmTime}ms`);
```

---

## 📞 Getting Help

### Before Reporting Issue

1. ✅ Check this troubleshooting guide
2. ✅ Review execution-flow.md for code path
3. ✅ Enable debug logging and capture logs
4. ✅ Try minimal reproduction case

### What to Include in Issue Report

```markdown
**Environment:**
- OS: macOS 14.2 / Windows 11 / Ubuntu 22.04
- VS Code version: 1.85.0
- Extension version: 0.1.0

**Configuration:**
- Router model: grok-beta
- Actor model: codestral-latest
- Strategy: prompt

**Steps to Reproduce:**
1. Open workspace with file.py
2. Send message: "Create hello.txt"
3. Extension shows error...

**Expected:** File should be created
**Actual:** Error: "Tool execution failed..."

**Logs:**
```
[Router] Classified: DIRECT
[ToolPolicyService] ALLOW: createFile
[PromptAdapter] WARNING: Patch did not match!
```

---

## 🛠️ Self-Help Checklist

Before asking for help:

- [ ] Checked Output panel for errors
- [ ] Verified config file syntax is valid YAML
- [ ] Confirmed API keys are set and valid
- [ ] Tested with minimal example
- [ ] Checked adapter matches model capability
- [ ] Reviewed recent code changes
- [ ] Reloaded VS Code window
- [ ] Tried with different model
- [ ] Checked firewall/proxy settings
- [ ] Reviewed execution-flow.md for code path

---

## 🔗 Related Documentation

- **Execution Flow**: `execution-flow.md` - Trace code paths
- **Configuration**: `configuration-guide.md` - Config syntax
- **Adapters**: `adapter-guide.md` - Adapter issues
- **Reference Tables**: `reference-tables.md` - Expected values

---

**Last Updated**: 2025-12-30  
**Covers**: Common issues, diagnostics, solutions  
**Support**: Check logs first, then documentation

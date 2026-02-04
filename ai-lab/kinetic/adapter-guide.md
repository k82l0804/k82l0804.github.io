# Model Adapter System Guide

## Overview

The Agent Workflow uses a **Model Adapter pattern** to normalize different LLM APIs into a common interface. This allows seamless switching between models with native function calling (GPT-4, Grok) and those requiring prompt engineering (Codestral, Llama).

---

## 🎯 Why Adapters?

### The Problem

Different LLM providers handle tool calling differently:

| Provider | Tool Calling | Format |
|----------|--------------|--------|
| OpenAI (GPT-4) | ✅ Native | `response.choices[0].message.tool_calls` |
| xAI (Grok) | ✅ Native (mostly) | Sometimes JSON-in-content |
| vLLM (Codestral) | ❌ None | Must output JSON text |
| Llama.cpp | ❌ None | Must output JSON text |

### The Solution

**Adapter Pattern**: Single interface, multiple implementations

```
AgentWorkflowViewProvider
  ↓
VercelAIClient
  ↓
ModelFactory.createAdapter(strategy)
  ├─ strategy="native" → NativeAdapter
  └─ strategy="prompt" → PromptAdapter
```

---

## 🏗️ Architecture

### Interface: ModelAdapter

**File**: `adapters/ModelAdapter.ts`

```typescript
export interface ModelAdapter {
    /**
     * Get the model name
     */
    getModelName(): string;
    
    /**
     * Generate a response with optional tool calling
     */
    generate(
        messages: Message[], 
        tools?: ToolDefinition[]
    ): Promise<ModelResponse>;
}
```

### Normalized Response

All adapters return the same structure:

```typescript
export interface ModelResponse {
    content?: string;              // Text response
    finish_reason: string;         // 'stop' or 'tool_calls'
    tool_calls?: ToolCall[];       // Normalized tool calls
    usage?: {
        prompt_tokens: number;
        completion_tokens: number;
        total_tokens: number;
    };
}

export interface ToolCall {
    id: string;
    name: string;
    arguments: Record<string, any>;  // Parsed JSON
}
```

---

## 🔧 Adapter 1: NativeAdapter

**File**: `adapters/NativeAdapter.ts`

### Purpose

For models with **native OpenAI-style function calling**:
- OpenAI (GPT-4, GPT-4o)
- xAI (Grok-2, Grok-beta)
- Anthropic Claude (via OpenAI compatibility SDK)

### How It Works

```typescript
async generate(messages: Message[], tools?: ToolDefinition[]) {
    // 1. Pass tools directly to API
    const params = {
        model: this.modelName,
        messages: messages,
        tools: tools,           // Native tool definitions
        tool_choice: 'auto'
    };
    
    // 2. Call API
    const response = await this.client.chat.completions.create(params);
    
    // 3. Validate response
    if (!response.choices || response.choices.length === 0) {
        throw new Error('LLM returned empty choices array');
    }
    
    const choice = response.choices[0];
    
    // 4. Normalize response
    const result: ModelResponse = {
        content: choice.message.content || undefined,
        finish_reason: choice.finish_reason === 'tool_calls' ? 'tool_calls' : 'stop',
        usage: response.usage
    };
    
    // 5. Extract tool calls if present
    if (choice.message.tool_calls && choice.message.tool_calls.length > 0) {
        result.tool_calls = choice.message.tool_calls.map(tc => ({
            id: tc.id,
            name: tc.function.name,
            arguments: JSON.parse(tc.function.arguments)  // Parse JSON string
        }));
        result.finish_reason = 'tool_calls';
    }
    
    return result;
}
```

### Grok Edge Case Handling

**Problem**: Grok sometimes embeds tool calls as JSON in content field instead of structured `tool_calls`.

**Example**:
```json
{
  "content": "{\"tool_calls\": [{\"name\": \"createFile\", ...}]}"
}
```

**Solution** (lines 59-112):

```typescript
// Fallback: Check for JSON-in-Content (Grok edge case)
if (result.content && result.content.trim().startsWith('{')) {
    try {
        const parsed = JSON.parse(result.content.trim());
        if (parsed.tool_calls && Array.isArray(parsed.tool_calls)) {
            // Extract tool calls from JSON
            result.tool_calls = parsed.tool_calls.map(tc => ({
                id: tc.id || generateRandomId(),
                name: tc.function?.name || tc.name,
                arguments: parseToolArguments(tc)
            }));
            result.finish_reason = 'tool_calls';
            result.content = undefined;  // Hide JSON from user
        }
    } catch (e) {
        // Not JSON, leave as content
    }
}
```

### When to Use

✅ Use NativeAdapter when:
- Model supports OpenAI-compatible function calling
- Config has `strategy: "native"`
- Examples: GPT-4, Grok, Claude

---

## 🎨 Adapter 2: PromptAdapter

**File**: `adapters/PromptAdapter.ts`

### Purpose

For models **without native function calling**:
- Codestral (Mistral's code model)
- Llama 3.x (via vLLM)
- Local open-source models

### How It Works

#### Step 1: Inject TypeScript Tool Definitions

```typescript
private generateToolDefinitions(tools?: ToolDefinition[]): string {
    return `
// --- TYPESCRIPT DEFINITIONS ---
type ToolName = "createFile" | "readFile" | "runCommand" | ...;

interface AgentAction {
    /** Your reasoning about what action to take */
    thought: string;
    /** The tool to execute */
    tool_name: ToolName;
    /** Arguments for the tool */
    tool_args: Record<string, any>;
}

// --- AVAILABLE TOOLS ---
/** Create a new file */
function createFile(args: { path: string; content: string }): void;

/** Read file contents */
function readFile(args: { path: string }): string;
...

// --- OUTPUT FORMAT ---
// Respond with a single valid JSON object implementing AgentAction.
// Example: {"thought": "Creating hello.py", "tool_name": "createFile", "tool_args": {...}}
`;
}
```

**Why TypeScript?**
- Codestral is a code model, understands TS better than English
- Type signatures are unambiguous
- Natural for function calling representation

#### Step 2: Aggressive Prompt Patching

**Problem**: Base prompts may contain conflicting instructions:
- "OUTPUT FORMAT: Respond conversationally" (blocks JSON)
- "Do not output JSON" (contradicts tool needs)

**Solution**: Regex replacement to override conflicting sections

```typescript
private enhanceMessages(messages: Message[], tools?: ToolDefinition[]): Message[] {
    const systemIndex = enhanced.findIndex(m => m.role === 'system');
    let content = enhanced[systemIndex].content || '';
    
    // PATCH 1: Replace OUTPUT FORMAT section
    const outputFormatRegex = /OUTPUT FORMAT:[\s\S]*?(?=(FUNCTION CALLING:|PROHIBITIONS:|$))/;
    if (content.match(outputFormatRegex)) {
        content = content.replace(outputFormatRegex, `OUTPUT FORMAT:
You MUST respond with a single valid JSON object matching the AgentAction schema.
Do NOT output conversational text.
`);
    }
    
    // PATCH 2: Replace FUNCTION CALLING section
    const nativeInstructionsRegex = /FUNCTION CALLING:[\s\S]*?(?=PROHIBITIONS:|$)/;
    if (content.match(nativeInstructionsRegex)) {
        content = content.replace(nativeInstructionsRegex, `
FUNCTION CALLING (JSON MODE):
You serve a JSON-based runtime. You DO NOT have native tool_calls.
Instead, you MUST output a single valid JSON object.
`);
    }
    
    // PATCH 3: Fix specific contradictory phrases
    content = content.replace(/ANSWER DIRECTLY in the chat/g, 'Call `notify_user`');
    content = content.replace(/Do not output tool calls as JSON/g, 'Output valid JSON');
    
    // Append tool definitions
    enhanced[systemIndex].content = content + '\n\n' + toolDefs;
    
    return enhanced;
}
```

#### Step 3: Use vLLM `guided_json`

**Purpose**: Force model to output valid JSON matching schema

```typescript
const params = {
    model: this.modelName,
    messages: enhancedMessages,
    // vLLM specific: enforce JSON schema
    extra_body: {
        guided_json: AGENT_ACTION_SCHEMA
    }
};
```

**Schema**:
```typescript
const AGENT_ACTION_SCHEMA = {
    type: "object",
    properties: {
        thought: { type: "string" },
        tool_name: { type: "string", enum: ["createFile", "readFile", ...] },
        tool_args: { type: "object" }
    },
    required: ["thought", "tool_name", "tool_args"]
};
```

#### Step 4: Parse JSON Response

**Challenge**: Model may output JSON in various formats:
- Pure JSON: `{"thought": "...", ...}`
- Markdown code block: ` ```json\n{...}\n``` `
- Mixed text + JSON: `"Let me create the file... {"thought": ...}"`
- Array of tool calls: `[{...}, {...}]`

**Solution**: Multi-strategy parsing (lines 183-303)

```typescript
// Try 1: Strip markdown code blocks
const codeBlockRegex = /```(?:\w+)?\s*([\s\S]*?)\s*```/g;
while ((match = codeBlockRegex.exec(content)) !== null) {
    try {
        const parsed = JSON.parse(match[1].trim());
        // Extract tool call from parsed JSON
        extractToolCall(parsed);
    } catch (e) {}
}

// Try 2: Scan for raw JSON arrays
const arrayRegex = /\[\s*\{[\s\S]*\}\s*\]/g;
while ((arrayMatch = arrayRegex.exec(content)) !== null) {
    try {
        const parsed = JSON.parse(arrayMatch[0]);
        // Process array of tool calls
        extractToolCalls(parsed);
    } catch (e) {}
}

// Try 3: Scan for individual JSON objects
let braceCount = 0;
for (let i = 0; i < content.length; i++) {
    if (content[i] === '{') braceCount++;
    else if (content[i] === '}') {
        braceCount--;
        if (braceCount === 0) {
            // Found complete JSON object
            tryParse(content.substring(startIndex, i + 1));
        }
    }
}
```

### When to Use

✅ Use PromptAdapter when:
- Model does NOT support native function calling
- Config has `strategy: "prompt"`
- Running local vLLM/llama.cpp models
- Examples: Codestral, Llama 3, Qwen

---

## 🔀 Adapter Selection

### ModelFactory

**File**: `adapters/ModelFactory.ts`

```typescript
export class ModelFactory {
    static createAdapter(
        client: OpenAI, 
        modelName: string, 
        strategy: 'native' | 'prompt'
    ): ModelAdapter {
        if (strategy === 'native') {
            return new NativeAdapter(client, modelName);
        } else {
            return new PromptAdapter(client, modelName, AGENT_ACTION_SCHEMA);
        }
    }
}
```

### Configuration-Driven

**File**: `kinetic_config.yaml`

```yaml
models:
  - name: "gpt-4-turbo"
    provider: "openai"
    strategy: "native"  # ← Uses NativeAdapter
    
  - name: "codestral-latest"
    provider: "vllm"
    strategy: "prompt"  # ← Uses PromptAdapter
```

### Usage

**File**: `VercelAIClient.ts`

```typescript
const modelConfig = this.router.getModelConfig(modelName);
const client = createOpenAI({ /* ... */ });

// Factory creates appropriate adapter
const adapter = ModelFactory.createAdapter(
    client, 
    modelConfig.model, 
    modelConfig.strategy  // 'native' or 'prompt'
);

// Uniform interface regardless of adapter
const response = await adapter.generate(messages, tools);
```

---

## 📊 Comparison Table

| Feature | NativeAdapter | PromptAdapter |
|---------|---------------|---------------|
| **Tool Format** | OpenAI native | TypeScript interfaces |
| **Prompt Modification** | None | Aggressive patching |
| **Output Enforcement** | Model's own | vLLM `guided_json` |
| **Parsing Complexity** | Low (structured) | High (multiple fallbacks) |
| **Reliability** | Very high | Medium (model-dependent) |
| **Token Overhead** | Low | +300-500 tokens (tool defs) |
| **Latency** | Minimal | +50-100ms (parsing) |
| **Suitable For** | GPT, Grok, Claude | Codestral, Llama, local |
| **Failure Mode** | Rare | Model outputs text instead of JSON |

---

## 🐛 Common Issues & Solutions

### Issue 1: "Model outputs explanation instead of tool calls"

**Symptom**: Model says "I will create the file..." instead of calling `createFile`

**Cause**: Using NativeAdapter with non-native model

**Solution**: Change to PromptAdapter
```yaml
strategy: "prompt"  # Not "native"
```

### Issue 2: "JSON parsing fails"

**Symptom**: Error: "Unexpected token in JSON"

**Cause**: Model outputs mixed text + JSON

**Solution**: PromptAdapter uses fallback parsers. If still fails:
1. Increase prompt clarity
2. Use stricter guided_json schema
3. Check if model supports structured output

### Issue 3: "Grok returns JSON in content"

**Symptom**: `tool_calls` is empty, but content has JSON

**Cause**: Grok edge case (sometimes embeds in content)

**Solution**: NativeAdapter already handles this (lines 59-112)

### Issue 4: "Codestral ignores tools"

**Symptom**: Model responds conversationally, doesn't use tools

**Cause**: Prompt patching failed (regex didn't match)

**Solution**: Check prompt structure matches expected format
- Must have `OUTPUT FORMAT:` section
- Must have `FUNCTION CALLING:` section

---

## 🎯 Best Practices

### 1. Match Strategy to Model Capability

❌ **Wrong**: `gpt-4-turbo` with `strategy: "prompt"`  
✅ **Right**: `gpt-4-turbo` with `strategy: "native"`

### 2. Test Both Adapters

```bash
# Test native
curl -X POST http://localhost:8000/chat \
  -d '{"model": "gpt-4-turbo", "message": "Create hello.txt"}'

# Test prompt
curl -X POST http://localhost:8000/chat \
  -d '{"model": "codestral-latest", "message": "Create hello.txt"}'
```

### 3. Monitor Adapter Performance

Add logging:
```typescript
console.log(`[Adapter] Using ${adapter.constructor.name} for ${modelName}`);
console.log(`[Adapter] Response parsed in ${parseTime}ms`);
```

### 4. Validate Tool Call Format

```typescript
// In adapter, before returning
if (result.tool_calls) {
    for (const call of result.tool_calls) {
        if (!call.name || !call.arguments) {
            console.error('Invalid tool call format:', call);
        }
    }
}
```

---

## 🔧 Extending Adapters

### Adding a New Adapter

Example: Anthropic Claude with native support

```typescript
// adapters/AnthropicAdapter.ts
export class AnthropicAdapter implements ModelAdapter {
    private client: Anthropic;
    private modelName: string;
    
    async generate(messages: Message[], tools?: ToolDefinition[]) {
        // Anthropic-specific API call
        const response = await this.client.messages.create({
            model: this.modelName,
            messages: this.convertMessages(messages),
            tools: this.convertTools(tools)
        });
        
        // Normalize to ModelResponse
        return this.normalizeResponse(response);
    }
    
    private convertMessages(messages: Message[]): any {
        // Convert from OpenAI format to Anthropic format
    }
}
```

Then register in `ModelFactory`:
```typescript
if (strategy === 'anthropic') {
    return new AnthropicAdapter(client, modelName);
}
```

---

## 🔗 Related Documentation

- **Configuration Guide**: `configuration-guide.md` - Setting `strategy` field
- **Prompt System**: `prompt-system-guide.md` - How prompts are patched
- **Reference Tables**: `reference-tables.md` - Model → Strategy mapping

---

**Last Updated**: 2025-12-30  
**Adapters**: NativeAdapter, PromptAdapter  
**Pattern**: Strategy Pattern (Gang of Four)

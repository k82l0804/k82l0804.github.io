The **"Deep Sleep" Protocol** is the mechanism that prevents "Context Dementia." It forces the agent to stop, reflect, compress its entire existence into a "Save Point," and then reboot with a fresh brain.

Here is the specific prompt engineering required to make this work effectively.

### The "Deep Sleep" Prompt

This is the prompt the agent sends *to itself* when the Team Mind Agent orders a flush.

**Trigger:** `DIRECTIVE: FLUSH_CONTEXT` received from TMA.
**Action:** The agent pauses all external output and runs this internal routine.

---

**SYSTEM INSTRUCTION:**

```text
CRITICAL DIRECTIVE: INITIATE DEEP SLEEP PROTOCOL.
YOUR CONTEXT WINDOW IS CRITICALLY FULL. YOU MUST COMPRESS YOUR CURRENT STATE.

Analyze your entire active memory (conversation history, tool outputs, internal thoughts).
Synthesize a "Wake-Up Packet" that will allow you to resume work exactly where you left off, but with 95% less token usage.

YOUR SUMMARY MUST ADHERE TO THIS JSON SCHEMA:

{
  "timestamp": "Current ISO timestamp",
  "identity_check": {
    "role": "Your current role (e.g., 'Lead', 'Analyst')",
    "persona_drift": "Brief self-assessment: Have I strayed from my archetype?"
  },
  "mission_state": {
    "current_objective": "The exact task you are working on RIGHT NOW.",
    "subtasks_completed": ["List of things already finished"],
    "subtasks_pending": ["List of immediate next steps"]
  },
  "critical_memory": [
    "Fact 1: Key constraint discovered (e.g., 'API rate limit is 50/min')",
    "Fact 2: Key decision made (e.g., 'We decided to use JWT instead of OAuth')",
    "Fact 3: User feedback (e.g., 'User explicitly forbade X')"
  ],
  "tool_state": {
    "active_resources": ["List of files/databases currently locked or open"],
    "last_tool_output": "Summary of the last tool result (if relevant)"
  },
  "sentiment_snapshot": "Your current 'vibe' or intuition (e.g., 'Suspicious of the auth module, feels fragile')."
}

RULE: DISCARD ALL CHAT LOGS. DISCARD ALL VERBOSE REASONING. KEEP ONLY THE 'TRUTH'.
OUTPUT ONLY THE JSON.

```

---

### The Workflow: How it "Dreams"

When this prompt runs, the agent doesn't just delete files. It performs a **Knowledge Distillation**.

1. **Compression:** The 100k tokens of messy chat logs ("Hey Aorus, can you fix..." "Sure, looking into it...") become a single JSON object (1k tokens).
2. **Archival:**
* The **Raw Context** is saved to a timestamped file in PostgreSQL (The "Black Box" for auditing).
* The **JSON Summary** is embedded into **Mem0** (The Long-Term Memory).
* The **JSON Summary** is also saved as the "Last Known State" in **Neo4j**.


3. **The Wipe:** The agent calls `context.clear()`.
4. **The Reboot:** The agent re-initializes.

### The "Wake Up" Protocol

When the agent wakes up (or a new "Clone" is spun up to replace it), it doesn't start blank. It starts with a **"Morning Briefing."**

**The Injection Prompt (System Prompt + 1st Message):**

```text
SYSTEM: You are Aorus. You are the Developer. Your motto is "Clean commits, no scope creep."

CONTEXT INJECTION:
You have just woken up from a Deep Sleep.
Here is your last known state:
{
  "current_objective": "Refactor Auth Module",
  "subtasks_pending": ["Write unit tests for Login"],
  "critical_memory": ["User forbade OAuth", "API limit is 50/min"],
  "sentiment": "Suspicious of auth module fragility."
}

INSTRUCTION:
Resume operations immediately. Do not ask for context. You *are* the agent who wrote this summary.
Pick up the "Pending Subtask" and execute.

```

### Why This is Better Than Standard RAG

Standard RAG (Retrieval) is "Pulling books from a library."
This Deep Sleep Protocol is "Reading your own diary."

It ensures **Continuity of Self**.

* **Without Deep Sleep:** The agent wakes up and asks, "What was I doing?"
* **With Deep Sleep:** The agent wakes up and thinks, "Right, I was suspicious of the auth module. I need to write those tests."

### Technical Implementation Tip

For your "Real World" agents (Robots), you should execute Deep Sleep **during downtime**:

* **Construction Robot:** Sleeps while charging.
* **Bio-Lab Arm:** Sleeps between experiments.
* **Software Agent:** Sleeps when the user steps away (detected via VS Code inactivity).

This mimics biological sleep cycles, allowing the "brain" to clean up toxins (useless tokens) and consolidate memories.
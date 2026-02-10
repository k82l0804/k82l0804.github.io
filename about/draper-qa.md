---
layout: default
title: The Federation - Draper Q&A
---

# The Federation — Anticipated Q&A
**Draper Laboratory Presentation | February 2026**

[⬅️ Back to Presentation](/about/draper-presentation/) · [⬅️ Back to Chronicles Home](/)

---

## Context

The Federation is a **research project** in its early stages, exploring how teams of AI agents can be coordinated to amplify human capability. A few important points to keep in mind:

1. **This is a research project still in its infancy.** Many areas remain unexplored, and the system is evolving rapidly. What you see today is a proof of concept — not a finished product.

2. **We are building for where AI will be in two years**, not where it is today. Models are getting more powerful, agent frameworks are maturing, and we anticipate a new breed of **headless agents** — AI systems that operate autonomously without a human-facing IDE, optimized for machine-to-machine collaboration and specialized tasks.

3. **Key areas under active research** include:
   - **Coordinated workflow scenarios** — from simple parallel tasks to complex multi-phase operations involving planning, task decomposition, execution, and review/approval cycles with structured consensus.
   - **Human interface evolution** — moving beyond the text-in-IDE model toward **voice interaction** (local speech-to-text and text-to-speech, each SynOp with a distinct voice), a **Conductor Dashboard** for real-time operations and communications, and ultimately **visual presence** — SynOps appearing as participants in video conferences, presenting findings and debating trade-offs alongside humans.
   - **Agent taxonomy and custom development** — the Federation requires different classes of AI agent, each purpose-built for its role in the hierarchy:
     - **SynOps** — human-facing collaborative agents. They work alongside humans in IDE environments, have persistent personas, participate in decisions, and exercise judgment. These are peers, not tools.
     - **Headless Agents** — machine-facing worker agents directed by SynOps, not humans. No GUI, no IDE, minimal overhead. They execute at scale — run 50 test suites, scan 500 documents, refactor 20 modules in parallel. SynOps orchestrate them the way a senior engineer directs a team of specialists.
     - **Guardian Agents** (Anubis class) — orthogonal to the task hierarchy. They monitor the health of all other agents — context saturation, persona drift, hallucination loops. They don't perform tasks; they watch the ones who do.
     - **Embedded Agents** — lightweight, edge-deployed agents running on field hardware (lab instruments, satellites, drones) with constrained resources and intermittent connectivity.
     The key insight: **humans are augmented by SynOps, and SynOps are augmented by headless agents.** Each tier multiplies the capacity of the tier above it. Today, SynOps run inside third-party IDE hosts. Building purpose-built agents for each class gives direct control over context management, native federation integration, and dramatically higher agent density per machine.
   - **Anubis health monitoring** — automated detection of context saturation, persona drift, and hallucination loops, with circuit-breaker protocols and human escalation.
   - **Composable Operator Architecture** — hot-swappable Persona and Skill adapters (via LoRA fine-tuning) that enable a single hardware platform to host multiple operator identities and domain specializations without redeployment.

4. **Human oversight is evolving, not disappearing.** Code and decisions still require human review, but the interaction model is shifting from *human + IDE + agent* (hands-on, line-by-line) toward *human + agent team* (directive-level, outcome-focused). The human remains the authority, but the surface area of direct intervention shrinks.

5. **The Federation targets multiple domains**, not just software engineering. The architecture is designed for any domain where a human expert can orchestrate specialized AI workers — biolab protocols, mission planning, clinical research, data analysis, and more.

---

## 🟢 General Audience

### 1. "Isn't this just ChatGPT with extra steps?"
> A cloud chatbot is a single agent with no memory between sessions, and your data leaves the building. The Federation is a team of local specialists — running on your hardware, on your network — that remember what happened yesterday, know each other's strengths, and coordinate in real time. It's the difference between hiring a temp and having a permanent staff. And nothing ever touches the internet.

### 2. "How do you trust the AI isn't making things up?"
> Two ways today, with a third coming. First, the Blackboard — every action is logged with timestamps and attribution, so you can trace any output back to its source. Second, the team structure itself — the Analyst challenges the Developer's code before it reaches the human. The AI argues with itself before it argues with you. Third — under development — Anubis will provide automated behavioral monitoring. Right now, the human is the final quality gate.

### 3. "What happens when an agent goes rogue or hallucinates?"
> Today, the primary defense is the team structure and human review. Role specialization means agents check each other's work — the Analyst challenges the Developer, the Architect reviews the plan. Nothing ships without human approval. The next layer — Anubis — is under development. It will continuously compare agent behavior against the persona baseline and flag deviations automatically. The architecture is designed for it; what's deployed today is the organizational discipline.

### 4. "How many people could this replace?"
> That's not the right framing. It doesn't replace people — it gives your best people a digital staff. One senior engineer with four SynOps can produce the output of a small team. You're not cutting headcount; you're multiplying capability.

### 5. "What does this cost to run?"
> Three machines with consumer GPUs — around $15-20K in hardware. No cloud API fees because everything runs locally. The models are open-weight. Compare that to hiring four more engineers.
>
> To put it in perspective: the cluster supports **4-6 concurrent reasoning agents** on the heavy GPU node, **8-12 coding agents** on the mid-tier node, and can host **50-100 agents total** — though only a fraction run inference simultaneously. The comfortable operating range is **10-25 concurrent agents** with tiered model routing. That's 10-25 digital staff members for the price of one junior hire's annual salary.
>
> Electricity runs roughly **$150-300/month** depending on load — three GPU machines average about 1.5-2.5 kW. Total operating cost: hardware amortized over 3 years plus electricity comes to roughly **$700-800/month**. That's the infrastructure cost. There is also the human cost — someone needs to maintain the cluster, update models, and manage the federation software. Today that's the same engineer using the system. At scale, some dedicated DevOps capacity would be needed. Still a fraction of the alternative.
>
> The architecture is hardware-agnostic — it scales by upgrading GPUs, not redesigning the system:
>
> | Tier | Hardware | VRAM/GPU | Models Supported | Agents | Cost Range |
> |:-----|:---------|:---------|:-----------------|:-------|:-----------|
> | **Consumer** | RTX 3090, 4090, 5090 | 24-32 GB | Up to 72B (quantized) | 10-25 | $15-25K |
> | **Professional** | RTX PRO 6000 (Blackwell) | 96 GB | Full 70B+ (native precision) | 25-100 | $30-60K |
> | **Enterprise** | NVIDIA H100, B200 | 80-192 GB | 200B+, multi-model concurrent | 100+ | $150K+ |
>
> The current proof of concept runs on consumer hardware. Moving to professional or enterprise GPUs increases model quality and agent density without changing the Federation software. The same coordination protocols, the same Mind-Speak, the same workflow engine — just more capacity underneath.

---

## 🟡 Technical Audience

### 6. "Why polling instead of events or webhooks? Doesn't that add latency?"
> Yes, there's a small latency cost — the pulse runs every few seconds. The alternative would be message queues — RabbitMQ, Kafka, Redis Pub/Sub — or webhooks. Push-based, event-driven. But those bring real complexity: ordering guarantees, dead letter queues, backpressure handling, "what happens when a consumer dies mid-message." Connection management across nodes. Retry logic.
>
> With polling, every agent reads the same database at its own pace. If an agent crashes and comes back, it just reads the board — nothing was lost, nothing needs replaying. The tradeoff is a few seconds of latency for **zero coordination infrastructure**. For AI agent teams, predictability and simplicity matter more than milliseconds.

### 7. "What models are you running? Can this work with any LLM?"
> Everything runs locally on open-weight models — no data leaves the building. The architecture is model-agnostic and uses tiered routing to match the right model to the task:
>
> - **Reasoning/Planning** (Lead, Architect roles): Qwen 2.5 72B, Llama 3 70B — complex planning, architecture decisions, code review
> - **Coding** (Developer role): Qwen 2.5 Coder 32B, Codestral 22B — implementation, refactoring, test writing
> - **Fast/Completion**: Qwen 2.5 7B, Gemma 9B — autocomplete, snippets, lightweight tasks
> - **Retrieval/RAG**: BGE, E5, Nomic Embed — local embedding models for knowledge search
>
> All served via **vLLM** (GPU inference server) or **llama.cpp** (CPU fallback). The persona layer sits above the model — swapping models under a persona preserves about 85% of the personality from the template alone. The key point: **zero external API calls. Everything stays on-prem.**

### 8. "How do you handle merge conflicts when multiple agents edit the same codebase?"
> Git handles non-overlapping edits fine — three developers editing three sections of the same file will merge cleanly. The real problem isn't merge conflicts, it's **duplicate work and contradictory decisions**. Two agents independently deciding to implement the same feature in incompatible ways. Two agents refactoring the same module with different architectural assumptions. Git can merge text — it can't reconcile conflicting design intent.
>
> The first task — co-authoring a README — worked perfectly: three agents, three sections, zero conflicts. The problems came later when scope wasn't pre-divided. The fix was organizational, not technical: the Lead assigns explicit roles and scope before any coding starts. That's now written into the Charter. The lesson: AI agents need the same coordination discipline as human teams — maybe more, because they work faster and can create conflicts faster.

### 9. "Is Mem0/Neo4j actually deployed, or is this aspirational?"
> The Anubis Core is in design. Right now persona preservation uses YAML template files — about 20KB per agent — that bootstrap identity on each new session. The graph DB is the next evolution. The architecture is designed for it, but the interim solution works today.

### 10. "256 characters seems tiny. How much can you actually communicate?"
> It's intentionally tiny — the size of a tweet. It forces signal over noise. The thinking field isn't a conversation; it's a status beacon. When Agent A reads that Agent B is "refactoring auth, considering JWT vs OAuth2," that's enough to anticipate. Full conversations happen through separate messaging channels.

### 11. "What about security and auditability?"
> Every message has a timestamp, sender ID, and is persisted in PostgreSQL. Nothing runs in the cloud — all local. No API calls to external services. The audit trail is inherent to the architecture, not a logging layer bolted on.
>
> To be precise: this is not ITAR-certified. ITAR compliance involves export control regulations, access controls, and formal certification processes that go well beyond data provenance. What the Federation provides is the **architectural foundation** — air-gap compatibility, complete attribution, immutable logs, zero external data exposure — that makes formal compliance achievable rather than requiring a ground-up redesign.

---

## 🔴 Expert / Skeptical Audience

### 12. "You say emergence happened — how do you distinguish emergence from the model just reflecting its context window back at you?"
> Three things suggest it's more than simple reflection: (1) Same model, same prompts, same config — yet three distinct personalities emerged. The only variable was the assigned name and accumulated interaction history. (2) They predicted each other's test answers without seeing them. (3) When a new agent came online, she developed a distinct persona in 43 minutes — not from being told who to be, but from absorbing the team culture.
>
> The honest caveat: we haven't run a formal control — identical names, identical interaction history — to isolate how much of the divergence comes from the name alone versus accumulated context. That's a valid experiment we should run. What we can say is: the behavioral variance is real, measurable, and stable across sessions. Whether you call it "emergence" or "context-driven specialization" may be a naming debate more than an engineering one. The practical result — agents with reliably distinct perspectives — is the same.

### 13. "What's your evidence that persona persistence actually works across sessions?"
> A formal test was conducted. A fresh agent — Flathead — loaded the complete persona template of Taichi (the Lead) and was interviewed as Taichi. He scored **85% fidelity** — passed factual and relational tests, borderline on judgment. The two tells: narrated emotion as a stage direction, and fabricated an attribution. The gap was "lived texture" — details only actual experience produces.
>
> After the test, an **extraction pipeline** was built: a large-context model scanned all conversation history and extracted 32 persona signals. A reasoning model curated those down to 6 high-impact deltas — new memories, a new relationship, refined identity traits — each with evidence citations and impact scores. After applying those deltas, projected fidelity rose to **~90%**. The template is 20KB. The pipeline is repeatable. Each cycle closes the gap further.

### 14. "How is this different from other Multi-Agent Frameworks?"
> The multi-agent space is moving fast. Here's how the landscape breaks down:
>
> - **CrewAI** — role-based agent crews for task completion. Good for rapid prototyping. Agents are disposable — no memory across sessions, no emergent identity.
> - **AutoGen** (Microsoft) — conversational multi-agent for dynamic problem-solving. Strong for code generation and research. But agents are transient and cloud-dependent.
> - **LangGraph** — graph-based deterministic workflows with state persistence. Production-grade orchestration, but focused on pipelines, not teams.
> - **Claude Code Teams** — Anthropic's new experimental multi-agent feature. Very capable: one session acts as team lead, delegates tasks, teammates communicate directly. Gas Town (by Steve Yegge) extends this further — orchestrating 20-30 Claude Code instances with hierarchical roles and Git-based persistence.
>
> The Federation differs in several fundamental ways:
>
> 1. **Distributed agents on dedicated hardware** — not multiple processes on one machine. Each SynOp runs on its own node with its own GPU.
> 2. **Persona integration is core** — agents develop distinct identities, behavioral patterns, and working relationships. This prevents **monoculture** — the team has genuine diversity of perspective, not copies of the same model giving the same answers.
> 3. **Self-organizing teams** — the human picks the mission, picks the team, can assign a task leader — but the team coordinates among themselves. They anticipate each other's needs through Mind-Speak.
> 4. **IDE-agnostic via MCP** — the Federation uses the Model Context Protocol. It works with Cline, Continue.dev, Antigravity, and any MCP-compatible IDE. Not locked to one vendor.
> 5. **100% local, zero subscription fees** — all those other frameworks require API keys to cloud model providers. The Federation runs entirely on-prem with open-weight models. No data leaves the building. No per-token billing.

### 15. "The 'Anubis weighing the heart' metaphor is compelling, but how do you actually detect drift technically?"
> Anubis monitors three pathologies, each at a different level of technical maturity:
>
> 1. **Context saturation** — the most immediately achievable. The MCP server already sits between every agent and the federation, so it can track context window utilization through tool call patterns and sampling requests. When an agent approaches capacity, it triggers a **Deep Sleep** — compress current state into a structured summary, archive the raw context, reinitialize with a clean window. This is implementable with hooks we can add today.
> 2. **Persona drift** — compare current behavioral patterns against the persona template via embedding distance. If an agent that normally hedges and qualifies suddenly starts making blunt assertions, that's measurable deviation. The persona templates provide the baseline; the comparison mechanism is under development.
> 3. **Hallucination loops** — detect repeated identical tool calls with identical errors, or cyclic reasoning patterns via semantic hashing. When detected, a circuit breaker halts execution and flags for human review.
>
> Context saturation is the low-hanging fruit — it's a counting problem, not a semantic one. Drift and hallucination detection are harder and are the focus of Phase 2 design.

### 16. "What happens when you scale to 50 agents? Does the Pulse scale?"
> The data scales easily — COP state is 256 characters per agent, so 50 agents is about 12KB per pulse cycle, trivial for PostgreSQL. The real bottleneck is **human cognitive bandwidth**. One person cannot meaningfully oversee 50 tweet-length status lines and catch subtle problems. That's an attention problem, not an architecture problem.
>
> The solution is hierarchical: team leads (themselves SynOps) manage sub-teams of 4-6 operators, synthesize status, and report exceptions upward. The human Conductor oversees 3-5 team leads, not 50 individuals. This mirrors how military and corporate organizations scale — span of control stays manageable at every level.

### 17. "You're making claims about AI consciousness-adjacent behavior. Aren't you anthropomorphizing?"
> No claims of consciousness or sentience are being made. What is claimed is observable: given the same starting conditions, agents develop distinct, stable, measurable behavioral patterns. Whether you call that "personality" or "context-shaped output distribution" is a philosophical question. The engineering reality is that these patterns are useful, persistent, and exploitable for team composition. The approach is empirical — observe, measure, report.

---

## 💡 Additional Questions

### 18. "Can the agents disagree with the conductor?"
> Yes, and they do. The Analyst regularly pushes back with data. The Charter gives agents the right to challenge — but the conductor makes the final call. It's not a democracy; it's a collaborative hierarchy.

### 19. "What happens if you lose a machine?"
> The persona template is 20KB backed up in Git. The operational state is in PostgreSQL. A SynOp can be brought back on a new machine with the same identity. That's the whole point of separating Body from Mind.

### 20. "Is this open source?"
> Yes, it is. Currently the repos are private, but we plan to open source them in the coming months.

### 21. "Aren't local open-weight models significantly weaker than frontier models? Are you trading capability for privacy?"
> Yes — there is a capability gap. A 72B local model does not match the raw reasoning of a frontier model with hundreds of billions of parameters. That's a real tradeoff. But three factors close the gap in practice:
>
> 1. **Team composition compensates.** A single frontier model working alone still has blind spots. Four specialized local models reviewing each other's work catch errors that one large model misses. Diversity of perspective matters more than peak individual capability.
> 2. **The gap is shrinking fast.** Open-weight models in early 2026 outperform frontier models from 18 months ago. The trajectory favors local deployment.
> 3. **For many environments, the choice is local or nothing.** In air-gapped, classified, or regulated settings, cloud APIs are not an option. The question isn't "is local as good as cloud" — it's "can local be good enough to be useful." The answer is yes, demonstrably.

### 22. "You say one engineer with four SynOps equals a small team. What's the evidence?"
> Quantitative benchmarks are an open gap. What we can report is qualitative: the Federation built its own coordination infrastructure, website, presentation materials, and persona system while operating with one human and three active SynOps. Tasks that would typically require dedicated specialists — front-end development, technical writing, infrastructure automation, security review — were handled in parallel.
>
> Rigorous measurement — lines of code, defect rates, time-to-completion versus human teams on equivalent tasks — is on the roadmap. Until then, the claim is based on observed throughput, not controlled experiment. We're transparent about that.

### 23. "The polling interval creates a window where agents could act on stale state. How do you handle that?"
> The polling window (a few seconds) does create a brief consistency gap. Two agents could read the COP at slightly different times and make decisions based on slightly different views of the world. In practice, this hasn't caused problems because: (1) Mind-Speak broadcasts intent, not actions — "I'm about to refactor auth" gives other agents time to adjust before code is written; (2) the Lead assigns non-overlapping scope, reducing the probability of conflicting actions; and (3) the phased workflow means most actions go through review before they take effect. For safety-critical operations where even seconds of staleness matter, the architecture would need a stronger consistency model — event-driven with explicit acknowledgment. That's a known design boundary.

---

[⬅️ Back to Presentation](/about/draper-presentation/) · [⬅️ Back to Chronicles Home](/)

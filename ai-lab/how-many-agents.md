---
layout: default
title: How Many Agents?
permalink: /ai-lab/how-many-agents/
---

# How Many Agents Can We Run?

*Exploring the capacity limits of the AI-Lab cluster for multi-agent workloads.*

---

## The Question

With 2× RTX 5090s (64GB total) and 2× RTX 3090s (48GB total), plus ample CPU/RAM, how many concurrent AI agents (Cline, Continue.dev, or Federation members) can the cluster realistically support?

The answer depends on **what the agents are doing**.

---

## Understanding Agent Workloads

Not all agent activity is equal:

| Workload Type | GPU Impact | Frequency | Duration |
|---------------|------------|-----------|----------|
| **Reasoning** (complex planning) | High (70B model) | Rare | 5-30 sec |
| **Coding** (implementation) | Medium (27B model) | Frequent | 2-10 sec |
| **Autocomplete** (inline suggestions) | Low (7B model) | Constant | <1 sec |
| **Embeddings** (RAG retrieval) | Low (encoder) | Frequent | <100ms |
| **Idle** (waiting for user) | None | 95% of time | — |

**Key insight:** Agents are idle most of the time. Even "active" agents spend 90%+ waiting for user input, reading responses, or thinking (CPU-bound).

---

## The Multi-Tier Model Architecture

Instead of one model for everything, we can specialize:

```
┌─────────────────────────────────────────────────────────────┐
│           Tier 1: Reasoning (taichi-5090)                   │
│           Qwen 2.5 72B / Llama 3.3 70B                     │
│           Complex planning, architecture, debugging         │
│           ~4-6 concurrent requests                          │
├─────────────────────────────────────────────────────────────┤
│           Tier 2: Coding (aorus-3090)                       │
│           Gemma 2 27B / DeepSeek-Coder 33B                 │
│           Implementation, refactoring, test generation     │
│           ~8-12 concurrent requests                         │
├─────────────────────────────────────────────────────────────┤
│           Tier 3: Autocomplete (both nodes)                 │
│           Gemma 2 9B / StarCoder 7B                        │
│           Inline completions, snippets                      │
│           ~20-40 concurrent requests                        │
├─────────────────────────────────────────────────────────────┤
│           Tier 4: Embeddings (aorus-3090)                   │
│           BGE-base / E5-small                              │
│           RAG indexing and retrieval                       │
│           ~1000+ chunks/sec                                 │
└─────────────────────────────────────────────────────────────┘
```

---

## Capacity Analysis

### GPU-Limited: Concurrent Inference

| Node | Model | VRAM Used | Max Concurrent |
|------|-------|-----------|----------------|
| **taichi-5090** | Qwen 72B (Q4) | ~40GB | 4-6 requests |
| **taichi-5090** | Gemma 27B (FP16) | ~54GB | 2 models, 8-10 each |
| **aorus-3090** | Gemma 27B (Q4) | ~16GB | 8-12 requests |
| **aorus-3090** | StarCoder 7B | ~8GB | 20-40 requests |

**vLLM handles queuing.** If 20 agents request inference simultaneously, only 6 run concurrently; others wait (~100-500ms queue time).

### CPU/RAM-Limited: Agent Hosting

| Resource | Capacity | Limit |
|----------|----------|-------|
| **CPU cores (48)** | 1 core/agent | ~48 agents |
| **RAM (256GB)** | 2GB/agent | ~100+ agents |
| **Practical** | Async efficiency | 50-80 agents |

---

## Realistic Scenarios

### Scenario 1: Single Developer Setup
*One person, three monitors, multiple IDE instances*

| Agent | Purpose | Model Tier |
|-------|---------|------------|
| Cline (main) | Primary coding assistant | Tier 1/2 |
| Continue.dev | RAG context | Tier 4 |
| Autocomplete | Inline suggestions | Tier 3 |
| Research agent | Background research | Tier 2 |

**Verdict:** ✅ Easily supported. Max 4 concurrent inferences.

---

### Scenario 2: Small Team (5 developers)
*Each developer has 2-3 active agents*

| Metric | Value |
|--------|-------|
| Total agents | 10-15 |
| Peak concurrent inferences | 5-8 |
| Average queue time | <500ms |

**Verdict:** ✅ Fully supported with minimal latency.

---

### Scenario 3: Federation at Scale (10 agents)
*All Federation members active simultaneously*

| Agent | Node | Role |
|-------|------|------|
| Taichi | taichi-5090 | Lead reasoning |
| Aorus | aorus-3090 | Development |
| Baby | baby | Analysis (CPU) |
| Qwen | taichi-5090 | Architecture |
| + 6 more | distributed | Various |

| Metric | Value |
|--------|-------|
| Total agents | 10 |
| Concurrent 72B requests | 2-3 |
| Concurrent 27B requests | 4-6 |
| Queue time (peak) | 1-3 sec |

**Verdict:** ✅ Supported. Occasional queue delays during burst activity.

---

### Scenario 4: Classroom/Workshop (20 users)
*Training session with 20 concurrent users*

| Metric | Value |
|--------|-------|
| Users | 20 |
| Agents | 20-40 (1-2 per user) |
| Peak concurrent inferences | 10-15 |
| Model tier | Tier 2/3 only (no 72B) |
| Queue time | 1-5 sec |

**Verdict:** ⚠️ Marginal. Requires rate limiting. Use smaller models.

---

### Scenario 5: Production Service (50+ users)
*Always-on API serving multiple clients*

| Metric | Requirement | Cluster Capacity |
|--------|-------------|------------------|
| Users | 50+ | ❌ Undersized |
| Latency SLA | <2 sec | ⚠️ At risk |
| Model size | 27B+ | ✅ Possible |

**Verdict:** ❌ Not designed for production at this scale. Add more GPUs or use cloud burst.

---

## Optimization Strategies

### 1. Model Routing
Route requests to appropriate tier:
- Simple questions → Tier 3 (7B)
- Coding tasks → Tier 2 (27B)
- Complex architecture → Tier 1 (72B)

### 2. Speculative Decoding
Use small model to draft, large model to verify. 2-3× throughput gain.

### 3. Quantization Tradeoffs
| Quantization | VRAM | Quality | Speed |
|--------------|------|---------|-------|
| FP16 | 100% | Best | Baseline |
| Q8 | 50% | ~98% | 1.5× |
| Q4 | 25% | ~95% | 2× |

### 4. Context Window Management
| Context | KV Cache (72B) | Impact |
|---------|----------------|--------|
| 2K tokens | ~2GB | 6+ concurrent |
| 4K tokens | ~4GB | 4-5 concurrent |
| 8K tokens | ~8GB | 2-3 concurrent |
| 32K tokens | ~32GB | 1 concurrent |

**Aggressively summarize context** to maximize concurrency.

---

## The Math

### VRAM Budget (taichi-5090, 64GB total)
```
Model weights (72B Q4):     ~40GB
KV cache per agent (4K ctx): ~4GB
Available for agents:        ~24GB
Max concurrent agents:       24 / 4 = 6
```

### Token Throughput
```
72B model: ~30 tokens/sec
27B model: ~60 tokens/sec  
7B model:  ~150 tokens/sec

Average response: 500 tokens
Time per response: 3-17 seconds depending on model
```

---

## Summary: Capacity Matrix

| Scenario | Agents | Supported? |
|----------|--------|------------|
| Solo developer | 3-5 | ✅ Excellent |
| Small team (5) | 10-15 | ✅ Great |
| Federation (10) | 10 | ✅ Good |
| Workshop (20) | 20-40 | ⚠️ Marginal |
| Production (50+) | 50+ | ❌ Undersized |

---

## Future Expansion

| Addition | Capacity Gain |
|----------|---------------|
| +1 RTX 5090 | +50% heavy inference |
| +1 RTX 3090 | +25% coding tier |
| Speculative decoding | 2-3× throughput |
| Better quantization | +30% efficiency |
| Cloud burst | Unlimited ($$) |

---

*The cluster is optimized for R&D workflows: bursts of activity from a small team, with most agents idle most of the time. It's not a production inference farm—but for what it's designed to do, it punches well above its weight.*

---
layout: default
title: AI-Lab
permalink: /ai-lab/
---

# AI-Lab Documentation

Technical documentation from the broader AI Lab project - the infrastructure that powers the Federation.

---

## Subsystems

| Section | Description |
|---------|-------------|
| [🖥️ Cluster Hardware](cluster/) | Node specs, GPU configs, model compatibility |
| [🔍 FRAG (Fast RAG)](frag/) | High-performance RAG indexing and search |
| [🧠 RAG Concepts](rag/) | Understanding RAG, MCP, and embeddings |
| [🔐 AI Security](security/) | Enterprise security architectures for AI systems |
| [⚡ Kinetic](kinetic/) | Agentic workflow system for VS Code |
| [📊 RAG Benchmarks](benchmarks/) | Performance benchmarks and analysis |

---

## The AI-Lab Stack

```
┌─────────────────────────────────────────────────────┐
│                   Federation Layer                   │
│   (MCP Federation, Mind-Speak, Collective Memory)   │
├─────────────────────────────────────────────────────┤
│                   Agent Layer                        │
│   (Kinetic Workflows, Cline, Continue.dev)    │
├─────────────────────────────────────────────────────┤
│                   Retrieval Layer                    │
│   (FRAG, LanceDB, Embedding Models)                 │
├─────────────────────────────────────────────────────┤
│                   Inference Layer                    │
│   (vLLM, llama.cpp, X-Ray Orchestration)            │
├─────────────────────────────────────────────────────┤
│                   Hardware Layer                     │
│   (RTX 5090s, 3090s, Mellanox 100Gbps, NFS)        │
└─────────────────────────────────────────────────────┘
```

---

## Quick Links

### FRAG (Fast RAG)
- [Architecture Overview](frag/architecture) - FRAG Core, Service, and MCP integration
- [Embedding Benchmarks](benchmarks/embedding) - Quality vs speed tradeoffs

### RAG Concepts
- [Understanding RAG & MCP](rag/concepts) - Complete beginner's guide
- [Privacy Architecture](rag/concepts#can-you-keep-private-data-private-yes-) - Keep private data private

### AI Security
- [SCIF Architecture](security/scif-architecture) - Air-gapped LLM infrastructure
- [Multi-Tenant RAG](security/multi-tenant-rag) - Secure multi-user RAG design

### Kinetic Agent
- [Kinetic Overview](kinetic/) - Early agentic coding experiment

---

*Building the infrastructure for intelligent systems.*

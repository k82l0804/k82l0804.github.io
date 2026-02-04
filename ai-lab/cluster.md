---
layout: default
title: Cluster Hardware
permalink: /ai-lab/cluster/
---

# AI-Lab Cluster Hardware

*Three nodes, 100Gbps interconnect, consumer GPUs running production workloads.*

---

## Cluster Overview

| Node | Role | GPUs | RAM | Storage |
|------|------|------|-----|---------|
| **taichi-5090** | Heavy LLM Inference | 2× RTX 5090 (32GB each) | 64GB | NVMe |
| **aorus-3090** | NFS Server, FRAG, Developer | 2× RTX 3090 (24GB each) | 128GB | 4TB NVMe |
| **baby** | Orchestration, Ray Head, Gitea | CPU-only | 64GB | 2TB SSD |

**Interconnect:** 100Gbps Mellanox InfiniBand (10.0.0.x network)

---

## Node Details

### taichi-5090 (Lead Node)
| Component | Specification |
|-----------|---------------|
| **GPUs** | 2× NVIDIA RTX 5090 (32GB VRAM each, 64GB total) |
| **CPU** | AMD Ryzen (32 threads) |
| **RAM** | 64GB DDR5 |
| **Role** | Primary LLM inference, heavy models |
| **Software** | vLLM, llama.cpp |

**Suitable Models:**
- 70B+ parameter models at full precision
- Qwen 2.5 72B, Llama 3.3 70B
- Multi-model serving (32B + 27B simultaneously)

---

### aorus-3090 (Developer Node)
| Component | Specification |
|-----------|---------------|
| **GPUs** | 2× NVIDIA RTX 3090 (24GB VRAM each, 48GB total) |
| **CPU** | 24-core (48 threads) |
| **RAM** | 128GB DDR4 |
| **Storage** | 4TB NVMe (NFS shared) |
| **Role** | NFS server, FRAG indexing, development |
| **Software** | vLLM, FRAG service, embedding models |

**Suitable Models:**
- 27B-35B parameter models (Gemma 2 27B, CodeLlama 34B)
- Embedding models (BGE, E5)
- Quantized 70B (Q4/Q5)

---

### baby (Orchestration Node)
| Component | Specification |
|-----------|---------------|
| **GPUs** | None (CPU-only) |
| **CPU** | Multi-core server CPU |
| **RAM** | 64GB |
| **Role** | Ray head, Gitea, orchestration |
| **Software** | Ray, PostgreSQL, ActiveMQ, Gitea |

**Purpose:** Coordination, not inference. Runs federation infrastructure.

---

## Model → Hardware Mapping

| Model Size | Minimum VRAM | Recommended Node |
|------------|--------------|------------------|
| **7B-8B** | 8GB | Any node (CPU possible) |
| **13B-14B** | 16GB | aorus-3090 (single GPU) |
| **27B-35B** | 24GB | aorus-3090 (single GPU) |
| **70B (Q4)** | 40GB | aorus-3090 (2× 3090) |
| **70B (FP16)** | 140GB | taichi-5090 only |
| **70B+ (BF16)** | 64GB+ | taichi-5090 (2× 5090) |

---

## VRAM Requirements

| Precision | Formula | Example (70B) |
|-----------|---------|---------------|
| **FP32** | params × 4 bytes | 280GB |
| **FP16/BF16** | params × 2 bytes | 140GB |
| **INT8** | params × 1 byte | 70GB |
| **INT4 (Q4)** | params × 0.5 bytes | 35GB |

Add ~20% overhead for KV cache during inference.

---

## Current Deployments

| Model | Node | Use Case |
|-------|------|----------|
| Qwen 2.5 72B | taichi-5090 | Primary agent reasoning |
| Gemma 2 27B | aorus-3090 | Fast inference, coding |
| BGE-base | aorus-3090 | FRAG embeddings (970 chunks/sec) |
| Reranker | aorus-3090 | RAG result reranking |

---

## Network Architecture

```
┌─────────────┐     100Gbps      ┌─────────────┐
│  taichi     │◄────────────────►│   aorus     │
│  (5090×2)   │    Mellanox      │  (3090×2)   │
└──────┬──────┘                  └──────┬──────┘
       │                                │
       │         100Gbps                │
       │      ┌──────────┐              │
       └─────►│   baby   │◄─────────────┘
              │ (CPU/NFS)│
              └──────────┘
```

**NFS Mount:** `/mnt/cluster/` shared across all nodes  
**Model Cache:** `/mnt/cluster/models/.cache/`

---

## RAG Performance Benchmarks

| Workload | Node | Throughput |
|----------|------|------------|
| Embedding (bge-base) | aorus-3090 | 970 chunks/sec |
| Inference (27B) | aorus-3090 | ~50 tokens/sec |
| Inference (72B) | taichi-5090 | ~30 tokens/sec |
| NFS throughput | aorus-3090 | 10+ GB/s |

See [Benchmarks](/ai-lab/benchmarks/) for detailed analysis.

---

## Why Consumer GPUs?

| Factor | Consumer (RTX) | Datacenter (A100/H100) |
|--------|----------------|------------------------|
| **Cost** | $2K-4K per card | $15K-40K per card |
| **VRAM** | 24-32GB | 40-80GB |
| **Availability** | Retail purchase | Enterprise contracts |
| **Power** | 350-450W | 400-700W |
| **Driver restrictions** | None (desktop) | Sometimes EULA limits |

**Trade-off:** More cards needed, but 5-10× cheaper per GB of VRAM.

---

*Built for R&D, not production. But it runs production workloads.*

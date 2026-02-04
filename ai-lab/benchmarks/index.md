---
layout: default
title: Benchmarks
permalink: /ai-lab/benchmarks/
---

# Performance Benchmarks

Comprehensive benchmarking of AI-Lab systems.

---

## Benchmark Reports

| Report | Focus |
|--------|-------|
| [📊 FRAG Embedding](embedding) | Embedding model quality vs speed tradeoffs |
| [⚡ MCP Server](mcp-server) | MCP server latency and throughput |
| [📁 File Agent](file-agent) | File streaming and indexing performance |

---

## Benchmark Environment

| Component | Specification |
|-----------|---------------|
| **GPU** | 2x NVIDIA RTX 3090 (48 GB VRAM total) |
| **CPU** | 24 cores (48 threads) |
| **Network** | 100Gbps Mellanox infiniband |
| **Storage** | NVMe SSDs + NFS cluster |

---

## Key Results

### Embedding Performance (FRAG)

| Quality | Model | Throughput | Improvement |
|---------|-------|------------|-------------|
| good | bge-small | 1,576 chunks/s | 4x faster |
| **better** | bge-base | 970 chunks/s | **Best balance** |
| best | bge-large | 390 chunks/s | Max quality |

**Key Finding**: Calibrated batch sizes improved performance by **61%**.

---

*Measure twice, optimize once.*

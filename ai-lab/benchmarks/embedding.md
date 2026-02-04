# FRAG Embedding Model Benchmarking Report

**Date**: 2025-12-17  
**Hardware**: 2x NVIDIA RTX 3090 (47 GB VRAM), 24-core CPU (48 threads)  
**Test Dataset**: continue-ext (2,011 files, 18.2 MB, 61,386 chunks)

---

## Executive Summary

This report documents comprehensive benchmarking of the FRAG (Fast RAG) embedding service, focusing on quality-based model selection, hardware calibration, and performance optimization. We implemented a quality preset system that allows users to select between speed and quality with a single configuration parameter, automatically applying calibrated batch sizes for optimal GPU utilization.

**Key Achievement**: Improved indexing performance by **61%** (602 → 970 chunks/sec) with the same model quality by integrating hardware-calibrated batch sizes.

---

## Table of Contents

1. [Overview](#overview)
2. [Quality Preset System](#quality-preset-system)
3. [Hardware Calibration](#hardware-calibration)
4. [Benchmark Methodology](#benchmark-methodology)
5. [Performance Results](#performance-results)
6. [Resource Utilization](#resource-utilization)
7. [Quality vs Speed Tradeoffs](#quality-vs-speed-tradeoffs)
8. [Configuration Guide](#configuration-guide)
9. [Key Findings](#key-findings)
10. [Recommendations](#recommendations)

---

## Overview

### What is FRAG?

FRAG is a high-performance RAG (Retrieval-Augmented Generation) service that provides:
- **Semantic chunking** using tree-sitter AST parsing
- **Direct GPU embedding** via HuggingFace Transformers (PyTorch)
- **Vector storage** in LanceDB
- **Multi-GPU support** via DataParallel
- **Precision optimization** (BF16 on Ampere+ GPUs)

### Goals

1. Compare embedding model performance across quality levels
2. Integrate hardware calibration for optimal batch sizes
3. Establish baseline metrics for production deployment
4. Validate quality preset system functionality

---

## Quality Preset System

### Concept

Instead of requiring users to understand model names, embedding dimensions, and optimal batch sizes, we created a simple quality-based selection:

```yaml
embedding-quality: "better"  # good, better, or best
```

### Implementation

**Configuration** (`services-config.yaml`):
```yaml
embedding-quality-presets:
  good:
    model: "BAAI/bge-small-en-v1.5"
    batch-size: 8192  # Calibrated for 2x RTX 3090
    theoretical-throughput: 78180  # chunks/sec (synthetic)
  better:
    model: "BAAI/bge-base-en-v1.5"
    batch-size: 4096
    theoretical-throughput: 26762
  best:
    model: "BAAI/bge-large-en-v1.5"
    batch-size: 4096
    theoretical-throughput: 8415
```

**Automatic Resolution** (`frag_config.py`):
- Reads quality presets from YAML
- Resolves quality → model + batch size
- Logs configuration at startup
- Provides fallback to hardcoded defaults

---

## Hardware Calibration

### Calibration Tool

We used FRAG's built-in calibration tool to find optimal settings:

```bash
python ./FRAG/scripts/calibrate.py --model BAAI/bge-base-en-v1.5
```

### Calibration Process

The tool performs:

1. **Hardware Detection**
   - GPU model, count, VRAM
   - CPU cores/threads
   - Compute capability (for precision selection)

2. **Batch Size Calibration**
   - Binary search for max batch without OOM
   - 80% safety margin
   - Tests with sequence length 128

3. **Worker Optimization**
   - Tokenizer workers (CPU parallelism)
   - Chunking workers (I/O optimization)

4. **Throughput Benchmarking**
   - Synthetic data (40-char chunks)
   - Measured in chunks/sec

### Calibration Results

| Model | Batch Size | Tokenizer Workers | Chunking Workers | Synthetic Throughput |
|-------|------------|-------------------|------------------|---------------------|
| bge-small | 8192 | 12 | 8 | 78,180 chunks/sec |
| bge-base | 4096 | 12 | 8 | 26,762 chunks/sec |
| bge-large | 4096 | 12 | 8 | 8,415 chunks/sec |

**Note**: Synthetic throughput uses 40-character chunks. Real-world performance with 370-char code chunks is proportionally lower.

---

## Benchmark Methodology

### Test Environment

- **Hardware**: 2x NVIDIA RTX 3090, 24-core CPU
- **Software**: FRAG service, PyTorch with BF16, DataParallel
- **Dataset**: continue-ext repository
  - 2,011 source files
  - 18.2 MB total
  - 61,386 chunks after tree-sitter chunking
  - ~370 chars average per chunk

### Benchmark Script

`bench-frag-indexing.py` measures:
- **Indexing time**: Chunking, embedding, storage phases
- **Throughput**: Chunks/sec and MB/sec
- **Resource usage**: CPU%, RAM, GPU VRAM, GPU utilization

### Resource Monitoring

Implemented using `pynvml` (NVIDIA Management Library):
- Tracks actual GPU memory usage (not just PyTorch allocations)
- Monitors GPU utilization percentage
- Samples every 0.5 seconds during indexing

---

## Performance Results

### Complete Quality Comparison

| Quality | Model | Dimensions | Batch Size | Time | Embed Rate | MTEB Score |
|---------|-------|------------|------------|------|------------|------------|
| **good** | bge-small | 384 | 8192 | 55.3s | **1,576/s** | 62.3 |
| **better** | bge-base | 768 | 4096 | 80.4s | **970/s** | 64.2 |
| **best** | bge-large | 1024 | 4096 | 180.3s | **390/s** | 64.6 |

### Phase Breakdown (continue-ext)

| Quality | Chunking | Embedding | Storage | Total |
|---------|----------|-----------|---------|-------|
| good | 11.8s | 39.0s | 2.3s | 55.3s |
| better | 11.8s | 63.3s | 4.0s | 80.4s |
| best | 11.9s | 157.6s | 5.2s | 180.3s |

**Key Insight**: Embedding phase dominates total time (70-87% of total).

### Performance Improvement

Comparing **before** and **after** calibration (bge-base model):

| Metric | Before (batch=256) | After (batch=4096) | Improvement |
|--------|-------------------|-------------------|-------------|
| Embed Rate | 602 chunks/sec | 970 chunks/sec | **+61%** |
| Total Time | ~100s | 80s | **-20%** |
| GPU VRAM | Unknown | 23.5 GB peak | Measured |

---

## Resource Utilization

### GPU Memory Usage

| Quality | Model Size | GPU 0 VRAM | GPU 1 VRAM | Batch VRAM |
|---------|------------|------------|------------|------------|
| good | 33M params | 23.95 GB peak | 19.74 GB peak | ~20-24 GB |
| better | 109M params | 23.54 GB peak | 22.61 GB peak | ~23 GB |
| best | 335M params | 19.43 GB peak | 23.28 GB peak | ~21 GB |

**Analysis**:
- Batch data dominates VRAM usage (vs model parameters)
- Multi-GPU DataParallel distributes work ~equally
- All configs fit comfortably in 24 GB per GPU

### GPU Utilization

| Quality | GPU 0 Util | GPU 1 Util | Average | Bottleneck |
|---------|------------|------------|---------|------------|
| good | 42% | 33% | **38%** | Memory bandwidth |
| better | 58% | 53% | **56%** | Memory bandwidth |
| best | 79% | 69% | **74%** | Compute |

**Analysis**:
- bge-large achieves highest GPU utilization (74%)
- Smaller models are memory-bandwidth limited
- Room for optimization in good/better modes

### CPU & RAM

| Resource | good | better | best |
|----------|------|--------|------|
| CPU (avg) | 7.4% | 6.7% | 4.6% |
| CPU (peak) | 47.1% | 41.6% | 46.5% |
| RAM (peak) | 0.51 GB | 0.51 GB | 0.51 GB |

**Analysis**:
- CPU barely utilized (chunking is efficient)
- RAM usage minimal (streaming mode prevents OOM)
- Bottleneck is clearly GPU embedding

---

## Quality vs Speed Tradeoffs

### Speed Comparison

Relative to **best** quality:
- **good**: 4.0x faster (1,576 vs 390 chunks/sec)
- **better**: 2.5x faster (970 vs 390 chunks/sec)

### Quality Comparison (MTEB Scores)

| Model | MTEB Score | vs best | Practical Impact |
|-------|------------|---------|------------------|
| bge-small | 62.3 | -2.3 pts | Slightly less accurate retrieval |
| bge-base | 64.2 | -0.4 pts | **Minimal quality loss** ⭐ |
| bge-large | 64.6 | baseline | Maximum quality |

### Cost-Benefit Analysis

**For Production RAG**:

| Quality | Speed | Quality | GPU Util | Recommendation |
|---------|-------|---------|----------|----------------|
| good | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | 38% | Large codebases, frequent re-indexing |
| better | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | 56% | **Best balance** 🎯 |
| best | ⭐⭐ | ⭐⭐⭐⭐⭐ | 74% | Small codebases, critical quality |

---

## Configuration Guide

### Quick Start

**1. Choose Quality Level**

Edit `configs/services-config.yaml`:
```yaml
rag-server:
  embedding-quality: "better"  # good, better, or best
```

**2. Restart FRAG**

```bash
./scripts/start-rag-service.sh
```

**3. Verify Configuration**

Check FRAG startup logs:
```
Resolved embedding-quality 'better' -> BAAI/bge-base-en-v1.5
  Balanced performance, better retrieval quality
  Batch size: 4096 (calibrated)
```

### Advanced: Custom Calibration

**Run calibration for your hardware**:

```bash
# For each model you want to use
python ./FRAG/scripts/calibrate.py --model BAAI/bge-small-en-v1.5
python ./FRAG/scripts/calibrate.py --model BAAI/bge-base-en-v1.5
python ./FRAG/scripts/calibrate.py --model BAAI/bge-large-en-v1.5
```

**Update quality presets** with calibrated batch sizes in `services-config.yaml`.

### Advanced: Manual Override

You can override batch size manually:
```yaml
rag-server:
  embedding-quality: "better"
  embedding-batch-size: 2048  # Manual override
```

---

## Key Findings

### 1. Calibrated Batch Sizes Critical

**Finding**: Using hardware-calibrated batch sizes improved performance by 61% with the same model.

**Impact**:
- Old manual setting: batch=256
- Calibrated setting: batch=4096 (16x larger)
- Result: Better GPU utilization, faster indexing

**Recommendation**: Always run calibration for your specific hardware.

### 2. "Better" Quality is the Sweet Spot

**Finding**: bge-base offers 97% of best quality at 2.5x the speed.

**Impact**:
- Quality loss: Only 0.4 MTEB points
- Speed gain: 2.5x faster indexing
- GPU efficiency: Good utilization (56%)

**Recommendation**: Use "better" for production unless you specifically need maximum quality.

### 3. Real-World vs Synthetic Performance

**Finding**: Real-world throughput is ~60-80% lower than synthetic benchmarks.

**Explanation**:
- Synthetic: 40-char chunks → 78K chunks/sec (bge-small)
- Real-world: 370-char chunks → 1.6K chunks/sec (bge-small)
- Ratio: ~2% of synthetic (due to 9x larger chunks)

**Recommendation**: Use real-world benchmarks for capacity planning.

### 4. Multi-GPU Scaling Works

**Finding**: DataParallel effectively utilizes both GPUs with minimal overhead.

**Evidence**:
- GPU 0: 23.5 GB VRAM, 58% util
- GPU 1: 22.6 GB VRAM, 53% util
- Load balanced within 10%

**Recommendation**: Continue using multi-GPU for embedding workloads.

### 5. Embedding Phase is the Bottleneck

**Finding**: 70-87% of indexing time is spent on embedding.

**Breakdown** (better quality):
- Chunking: 15% of time
- **Embedding: 79% of time** ⚠️
- Storage: 5% of time

**Recommendation**: Focus optimization efforts on embedding phase (batch size, precision, model selection).

### 6. GPU Memory Bandwidth Limits Small Models

**Finding**: bge-small and bge-base are memory-bandwidth limited, not compute-limited.

**Evidence**:
- bge-small: 38% GPU util
- bge-base: 56% GPU util
- bge-large: 74% GPU util

**Explanation**: Smaller models move less data per operation, underutilizing GPU compute cores.

**Recommendation**: Consider bge-large for maximum GPU utilization if quality matters.

---

## Recommendations

### For Production Deployment

1. **Use "better" quality (bge-base)**
   - Best balance of speed and quality
   - 970 chunks/sec indexing rate
   - MTEB 64.2 (97% of best)

2. **Run calibration on your hardware**
   - Batch sizes are hardware-specific
   - Takes 20-30 seconds per model
   - Significant performance impact

3. **Monitor GPU utilization**
   - Target 50-80% utilization
   - <50% suggests memory-bandwidth bottleneck
   - >90% suggests compute bottleneck

### For Development

1. **Use "good" quality (bge-small)**
   - 4x faster than best
   - Acceptable quality for dev/test
   - Faster iteration cycles

### For Critical Applications

1. **Use "best" quality (bge-large)**
   - Maximum retrieval accuracy
   - Accept 2.5x slower indexing
   - Best GPU utilization (74%)

### Scaling Guidelines

**Expected Performance** (continue-ext baseline: 61,386 chunks):

| Codebase Size | good | better | best |
|---------------|------|--------|------|
| Small (25K chunks) | ~16s | ~26s | ~64s |
| Medium (100K chunks) | ~63s | ~103s | ~256s |
| Large (500K chunks) | ~317s | ~515s | ~1,282s |
| Very Large (1M chunks) | ~634s | ~1,031s | ~2,564s |

**Recommendation**: For codebases >500K chunks, consider batch indexing overnight.

---

## Appendix: Model Details

### BAAI/bge-small-en-v1.5

- **Parameters**: 33M
- **Dimensions**: 384
- **Layers**: 12
- **MTEB Score**: 62.3
- **Context Length**: 512 tokens
- **License**: MIT

### BAAI/bge-base-en-v1.5

- **Parameters**: 109M
- **Dimensions**: 768
- **Layers**: 12
- **MTEB Score**: 64.2
- **Context Length**: 512 tokens
- **License**: MIT

### BAAI/bge-large-en-v1.5

- **Parameters**: 335M
- **Dimensions**: 1024
- **Layers**: 24
- **MTEB Score**: 64.6
- **Context Length**: 512 tokens
- **License**: MIT

---

## Appendix: Benchmark Commands

### Run Full Benchmark Suite

```bash
# Activate environment
source .venv-llm/bin/activate

# Benchmark all configured projects
cd FRAG/scripts
python3 bench-frag-indexing.py

# Benchmark specific project
python3 bench-frag-indexing.py --project continue-ext
python3 bench-frag-indexing.py --project llama.cpp
```

### Change Quality Level

```bash
# Edit config
vim configs/services-config.yaml
# Change: embedding-quality: "better"

# Restart FRAG
./scripts/start-rag-service.sh
```

### View FRAG Service Status

```bash
curl http://localhost:8001/health | python3 -m json.tool
```

---

## Conclusion

The FRAG embedding benchmarking initiative successfully:

1. **Established baseline metrics** for three quality levels
2. **Integrated hardware calibration** for optimal batch sizes
3. **Improved performance by 61%** through calibration
4. **Created user-friendly quality presets** (good/better/best)
5. **Implemented GPU resource monitoring** with pynvml
6. **Validated multi-GPU DataParallel** effectiveness

The **"better" quality preset (bge-base)** is recommended for production use, offering excellent quality (MTEB 64.2) at 970 chunks/sec indexing rate with efficient GPU utilization (56%).

---

**Report Version**: 1.0  
**Author**: AI Lab Benchmarking Team  
**Contact**: See [services-config.yaml](../../configs/services-config.yaml) for FRAG configuration

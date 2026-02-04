# MCP Server Performance Benchmarks

**Date**: 2025-12-21  
**Hardware**: Server: aorus-3090 (RTX 5090 x2) | Client: baby (Intel i3, Wired LAN)  
**Dataset**: `mcp-rag-server-test-area` (5,986 files, ~90k chunks)

---

## Executive Summary

The MCP Server wraps the optimization File Agent (WebSocket) in a standardized JSON-RPC protocol. Benchmarks confirm it retains the high performance of the underlying transport while adding **Incremental Sync** capabilities that make it suitable for real-time IDE usage.

## Test Results

### 1. Full vs Incremental Indexing

We compared a fresh "Cold Start" index against an incremental update (modifying 1 file).

| Operation | Files Scanned | Files Sent | Total Time | Throughput | Notes |
|-----------|---------------|------------|------------|------------|-------|
| **Full Index** | 5,986 | 5,986 | **95.05s** | ~1,092 chunks/s | Network Latency Bound |
| **Incremental** | 5,986 | 1 | **1.74s** | Instant | < 2s Response Time |

**Key Finding**: The Incremental Sync is **54x faster** than a full re-index for typical edits, consuming virtually no resources.

### 2. Network Overhead

Detailed analysis of the Full Index run:
- **Total Duration**: 95.05s
- **File Count**: 5,986
- **Avg Time/File**: ~15.8ms

This 15.8ms includes:
1.  **Client**: `os.walk` + `gitignore` filtering
2.  **Transport**: JSON-RPC Serialization + WebSocket Send
3.  **Network**: Wired LAN RTT (~0.2ms)
4.  **Server**: Hash Comparison + Task Queueing

The system is efficient enough that the "Per File" overhead is minimal, enabling smooth background indexing even on modest client hardware.

## Methodology

Benchmarks were run using `tests/bench_mcp_stdio.py`:

```bash
# Full Index
python mcp-rag-server/tests/bench_mcp_stdio.py --path mcp-rag-server-test-area --name mcp_bench

# Modify 1 file...

# Incremental Index (Same ID)
python mcp-rag-server/tests/bench_mcp_stdio.py --path mcp-rag-server-test-area --name mcp_bench --id mcp_bench
```

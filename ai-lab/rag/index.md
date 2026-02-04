---
layout: default
title: RAG Concepts
permalink: /ai-lab/rag/
---

# RAG Concepts

Understanding Retrieval-Augmented Generation, MCP, and embeddings.

---

## Documentation

| Document | Description |
|----------|-------------|
| [📚 Understanding RAG & MCP](concepts) | Complete beginner's guide to RAG and MCP servers |

---

## What is RAG?

**RAG (Retrieval-Augmented Generation)** enhances LLM responses by:

1. **Retrieving** relevant context from a knowledge base
2. **Augmenting** the LLM prompt with that context  
3. **Generating** better, more informed responses

> Think of it as giving the LLM a "cheat sheet" of relevant code before answering.

---

## The RAG Pipeline

```
Phase 1: Indexing (Offline)
  Your Codebase → Chunking → Embedding → Vector Storage

Phase 2: Query (Every Question)
  Your Question → Embed Query → Search Vectors → Retrieve Chunks → Augment Prompt → Generate Answer
```

---

## Can You Keep Data Private? YES! ✅

The entire pipeline can run locally:
- ✅ Local embeddings model (sentence-transformers, llama.cpp)
- ✅ Local vector database (LanceDB)
- ✅ Local MCP server
- ✅ Local LLM (vLLM, llama-server)
- **Nothing leaves your machine!**

[Full privacy architecture →](concepts#can-you-keep-private-data-private-yes-)

---

*Private, context-aware coding assistance.*

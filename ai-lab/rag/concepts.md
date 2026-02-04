# Understanding RAG and MCP Servers for Continue.dev

## What is RAG (Retrieval-Augmented Generation)?

**RAG** is a technique that enhances LLM responses by:
1. **Retrieving** relevant context from a knowledge base (your codebase)
2. **Augmenting** the LLM prompt with that context
3. **Generating** better, more informed responses

Think of it as giving the LLM a "cheat sheet" of relevant code before answering your question.

---

## The RAG Pipeline - Step by Step

### Phase 1: Indexing (Done Once, Offline)

This happens **before** you ask questions, typically as a batch process:

```
Your Codebase
    ↓
1. CHUNKING: Break files into meaningful pieces
    ↓
2. EMBEDDING: Convert each chunk to a vector using an embeddings model
    ↓
3. STORAGE: Store vectors in a vector database (e.g., LanceDB)
```

**When embeddings happen (Phase 1):**
- During initial indexing of your codebase
- When you add/modify files and re-index
- This is **NOT** real-time during chat

**Your control:**
- ✅ You choose the embeddings model (local or remote)
- ✅ You run the indexing script when you want
- ✅ You control what data gets indexed

### Phase 2: Query (Happens During Chat)

This happens **every time** you ask the Continue.dev extension a question:

```
Your Question: "How does authentication work?"
    ↓
1. EMBEDDING: Convert your question to a vector (using SAME embeddings model)
    ↓
2. SEARCH: Find similar vectors in the database (vector similarity search)
    ↓
3. RETRIEVE: Get the actual code chunks for top matches
    ↓
4. AUGMENT: Add those chunks to the LLM prompt
    ↓
5. GENERATE: LLM responds with context-aware answer
```

**When embeddings happen (Phase 2):**
- Every time you ask a question (query embedding)
- Uses the **same** embeddings model as Phase 1

---

## What is an MCP Server?

**MCP (Model Context Protocol) Server** is a **query interface** - it's like a waiter at a restaurant:

```
Continue.dev Extension (Customer)
    ↓
    "I need code related to authentication"
    ↓
MCP Server (Waiter)
    ↓
    Takes order, queries database
    ↓
Vector Database (Kitchen)
    ↓
    Prepares the food (retrieves chunks)
    ↓
MCP Server (Waiter)
    ↓
    Returns results
    ↓
Continue.dev Extension (Customer)
    ↓
    Feeds results to LLM
```

### What MCP Server DOES:
✅ Provides tools/functions that Continue.dev can call
✅ Queries the vector database
✅ Formats and returns relevant code chunks
✅ Handles the "retrieval" part of RAG

### What MCP Server DOES NOT DO:
❌ Does NOT create embeddings (you do that in indexing script)
❌ Does NOT store data (vector database does that)
❌ Does NOT call the LLM (Continue.dev does that)
❌ Does NOT generate responses (LLM does that)

**Analogy:**
- **Indexing Script** = Chef preparing ingredients and storing them
- **Vector Database** = The refrigerator storing ingredients
- **MCP Server** = The waiter taking orders and fetching from the fridge
- **Continue.dev** = The customer (asks questions, gets food)
- **LLM** = Your stomach (digests the food to give you energy)

---

## Can You Keep Private Data Private? YES! ✅

### Architecture for Privacy:

```
┌─────────────────────────────────────────────────┐
│  PRIVATE CODEBASE (Never Leaves Your Computer)  │
└─────────────────────────────────────────────────┘
              ↓
    ┌─────────────────────┐
    │  Indexing Script    │
    │  (You control this) │
    └─────────────────────┘
              ↓
    ┌─────────────────────────────┐
    │  LOCAL Embeddings Model     │  ← Run locally via llama.cpp or vLLM
    │  (e.g., nomic-embed-text)   │     Your data NEVER sent externally
    └─────────────────────────────┘
              ↓
    ┌─────────────────────────┐
    │  Vector Database        │  ← Stored locally on your disk
    │  (LanceDB, local file)  │     /path/to/local/db
    └─────────────────────────┘
              ↓
    ┌─────────────────────┐
    │  MCP Server         │  ← Runs locally, queries local DB
    │  (Python script)    │
    └─────────────────────┘
              ↓
    ┌─────────────────────┐
    │  Continue.dev       │  ← VSCode extension
    │  Extension          │
    └─────────────────────┘
              ↓
    ┌─────────────────────────┐
    │  LOCAL Chat LLM         │  ← Your CodeLlama, GPT-OSS, etc.
    │  (llama-server/vLLM)    │     Running on your RTX 5090s
    └─────────────────────────┘

ALL OF THIS IS LOCAL - NOTHING LEAVES YOUR MACHINE
```

### For Public Data (Optional):

You **could** have a separate setup for public documentation:

```
PUBLIC DOCS (e.g., Python docs, React docs)
    ↓
External Embeddings API (e.g., Voyage AI, OpenAI)
    ↓
Separate Vector DB or same DB (different namespace)
    ↓
Same or different MCP Server
    ↓
Continue.dev uses both sources
```

---

## Complete Privacy Setup Example

### Step 1: Choose Local Embeddings Model

**Option A: Use llama.cpp with a GGUF embeddings model**
```bash
# Download a local embeddings model (e.g., nomic-embed-text)
./llama-server --model nomic-embed-text-v1.5.Q8_0.gguf --embedding --port 8082
```

**Option B: Use vLLM with an embeddings model**
```bash
vllm serve intfloat/e5-mistral-7b-instruct --port 8082
```

**Option C: Use sentence-transformers (Python library)**
```python
from sentence_transformers import SentenceTransformer
model = SentenceTransformer('nomic-ai/nomic-embed-text-v1.5')
embeddings = model.encode(["your code chunk"])
```

### Step 2: Indexing Script (Fully Local)

```python
#!/usr/bin/env python3
"""Local RAG indexing - all private data stays local"""

import os
import lancedb
from sentence_transformers import SentenceTransformer

# LOCAL embeddings model - runs on your machine
embedder = SentenceTransformer('nomic-ai/nomic-embed-text-v1.5')

# LOCAL database - stored on your disk
db = lancedb.connect("/home/k82l0804/.local/share/rag-db")

# Read your codebase
chunks = []
for root, dirs, files in os.walk("/home/k82l0804/workarea/bench/llamacpp"):
    for file in files:
        if file.endswith('.py'):
            filepath = os.path.join(root, file)
            with open(filepath) as f:
                content = f.read()
                # Simple chunking: one chunk per file (or implement smarter chunking)
                chunks.append({
                    "filename": filepath,
                    "text": content,
                    "vector": embedder.encode(content).tolist()
                })

# Store in local database
table = db.create_table("private_code", data=chunks, mode="overwrite")
print(f"Indexed {len(chunks)} files locally")
```

**Run this once:**
```bash
python3 index-private-codebase.py
# All embeddings generated locally, stored locally
```

### Step 3: MCP Server (Query Local DB)

```python
#!/usr/bin/env python3
"""Local MCP server - queries local database only"""

from mcp.server import Server
from mcp.server.stdio import stdio_server
from mcp.types import TextContent
import lancedb
from sentence_transformers import SentenceTransformer

# LOCAL database connection
db = lancedb.connect("/home/k82l0804/.local/share/rag-db")
table = db.open_table("private_code")

# LOCAL embeddings for queries
embedder = SentenceTransformer('nomic-ai/nomic-embed-text-v1.5')

app = Server("private-rag-server")

@app.tool()
async def search_private_code(query: str, limit: int = 5) -> list[TextContent]:
    """Search private codebase - never leaves your machine"""
    # Embed the query locally
    query_vector = embedder.encode(query).tolist()
    
    # Search local database
    results = table.search(query_vector).limit(limit).to_list()
    
    # Return results
    return [TextContent(
        type="text",
        text=f"File: {r['filename']}\n\n{r['text']}"
    ) for r in results]

if __name__ == "__main__":
    stdio_server(app).run()
```

### Step 4: Configure Continue.dev

Edit `.continue/config.yaml`:

```yaml
mcpServers:
  - name: private-rag
    command: python3
    args:
      - /home/k82l0804/workarea/bench/llamacpp/RAG/mcp-server-local.py
    env:
      # No external API keys needed!

models:
  - name: codellama 34B
    provider: openai
    model: TheBloke/CodeLlama-34B-Instruct-AWQ
    apiBase: http://localhost:8088/v1  # Your LOCAL server
    apiKey: sk-1234
    roles: [chat]
```

---

## When Embeddings Are Called

### During Indexing (One-time or periodic):
1. **You run** the indexing script manually
2. **Embeddings model** converts each code chunk to a vector
3. **Vectors stored** in database

**Frequency:** When you want (daily, weekly, on git push, etc.)

### During Query (Every question):
1. **You ask** Continue.dev a question
2. **Continue.dev calls** your MCP server
3. **MCP server embeds** your question using the same embeddings model
4. **Database searches** for similar vectors
5. **MCP returns** matching code chunks
6. **Continue.dev adds** chunks to LLM prompt
7. **LLM generates** answer with context

**Frequency:** Every chat message

---

## Dual Setup: Private + Public

You can absolutely have both:

```yaml
mcpServers:
  # Private data - all local
  - name: private-rag
    command: python3
    args:
      - /path/to/private-mcp-server.py
    
  # Public data - can use external embeddings
  - name: public-docs-rag
    command: python3
    args:
      - /path/to/public-mcp-server.py
    env:
      VOYAGE_API_KEY: ${VOYAGE_API_KEY}  # OK for public data
```

Continue.dev can use **both** MCP servers and combine results!

---

## Summary - Your Questions Answered

### Q: When do embeddings get called?
**A:** 
- **Indexing time** (offline): Convert code chunks to vectors
- **Query time** (every question): Convert your question to a vector

### Q: Can I use both local and external embeddings?
**A:** Yes! Use:
- **Local embeddings** for private codebase (sentence-transformers, llama.cpp)
- **External embeddings** for public docs (OpenAI, Voyage) if you want

### Q: Does MCP Server handle embeddings?
**A:** **Partially** - It typically:
- **Does NOT** create embeddings during indexing (your script does that)
- **DOES** create query embeddings when searching (can use local or remote)
- **DOES** query the database and return results

### Q: Can I keep private data private?
**A:** **YES! 100%** - Use:
- ✅ Local embeddings model (sentence-transformers, llama.cpp)
- ✅ Local vector database (LanceDB to local file)
- ✅ Local MCP server (Python script on your machine)
- ✅ Local chat LLM (your vLLM/llama-server)
- ✅ **NOTHING leaves your computer**

---

## Next Steps

1. **Choose** local embeddings model (e.g., nomic-embed-text)
2. **Write** indexing script to process your codebase
3. **Run** indexing to populate local database
4. **Create** MCP server to query the database
5. **Configure** Continue.dev to use your MCP server
6. **Enjoy** private, context-aware coding assistance!

Your setup will be **completely private** - no data, code, or queries ever leave your RTX 5090 machine! 🔒

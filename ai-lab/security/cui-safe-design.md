# Designing a CUI-Safe Corporate LLM Server

## Executive Summary

**Goal:** Build corporate LLM infrastructure that handles CUI safely while providing shared services to multiple users/projects.

**Key Principles:**
1. **Ephemeral Processing** - No persistent storage of prompts/responses
2. **Zero-Knowledge Architecture** - Server processes but doesn't "know" content
3. **User Isolation** - Complete separation between users/projects
4. **Minimal Logging** - Audit metadata only, not content
5. **Defense in Depth** - Multiple layers of security controls

---

## Architecture Overview

### Zero-Logging, Ephemeral Processing Design

```
┌────────────────────────────────────────────────────────────────┐
│ CLIENT WORKSTATION (User's Laptop)                             │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │ Continue.dev Extension                                   │ │
│  │  - Builds prompt locally                                 │ │
│  │  - Encrypts with user's project key                      │ │
│  └──────────────────────────────────────────────────────────┘ │
│                           ↓ TLS 1.3 + mTLS                     │
└────────────────────────────────────────────────────────────────┘
                            ↓
┌────────────────────────────────────────────────────────────────┐
│ CORPORATE NETWORK BOUNDARY                                      │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │ Hardware Security Module (HSM)                           │ │
│  │  - Certificate validation                                │ │
│  │  - Key management                                        │ │
│  │  - Decrypt only in secure enclave                        │ │
│  └──────────────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────────────┘
                            ↓
┌────────────────────────────────────────────────────────────────┐
│ API GATEWAY (CUI-Hardened)                                     │
│                                                                 │
│  ✅ Authentication (mTLS client certificates)                  │
│  ✅ Authorization (RBAC + project isolation)                   │
│  ✅ Rate limiting (per-user quotas)                            │
│  ✅ Audit logging (metadata ONLY - no content)                 │
│  ❌ NO request/response logging                                │
│  ❌ NO caching                                                  │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
                            ↓
┌────────────────────────────────────────────────────────────────┐
│ STATELESS INFERENCE CLUSTER (GPU Nodes)                       │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │ GPU Node (Isolated per Project)                         │ │
│  │                                                           │ │
│  │  ┌────────────────────────────────────────────────────┐ │ │
│  │  │ Secure Boot + TPM                                  │ │ │
│  │  │ Memory Encryption (AMD SEV / Intel TDX)            │ │ │
│  │  └────────────────────────────────────────────────────┘ │ │
│  │                                                           │ │
│  │  ┌────────────────────────────────────────────────────┐ │ │
│  │  │ vLLM / llama.cpp Server                            │ │ │
│  │  │  - Model loaded in GPU VRAM                        │ │ │
│  │  │  - Process request in memory ONLY                  │ │ │
│  │  │  - NO disk writes                                   │ │ │
│  │  │  - NO logging                                       │ │ │
│  │  │  - VRAM cleared after response                     │ │ │
│  │  └────────────────────────────────────────────────────┘ │ │
│  │                                                           │ │
│  │  ┌────────────────────────────────────────────────────┐ │ │
│  │  │ /tmp on tmpfs (RAM-only filesystem)               │ │ │
│  │  │  - No persistent storage                           │ │ │
│  │  │  - Wiped on reboot                                 │ │ │
│  │  └────────────────────────────────────────────────────┘ │ │
│  └──────────────────────────────────────────────────────────┘ │
│                                                                 │
│  Container Isolation:                                          │
│   - Project A gets dedicated GPU nodes                         │
│   - Project B gets different GPU nodes                         │
│   - No sharing between projects                                │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
                            ↓
┌────────────────────────────────────────────────────────────────┐
│ AUDIT SYSTEM (Metadata Only)                                   │
│                                                                 │
│  Logs stored (NO PROMPT/RESPONSE CONTENT):                     │
│   - User ID (who)                                              │
│   - Timestamp (when)                                           │
│   - Token count (how much)                                     │
│   - Model name (what)                                          │
│   - Latency (performance)                                      │
│   - Success/failure (status)                                   │
│   - Project ID (isolation tracking)                            │
│                                                                 │
│  ❌ NOT LOGGED:                                                 │
│   - Prompt text                                                │
│   - Response text                                              │
│   - Any CUI content                                            │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
```

---

## Core Security Mechanisms

### 1. Ephemeral Processing (No Persistent Storage)

**Principle:** Data exists only during active processing, then is destroyed.

#### Implementation:

**A. GPU VRAM Management**
```python
# vLLM/llama.cpp configuration for ephemeral processing

# Server startup config
{
  "gpu_memory_utilization": 0.95,  # Use most of VRAM
  "swap_space": 0,  # NO swap to disk
  "disable_log_requests": true,  # NO request logging
  "disable_log_stats": true,  # NO statistics with content
  "max_num_seqs": 256,  # Concurrent requests
  "max_model_len": 32768  # Context window
}

# Process flow
def process_request(prompt):
    # 1. Load into VRAM from network
    tokens = tokenize(prompt)  # In VRAM only
    
    # 2. Generate in VRAM
    response_tokens = model.generate(tokens)  # In VRAM only
    
    # 3. Decode in VRAM
    response_text = detokenize(response_tokens)  # In VRAM only
    
    # 4. Send response
    return response_text
    
    # 5. VRAM automatically freed for next request
    # No disk writes at any step
```

**B. RAM-Only Filesystem**
```bash
# Mount /tmp as tmpfs (RAM-only, no disk)
mount -t tmpfs -o size=32G,mode=1777 tmpfs /tmp

# vLLM working directory
export TMPDIR=/tmp/vllm-working
mkdir -p $TMPDIR
# This directory exists only in RAM, wiped on reboot

# Verify no disk writes
iotop -o  # Should show zero disk I/O during inference
```

**C. Containerization with No Persistent Volumes**
```yaml
# Docker/Kubernetes config
apiVersion: v1
kind: Pod
metadata:
  name: vllm-cui-safe
spec:
  containers:
  - name: vllm
    image: vllm-cui-hardened:latest
    
    # NO persistent volumes mounted
    volumeMounts:
    - name: ram-disk
      mountPath: /tmp
      
    # RAM disk only
    volumes:
    - name: ram-disk
      emptyDir:
        medium: Memory  # tmpfs in RAM
        sizeLimit: 32Gi
    
    # Security context
    securityContext:
      readOnlyRootFilesystem: true  # Cannot write to disk
      runAsNonRoot: true
      capabilities:
        drop:
        - ALL  # Drop all capabilities
    
    # No logging
    env:
    - name: VLLM_LOGGING_LEVEL
      value: "CRITICAL"
    - name: PYTHONUNBUFFERED
      value: "1"
```

---

### Backend Options: vLLM vs llama.cpp Server

#### Comparison Matrix

| Feature | vLLM | llama.cpp Server | Winner for CUI |
|---------|------|------------------|----------------|
| **Model Format** | safetensors, HuggingFace | GGUF | Depends on source |
| **Quantization** | AWQ, GPTQ, BitsAndBytes | GGUF (Q4, Q5, Q8) | llama.cpp (more options) |
| **Memory Efficiency** | Good (PagedAttention) | Excellent (GGUF quantization) | llama.cpp |
| **Throughput** | Excellent (batching) | Good | vLLM |
| **Latency (single)** | Good | Excellent | llama.cpp |
| **CPU Fallback** | Limited | Excellent | llama.cpp |
| **Dependencies** | Python, CUDA | C++, minimal | llama.cpp (simpler) |
| **Security Audit Surface** | Larger (Python ecosystem) | Smaller (C++) | llama.cpp |
| **Logging Control** | Good (can disable) | Excellent (none by default) | llama.cpp |
| **Multi-GPU** | Excellent (tensor parallel) | Good (layer split) | vLLM |
| **Continuous Batching** | Yes | Yes | Tie |
| **OpenAI API Compat** | Excellent | Excellent | Tie |

#### When to Use Each

**Use vLLM when:**
- ✅ High throughput needed (many concurrent users)
- ✅ Using safetensors/HuggingFace models
- ✅ AWQ/GPTQ quantization preferred
- ✅ Tensor parallelism across many GPUs
- ✅ Python ecosystem is acceptable

**Use llama.cpp server when:**
- ✅ GGUF models available
- ✅ Memory efficiency critical
- ✅ Lower latency for single requests
- ✅ CPU inference needed (hybrid CPU/GPU)
- ✅ Simpler deployment preferred
- ✅ Minimal attack surface desired
- ✅ C++ security audit easier than Python

#### CUI-Safe Configuration Examples

**OPTION 1: vLLM (Python-based)**

```bash
# Launch vLLM with CUI-safe settings
vllm serve /mnt/models/codellama-34b-instruct-awq \
  --host 0.0.0.0 \
  --port 8088 \
  --disable-log-requests \
  --disable-log-stats \
  --gpu-memory-utilization 0.95 \
  --swap-space 0 \
  --max-num-seqs 256 \
  --max-model-len 32768 \
  --tensor-parallel-size 2 \
  --trust-remote-code false \
  --api-key sk-cui-project-alpha
```

**Kubernetes Deployment:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: vllm-cui-safe
spec:
  containers:
  - name: vllm
    image: vllm/vllm-openai:latest
    command:
      - python3
      - -m
      - vllm.entrypoints.openai.api_server
    args:
      - --model=/models/codellama-34b-awq
      - --disable-log-requests
      - --disable-log-stats
      - --gpu-memory-utilization=0.95
      - --swap-space=0
      - --max-num-seqs=256
    
    volumeMounts:
    - name: ram-disk
      mountPath: /tmp
    - name: models
      mountPath: /models
      readOnly: true
    
    securityContext:
      readOnlyRootFilesystem: true
      runAsNonRoot: true
      capabilities:
        drop: [ALL]
    
    env:
    - name: VLLM_LOGGING_LEVEL
      value: "CRITICAL"  # Minimal logging
    - name: TMPDIR
      value: "/tmp"
    
    resources:
      limits:
        nvidia.com/gpu: 2
  
  volumes:
  - name: ram-disk
    emptyDir:
      medium: Memory
      sizeLimit: 32Gi
  - name: models
    hostPath:
      path: /mnt/models
      type: Directory
```

**OPTION 2: llama.cpp Server (C++)**

```bash
# Launch llama-server with CUI-safe settings
./llama-server \
  --model /mnt/models/codellama-34b-instruct-q5_k_m.gguf \
  --host 0.0.0.0 \
  --port 8088 \
  --ctx-size 32768 \
  --n-gpu-layers 999 \
  --parallel 4 \
  --cont-batching \
  --cache-type-k f16 \
  --cache-type-v f16 \
  --mlock \
  --numa \
  --no-log \
  --no-slots-report \
  --api-key sk-cui-project-alpha
```

**Configuration breakdown:**
- `--no-log` - Disable all logging (CUI safe!)
- `--no-slots-report` - Don't log slot status
- `--mlock` - Lock model in RAM (no swap to disk)
- `--numa` - NUMA optimization for multi-socket
- `--n-gpu-layers 999` - Offload all layers to GPU
- `--cont-batching` - Continuous batching for throughput
- `--parallel 4` - 4 parallel sequences
- `--cache-type-k/v f16` - KV cache precision

**Kubernetes Deployment:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: llama-server-cui-safe
spec:
  containers:
  - name: llama-server
    image: ghcr.io/ggerganov/llama.cpp:server
    command:
      - /app/llama-server
    args:
      - --model=/models/codellama-34b-q5_k_m.gguf
      - --host=0.0.0.0
      - --port=8088
      - --ctx-size=32768
      - --n-gpu-layers=999
      - --parallel=4
      - --cont-batching
      - --mlock
      - --numa
      - --no-log
      - --no-slots-report
      - --api-key=sk-cui-project-alpha
    
    volumeMounts:
    - name: ram-disk
      mountPath: /tmp
    - name: models
      mountPath: /models
      readOnly: true
    
    securityContext:
      readOnlyRootFilesystem: true
      runAsNonRoot: true
      capabilities:
        drop: [ALL]
        add: [IPC_LOCK]  # For mlock privilege
    
    resources:
      limits:
        nvidia.com/gpu: 1
        memory: 64Gi  # For mlock
  
  volumes:
  - name: ram-disk
    emptyDir:
      medium: Memory
      sizeLimit: 8Gi  # llama.cpp uses less temp space
  - name: models
    hostPath:
      path: /mnt/models
      type: Directory
```

#### Multi-GPU Configuration

**vLLM (Tensor Parallelism):**
```bash
# Split model across 2 GPUs
vllm serve model.safetensors \
  --tensor-parallel-size 2 \
  --disable-log-requests
```

**llama.cpp (Layer Splitting):**
```bash
# Split layers across 2 GPUs
./llama-server \
  --model model.gguf \
  --n-gpu-layers 80 \
  --tensor-split 1,1 \
  --main-gpu 0 \
  --no-log
```

#### Logging Verification

**Verify vLLM isn't logging:**
```bash
# Check for log files
find /var/log /tmp -name "*vllm*" -o -name "*llm*" 2>/dev/null

# Monitor disk I/O
iotop -o  # Should show zero writes during inference

# Check environment
env | grep -i log
```

**Verify llama-server isn't logging:**
```bash
# llama-server doesn't log by default with --no-log
# Verify no output files
ls -la /tmp/llama* 2>/dev/null

# Check process output (should be minimal)
ps aux | grep llama-server
```

#### Performance Tuning for CUI Workloads

**vLLM Optimization:**
```python
# API server configuration
{
  "model": "/models/codellama-34b-awq",
  "disable_log_requests": true,
  "disable_log_stats": true,
  "gpu_memory_utilization": 0.95,
  "max_num_seqs": 256,  # High for many users
  "max_model_len": 32768,
  "tensor_parallel_size": 2,
  "dtype": "auto",
  "quantization": "awq",
  "enforce_eager": false,  # Use CUDA graphs for speed
  "enable_prefix_caching": false  # Disable caching (CUI!)
}
```

**llama.cpp Optimization:**
```bash
./llama-server \
  --model /models/model.gguf \
  --ctx-size 32768 \
  --n-gpu-layers 999 \
  --parallel 4 \  # Lower than vLLM, still good
  --batch-size 512 \
  --ubatch-size 512 \  # Micro-batch size
  --cont-batching \
  --mlock \
  --numa \
  --no-log \
  --flash-attn \  # FlashAttention if available
  --cache-type-k f16 \
  --cache-type-v f16
```

---

### RAM-Only Filesystem (Both Backends)

### 2. Zero-Knowledge Architecture

**Principle:** Server infrastructure doesn't "know" content, only processes it.

#### A. Client-Side Encryption (Optional, High Security)

```python
# On user's laptop - encrypt prompt before sending
from cryptography.fernet import Fernet

# User's project-specific key (never leaves laptop)
user_key = load_key_from_secure_storage()
cipher = Fernet(user_key)

# Encrypt prompt
plaintext_prompt = "How does our CUI authentication work? [CUI code...]"
encrypted_prompt = cipher.encrypt(plaintext_prompt.encode())

# Send to corporate LLM
response_encrypted = corporate_llm.chat(encrypted_prompt)

# Decrypt response on laptop
response = cipher.decrypt(response_encrypted).decode()
```

**Problem:** LLM can't process encrypted text!

**Solution:** Use **homomorphic encryption** (future) or **trusted execution environment** (TEE)

#### B. Trusted Execution Environment (TEE)

```
┌──────────────────────────────────────────────────┐
│ Intel SGX / AMD SEV / ARM TrustZone Enclave      │
│                                                   │
│  • Prompt decrypted INSIDE secure enclave        │
│  • LLM inference runs INSIDE secure enclave      │
│  • Response encrypted INSIDE secure enclave      │
│  • Server OS/admins CANNOT access enclave memory │
│                                                   │
│  Even root access cannot see CUI content!        │
└──────────────────────────────────────────────────┘
```

**Technologies:**
- **Intel SGX** (Software Guard Extensions)
- **AMD SEV-SNP** (Secure Encrypted Virtualization)
- **NVIDIA Confidential Computing** (H100 GPUs)
- **Google Confidential VMs**

**Benefit:** Admins manage infrastructure but can't see CUI content

---

### 3. Granular User/Project Isolation

**Principle:** Each project gets logically/physically separated resources.

#### A. Dedicated GPU Nodes per Project

```yaml
# Kubernetes node affinity
apiVersion: v1
kind: Pod
metadata:
  name: vllm-project-alpha
  labels:
    project: project-alpha
    classification: cui
spec:
  # Only schedule on Project Alpha nodes
  nodeSelector:
    project: project-alpha
    gpu: nvidia-h100
  
  # Anti-affinity: Don't schedule with other projects
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: project
            operator: NotIn
            values:
            - project-alpha
        topologyKey: kubernetes.io/hostname
```

**Result:**
- Project Alpha's CUI only processes on Alpha nodes
- Project Beta uses different physical GPUs
- No cross-contamination

#### B. Network Segmentation (VLANs/VPCs)

```
┌──────────────────────────────────────┐
│ Project Alpha VLAN (VLAN 100)        │
│  - Alpha users only                  │
│  - Alpha GPU nodes only              │
│  - Firewall rules: no access to VLAN 200
└──────────────────────────────────────┘

┌──────────────────────────────────────┐
│ Project Beta VLAN (VLAN 200)         │
│  - Beta users only                   │
│  - Beta GPU nodes only               │
│  - Firewall rules: no access to VLAN 100
└──────────────────────────────────────┘
```

#### C. Namespace Isolation in Vector Database

```python
# If using corporate RAG/embeddings service

# Project Alpha's vector database
alpha_db = lancedb.connect("s3://corporate-rag-db/project-alpha")
alpha_table = alpha_db.open_table("code_chunks")

# Project Beta CANNOT access Alpha's namespace
# Enforced by:
# - IAM roles (AWS/Azure)
# - Database ACLs
# - Network policies
```

---

### 4. Minimal Audit Logging (Metadata Only)

**Principle:** Log enough for compliance, nothing with CUI content.

#### A. What to Log (Metadata)

```json
{
  "event_type": "llm_request",
  "timestamp": "2025-11-26T06:20:05Z",
  "user_id": "alice@project-alpha.mil",
  "user_cert_fingerprint": "SHA256:abc123...",
  "project_id": "project-alpha",
  "model_name": "codellama-34b-cui",
  "classification_level": "CUI",
  "request_id": "req-550e8400-e29b-41d4-a716-446655440000",
  "token_count": {
    "prompt_tokens": 1250,
    "completion_tokens": 450,
    "total_tokens": 1700
  },
  "latency_ms": 1834,
  "gpu_node": "gpu-alpha-03",
  "status": "success",
  "error": null
}
```

**Notice:** No prompt/response content!

#### B. What NOT to Log

```json
{
  // ❌ DO NOT LOG THESE:
  "prompt_text": "How does auth work? [CUI code...]",
  "response_text": "The authentication uses...",
  "rag_chunks": ["chunk1 with CUI", "chunk2 with CUI"],
  "embedding_vector": [0.123, 0.456, ...]
}
```

#### C. Log Rotation and Retention

```yaml
logging_policy:
  retention:
    audit_metadata: 90_days  # Minimum for compliance
    performance_metrics: 30_days
    error_logs: 7_days
  
  automatic_deletion:
    enabled: true
    grace_period: 7_days  # After retention period
  
  encryption:
    at_rest: AES-256
    in_transit: TLS 1.3
  
  access_control:
    viewers: [security-officer, compliance-team]
    no_project_admins: true  # Admins can't see even metadata
```

---

### 5. Encryption Everywhere

#### A. In-Transit Encryption

**Mutual TLS (mTLS) with Client Certificates**

```yaml
# API Gateway configuration
tls:
  version: "1.3"  # TLS 1.3 only
  cipher_suites:
    - TLS_AES_256_GCM_SHA384
    - TLS_CHACHA20_POLY1305_SHA256
  
  # Require client certificates
  client_auth:
    required: true
    ca_cert: /etc/ssl/project-alpha-ca.crt
    verify_depth: 2
  
  # Perfect forward secrecy
  prefer_server_ciphers: false
  
  # HSTS
  hsts:
    enabled: true
    max_age: 31536000
    include_subdomains: true
```

**User Configuration:**
```yaml
# User's Continue.dev config
models:
  - name: corporate-cui-llm
    provider: openai
    apiBase: https://llm-cui.defenselab.mil/v1
    apiKey: ${PROJECT_ALPHA_TOKEN}
    
    # Client certificate for mTLS
    tls:
      client_cert: ~/.certs/project-alpha-client.crt
      client_key: ~/.certs/project-alpha-client.key
      ca_cert: ~/.certs/defenselab-ca.crt
```

#### B. At-Rest Encryption

Even though we minimize disk writes, protect what little exists:

```bash
# Encrypt model weights on disk (at rest)
# Using LUKS for Linux

# Create encrypted partition
cryptsetup luksFormat /dev/nvme0n1
cryptsetup open /dev/nvme0n1 cui-models

# Mount with encryption
mount /dev/mapper/cui-models /mnt/models

# Store model files
cp codellama-34b-cui.safetensors /mnt/models/

# At boot, require key/password to unlock
```

#### C. Memory Encryption (AMD SEV / Intel TDX)

```bash
# AMD SEV (Secure Encrypted Virtualization)
# Encrypts VM memory, even from hypervisor

# VM configuration
qemu-system-x86_64 \
  -machine q35,memory-encryption=sev0 \
  -object sev-guest,id=sev0,cbitpos=47,reduced-phys-bits=1 \
  -drive file=vm-disk.qcow2 \
  ...

# Result: Even cloud provider admin can't read VM memory
```

---

### 6. Access Controls

#### A. Role-Based Access Control (RBAC)

```yaml
# Project Alpha access control
roles:
  - name: project-alpha-developer
    permissions:
      - llm:query  # Can send prompts
      - llm:read_own_history  # Can see own metadata
    restrictions:
      - no_admin_access
      - no_log_access
      - no_other_project_access
  
  - name: project-alpha-lead
    permissions:
      - llm:query
      - llm:read_project_metadata  # See project usage stats
      - llm:manage_quotas
    restrictions:
      - no_content_access  # Cannot see prompts/responses
  
  - name: security-officer
    permissions:
      - audit:read_metadata  # See who/when/what
      - audit:generate_reports
    restrictions:
      - no_content_access  # Cannot see CUI content
      - read_only  # Cannot modify
```

#### B. Attribute-Based Access Control (ABAC)

```python
# Policy: User can only access LLM if:
# 1. User has active clearance
# 2. User is assigned to the project
# 3. Request is from approved network
# 4. Request is during business hours (optional)

def authorize_request(user, request):
    # Check clearance
    if not user.clearance_active():
        return deny("Clearance inactive")
    
    # Check project assignment
    if request.project_id not in user.assigned_projects:
        return deny("Not assigned to project")
    
    # Check network
    if not is_approved_network(request.source_ip):
        return deny("Request from unapproved network")
    
    # Check classification compatibility
    if request.classification > user.clearance_level:
        return deny("Insufficient clearance level")
    
    return allow()
```

---

### 7. Hardware Security

#### A. Trusted Platform Module (TPM)

```bash
# Use TPM for:
# 1. Secure boot verification
# 2. Key storage
# 3. Attestation (prove node hasn't been tampered with)

# Enable TPM in BIOS
# Then use for disk encryption keys

cryptsetup luksFormat /dev/nvme0n1 \
  --type luks2 \
  --pbkdf pbkdf2 \
  --key-file /dev/tpm0
```

#### B. Hardware Security Module (HSM)

```python
# Use HSM for:
# - Certificate management
# - Key generation
# - Cryptographic operations

# Example: Azure Key Vault / AWS CloudHSM
from azure.keyvault.keys import KeyClient
from azure.identity import DefaultAzureCredential

client = KeyClient(
    vault_url="https://cui-keys.vault.azure.net/",
    credential=DefaultAzureCredential()
)

# Generate encryption key in HSM (never leaves hardware)
key = client.create_rsa_key("project-alpha-encryption-key", size=4096)

# Use for encrypting sensitive config
encrypted_config = key.encrypt(sensitive_data)
```

---

## Do You Still Need Local Embeddings?

### Threat Model Analysis

#### Scenario 1: Fully Trusted Corporate Infrastructure

**IF all of the following are true:**
- ✅ Ephemeral processing (no logs)
- ✅ TEE/SGX (admins can't see content)
- ✅ mTLS (encrypted transit)
- ✅ Project isolation (no cross-contamination)
- ✅ Cleared admins only
- ✅ Regular security audits
- ✅ Compliance certification (FedRAMP, etc.)

**THEN:**
- 🟢 **Local embeddings NOT required** for security
- 🟢 Corporate embeddings service can be used safely
- 🟢 CUI is protected by infrastructure controls

**Benefits of corporate embeddings:**
- Faster (GPU-accelerated on cluster)
- Centralized model management
- Consistent embeddings across org
- Easier to upgrade/maintain

#### Scenario 2: Defense in Depth (Recommended)

**Even with trusted corporate infrastructure:**

- 🟡 **Use local embeddings ANYWAY** for defense in depth
- 🟡 Reduces attack surface
- 🟡 Protects against future policy changes
- 🟡 Protects against insider threats
- 🟡 Ensures control even if corporate service is compromised

**Hybrid Approach:**
```yaml
# For UNCLASSIFIED code
embeddings:
  provider: corporate
  endpoint: https://embeddings.defenselab.mil

# For CUI code
embeddings:
  provider: local
  model: sentence-transformers/nomic-embed-text
```

#### Scenario 3: Zero-Trust Model (Maximum Security)

**Assume NOTHING is fully trusted:**

- 🔴 **Local embeddings REQUIRED**
- 🔴 Local vector database REQUIRED
- 🔴 Local MCP server REQUIRED
- 🔴 Corporate LLM used only for generation (final step)

**Reasoning:**
- Only the final synthesis happens on corporate infrastructure
- All retrieval/indexing stays local
- Minimal CUI exposure (only what's needed for answer)

---

## Complete Reference Architecture

### CUI-Safe Corporate LLM (Full Specification)

```yaml
# Corporate LLM Infrastructure Specification
# Classification: CUI-Approved

infrastructure:
  name: DefenseLab CUI LLM Service
  classification: CUI
  compliance:
    - NIST-SP-800-171
    - CMMC-Level-3
    - FedRAMP-Moderate (in progress)

network:
  segmentation:
    - vlan_per_project: true
    - network_policies: strict
    - firewall_rules: deny_all_except_explicit
  
  encryption:
    transit: TLS-1.3
    client_auth: mTLS-required
    cipher_suites: [TLS_AES_256_GCM_SHA384]

compute:
  gpu_nodes:
    - type: NVIDIA-H100-80GB
    - count: 24
    - allocation: dedicated_per_project
    - memory_encryption: AMD-SEV-SNP
    - secure_boot: enabled
    - tpm: required
  
  containerization:
    runtime: containerd
    isolation: kata-containers  # VM-level isolation
    security_context:
      read_only_root: true
      no_persistent_volumes: true
      capabilities_drop: ALL

software:
  llm_engine:
    # OPTION 1: vLLM (Python-based, high throughput)
    backend_vllm:
      engine: vllm
      version: ">=0.4.0"
      config:
        disable_log_requests: true
        disable_log_stats: true
        gpu_memory_utilization: 0.95
        swap_space: 0  # No disk swap
        trust_remote_code: false  # Security
        max_num_seqs: 256  # Concurrent requests
        
    # OPTION 2: llama.cpp server (C++, efficient, GGUF support)
    backend_llamacpp:
      engine: llama-server
      version: ">=b2354"  # Latest stable
      config:
        no_log: true  # No logging
        no_slots_report: true  # No slot status logging
        cont_batching: true  # Continuous batching
        parallel: 4  # Parallel sequences
        cache_type_k: f16  # KV cache precision
        cache_type_v: f16
        mlock: true  # Lock model in RAM
        numa: true  # NUMA optimization
  
  models:
    # vLLM example (safetensors/AWQ)
    - name: codellama-34b-cui-vllm
      backend: vllm
      path: /mnt/models/codellama-34b-instruct-awq
      format: safetensors
      quantization: AWQ
      context_length: 32768
      tensor_parallel: 2  # Multi-GPU
      
    # llama.cpp example (GGUF)
    - name: codellama-34b-cui-gguf
      backend: llama-server
      path: /mnt/models/codellama-34b-instruct-q5_k_m.gguf
      format: GGUF
      quantization: Q5_K_M
      context_length: 32768
      n_gpu_layers: 999  # Offload all layers to GPU

security:
  authentication:
    method: mTLS-client-certificates
    ca: defenselab-root-ca
    cert_rotation: 90_days
  
  authorization:
    model: RBAC+ABAC
    project_isolation: mandatory
    clearance_check: required
  
  audit:
    log_type: metadata_only
    retention: 90_days
    encryption: AES-256
    access: security_officer_only
  
  data_handling:
    ephemeral_processing: enforced
    no_prompt_logging: enforced
    no_response_caching: enforced
    memory_wiping: automatic
  
  encryption:
    at_rest: LUKS2+TPM
    at_rest_key_management: HSM
    in_transit: TLS-1.3+mTLS
    in_memory: AMD-SEV-SNP

compliance:
  frameworks:
    - NIST-SP-800-171: compliant
    - CMMC-Level-3: in_progress
    - FISMA: moderate
  
  assessments:
    penetration_testing: quarterly
    security_audits: monthly
    compliance_review: annual
  
  documentation:
    ssp: security_system_plan.pdf
    poam: plan_of_action.xlsx
    risk_assessment: risk_register.pdf

operations:
  monitoring:
    metrics: prometheus
    alerts: pagerduty
    dashboards: grafana
    log_analytics: splunk  # Metadata only
  
  incident_response:
    playbook: incident_response_plan.pdf
    contacts: [security_team, project_leads]
    escalation: 24x7_soc
  
  backup:
    model_weights: encrypted_backup
    user_data: none  # Ephemeral only
    audit_logs: encrypted_s3

support:
  documentation: https://llm-docs.defenselab.mil
  ticket_system: jira
  sla:
    availability: 99.5%
    response_time_p95: 2000ms
```

---

## Implementation Roadmap

### Phase 1: Minimum Viable CUI-Safe LLM (Months 1-3)

**Goals:**
- ✅ Ephemeral processing (no persistent logs)
- ✅ mTLS authentication
- ✅ Basic project isolation
- ✅ Metadata-only audit logging

**Deliverables:**
1. vLLM cluster with logging disabled
2. mTLS gateway with client certificates
3. Kubernetes namespaces per project
4. Metadata audit system

### Phase 2: Enhanced Security Controls (Months 4-6)

**Goals:**
- ✅ Memory encryption (AMD SEV)
- ✅ Hardware TPM integration
- ✅ Advanced network segmentation
- ✅ ABAC authorization

**Deliverables:**
1. SEV-enabled  compute nodes
2. TPM-backed disk encryption
3. VLAN segmentation per project
4. Policy engine for fine-grained access

### Phase 3: Compliance Certification (Months 7-12)

**Goals:**
- ✅ CMMC Level 3 compliance
- ✅ FedRAMP Moderate (optional)
- ✅ Security audits completed
- ✅ Documentation complete

**Deliverables:**
1. System Security Plan (SSP)
2. Risk assessment
3. Audit reports
4. Compliance certification

---

## Cost-Benefit Analysis

### Corporate CUI-Safe LLM vs Laptop-Only

| Aspect | Corporate LLM (Hardened) | Laptop LLM | Winner |
|--------|-------------------------|------------|---------|
| **Security** | Very High (with TEE/SEV) | Highest (air-gapped) | Laptop |
| **Performance** | High (cluster GPUs) | Medium (laptop GPU) | Corporate |
| **Cost per User** | Low (shared) | High (GPU per user) | Corporate |
| **Availability** | High (99.5% SLA) | Medium (laptop only) | Corporate |
| **Scalability** | Excellent | Poor (1 user = 1 laptop) | Corporate |
| **Maintenance** | Centralized (IT team) | Distributed (each user) | Corporate |
| **Compliance** | Complex but achievable | Simple (isolated) | Laptop |
| **Flexibility** | Configuration drift risk | User-controlled | Laptop |
| **Offline Use** | No (network required) | Yes | Laptop |

**Recommendation:** 
- **Small teams (<10 users):** Laptop LLMs
- **Large orgs (>50 users):** Corporate CUI-Safe LLM
- **Hybrid:** Corporate for general use, laptop for highly sensitive projects

---

## Final Recommendations

### For Local Embeddings Question:

**With properly hardened corporate LLM:**
- 🟢 **Local embeddings NOT strictly required** for security
- 🟡 **Local embeddings still RECOMMENDED** for defense in depth
- 🟢 **Corporate embeddings CAN be used** if infrastructure includes:
  - Ephemeral processing
  - Memory encryption (TEE/SEV)
  - No logging
  - Project isolation

**Decision Matrix:**

| Corporate Infrastructure Maturity | Recommendation |
|----------------------------------|----------------|
| ✅ All security controls from this doc | Corporate embeddings OK |
| 🟡 Most controls, but no TEE/SEV | Local embeddings preferred |
| ❌ Standard corporate LLM (logs enabled) | Local embeddings REQUIRED |

### Best Practice:

**Tiered approach based on data sensitivity:**

```yaml
# Unclassified
embeddings: corporate
vector_db: corporate
llm: corporate

# CUI (Basic)
embeddings: corporate (if hardened)
vector_db: local
llm: corporate (if hardened)

# CUI (Sensitive)
embeddings: local
vector_db: local
llm: corporate (if hardened)

# Classified
embeddings: local
vector_db: local-classified
llm: local-classified (air-gapped)
```

---

## Conclusion

**Yes, a properly designed corporate LLM can handle CUI safely:**

✅ Ephemeral processing eliminates persistent storage risk  
✅ TEE/SEV prevents admin access to content  
✅ mTLS protects in transit  
✅ Project isolation prevents spillage  
✅ Metadata-only logging enables compliance without exposure  

**But:**

⚠️ Requires significant infrastructure investment  
⚠️ Needs ongoing security maintenance  
⚠️ Trust boundary includes corporate IT  

**For maximum security:** Local embeddings + local LLM still wins  
**For practical enterprise use:** Hardened corporate LLM is acceptable  

🎯 **The answer isn't binary - it's a risk/benefit tradeoff based on your threat model and resources.**

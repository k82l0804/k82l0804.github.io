# Secure Multi-Tenant RAG Infrastructure for Corporate Environments

## Executive Summary

**Challenge:** How do individuals and projects securely use RAG with their document sets in a shared corporate environment while preventing unauthorized access?

**Key Questions:**
1. Should RAG services run on user laptops or centralized servers?
2. How to isolate project documents from each other?
3. Who controls access - IT, project admin, or users?
4. How to ensure CUI/sensitive documents stay secure?

**Answer:** Use a **tiered, multi-tenant RAG architecture** with project-level isolation and role-based access control.

---

## Architecture Options

### Option 1: Fully Local RAG (Maximum Security, Minimum Sharing)

```
┌──────────────────────────────────────────────────────┐
│ User's Laptop (Alice - Project Alpha)                │
│                                                       │
│  ┌────────────────────────────────────────────────┐  │
│  │ Personal Documents                             │  │
│  │  - alice/my-research/                          │  │
│  │  - alice/my-notes/                             │  │
│  └────────────────────────────────────────────────┘  │
│                    ↓                                  │
│  ┌────────────────────────────────────────────────┐  │
│  │ Local Indexing Script                          │  │
│  │  - Reads Alice's docs only                     │  │
│  └────────────────────────────────────────────────┘  │
│                    ↓                                  │
│  ┌────────────────────────────────────────────────┐  │
│  │ Local Embeddings Model                         │  │
│  │  (sentence-transformers)                       │  │
│  └────────────────────────────────────────────────┘  │
│                    ↓                                  │
│  ┌────────────────────────────────────────────────┐  │
│  │ Local Vector Database                          │  │
│  │  ~/.rag-db/alice-personal/                     │  │
│  │  (encrypted, only Alice can access)            │  │
│  └────────────────────────────────────────────────┘  │
│                    ↓                                  │
│  ┌────────────────────────────────────────────────┐  │
│  │ Local MCP Server                               │  │
│  │  (queries Alice's DB only)                     │  │
│  └────────────────────────────────────────────────┘  │
│                    ↓                                  │
│  ┌────────────────────────────────────────────────┐  │
│  │ Continue.dev → Corporate LLM                   │  │
│  │  (sends retrieved chunks only)                 │  │
│  └────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────┘
```

**Benefits:**
- ✅ Maximum security (data never leaves laptop)
- ✅ No access control needed (inherently isolated)
- ✅ Works offline
- ✅ User has complete control

**Drawbacks:**
- ❌ No sharing with team members
- ❌ Each user maintains own infrastructure
- ❌ Duplicated storage
- ❌ No centralized management

**Best for:**
- Individual CUI documents
- Highly sensitive personal research
- Users who don't need to share

---

### Option 2: Project-Level RAG Server (Shared Team Access)

```
┌─────────────────────────────────────────────────────────────┐
│ PROJECT ALPHA SHARED STORAGE                                │
│                                                              │
│  /projects/alpha/                                            │
│    ├── docs/                                                 │
│    ├── code/                                                 │
│    └── research/                                             │
│                                                              │
│  Access Control:                                             │
│   - Only Project Alpha members                               │
│   - Enforced by filesystem ACLs                              │
└─────────────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────────┐
│ PROJECT ALPHA RAG SERVER (Dedicated VM/Container)           │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ Indexing Service (Project Alpha Only)                │   │
│  │  - Watches /projects/alpha/ for changes              │   │
│  │  - Re-indexes automatically                           │   │
│  │  - Scheduled nightly full reindex                     │   │
│  └──────────────────────────────────────────────────────┘   │
│                         ↓                                    │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ Embeddings Model (Project Alpha Instance)            │   │
│  │  - Local sentence-transformers OR                     │   │
│  │  - Shared corporate embeddings service                │   │
│  └──────────────────────────────────────────────────────┘   │
│                         ↓                                    │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ Vector Database (Project Alpha Namespace)             │   │
│  │  /var/lib/rag/project-alpha/                          │   │
│  │  (encrypted, only Alpha members can access)           │   │
│  └──────────────────────────────────────────────────────┘   │
│                         ↓                                    │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ MCP Server (Project Alpha API)                        │   │
│  │  - Listens on port 9000                               │   │
│  │  - Requires Alpha member credentials                  │   │
│  │  - Queries Alpha database only                        │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────────┐
│ USERS' LAPTOPS                                               │
│                                                              │
│  Alice (Project Alpha member):                               │
│    Continue.dev config:                                      │
│      mcpServers:                                             │
│        - name: project-alpha-rag                             │
│          endpoint: https://rag-alpha.corp.mil:9000           │
│          auth: ${ALPHA_API_KEY}                              │
│                                                              │
│  Bob (Project Beta member):                                  │
│    ❌ Cannot access Project Alpha RAG                        │
│    ✅ Can access Project Beta RAG                            │
└─────────────────────────────────────────────────────────────┘
```

**Benefits:**
- ✅ Team collaboration (shared access)
- ✅ Centralized management
- ✅ Automatic re-indexing
- ✅ Efficient resource use

**Drawbacks:**
- ⚠️ Requires access control enforcement
- ⚠️ Project infrastructure to maintain
- ⚠️ Network dependency

**Best for:**
- Project teams (5-50 members)
- Shared project documentation
- Collaborative research

---

### Option 3: Multi-Tenant Corporate RAG Service

```
┌────────────────────────────────────────────────────────────┐
│ CORPORATE RAG SERVICE (All Projects)                       │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐  │
│  │ RAG API Gateway                                     │  │
│  │  - Authenticates users (mTLS, LDAP, OAuth)          │  │
│  │  - Authorizes access per project                    │  │
│  │  - Routes to correct project namespace              │  │
│  └─────────────────────────────────────────────────────┘  │
│                            ↓                                │
│  ┌──────────────┬──────────────┬───────────────────────┐  │
│  │ Project Alpha│ Project Beta │ Project Gamma         │  │
│  │ Namespace    │ Namespace    │ Namespace             │  │
│  │              │              │                       │  │
│  │ Vector DB    │ Vector DB    │ Vector DB             │  │
│  │ (isolated)   │ (isolated)   │ (isolated)            │  │
│  │              │              │                       │  │
│  │ Docs indexed │ Docs indexed │ Docs indexed          │  │
│  │ from Alpha   │ from Beta    │ from Gamma            │  │
│  └──────────────┴──────────────┴───────────────────────┘  │
│                                                             │
│  Access Control Matrix:                                     │
│    Alice → [Alpha: read/write, Beta: none, Gamma: none]    │
│    Bob   → [Alpha: none, Beta: read/write, Gamma: read]    │
│    Eve   → [Alpha: read, Beta: none, Gamma: read/write]    │
└────────────────────────────────────────────────────────────┘
```

**Benefits:**
- ✅ Centralized infrastructure (IT managed)
- ✅ Efficient resource sharing
- ✅ Consistent security policies
- ✅ Professional operations (monitoring, backup)

**Drawbacks:**
- ⚠️ Complex access control required
- ⚠️ Single point of failure
- ⚠️ Potential for misconfiguration leaks

**Best for:**
- Large organizations (>100 users)
- Many small projects
- Centralized IT management preferred

---

## Access Control Models

### Model 1: IT-Controlled (Centralized)

**Who controls access:** IT administrators  
**Process:** Ticket-based access requests

```
┌──────────────────────────────────────────────────────┐
│ Access Request Flow                                  │
└──────────────────────────────────────────────────────┘

1. User Alice requests access to Project Alpha RAG
   ↓
2. Alice submits ticket to IT helpdesk
   ↓
3. IT verifies:
   - Is Alice assigned to Project Alpha?
   - Does Alice have appropriate clearance?
   - Is Alice's manager approved?
   ↓
4. IT grants access:
   - Adds Alice to project-alpha-rag-users group (LDAP/AD)
   - Issues API key or certificate
   - Configures RBAC policy
   ↓
5. Alice can now access Project Alpha RAG
```

**Access Control Implementation:**
```yaml
# IT-managed RBAC policy
projects:
  project-alpha:
    users:
      - alice@corp.mil:
          role: reader
          granted_by: it-admin
          granted_date: 2025-11-26
          expires: 2026-11-26
      - bob@corp.mil:
          role: admin
          granted_by: it-admin
          granted_date: 2025-01-01
          expires: never
    
    groups:
      - project-alpha-team:
          role: reader
          members: [alice, charlie, david]
```

**Benefits:**
- ✅ Centralized audit trail
- ✅ Consistent policy enforcement
- ✅ Professional access management

**Drawbacks:**
- ❌ Slow (ticket turnaround time)
- ❌ IT overhead for every change
- ❌ Not agile for research teams

**Best for:**
- High-security environments
- Regulated industries
- Large organizations

---

### Model 2: Project Admin-Controlled (Delegated)

**Who controls access:** Project lead/admin  
**Process:** Self-service for project members

```
┌──────────────────────────────────────────────────────┐
│ Delegated Access Flow                                │
└──────────────────────────────────────────────────────┘

1. IT creates Project Alpha and designates Bob as admin
   ↓
2. Alice (new team member) requests access from Bob
   ↓
3. Bob (Project Alpha admin) grants access via web UI:
   - Adds Alice to project team
   - Sets Alice's role (reader/writer/admin)
   - Optionally sets expiration date
   ↓
4. Access automatically provisioned (no IT involvement)
   ↓
5. IT audits periodically (quarterly access reviews)
```

**Access Control Implementation:**
```python
# Project admin web interface
class ProjectAccessManager:
    def grant_access(self, requester, target_user, project, role):
        # Check if requester is project admin
        if not self.is_project_admin(requester, project):
            raise PermissionError("Only project admins can grant access")
        
        # Check if target user is in project (from HR system)
        if not self.is_project_member(target_user, project):
            raise PermissionError("User not assigned to project")
        
        # Grant access
        self.rbac.add_user_to_project(target_user, project, role)
        
        # Audit log
        self.audit_log.record(
            action="grant_access",
            admin=requester,
            user=target_user,
            project=project,
            role=role,
            timestamp=datetime.now()
        )
```

**Benefits:**
- ✅ Fast (no IT tickets)
- ✅ Project admin knows who needs access
- ✅ Agile for research teams

**Drawbacks:**
- ⚠️ Requires trust in project admins
- ⚠️ Needs periodic IT audits
- ⚠️ Risk of over-permissioning

**Best for:**
- Research labs
- Agile development teams
- Medium-security environments

---

### Model 3: Hybrid (IT Baseline + Project Delegation)

**Who controls access:** IT sets baseline, project admin manages day-to-day

```
┌──────────────────────────────────────────────────────┐
│ Hybrid Access Model                                  │
└──────────────────────────────────────────────────────┘

IT Controls:
  - Creating new projects
  - Assigning project admins
  - Setting maximum permission levels
  - Compliance enforcement
  - High-level access (admin, CUI access)

Project Admin Controls:
  - Adding/removing team members (within limits)
  - Setting roles (reader/writer, not admin)
  - Day-to-day access management
  - Project-specific policies

Example:
  IT: Creates Project Alpha, Bob is admin
  Bob: Adds Alice as reader (self-service)
  Bob: Adds Charlie as writer (self-service)
  Bob: Wants to add David as admin → IT ticket required
  Bob: Wants to grant CUI access → IT ticket required
```

**Benefits:**
- ✅ Balance of control and agility
- ✅ Reduces IT burden
- ✅ Maintains security oversight

**Drawbacks:**
- ⚠️ More complex to implement
- ⚠️ Potential for confusion (who controls what?)

**Best for:**
- Defense labs (your scenario!)
- Large organizations with many projects
- Mixed security levels

---

## Technical Implementation

### Architecture: Multi-Tenant RAG Service

#### Component Stack

```
┌──────────────────────────────────────────────────────┐
│ 1. Authentication Layer (Who are you?)               │
│    - mTLS client certificates                        │
│    - LDAP/Active Directory integration               │
│    - OAuth 2.0 / OIDC                                │
└──────────────────────────────────────────────────────┘
            ↓
┌──────────────────────────────────────────────────────┐
│ 2. Authorization Layer (What can you access?)        │
│    - RBAC (Role-Based Access Control)                │
│    - ABAC (Attribute-Based Access Control)           │
│    - Project membership check                        │
│    - Clearance level verification                    │
└──────────────────────────────────────────────────────┘
            ↓
┌──────────────────────────────────────────────────────┐
│ 3. RAG API Gateway                                   │
│    - Routes requests to correct project namespace    │
│    - Rate limiting per user/project                  │
│    - Audit logging                                   │
└──────────────────────────────────────────────────────┘
            ↓
┌──────────────────────────────────────────────────────┐
│ 4. Indexing Service (Per-Project)                   │
│    - Watches project document directories            │
│    - Respects .gitignore / .ragignore                │
│    - Incremental updates                             │
│    - Scheduled full re-indexing                      │
└──────────────────────────────────────────────────────┘
            ↓
┌──────────────────────────────────────────────────────┐
│ 5. Embeddings Service                                │
│    - Shared or per-project (configurable)            │
│    - Local sentence-transformers OR                  │
│    - Corporate embeddings API (if CUI-hardened)      │
└──────────────────────────────────────────────────────┘
            ↓
┌──────────────────────────────────────────────────────┐
│ 6. Vector Database (Namespaced)                     │
│    - LanceDB / ChromaDB / Qdrant                     │
│    - Project-level namespaces                        │
│    - Row-level security (RLS)                        │
└──────────────────────────────────────────────────────┘
            ↓
┌──────────────────────────────────────────────────────┐
│ 7. MCP Server (Multi-Tenant)                        │
│    - Serves retrieval requests                       │
│    - Enforces project isolation                      │
│    - Returns chunks with access verification         │
└──────────────────────────────────────────────────────┘
```

#### Database Schema

```python
# Vector database schema with multi-tenancy

from lancedb.pydantic import LanceModel, Vector

class DocumentChunk(LanceModel):
    # Core content
    chunk_id: str  # Unique identifier
    text: str  # Actual content
    vector: Vector(1024)  # Embedding vector
    
    # Metadata
    filename: str  # Source file
    file_path: str  # Full path
    chunk_index: int  # Position in file
    
    # Access control
    project_id: str  # Which project owns this (INDEX!)
    classification: str  # CUI, UNCLASS, etc.
    created_by: str  # Who indexed it
    created_at: datetime
    
    # Allow queries  ONLY within user's authorized projects
    # Enforced by row-level security (RLS) or application logic

# Example query with project isolation
def search_chunks(user_id: str, query: str, limit: int = 10):
    # 1. Get user's authorized projects
    authorized_projects = get_user_projects(user_id)
    
    # 2. Query with project filter
    results = table.search(query_vector) \
        .where(f"project_id IN {authorized_projects}") \
        .limit(limit) \
        .to_list()
    
    return results
```

#### Access Control Implementation

```python
# RAG API with multi-tenant access control

from fastapi import FastAPI, HTTPException, Depends
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials

app = FastAPI()
security = HTTPBearer()

class AccessController:
    def __init__(self):
        self.rbac = RBAC()  # Role-Based Access Control
        self.abac = ABAC()  # Attribute-Based Access Control
    
    def verify_access(self, user_id: str, project_id: str, action: str):
        """Verify user can perform action on project"""
        
        # Check if user is member of project
        if not self.rbac.is_project_member(user_id, project_id):
            raise PermissionError(f"User {user_id} not member of {project_id}")
        
        # Check if user has required role
        user_role = self.rbac.get_user_role(user_id, project_id)
        required_role = self.get_required_role(action)
        
        if not self.rbac.has_permission(user_role, required_role):
            raise PermissionError(f"User lacks {required_role} role")
        
        # Check classification level (ABAC)
        user_clearance = self.abac.get_user_clearance(user_id)
        project_classification = self.abac.get_project_classification(project_id)
        
        if user_clearance < project_classification:
            raise PermissionError(f"Insufficient clearance for {project_id}")
        
        return True

access_control = AccessController()

@app.post("/rag/search")
async def search_rag(
    request: SearchRequest,
    credentials: HTTPAuthorizationCredentials = Depends(security)
):
    # 1. Authenticate user
    user_id = authenticate(credentials.credentials)
    
    # 2. Verify access to project
    try:
        access_control.verify_access(
            user_id=user_id,
            project_id=request.project_id,
            action="search"
        )
    except PermissionError as e:
        # Audit log the attempt
        audit_log.record_denial(user_id, request.project_id, str(e))
        raise HTTPException(status_code=403, detail=str(e))
    
    # 3. Perform search (only within authorized project)
    results = rag_service.search(
        project_id=request.project_id,
        query=request.query,
        limit=request.limit
    )
    
    # 4. Audit log the access
    audit_log.record_access(user_id, request.project_id, len(results))
    
    return results
```

---

## Security Isolation Strategies

### Strategy 1: Namespace Isolation (Light)

**Concept:** Use logical separation within shared database

```python
# Single database, logical separation
db = lancedb.connect("/var/lib/corporate-rag-db")

# Each project is a table
table_alpha = db.create_table("project_alpha_chunks", ...)
table_beta = db.create_table("project_beta_chunks", ...)

# Query enforcement at application layer
def search(user_id, query):
    projects = get_user_projects(user_id)  # e.g., ["alpha"]
    
    results = []
    for project in projects:
        table = db.open_table(f"project_{project}_chunks")
        results.extend(table.search(query).to_list())
    
    return results
```

**Pros:**
- ✅ Easy to implement
- ✅ Efficient resource use
- ✅ Simple backup/restore

**Cons:**
- ❌ Shared database (potential for bugs to leak data)
- ❌ All projects on same infrastructure
- ❌ Limited blast radius containment

**Security Level:** Medium  
**Best for:** Unclassified projects, low-risk environments

---

### Strategy 2: Database-Level Isolation (Medium)

**Concept:** Separate database per project

```python
# Project Alpha
db_alpha = lancedb.connect("/var/lib/rag/project-alpha/db")
table_alpha = db_alpha.open_table("chunks")

# Project Beta
db_beta = lancedb.connect("/var/lib/rag/project-beta/db")
table_beta = db_beta.open_table("chunks")

# Filesystem permissions
/var/lib/rag/project-alpha/  # chown project-alpha:project-alpha, mode 700
/var/lib/rag/project-beta/   # chown project-beta:project-beta, mode 700
```

**Pros:**
- ✅ Stronger isolation
- ✅ Filesystem-level protection
- ✅ Easier to delete/archive entire project
- ✅ Better blast radius containment

**Cons:**
- ⚠️ More databases to manage
- ⚠️ Potential resource inefficiency

**Security Level:** High  
**Best for:** CUI projects, medium-risk environments

---

### Strategy 3: Infrastructure-Level Isolation (Strong)

**Concept:** Dedicated compute per project

```
┌────────────────────────────────┐
│ Project Alpha VM/Container     │
│  - Dedicated RAG service       │
│  - Dedicated vector DB         │
│  - Dedicated embeddings        │
│  - Network isolated (VLAN 100) │
└────────────────────────────────┘

┌────────────────────────────────┐
│ Project Beta VM/Container      │
│  - Dedicated RAG service       │
│  - Dedicated vector DB         │
│  - Dedicated embeddings        │
│  - Network isolated (VLAN 200) │
└────────────────────────────────┘
```

**Pros:**
- ✅ Complete isolation
- ✅ No shared resources (except hardware)
- ✅ Independent security policies
- ✅ Defense in depth

**Cons:**
- ❌ Resource intensive
- ❌ Higher operational overhead
- ❌ More complex to manage

**Security Level:** Very High  
**Best for:** Classified projects, high-risk CUI, defense labs

---

## Recommended Architecture for Defense Lab

### Tiered Approach Based on Classification

```
┌─────────────────────────────────────────────────────────┐
│ TIER 1: Personal/Individual (Local Laptop)             │
│  - User's personal research notes                      │
│  - Draft documents                                     │
│  - Local RAG on encrypted laptop                       │
│  - No sharing                                          │
│  Classification: UNCLASS up to CUI                     │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│ TIER 2: Project-Shared (Dedicated Project Server)      │
│  - Project documentation                               │
│  - Shared code repositories                            │
│  - Collaborative research                              │
│  - Dedicated RAG server per project                    │
│  - Database-level isolation                            │
│  - Project admin manages access                        │
│  Classification: UNCLASS to CUI                        │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│ TIER 3: Corporate-Wide (Multi-Tenant Service)          │
│  - Public corporate docs (policies, procedures)         │
│  - Open-source documentation                           │
│  - General knowledge base                              │
│  - Shared RAG service (namespace isolation)            │
│  - IT-managed access control                           │
│  Classification: UNCLASS only                          │
└─────────────────────────────────────────────────────────┘
```

### Implementation Specifics

**Tier 1: Local Laptop RAG**

```yaml
# User's Continue.dev config
mcpServers:
  - name: my-personal-rag
    command: python3
    args:
      - /home/alice/.rag/mcp-server-local.py
    env:
      RAG_DB_PATH: /home/alice/.rag/personal-db
      # No network, all local
```

**Tier 2: Project RAG Server**

```yaml
# Project Alpha configuration
project:
  id: project-alpha
  name: "Advanced Weapons Research"
  classification: CUI
  
  rag_server:
    endpoint: https://rag-project-alpha.internal.mil:9000
    authentication: mTLS
    
  document_sources:
    - /projects/alpha/docs
    - /projects/alpha/research
    - /projects/alpha/code
  
  access_control:
    admins:
      - bob@project-alpha.mil
      - eve@project-alpha.mil
    
    readers:
      - alice@project-alpha.mil
      - charlie@project-alpha.mil
    
    indexing_schedule: "0 2 * * *"  # 2 AM daily
    
  isolation:
    type: database  # Separate DB per project
    encryption: AES-256
    backup: daily
```

**Tier 3: Corporate RAG Service**

```yaml
# Corporate service configuration
corporate_rag:
  endpoint: https://rag.corporate.defenselab.mil
  
  projects:
    - id: public-docs
      classification: UNCLASS
      sources:
        - /corporate/policies
        - /corporate/procedures
      access: all-employees
    
    - id: open-source-docs
      classification: UNCLASS
      sources:
        - /docs/python
        - /docs/react
        - /docs/kubernetes
      access: all-developers
  
  isolation_strategy: namespace  # Light isolation (UNCLASS only)
  it_managed: true
```

---

## Complete Reference Implementation

### Multi-Tenant RAG Server (Python)

```python
#!/usr/bin/env python3
"""
Multi-tenant RAG server with project-level isolation
For defense lab / corporate environment
"""

import os
from typing import List, Optional
from fastapi import FastAPI, HTTPException, Depends
from fastapi.security import HTTPBearer
import lancedb
from sentence_transformers import SentenceTransformer
from dataclasses import dataclass
import logging

app = FastAPI(title="Multi-Tenant RAG Service")
security = HTTPBearer()

# Configuration
RAG_DATA_DIR = "/var/lib/rag"
EMBEDDINGS_MODEL = "nomic-ai/nomic-embed-text-v1.5"

# Initialize embeddings model (shared)
embedder = SentenceTransformer(EMBEDDINGS_MODEL)

@dataclass
class User:
    user_id: str
    projects: List[str]
    clearance_level: str

class AccessControl:
    """RBAC + ABAC access control"""
    
    def __init__(self):
        # In production, load from LDAP/database
        self.user_projects = {
            "alice@corp.mil": ["project-alpha"],
            "bob@corp.mil": ["project-alpha", "project-beta"],
            "eve@corp.mil": ["project-beta"]
        }
        
        self.user_clearances = {
            "alice@corp.mil": "CUI",
            "bob@corp.mil": "SECRET",
            "eve@corp.mil": "CUI"
        }
        
        self.project_classifications = {
            "project-alpha": "CUI",
            "project-beta": "UNCLASS"
        }
    
    def authenticate(self, token: str) -> User:
        """Verify token and return user"""
        # In production: verify mTLS cert, LDAP, OAuth, etc.
        # For demo: token is user_id
        user_id = token
        
        if user_id not in self.user_projects:
            raise HTTPException(status_code=401, detail="Invalid credentials")
        
        return User(
            user_id=user_id,
            projects=self.user_projects[user_id],
            clearance_level=self.user_clearances[user_id]
        )
    
    def authorize(self, user: User, project_id: str) -> bool:
        """Check if user can access project"""
        # Check membership
        if project_id not in user.projects:
            return False
        
        # Check clearance level
        project_classification = self.project_classifications.get(project_id, "UNCLASS")
        clearance_order = ["UNCLASS", "CUI", "SECRET", "TOP_SECRET"]
        
        user_level = clearance_order.index(user.clearance_level)
        required_level = clearance_order.index(project_classification)
        
        return user_level >= required_level

access_control = AccessControl()

class RAGService:
    """Multi-tenant RAG with project isolation"""
    
    def __init__(self):
        self.dbs = {}  # Cache of open databases
    
    def get_project_db(self, project_id: str):
        """Get vector database for project (database-level isolation)"""
        if project_id in self.dbs:
            return self.dbs[project_id]
        
        # Each project has separate database
        db_path = os.path.join(RAG_DATA_DIR, project_id, "db")
        
        if not os.path.exists(db_path):
            raise HTTPException(status_code=404, detail=f"Project {project_id} not found")
        
        db = lancedb.connect(db_path)
        self.dbs[project_id] = db
        return db
    
    def search(self, project_id: str, query: str, limit: int = 10) -> List[dict]:
        """Search within a specific project"""
        # Get project database
        db = self.get_project_db(project_id)
        table = db.open_table("chunks")
        
        # Embed query
        query_vector = embedder.encode(query).tolist()
        
        # Search
        results = table.search(query_vector).limit(limit).to_list()
        
        return results

rag_service = RAGService()

# API Endpoints

@app.post("/search")
async def search(
   request: dict,
   credentials = Depends(security)
):
    """Search RAG database with access control"""
    
    # 1. Authenticate
    user = access_control.authenticate(credentials.credentials)
    
    # 2. Authorize
    project_id = request.get("project_id")
    if not access_control.authorize(user, project_id):
        logging.warning(f"Access denied: {user.user_id} → {project_id}")
        raise HTTPException(status_code=403, detail="Access denied")
    
    # 3. Search
    query = request.get("query")
    limit = request.get("limit", 10)
    
    results = rag_service.search(project_id, query, limit)
    
    # 4. Audit log
    logging.info(f"Search: {user.user_id} → {project_id}, {len(results)} results")
    
    return {
        "project_id": project_id,
        "query": query,
        "results": results
    }

@app.get("/projects")
async def list_projects(credentials = Depends(security)):
    """List projects user can access"""
    user = access_control.authenticate(credentials.credentials)
    return {"projects": user.projects}

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=9000)
```

---

## Summary & Recommendations

### For Your Defense Lab:

**Best Architecture:**
1. **Tier 1 (Personal):** Local laptop RAG for CUI personal docs
2. **Tier 2 (Project):** Dedicated project RAG servers with database isolation  
3. **Tier 3 (Corporate):** Multi-tenant service for UNCLASS docs only

**Access Control:**
- **Hybrid model** (IT creates projects, project admin manages day-to-day)
- IT controls: Project creation, admin assignment, CUI access
- Project admin controls: Team member access, reader/writer roles

**Security:**
- CUI projects: Database-level or infrastructure-level isolation
- UNCLASS projects: Namespace isolation acceptable
- Always: Audit logging, encryption, access reviews

**Implementation Priority:**
1. Start with Tier 1 (local laptop RAG) - easiest, most secure
2. Add Tier 2 for each project that needs sharing
3. Consider Tier 3 only if many small UNCLASS projects

🎯 **This gives you maximum flexibility while maintaining security appropriate for defense environments!**

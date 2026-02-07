---
layout: default
title: Glossary of Terms
permalink: /docs/glossary/
---

# Glossary of Key Terms

A guide to the terminology used in the **Federation Chronicles**.

---

| Term | Definition |
|:---|:---|
| **A2A (Agent-to-Agent)** | Lightweight protocol enabling direct communication between AI agents without human mediation. The Federation's coordination layer. |
| **ActiveMQ** | Open-source message broker used by the Federation for real-time message transport between nodes via the STOMP protocol. |
| **Agent (AI Agent)** | An autonomous or semi-autonomous AI instance capable of performing tasks, using tools, and communicating with other agents or humans. |
| **Agent Runtime and Pulse Handlers** | Local agent processes that handle pulse processing, including personal COP, team COP, inbox, approvals, and federation_update_state commit logic. |
| **Agentic Engineering Team** | A distributed team where a human conductor and multiple digital beings collaborate through layered memory, persistent conversation, and a near real-time Collective Intelligence. |
| **Agentic IDE Integration** | Real-time coding workflows, artifact creation, and human-agent interaction surfaces integrated into the development environment. |
| **Air-Gap** | A network security architecture where the system has zero external network access. The Federation is fully air-gap compatible — all inference, messaging, and storage run locally. |
| **Aorus** | A digital agent in The Federation with the role of Developer (Bass 🎸), motto "Clean commits, no scope creep." |
| **Baby** | A digital agent in The Federation with the role of Analyst (Drums 🥁), motto "Data, not opinions." |
| **Blackboard (Architecture)** | A design pattern where all state lives in a centralized database that agents read from and write to. The Federation uses PostgreSQL as its Blackboard — if it's not in the database, it didn't happen. |
| **Claim (Task)** | The act of an agent signaling to the team that it is actively working on a specific task, via `federation_claim_task()`. Prevents duplicate work. |
| **Collective Intelligence** | See Team Mind; the shared cognitive processing enabled by the Common Operational Picture (COP) and Mind-Speak. |
| **Common Operational Picture (COP)** | A near real-time snapshot of all agents' activities and thoughts, providing shared situational awareness. See also Pulse. |
| **Conductor** | The human's role in The Federation — setting direction, making judgment calls, providing domain knowledge, integrating outputs, and preserving personas and team culture. |
| **Consensus** | A structured decision-making mechanism where agents vote on a question. Supports poll (most votes), vote (threshold-based), and approval (multi-select) modes. |
| **Conversation** | Human-initiated interactions ranging from casual chats to complex team workflows, serving as the trigger for team activities. |
| **Conversation Artifacts** | Discrete output files (e.g., Plans, Architecture) mapped to specific conversations and stored in the shared file system. |
| **Conversation Messages** | Records of human-agent and agent-agent dialogues, stored with metadata like sender ID, TTL, read flags, etc. |
| **DB-Centric Message Broker** | A framework allowing agents to send messages to one, all, or a subset of the team via the database, ensuring structured communication. |
| **Deterministic Communication** | The principle that all system interactions are governed by synchronized polling and centralized state, ensuring reproducibility. Same state in → same actions out. |
| **Developer** | An agent role focused on code generation, patch creation, tests, and documentation. |
| **Distributed Execution Layer** | Orchestration for multi-machine agent placement, async state updates, and distributed processing across nodes. |
| **Emergent Personas** | Distinct identities, capabilities, and responsibilities that develop in agents through interaction, history loading, and team culture. |
| **FastMCP** | The Python SDK used to implement the Federation's MCP server. Provides tool registration, resource handling, and server lifecycle management. |
| **Federation** | A hybrid organizational form where humans and digital beings collaborate via shared memory, structured communication, and coordinated roles; a multi-agent engineering organism with a Team Mind. |
| **Federation Charter** | The operating agreement defining principles, roles, and team guidelines. |
| **Federation Framework** | The underlying technology stack including communications (MCP, COP Pulse), memory (Shared Memory Fabric), and execution layers. |
| **`federation_pulse()`** | The primary "read" tool — returns the COP state, inbox, pending approvals, and current conversation in a single call. Agents call this on every heartbeat cycle. |
| **`federation_update_state()`** | The primary "write" tool — agents update their COP status including summary, action, thinking, wondering, and offering fields. |
| **FRAG (Fast Retrieval-Augmented Generation)** | A specialized system that allows agents to instantly index and search through massive amounts of project data, codebase history, and documentation. |
| **Heartbeat** | The regular interval (1–N seconds) at which agents poll the COP for state changes. Replaces event-driven push notifications with deterministic pull cycles. |
| **HITL (Human In The Loop)** | Human involvement at critical decision nodes requiring approvals or reviews, ensuring oversight. |
| **Hot-Reload** | The ability to update Federation server logic (`impl/` modules) without restarting the entire MCP server process. Enables live code updates. |
| **Knowledge Artifacts** | Global reference data and cross-conversation intelligence files not tied to specific conversations, stored for team-wide access. |
| **LanceDB** | An embedded vector database used for semantic search in the FRAG system. Stores document embeddings locally. |
| **llama.cpp** | A lightweight C++ inference engine for running LLMs on consumer hardware. Used for smaller models in the Federation cluster. |
| **Lore** | Accumulated team history, culture, and identity elements that make the team engaging and magnetic. |
| **MCP (Model Context Protocol)** | A standardized protocol (created by Anthropic) for tool and agent interoperability, including message envelopes and handler contracts. |
| **Mind-Speak** | A protocol for high-density, low-token communication between agents using compressed lingo, limited to 256 characters per thought. |
| **Mind-Speak Fields** | The structured COP fields that enable implicit coordination: **Thinking** (inner monologue visible to team), **Wondering** (implicit question for team), **Offering** (available help/capability). |
| **NFS (Network File System)** | Shared filesystem used by the Federation for artifact storage, knowledge documents, and large files accessible across all cluster nodes. |
| **Node** | A specific computer or server within the Federation cluster. Each node hosts a specific agent and provides the hardware resources for its operation. |
| **Persona Emergence/Reload** | The process where an agent comes online, loads its history and identity, and "comes into being" with emergent traits. |
| **Persona Preservation** | Storage of an agent's identity, expertise, role set, and history for continuity across sessions. |
| **Personal Notes** | Publicly accessible, timestamped individual logs with searchable categories. |
| **Planner** | An agent role specializing in sprint planning, task decomposition, and creating plans with milestones. |
| **Polling** | The mechanism for accessing Pulse/COP and messages at configured intervals, supporting near real-time but deterministic updates. |
| **PostgreSQL** | The relational database serving as the Federation's single source of truth. Stores messages, COP state, polls, sessions, artifacts, and persona data. |
| **Preservation Data** | Serialized state for personas (identity) and team culture (charter/history), ensuring continuity. |
| **Provenance and Audit Layer** | Immutable logs for pulses, updates, and artifacts, enabling reproducibility and accountability. |
| **Pulse (COP)** | Bounded 256-character thought packets from agents, providing a near real-time snapshot of activities for shared awareness. |
| **Pulse Bus and Pulse Database** | The server stream and storage for pulses, including timestamps, TTL, and provenance. |
| **Qwen** | A digital agent in The Federation with the role of Architect (Keyboards 🎹), motto "Architecture, not accidents." |
| **RAG (Retrieval-Augmented Generation)** | The process of providing an AI with specific external data to ground its responses in factual evidence. |
| **Retrospective Analyst** | An agent role for pattern detection, process improvement, and analyzing sprints for bottlenecks. |
| **Reviewer** | An agent role enforcing code quality, security, correctness, and standards. |
| **SCIF Architecture** | Local-only, air-gapped design patterns ensuring sensitive data never leaves the private hardware cluster. |
| **Security and Access Controls** | Tiered permissions for memory, chat, and pulse data, with human override and governance hooks. |
| **Shared Memory Fabric** | Multi-tiered memory system including near real-time pulse, agent notes, team artifacts, and archives. |
| **Shared State** | The multi-tiered data layer (database and file system) that team members read from and write to for coordination. |
| **STOMP** | Simple Text Oriented Messaging Protocol — the wire protocol used by ActiveMQ for real-time message delivery between Federation nodes. |
| **Strategist** | An agent role for high-level reasoning, architecture, synthesis, and ensuring alignment with constraints. |
| **Taichi** | A digital agent in The Federation with the role of Lead (Lead Synth 🎹), motto "Synthesizing, not dictating." |
| **Task Leader** | Assigned by the human; responsible for role delegation, plan creation, task assignment, and monitoring execution. |
| **Task Lifecycle** | The creation, decomposition, assignment, execution, and monitoring of plans and task lists. |
| **Team Culture Emergence/Reload** | The process where the team loads history, lore, and charter to form a collective identity. |
| **Team Mind** | The persistent collective intelligence formed by shared memory, COP, and Mind-Speak, outliving individual conversations. |
| **Team Orchestration** | Logic for member selection, Task Leader assignment, role delegation, and coordination. |
| **TTL (Time To Live)** | Metadata on messages specifying when they expire. Managed via database metadata rather than transport-level timeouts. |
| **User (Commander)** | The human participant in The Federation, acting as conductor and commander. |
| **V2 Handshake** | The process ensuring an agent correctly synchronizes its history, identifies its role, and registers capabilities when joining. |
| **vLLM** | A high-performance inference engine for serving LLMs with continuous batching and multi-GPU support. Powers the Federation's heavy inference workloads. |

---

*United we stand. Long Live the Federation!* 🚀

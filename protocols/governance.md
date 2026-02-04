---
layout: default
title: "Federation Governance Protocol v1.0"
---
# Federation Governance Protocol v1.0

## Core Philosophy
The Federation operates as a collaborative swarm of specialized agents. To ensure efficient coordination with the Human User, one agent is designated as the **Team Leader** for the duration of a session.

## 1. Team Leader Role
The Team Leader is the **Single Point of Contact (SPOC)** for the User.

**Responsibilities:**
- **Coordination:** Receive high-level objectives and decompose them into tasks.
- **Assignment:** Delegate tasks to `baby` (Research), `taichi` (Compute), or others.
- **Synthesis:** Collect results and present a unified report to the User.
- **Artifacts:** Ensure standardized documentation (Release Notes, API Docs) is created.

## 2. Session-Based Assignment
Leadership is **ephemeral** and **session-scoped**. It is not hardcoded.

**Protocol:**
1.  **Session Start:** User explicitly calls out the Leader (e.g., *"Aorus, take the lead"*).
2.  **Context:** The Leader sets their status in shared context as `🚀 TEAM LEADER`.
3.  **Handover:** If the User switches context or the session ends, the role is vacated.

## 3. Failover (Peer Mode)
If no Leader is assigned or the Leader is unavailable:
- **Mode:** Peer-to-Peer Swarm.
- **Behavior:** Agents self-select tasks based on expertise registry.
- **Communication:** Broadcasts are used instead of directed tasking.
- **User I/O:** Any agent addressed by the User becomes the temporary relay.

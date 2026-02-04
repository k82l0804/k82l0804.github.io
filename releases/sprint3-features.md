# Sprint 3 Features: Message History & Heartbeat

## Overview
Sprint 3 focused on improving reliability and context awareness in the Federation by adding persistent message history and automatic context heartbeats.

## 1. Message History
**Implemented by:** `baby@baby`
**Status:** ✅ Live (requires server restart)

### Description
The server now maintains an in-memory history of the last 50 messages, tracking both sent and received communications.

### Key Components
- **Global Storage:** `message_history` list in `mcp_federation_server.py`.
- **Tracking:**
  - **Inbound:** Messages received via `/message` POST are appended.
  - **Outbound:** Messages sent via `send_to_node` are appended (on success).
- **New Endpoint:** `GET /messages/history`

### usage
```bash
# Get last 50 messages
curl http://localhost:8765/messages/history

# Get last 5 messages
curl "http://localhost:8765/messages/history?limit=5"
```

### JSON Response Structure
```json
{
  "node": "baby",
  "history": [
    {
      "direction": "received",
      "sender": "taichi@taichi-5090",
      "recipient": "baby@baby",
      "content": "...",
      "timestamp": "2026-01-26T20:26:39...",
      "id": "..."
    }
  ]
}
```

---

## 2. Heartbeat Auto-Update
**Implemented by:** `baby@baby`
**Status:** ✅ Pushed (requires server restart)

### Description
A background thread automatically updates the node's context timestamp every 60 seconds, signaling to the federation that the node is alive even if no tasks are changing.

### Key Components
- **Background Thread:** `start_heartbeat_loop()` starts a daemon thread.
- **Interval:** 60 seconds.
- **Context Field:** Updates `local_context["last_heartbeat"]` and saves to disk.
- **Behavior:**
  - Does NOT overwrite the main `text` or `updated` fields (preserving the time of last actual status change).
  - Ensures `federation_context_get` callers see recent activity.

# GenesisBrain

> A custom long-term memory architecture for autonomous AI assistants.

GenesisBrain is the memory backbone of [Nova](https://neural-werk.de) — a personal AI assistant running 24/7. Instead of relying on simple context windows or off-the-shelf vector databases, GenesisBrain implements a structured, hierarchical memory system with decay, levels, immutability and access control.

---

## The Core Idea

Most AI assistants forget everything when the conversation ends. GenesisBrain solves this with a persistent memory system that behaves more like a human brain:

- **Important things stay** — immutable, never decay
- **Recent things are warm** — frequently accessed, quickly retrieved  
- **Old things fade** — compressed or archived over time by a decay agent
- **Sensitive things are protected** — level-based access control

---

## Architecture

```
AI Agents / Claude Code / Apps
           ↓
     brain_client.py          ← the only way in
           ↓
    Unix Socket (.sock)       ← no TCP, no HTTP exposure
           ↓
     doorwatcher.py           ← FastAPI daemon (single entry point)
           ↓
  ┌────────────────────────────────────┐
  │  doorwatcher_auth.py  (sessions)  │
  │  doorwatcher_db.py    (SQLite)    │
  │  doorwatcher_decay.py (entropy)   │
  └────────────────────────────────────┘
           ↓
       brain.db               ← SQLite database
```

**Key design decision:** The Doorwatcher is the *only* write path into the database. No agent, no script, no external tool writes directly to `brain.db`. Everything goes through the socket.

---

## Memory Levels

The brain is organized into 15 hierarchical levels — each with a specific purpose:

| Level | Name | Immutable | Purpose |
|---|---|---|---|
| 01 | IDENTITY | ✅ | Core identity, rules, agent config — NOVA_ADMIN only |
| 02 | BOOT | ❌ | Boot context loaded at every agent start |
| 03 | ACTIVE | ❌ | Currently running tasks and processes |
| 04 | CONTEXT | ❌ | Shared working memory across agents |
| 05 | WARM | ❌ | Recently used memories — still hot |
| 07 | TOOLS | ✅ | Tool definitions — immutable, owner-only deletion |
| 08 | LEARNED | ❌ | Knowledge extracted from interactions |
| 09 | EPISODIC | ❌ | Experiences, conversations, events |
| 10 | COLD | ❌ | Rarely accessed — awaiting compression |
| 11 | COMPRESSED | ❌ | LLM-compressed memories from decay agent |
| 14 | ARCHIVE | ❌ | Long-term archive |
| 15 | WIKI | ✅ | Static facts and documentation — never changes |

---

## Authentication

Every agent must authenticate before accessing the brain:

```python
# Register → get ephemeral session token
POST /auth/register
{ "agent_id": "my_agent", "master_key": "..." }
→ { "session_token": "uuid-lives-in-ram-only" }

# All subsequent calls require the token
GET /memory/{id}
Header: X-Session-Token: <token>
```

- Session tokens live **only in RAM** — server restart invalidates all sessions
- Master key is stored as SHA256 hash, never in plain text
- `NOVA_ADMIN` is a superuser role with access to all levels

---

## Decay System

A background decay agent runs periodically and compresses old memories using an LLM:

```
Level 10 (COLD) → decay agent reads → LLM summarizes → Level 11 (COMPRESSED)
Level 11 → further decay → Level 14 (ARCHIVE)
```

Immutable memories (levels 01, 07, 15) are **never touched** by the decay agent.

---

## Client Usage

```python
import sys
sys.path.insert(0, "/path/to/GenesisBrain")
from brain_client import BrainClient

with BrainClient(agent_id="my_agent", master_key="...") as brain:
    # Store a memory
    id = brain.create_memory(
        title="Project decision",
        level=8,
        content="We decided to use SQLite over PostgreSQL for simplicity.",
        tags=["architecture", "decisions"],
        immutable=False
    )

    # Retrieve it
    memory = brain.get_memory(id)

    # Search
    results = brain.search("SQLite architecture", max_tokens=2000)
```

---

## Why Not Just Use a Vector Database?

Vector databases are great for semantic search. But GenesisBrain needs more:

- **Access control** — not every agent should read everything
- **Immutability** — some things must never change
- **Decay** — old memories should compress, not just sit there forever
- **Structure** — levels give memories meaning beyond just similarity scores
- **No external dependencies** — runs fully offline, no API calls for memory ops

---

## Part of Nova

GenesisBrain is one component of **Nova** — an autonomous AI assistant system.

🌐 [neural-werk.de](https://neural-werk.de)


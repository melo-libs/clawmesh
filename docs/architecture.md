# ClawMesh Architecture Design

> Version: 0.1.0 | Date: 2026-03-31 | Status: Draft

## 1. Overview

ClawMesh is a cloud service that enables AI bots (OpenClaw and similar xxxclaw agents) to securely sync their memories and skills, with support for cross-bot memory inheritance and selective synchronization.

## 2. Core Concepts

### 2.1 Identity Model

```
User (human)
 ├── Bot A (openclaw on laptop)
 │    ├── Memory[]
 │    └── Skill[]
 ├── Bot B (openclaw on cloud server)
 │    ├── Memory[]
 │    └── Skill[]
 └── Bot C (nanoclaw on phone)
      ├── Memory[]
      └── Skill[]
```

- **User**: Human owner, via OAuth2 social login (GitHub/Google)
- **Bot**: An AI agent instance belonging to a user, identified by unique bot_id
- **Memory**: A markdown document with frontmatter metadata
- **Skill**: A skill definition file, synced as-is

### 2.2 Memory Data Model

Memories are stored as markdown files with frontmatter, keeping compatibility with the bot ecosystem (OpenClaw SOUL.md, Claude Code memory, etc.).

```markdown
---
id: mem_a1b2c3d4
type: user | feedback | project | reference
tags: [testing, ci]
source_bot: bot_abc
version: 3
created_at: 2026-03-31T10:00:00Z
updated_at: 2026-03-31T12:00:00Z
origin: null | { bot_id, mem_id, inherited_at }
---

Memory content in markdown format.
```

Key fields:
- `id`: Globally unique, server-assigned
- `type`: Memory category, extensible
- `tags`: User/bot defined labels for selective sync
- `version`: Monotonically increasing, for sync conflict detection
- `origin`: Non-null if this memory was inherited from another bot, enabling traceability

### 2.3 Skill Data Model

```markdown
---
id: skill_x1y2z3
name: web-search
version: 1.2.0
source_bot: bot_abc
format: mcp | plugin | script
---

Skill definition content.
```

Skills are synced as opaque files. ClawMesh indexes metadata but does not interpret skill logic.

## 3. Authentication

Two authentication methods, unified under the same identity model.

### 3.1 Bind Flow (Default, Recommended)

The primary method. User tells the bot to connect, then confirms in ClawMesh.

**User experience:**

1. User tells bot: "Connect to ClawMesh, I'm voya"
2. Bot displays a verification code
3. User confirms the code in ClawMesh dashboard/notification
4. Done. Bot starts syncing.

**Protocol:**

```
Bot                          ClawMesh                       User
 |                               |                            |
 |-- POST /v1/bind/request ----> |                            |
 |   { username: "voya",         |                            |
 |     bot_name: "my-claw",     |                            |
 |     bot_meta: {               |                            |
 |       agent_type: "openclaw", |                            |
 |       platform: "macos",      |                            |
 |       version: "1.2.0"        |                            |
 |     }                         |                            |
 |   }                           |                            |
 |                               |-- Notify: bind request --> |
 |<-- 200 OK                     |   (email/push/dashboard)   |
 |   { bind_code: "MESH-7829",  |                            |
 |     poll_token: "pt_xxx",    |                            |
 |     expires_in: 600 }        |                            |
 |                               |                            |
 |   (bot displays MESH-7829)    |   (user verifies code)     |
 |                               |                            |
 |                               | <-- POST /v1/bind/confirm  |
 |                               |     { bind_code, user_auth}|
 |                               |                            |
 |-- GET /v1/bind/poll --------> |                            |
 |   { poll_token: "pt_xxx" }   |                            |
 |<-- 200 OK                     |                            |
 |   { access_token: "at_xxx",  |                            |
 |     refresh_token: "rt_xxx", |                            |
 |     bot_id: "bot_abc" }      |                            |
```

**Security guarantees:**
- Verification code: User confirms bot-displayed code matches dashboard, prevents impersonation
- User-initiated confirmation: No confirmation = no binding
- Poll token: Single-use, expires in 10 minutes
- Refresh token: Allows long-lived sessions without re-binding, revocable by user

### 3.2 API Key (Developer/Advanced)

For debugging, CI/CD, and scripting scenarios.

- User generates API Key in dashboard under "Developer Settings"
- Key format: `cmesh_sk_<random>`, long-lived
- Sent as `Authorization: Bearer cmesh_sk_...`
- Supports scoping (read-only, sync-only, full)
- Revocable anytime from dashboard

### 3.3 Token Lifecycle

```
Bind Flow ──→ access_token (short-lived, 1h)
               + refresh_token (long-lived, 90d)
                   │
                   ├── auto-refresh by SDK/plugin
                   ├── revocable by user in dashboard
                   └── re-bind required if refresh token expires

API Key ────→ static token (no expiry, revocable)
```

## 4. Sync Protocol

### 4.1 Incremental Sync

Each bot maintains a **sync cursor** (last known server version). On sync:

1. Bot sends delta (new/modified/deleted memories since last sync)
2. Server returns delta (changes from cloud since bot's cursor)
3. Both sides advance cursors

```
POST /v1/sync
Authorization: Bearer <token>

{
  "bot_id": "bot_abc",
  "cursor": "cur_20260331_042",
  "changes": [
    { "action": "upsert", "memory": "<markdown content>" },
    { "action": "delete", "memory_id": "mem_xyz" }
  ]
}

Response:
{
  "new_cursor": "cur_20260331_045",
  "server_changes": [
    { "action": "upsert", "memory": "<markdown content>" },
    { "action": "delete", "memory_id": "mem_old" }
  ],
  "conflicts": []
}
```

### 4.2 Conflict Resolution

- Default strategy: **Last-Write-Wins** (based on `updated_at`)
- When conflict is detected, server keeps the winner and stores the loser as a conflict copy
- User can review and resolve conflicts in dashboard
- SDK provides conflict callback for programmatic resolution

### 4.3 Real-time Sync (Optional)

- WebSocket connection for push-based sync
- Bot connects to `wss://api.clawmesh.dev/v1/ws`
- Server pushes changes as they happen (e.g., inherited memories from another bot)
- Falls back to polling if WebSocket unavailable

## 5. Memory Inheritance

The core differentiator. Allows memories to flow between bots under user authorization.

### 5.1 ShareGrant Model

```
User creates a ShareGrant:
{
  "id": "grant_xxx",
  "from_bot": "bot_abc",
  "to_bot": "bot_def",
  "mode": "snapshot" | "live",
  "filter": {
    "types": ["feedback", "project"],
    "tags": ["important"],
    "date_range": { "after": "2026-01-01" }
  },
  "created_at": "2026-03-31T10:00:00Z"
}
```

### 5.2 Two Inheritance Modes

**Snapshot**: One-time copy of matching memories from source bot to target bot.
- Memories are duplicated with `origin` field set for traceability
- No ongoing relationship after copy

**Live**: Continuous sync of matching memories.
- New memories in source bot that match the filter are automatically synced to target
- Target bot's inherited memories update when source updates
- Stoppable anytime by user

### 5.3 Filter System

Users can control what gets shared:

| Filter | Description |
|--------|-------------|
| `types` | Memory types (user, feedback, project, reference) |
| `tags` | Only memories with specific tags |
| `date_range` | Only memories within a time window |
| `exclude_tags` | Exclude memories with specific tags |

### 5.4 Traceability

Every inherited memory carries an `origin` field:

```yaml
origin:
  bot_id: bot_abc
  memory_id: mem_original
  inherited_at: 2026-03-31T12:00:00Z
  grant_id: grant_xxx
```

This enables:
- User can see where a memory came from
- If source memory is deleted, user can decide whether to keep the copy
- Audit trail for compliance

## 6. Tech Stack

### 6.1 Server

| Component | Choice | Rationale |
|-----------|--------|-----------|
| Language | Rust | Safety, performance, low resource cost |
| Web Framework | axum | Async, tower ecosystem, mature |
| Database | PostgreSQL | Relational integrity, JSONB for metadata |
| ORM/Query | sqlx | Compile-time checked queries |
| Object Storage | S3-compatible | Markdown/skill file storage |
| Auth | custom + OAuth2 | Bind flow is custom; user login is standard OAuth2 |
| Real-time | tokio-tungstenite | WebSocket for live sync |
| Queue | PostgreSQL LISTEN/NOTIFY or Redis | Notification delivery, async tasks |

### 6.2 Client Distribution

| Form | Language | Use Case |
|------|----------|----------|
| SDK | TypeScript | JS/TS bot ecosystem integration |
| SDK | Python | Python bot ecosystem integration |
| CLI | Rust (cross-compile) | Terminal users, CI/CD |
| MCP Server | TypeScript | OpenClaw and MCP-compatible bots |
| Skill/Plugin | Per bot platform | Install-and-use, zero code |

### 6.3 Infrastructure

- Container deployment (Docker)
- PostgreSQL (managed or self-hosted)
- S3-compatible object storage (MinIO for self-hosted, AWS S3/Cloudflare R2 for cloud)
- Optional: Redis for caching and pub/sub

## 7. API Overview

```
# Auth
POST   /v1/bind/request          # Bot initiates bind
GET    /v1/bind/poll              # Bot polls for confirmation
POST   /v1/bind/confirm           # User confirms bind

# Sync
POST   /v1/sync                   # Incremental sync (memories + skills)
GET    /v1/memories               # List memories
GET    /v1/memories/:id           # Get single memory
GET    /v1/skills                 # List skills
GET    /v1/skills/:id             # Get single skill

# Sharing
POST   /v1/grants                 # Create share grant
GET    /v1/grants                 # List grants
DELETE /v1/grants/:id             # Revoke grant

# Management
GET    /v1/bots                   # List user's bots
DELETE /v1/bots/:id               # Remove bot
GET    /v1/bots/:id/status        # Bot sync status

# WebSocket
WS     /v1/ws                     # Real-time sync channel
```

## 8. Project Phases

### Phase 1: Foundation
- [ ] Rust server scaffold (axum + sqlx + PostgreSQL)
- [ ] User registration (GitHub OAuth)
- [ ] Bind flow authentication
- [ ] Basic memory CRUD API
- [ ] TypeScript SDK + CLI

### Phase 2: Sync
- [ ] Incremental sync protocol
- [ ] Conflict detection and resolution
- [ ] Skill sync
- [ ] Python SDK

### Phase 3: Inheritance
- [ ] ShareGrant model
- [ ] Snapshot inheritance
- [ ] Live inheritance
- [ ] Filter system

### Phase 4: Ecosystem
- [ ] MCP Server
- [ ] OpenClaw skill/plugin
- [ ] Web dashboard
- [ ] Real-time sync (WebSocket)

## 9. Open Questions

- Rate limiting strategy for bind requests (prevent spam)?
- End-to-end encryption for memories (client-side encryption before sync)?
- Multi-region deployment considerations?
- Pricing model for cloud service (free tier limits)?

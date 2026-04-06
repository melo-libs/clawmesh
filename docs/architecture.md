# ClawMesh Architecture Design

> Version: 0.2.0 | Date: 2026-04-05 | Status: Draft

## 1. Overview

ClawMesh is a cloud service that enables AI bots (OpenClaw and similar xxxclaw agents) to securely sync their memories and skills, with support for cross-bot memory inheritance and selective synchronization.

## 2. Core Concepts

### 2.1 Identity Model

```
User (human)
 ├── Workspace: clawmesh (shared, multi-device sync)
 │    ├── CLI client on laptop
 │    ├── CLI client on desktop
 │    └── Memory[]
 ├── Bot A (openclaw on laptop, independent)
 │    ├── Memory[]
 │    └── Skill[]
 └── Bot B (openclaw on cloud server, independent)
      ├── Memory[]
      └── Skill[]
```

Two types of namespaces:

- **Workspace**: A shared namespace for syncing the same memories across multiple devices (e.g., Claude Code on laptop and desktop working on the same project). Multiple clients push/pull to the same namespace. Identified by `(user_id, workspace_name)`.
- **Bot**: An independent namespace for a single AI agent instance. Each bot has its own memories. Cross-bot sharing is explicit via snapshot inheritance.

Common concepts:

- **User**: Human owner, via OAuth2 social login (GitHub/Google)
- **Namespace**: Either a workspace or a bot. Server-side primary key for memories is `(namespace_id, file_path)`
- **Memory**: A markdown document with frontmatter metadata
- **Skill**: A skill definition file, synced as-is

### 2.2 Memory Data Model

Memories are stored as markdown files with frontmatter, keeping compatibility with the bot ecosystem (OpenClaw SOUL.md, Claude Code memory, etc.).

Each memory is identified by its **file path** (relative to the bot's memory directory), scoped per bot. No separate ID is needed.

**Local file (what the bot reads/writes):**

```markdown
---
type: user | feedback | project | reference
tags: [testing, ci]
---

Memory content in markdown format.
```

The SDK parses frontmatter to extract `type` and `tags` as plaintext metadata for sync. The **entire file** (including frontmatter) is encrypted as a single blob before upload. This way the server has plaintext metadata for filtering, and the client can reconstruct the original file exactly on pull.

**Server-side record:**

| Column | Description |
|--------|-------------|
| `namespace_id` | Owner namespace: workspace or bot (PK part 1) |
| `file_path` | Relative path, e.g., `role.md`, `project/infra.md` (PK part 2). Must not contain `..` or start with `/` |
| `version` | Monotonically increasing, server-assigned, for sync conflict detection |
| `encrypted_content` | Ciphertext blob (nonce + ciphertext + auth_tag) |
| `type` | Memory category, extracted from frontmatter |
| `tags` | Labels for filtering, extracted from frontmatter |
| `created_at` | Server-assigned |
| `updated_at` | Server-assigned |
| `origin` | Non-null if inherited from another bot (see Section 6.4) |

### 2.3 Skill Data Model

Skills are also identified by file path, same as memories.

```markdown
---
name: web-search
skill_version: 1.2.0
format: mcp | plugin | script
---

Skill definition content.
```

**Server-side primary key:** `(namespace_id, file_path)`

Skills are synced as opaque files. ClawMesh indexes metadata but does not interpret skill logic.

## 3. Encryption

All memory and skill content is end-to-end encrypted. The server never sees plaintext.

### 3.1 Key Architecture

```
Master Password (user remembers, never uploaded)
       │
       ▼ Argon2id (salt = user_id, memory=64MB, iterations=3, parallelism=4)
  Master Key (256-bit)
       │
       ├──▶ AES-256-GCM wrap → Encrypted DEK (stored on server)
       │
       └──▶ HKDF-SHA256(master_key, "pw-verify") → Password Verify Hash
             └── stored on server, used to verify master password correctness
```

- **Master Password**: Set by user at registration. Never leaves the client.
- **Master Key**: Derived via Argon2id from master password + user_id (UUID, unique per user). Used only to wrap/unwrap the DEK and derive the verify hash.
- **DEK**: Random 256-bit key, generated at registration. Wrapped with Master Key using AES-256-GCM, stored on server. Used for all content encryption.
- **Password Verify Hash**: Derived from Master Key via HKDF. Stored on server to let clients verify master password correctness before attempting DEK decryption.

### 3.2 Per-Memory Encryption

Each memory is encrypted individually, compatible with incremental sync and per-memory inheritance.

**Encrypt (client-side):**
1. Generate random 96-bit nonce
2. `AES-256-GCM(DEK, nonce, plaintext)` → `ciphertext + auth_tag(16 bytes)`
3. Store as: `nonce(12) || ciphertext || auth_tag(16)`
4. Upload this blob + plaintext metadata

**Decrypt (client-side):**
1. Split blob: first 12 bytes → nonce, last 16 bytes → auth_tag, middle → ciphertext
2. `AES-256-GCM-Decrypt(DEK, nonce, ciphertext, auth_tag)` → plaintext

> **Critical: Every encryption operation MUST use a fresh random nonce.** Nonce reuse in GCM completely breaks encryption. SDK implementations must use a cryptographically secure random generator (e.g., `crypto.getRandomValues`, `OsRng`).

### 3.3 Client Types

Two types of clients, same key system:

| Client | How it gets the DEK | When it decrypts |
|--------|---------------------|------------------|
| Bot | User provides master password during bind, DEK derived and cached locally | Each sync |
| Dashboard | User enters master password after login, DEK held in memory for session | Browsing/editing |

**Bot flow:**
1. During bind, user provides master password to bot
2. Bot derives Master Key → verifies via Password Verify Hash → downloads encrypted DEK → decrypts DEK
3. Bot caches DEK locally (platform secure storage: Keychain on macOS, keyring on Linux)
4. On sync: encrypt each memory with DEK before upload, decrypt after download

**Dashboard flow:**
1. User logs in via OAuth (gets access to account)
2. User enters master password → client-side JS derives Master Key → verifies → decrypts DEK
3. Memories decrypted and displayed in browser
4. Edits encrypted in browser before upload

### 3.4 What the Server Sees

| Data | Server visibility |
|------|-------------------|
| Memory/skill content | Ciphertext only (nonce + ciphertext + auth_tag) |
| Memory metadata (file_path, type, tags, version, timestamps) | Plaintext (for indexing, filtering, conflict resolution) |
| Encrypted DEK | Stored but cannot decrypt |
| Password Verify Hash | Stored, used for verification only |
| Master Password / Master Key / DEK | Never |

> **Note:** Tags and type metadata are stored in plaintext to enable server-side filtering for sync and inheritance. Users should be aware that metadata is not encrypted.

### 3.5 Recovery

Forgetting the master password means losing access to all encrypted content (true E2E guarantee). Mitigations:

- **Recovery Key**: Random 256-bit key, generated at registration, displayed once for user to save offline (similar to 1Password Emergency Kit). Server stores a second copy of DEK encrypted with this Recovery Key.
- **Recovery flow**: User provides Recovery Key → client decrypts DEK → user sets new master password → DEK re-wrapped with new Master Key.
- **No server-side recovery**: By design, the server cannot recover data without the master password or recovery key.

### 3.6 Key Rotation

If user changes master password:
1. Derive new Master Key from new password
2. Re-encrypt DEK with new Master Key
3. Derive new Password Verify Hash, upload together with new encrypted DEK
4. Content is NOT re-encrypted (DEK stays the same, only its wrapper changes)
5. All bots continue working (they cache the DEK, not the Master Key)

## 4. Authentication

Two authentication methods, unified under the same identity model.

### 4.1 Bind Flow (Default, Recommended)

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

**After bind succeeds, bot obtains DEK:**
1. Bot receives access_token + refresh_token from poll
2. User provides master password locally to the bot
3. Bot calls `GET /v1/keys/dek` to download encrypted DEK
4. Bot derives Master Key from password → decrypts DEK → caches DEK locally
5. Bot is now ready to sync (see Section 3.3 for details)

**Security guarantees:**
- Verification code: User confirms bot-displayed code matches dashboard, prevents impersonation
- User-initiated confirmation: No confirmation = no binding
- Poll token: Single-use, expires in 10 minutes
- Refresh token: Allows long-lived sessions without re-binding, revocable by user

### 4.2 CLI Login (OAuth Device Flow)

For the ClawMesh CLI, used to sync workspace memories (e.g., Claude Code projects across devices).

**Flow:**
1. User runs `clawmesh login`
2. CLI displays a URL and a device code
3. User opens URL in browser, enters device code, completes GitHub OAuth
4. CLI receives access_token + refresh_token
5. User enters master password locally -> CLI derives DEK, caches it
6. Done. CLI can now sync.

This is standard OAuth 2.0 Device Authorization Grant (RFC 8628), same flow as `gh auth login`.

### 4.3 API Key (Developer/Advanced)

For debugging, CI/CD, and scripting scenarios.

- User generates API Key in dashboard under "Developer Settings"
- Key format: `cmesh_sk_<random>`, long-lived
- Sent as `Authorization: Bearer cmesh_sk_...`
- Supports scoping (read-only, sync-only, full)
- Revocable anytime from dashboard

### 4.4 Token Lifecycle

```
Bind Flow ──→ access_token (short-lived, 1h)
               + refresh_token (long-lived, 90d)
                   │
                   ├── auto-refresh by SDK/plugin
                   ├── revocable by user in dashboard
                   └── re-bind required if refresh token expires

CLI Login ──→ access_token (short-lived, 1h)
               + refresh_token (long-lived, 90d)
                   │
                   └── same lifecycle as bind flow

API Key ────→ static token (no expiry, revocable)
```

## 5. Sync Protocol

Push and pull are separate operations. Push uploads local changes to the server; pull downloads server changes to the client.

### 5.1 Client-Side Change Detection (Diff)

Bots manage memories as local files (markdown with frontmatter). The SDK detects changes by diffing current files against a local snapshot, without requiring the bot to call SDK APIs for every write.

**Local storage:** A single JSON snapshot file (e.g., `.clawmesh/sync.json`). No SQLite or other database dependency.

```json
{
  "pull_cursor": "seq_100",
  "files": {
    "role.md": { "content_hash": "sha256:a3f...", "version": 3, "mtime": 1712100000 },
    "project/infra.md": { "content_hash": "sha256:7c1...", "version": 2, "mtime": 1712100000 }
  }
}
```

> **Design decision:** SQLite was considered but rejected. Diff is idempotent — interruption at any point is recovered by re-diffing on next sync, making a persistent mutation queue unnecessary. A JSON file has zero external dependencies, works on all platforms (including serverless), and is sufficient for the expected scale (tens to hundreds of memories per bot). Crash safety is achieved via atomic write (write temp file → rename).

**Diff algorithm (runs at sync start):**

1. Scan all `.md` files in memory directory
2. For each file, compute its relative path as `file_path`
3. Compare against snapshot file:
   - **mtime unchanged** → skip (fast path, no I/O)
   - **mtime changed** → compute SHA-256 → compare with `content_hash`
     - Hash unchanged → skip (file was touched but content identical)
     - Hash changed → parse frontmatter for metadata (type, tags) → encrypt content → add `upsert` to mutations list
   - **File not in snapshot** → new memory → parse frontmatter → encrypt → add `upsert` to mutations list
4. For each snapshot entry with no matching file on disk → add `delete` to mutations list

Mutations are held in memory, not persisted. If sync is interrupted, next sync re-diffs and regenerates them.

### 5.2 Push (Client → Server)

Client sends mutations in batches. Server acknowledges each item individually.

```
POST /v1/sync/push
Authorization: Bearer <token>

{
  "namespace_id": "ws_clawmesh",
  "changes": [
    {
      "action": "upsert",
      "file_path": "feedback_testing.md",
      "base_version": 3,
      "encrypted_content": "<nonce || ciphertext || auth_tag>",
      "metadata": { "type": "feedback", "tags": ["testing"] }
    },
    {
      "action": "delete",
      "file_path": "project/old_notes.md",
      "base_version": 1
    }
  ]
}

Response:
{
  "results": [
    { "file_path": "feedback_testing.md", "status": "accepted", "new_version": 4 },
    { "file_path": "project/old_notes.md", "status": "accepted" }
  ]
}
```

After receiving response:
- Update snapshot file with new versions and content hashes for accepted items
- If network fails before response, next sync re-diffs and re-pushes (idempotent)
- Server uses `file_path + base_version` for dedup (re-push is safe)
- For new files (not in snapshot), `base_version` is `null` — server rejects if file already exists (e.g., created via dashboard)

### 5.3 Pull (Server → Client)

Client pulls server changes using a cursor with pagination.

```
GET /v1/sync/pull?cursor=seq_100&limit=100
Authorization: Bearer <token>

Response:
{
  "changes": [
    {
      "action": "upsert",
      "file_path": "project/infra.md",
      "version": 2,
      "encrypted_content": "<nonce || ciphertext || auth_tag>",
      "metadata": { "type": "project", "tags": ["infra"] }
    }
  ],
  "next_cursor": "seq_108",
  "has_more": false
}
```

Client processing:
1. For each change, check if local memory already has same version → skip
2. Decrypt content, write to local file
3. Update snapshot entry for that file
4. After all pages consumed, save `next_cursor` to snapshot file
5. If interrupted, resume from last saved cursor

### 5.4 Full Sync Flow

```
SDK sync():
  1. Diff: scan files vs snapshot → generate mutations (in memory)
  2. Push: send mutations in batches
     └── for each batch: POST /v1/sync/push → save snapshot (atomic write)
  3. Pull: loop with cursor + pagination
     └── for each page: GET /v1/sync/pull → decrypt → write files → save snapshot (atomic write)
```

> Snapshot is saved after every batch/page (not just at the end) to avoid false conflicts if sync is interrupted midway. Each save is atomic (write temp file → rename).

### 5.5 Conflict Resolution

- Server checks `base_version` on each push: if server version ≠ base_version, conflict detected
- Default strategy: **Last-Write-Wins** (based on metadata `version`)
- Server resolves conflicts using metadata only (content is encrypted, server cannot read it)
- When conflict is detected, server keeps the winner and stores the loser as a conflict copy
- User can review and resolve conflicts in dashboard (decrypted client-side)
- SDK provides conflict callback for programmatic resolution

### 5.6 Real-time Sync (Optional)

- WebSocket connection for push-based notifications
- Bot connects to `wss://api.clawmesh.dev/v1/ws`
- Server pushes change notifications (bot then pulls via normal flow)
- Falls back to periodic polling if WebSocket unavailable

## 6. Memory Inheritance

Allows users to copy memories from one bot to another. This is a one-time, user-initiated operation (snapshot), not continuous sync.

> **Design decision:** Live (continuous) inheritance was considered and rejected. Different bots operate in different contexts and may hold legitimately different views on the same topic. Automatically pushing memories between bots would introduce contradictions and make bot behavior unpredictable. Snapshot inheritance keeps the user in control.

### 6.1 ShareGrant Model

```
User creates a ShareGrant:
{
  "id": "grant_xxx",
  "from_bot": "bot_abc",
  "to_bot": "bot_def",
  "filter": {
    "types": ["feedback", "project"],
    "tags": ["important"],
    "date_range": { "after": "2026-01-01" }
  },
  "created_at": "2026-03-31T10:00:00Z"
}
```

### 6.2 Snapshot Inheritance

One-time copy of matching memories from source bot to target bot.

- Memories are duplicated with `origin` field set for traceability
- No ongoing relationship after copy — source and target evolve independently
- Target bot can freely modify or delete inherited memories

### 6.3 Filter System

Users can control what gets shared:

| Filter | Description |
|--------|-------------|
| `types` | Memory types (user, feedback, project, reference) |
| `tags` | Only memories with specific tags |
| `date_range` | Only memories within a time window |
| `exclude_tags` | Exclude memories with specific tags |

### 6.4 Traceability

Every inherited memory carries an `origin` field:

```yaml
origin:
  namespace_id: bot_abc
  file_path: role.md
  inherited_at: 2026-03-31T12:00:00Z
  grant_id: grant_xxx
```

This enables:
- User can see where a memory came from
- Audit trail for compliance

## 7. Tech Stack

### 7.1 Server

| Component | Choice | Rationale |
|-----------|--------|-----------|
| Language | Rust | Safety, performance, low resource cost |
| Web Framework | axum | Async, tower ecosystem, mature |
| Database | PostgreSQL | Relational integrity, JSONB for metadata, BYTEA for encrypted content |
| ORM/Query | sqlx | Compile-time checked queries |
| Auth | custom + OAuth2 | Bind flow is custom; user login is standard OAuth2 |
| Real-time | tokio-tungstenite | WebSocket for real-time sync notifications |
| Queue | PostgreSQL LISTEN/NOTIFY or Redis | Notification delivery, async tasks |

### 7.2 Client Distribution

| Form | Language | Use Case |
|------|----------|----------|
| SDK | TypeScript | JS/TS bot ecosystem integration |
| SDK | Python | Python bot ecosystem integration |
| CLI | Rust (cross-compile) | Workspace sync (Claude Code, etc.), terminal users, CI/CD |
| MCP Server | TypeScript | Optional integration for MCP-compatible bots |
| Skill/Plugin | Per bot platform | Install-and-use, zero code |

### 7.3 Storage

All data stored in PostgreSQL. No separate object storage required.

- **Metadata** (file_path, type, tags, version, timestamps): regular columns, indexed for filtering and sync queries
- **Encrypted content**: `BYTEA` column. PG TOAST handles compression and out-of-line storage automatically for values > ~2KB
- **Encrypted DEK, recovery-encrypted DEK**: `BYTEA` columns in user table
- **Query optimization**: list/filter queries should SELECT only metadata columns to avoid TOAST reads; pull content only when needed

Estimated scale: memory files are typically 1-10KB. 1000 users × 1000 memories ≈ 1-5GB, well within PG comfort zone. If large file support is needed in the future (e.g., skill attachments), S3 can be introduced at that point.

### 7.4 Infrastructure

- Container deployment (Docker)
- PostgreSQL (managed or self-hosted)
- Optional: Redis for caching and pub/sub

## 8. API Overview

```
# Auth (Bot bind)
POST   /v1/bind/request          # Bot initiates bind
GET    /v1/bind/poll              # Bot polls for confirmation
POST   /v1/bind/confirm           # User confirms bind

# Auth (CLI login - OAuth Device Flow)
POST   /v1/auth/device            # Start device auth, returns device_code + user_code
POST   /v1/auth/device/token      # Poll for token after user completes browser auth
POST   /v1/auth/refresh           # Refresh access_token using refresh_token

# Encryption
GET    /v1/keys/dek               # Get encrypted DEK
PUT    /v1/keys/dek               # Update encrypted DEK (key rotation)
POST   /v1/keys/setup             # Initial master password setup (store encrypted DEK)

# Sync
POST   /v1/sync/push              # Push local changes (batched)
GET    /v1/sync/pull              # Pull server changes (cursor + pagination)

# Memories (CRUD, primarily for dashboard)
GET    /v1/memories               # List memories (metadata only). ?ns= for cross-namespace read (same user)
POST   /v1/memories               # Create memory
GET    /v1/memories/*path         # Get single memory. ?ns= for cross-namespace read (same user)
PATCH  /v1/memories/*path         # Update memory
DELETE /v1/memories/*path         # Delete memory

# Skills
GET    /v1/skills                 # List skills
GET    /v1/skills/*path           # Get single skill

# Sharing
POST   /v1/grants                 # Create share grant
GET    /v1/grants                 # List grants
DELETE /v1/grants/:id             # Revoke grant

# Workspaces
POST   /v1/workspaces             # Create workspace
GET    /v1/workspaces             # List user's workspaces
DELETE /v1/workspaces/:name       # Delete workspace

# Bots
GET    /v1/bots                   # List user's bots
PATCH  /v1/bots/:id              # Update bot info
DELETE /v1/bots/:id               # Remove bot
GET    /v1/bots/:id/status        # Bot sync status

# WebSocket
WS     /v1/ws                     # Real-time sync channel
```

## 9. Project Phases

### Phase 1: Foundation
- [ ] Rust server scaffold (axum + sqlx + PostgreSQL)
- [ ] User registration (GitHub OAuth)
- [ ] Master password setup + encrypted DEK storage
- [ ] Bind flow authentication (with DEK delivery)
- [ ] CLI login (OAuth Device Flow)
- [ ] Workspace CRUD
- [ ] Basic memory CRUD API (encrypted content)
- [ ] TypeScript SDK + CLI (with client-side encryption)
- [ ] CLI: `clawmesh login`, `clawmesh init`, `clawmesh sync`

### Phase 2: Sync
- [ ] Incremental sync protocol (push/pull + diff)
- [ ] Conflict detection and resolution
- [ ] Skill sync
- [ ] Claude Code hook integration
- [ ] Python SDK

### Phase 3: Inheritance
- [ ] ShareGrant model
- [ ] Snapshot inheritance
- [ ] Filter system
- [ ] Cross-namespace memory read API

### Phase 4: Ecosystem
- [ ] MCP Server (optional integration layer)
- [ ] OpenClaw skill/plugin
- [ ] Web dashboard
- [ ] Real-time sync (WebSocket)
- [ ] CLI daemon mode (background file watcher + auto-sync)

## 10. Open Questions

- Multi-region deployment considerations?
- Master password strength requirements (minimum entropy)?
- Should tags be encrypted too (trading off server-side filtering for full privacy)?
- Multi-bot collaboration: messaging/events between bots, shared context, task delegation. Current architecture (WebSocket + auth model) is forward-compatible -- design when use cases are clearer.
- DEK rotation (full re-encryption) for compromised device scenarios -- Phase 1 deferred.

> Rate limiting, pricing/free tier, and account lifecycle are now defined in `docs/user-stories.md` Sections 3-6.

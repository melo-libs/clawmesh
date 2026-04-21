# MemKeep Architecture Design

> Version: 0.6.0 | Date: 2026-04-21 | Status: Draft

## 1. Overview

This document defines the technical architecture of MemKeep: identity model, end-to-end encryption, authentication, sync protocol, memory copy, and tech stack. The architecture is general enough to serve individual developers (MVP) and scale into a third-party MaaS API (Phase 4).

## 2. Core Concepts

### 2.1 Identity Model

Data belongs to one of two entity types:

- **Project**: Project-centric knowledge. Identified by `(user_id, project_name)`, name unique per account.
- **Agent**: Agent-centric knowledge. Agents can join Projects (permission model: gains read/write access to Project documents).

Core concepts:

- **User**: Human owner, via OAuth2 social login (GitHub/Google)
- **Document**: Encrypted file with metadata (unified storage primitive). Belongs to one Project or Agent. Server-side primary key is `(owner_id, file_path)`, where `owner_id` refers to the owning Project or Agent.
- **Document metadata**: type (memory | context | conversation | skill), tags, ttl, source. Recommended format is Markdown + frontmatter, not enforced.

### 2.2 Document Data Model

Documents are stored as markdown files with frontmatter (recommended but not enforced), maintaining compatibility with the agent ecosystem (OpenClaw SOUL.md, Claude Code memory, etc.).

Each Document is identified by its **file path** (relative to the memory directory), scoped by owning Project or Agent. No separate ID is needed.

**Local file (what the agent reads/writes):**

```markdown
---
type: memory | context | conversation | skill
tags: [testing, ci]
---

Document content in markdown format.
```

The SDK parses frontmatter to extract `type` and `tags` as plaintext metadata for sync. The **entire file** (including frontmatter) is encrypted as a single blob before upload. This way the server has plaintext metadata for filtering, and the client can reconstruct the original file exactly on pull.

**Server-side record:**

| Column | Description |
|--------|-------------|
| `owner_id` | Owning Project or Agent ID (PK part 1) |
| `file_path` | Relative path, e.g., `role.md`, `project/infra.md` (PK part 2). Must not contain `..` or start with `/` |
| `version` | Monotonically increasing, server-assigned, for sync conflict detection |
| `encrypted_content` | Ciphertext blob (nonce + ciphertext + auth_tag) |
| `type` | Document category (memory | context | conversation | skill), extracted from frontmatter |
| `tags` | Labels for filtering, extracted from frontmatter |
| `created_at` | Server-assigned |
| `updated_at` | Server-assigned |
| `origin` | Non-null if copied from another Project/Agent (see Section 6.4) |

## 3. Encryption

All Document content is end-to-end encrypted. The server never sees plaintext.

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

### 3.2 Per-Document Encryption

Each Document is encrypted individually, compatible with incremental sync and per-document copy.

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

Three types of clients, same key system:

| Client | How it gets the DEK | When it decrypts |
|--------|---------------------|------------------|
| Agent | User provides master password during bind, DEK derived and cached locally | Each sync |
| CLI | User provides master password during `memkeep login`, DEK cached locally | Each sync |
| Dashboard | User enters master password after login, DEK held in memory for session | Browsing/editing |

**Agent flow:**
1. During bind, user provides master password to agent
2. Agent derives Master Key → verifies via Password Verify Hash → downloads encrypted DEK → decrypts DEK
3. Agent caches DEK locally (platform secure storage: Keychain on macOS, keyring on Linux)
4. On sync: encrypt each Document with DEK before upload, decrypt after download

**Dashboard flow:**
1. User logs in via OAuth (gets access to account)
2. User enters master password → client-side JS derives Master Key → verifies → decrypts DEK
3. Documents decrypted and displayed in browser
4. Edits encrypted in browser before upload

### 3.4 What the Server Sees

| Data | Server visibility |
|------|-------------------|
| Document content | Ciphertext only (nonce + ciphertext + auth_tag) |
| Document metadata (file_path, type, tags, version, timestamps) | Plaintext (for indexing, filtering, conflict resolution) |
| Encrypted DEK | Stored but cannot decrypt |
| Password Verify Hash | Stored, used for verification only |
| Master Password / Master Key / DEK | Never |

> **Note:** Tags and type metadata are stored in plaintext to enable server-side filtering for sync and copy. Users should be aware that metadata is not encrypted.

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
5. All agents continue working (they cache the DEK, not the Master Key)

### 3.7 Security Threat Model

**What we protect against:**

| Threat | Mitigation | Effectiveness |
|--------|-----------|---------------|
| Database breach (attacker dumps PG) | E2E encryption -- all content is ciphertext, DEK is wrapped | Full. Content unrecoverable without master password |
| Server compromise (attacker controls server) | E2E encryption -- server never sees plaintext. Password Verify Hash is derived (HKDF), not the master password itself | Full for content. Attacker could tamper with metadata (type, tags) but not decrypt content |
| Man-in-the-middle | TLS for transport + E2E encryption for content | Full |
| Brute-force master password (offline, from stolen encrypted DEK) | Argon2id with high memory cost (64MB) makes each guess expensive | Strong. ~$10K+ to crack a decent password. Weak passwords remain vulnerable |
| Token theft (attacker steals access_token) | Short-lived (1h), auto-refresh. Revocable in dashboard | Partial. Attacker has access until token expires or is revoked |
| Replay attacks on sync | base_version checking, server-assigned versions | Full |

**What we do NOT protect against (MVP):**

| Threat | Risk | Mitigation path |
|--------|------|----------------|
| Device physically compromised | Cached DEK can be extracted from local storage. Attacker can decrypt all synced content | Phase 1: revoke agent access (stops future sync). Future: DEK rotation (re-encrypt all content with new DEK) |
| Weak master password | Argon2id slows brute force but cannot prevent it for very weak passwords (e.g., "123456") | Enforce minimum password strength at registration. Future: support hardware keys (FIDO2) |
| Metadata analysis | Tags, types, file paths, timestamps are plaintext. Attacker can infer topics without reading content | Accepted trade-off for server-side filtering. Future: encrypted tags (see Open Questions) |
| Malicious agent/SDK | A compromised agent with valid DEK could exfiltrate all Documents | Out of scope -- agent security is the agent platform's responsibility, not MemKeep's |
| Side-channel attacks on clients | Timing attacks on Argon2id, memory dumps during decryption | Standard platform security. Out of scope for Phase 1 |

## 4. Authentication

Three authentication methods, unified under the same identity model.

### 4.1 Bind Flow (Default, Recommended)

The primary method. User tells the agent to connect, then confirms in MemKeep.

**User experience:**

1. User tells agent: "Connect to MemKeep, I'm voya"
2. Agent displays a verification code
3. User confirms the code in MemKeep dashboard/notification
4. Done. Agent starts syncing.

**Protocol:**

```
Agent                        MemKeep                       User
 |                               |                            |
 |-- POST /v1/bind/request ----> |                            |
 |   { username: "voya",         |                            |
 |     agent_name: "aria",       |                            |
 |     agent_meta: {             |                            |
 |       agent_type: "openclaw", |                            |
 |       platform: "macos",      |                            |
 |       version: "1.2.0"        |                            |
 |     }                         |                            |
 |   }                           |                            |
 |                               |-- Notify: bind request --> |
 |<-- 200 OK                     |   (email/push/dashboard)   |
 |   { bind_code: "MK-7829",  |                            |
 |     poll_token: "pt_xxx",    |                            |
 |     expires_in: 600 }        |                            |
 |                               |                            |
 |   (agent displays MK-7829)  |   (user verifies code)     |
 |                               |                            |
 |                               | <-- POST /v1/bind/confirm  |
 |                               |     { bind_code, user_auth}|
 |                               |                            |
 |-- GET /v1/bind/poll --------> |                            |
 |   { poll_token: "pt_xxx" }   |                            |
 |<-- 200 OK                     |                            |
 |   { access_token: "at_xxx",  |                            |
 |     refresh_token: "rt_xxx", |                            |
 |     agent_id: "agent_abc" }  |                            |
```

**After bind succeeds, agent obtains DEK:**
1. Agent receives access_token + refresh_token from poll
2. User provides master password locally to the agent
3. Agent calls `GET /v1/keys/dek` to download encrypted DEK
4. Agent derives Master Key from password → decrypts DEK → caches DEK locally
5. Agent is now ready to sync (see Section 3.3 for details)

**Security guarantees:**
- Verification code: User confirms agent-displayed code matches dashboard, prevents impersonation
- User-initiated confirmation: No confirmation = no binding
- Poll token: Single-use, expires in 10 minutes
- Refresh token: Allows long-lived sessions without re-binding, revocable by user

### 4.2 CLI Login (OAuth Device Flow)

For the MemKeep CLI, used to sync Project documents (e.g., Claude Code projects across devices).

**Flow:**
1. User runs `memkeep login`
2. CLI displays a URL and a device code
3. User opens URL in browser, enters device code, completes GitHub OAuth
4. CLI receives access_token + refresh_token
5. User enters master password locally → CLI derives DEK, caches it
6. Done. CLI can now sync.

This is standard OAuth 2.0 Device Authorization Grant (RFC 8628), same flow as `gh auth login`.

### 4.3 API Key (Developer/Advanced)

For debugging, CI/CD, and scripting scenarios.

- User generates API Key in dashboard under "Developer Settings"
- Key format: `mk_sk_<random>`, long-lived
- Sent as `Authorization: Bearer mk_sk_...`
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

Agents manage Documents as local files (markdown with frontmatter). The SDK detects changes by diffing current files against a local snapshot, without requiring the agent to call SDK APIs for every write.

**Local storage:** A single JSON snapshot file (e.g., `.memkeep/sync.json`). No SQLite or other database dependency.

```json
{
  "pull_cursor": "seq_100",
  "files": {
    "role.md": { "content_hash": "sha256:a3f...", "version": 3, "mtime": 1712100000 },
    "project/infra.md": { "content_hash": "sha256:7c1...", "version": 2, "mtime": 1712100000 }
  }
}
```

> **Design decision:** SQLite was considered but rejected. Diff is idempotent — interruption at any point is recovered by re-diffing on next sync, making a persistent mutation queue unnecessary. A JSON file has zero external dependencies, works on all platforms (including serverless), and is sufficient for the expected scale (tens to hundreds of Documents per Project/Agent). Crash safety is achieved via atomic write (write temp file → rename).

**Diff algorithm (runs at sync start):**

1. Scan all `.md` files in Document directory
2. For each file, compute its relative path as `file_path`
3. Compare against snapshot file:
   - **mtime unchanged** → skip (fast path, no I/O)
   - **mtime changed** → compute SHA-256 → compare with `content_hash`
     - Hash unchanged → skip (file was touched but content identical)
     - Hash changed → parse frontmatter for metadata (type, tags) → encrypt content → add `upsert` to mutations list
   - **File not in snapshot** → new Document → parse frontmatter → encrypt → add `upsert` to mutations list
4. For each snapshot entry with no matching file on disk → add `delete` to mutations list

Mutations are held in memory, not persisted. If sync is interrupted, next sync re-diffs and regenerates them.

### 5.2 Push (Client → Server)

Client sends mutations in batches. Server acknowledges each item individually.

```
POST /v1/sync/push
Authorization: Bearer <token>

{
  "owner_id": "proj_memkeep",
  "changes": [
    {
      "action": "upsert",
      "file_path": "feedback_testing.md",
      "base_version": 3,
      "encrypted_content": "<nonce || ciphertext || auth_tag>",
      "metadata": { "type": "memory", "tags": ["feedback", "testing"] }
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
      "metadata": { "type": "memory", "tags": ["project", "infra"] }
    }
  ],
  "next_cursor": "seq_108",
  "has_more": false
}
```

Client processing:
1. For each change, check if local Document already has same version → skip
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
- Server resolves conflicts using metadata only (content is encrypted, server cannot read it)
- When conflict is detected, server keeps its version (server wins) and stores the client's version as a conflict copy
- MVP: conflicts are auto-resolved (server wins). Agent may notify user: "1 conflict detected"
- Phase 3: dashboard provides UI to review conflict copies side-by-side and resolve manually
- SDK provides conflict callback for programmatic resolution

### 5.6 Real-time Sync (Optional)

- WebSocket connection for push-based notifications
- Agent connects to `wss://api.memkeep.ai/v1/ws`
- Server pushes change notifications (agent then pulls via normal flow)
- Falls back to periodic polling if WebSocket unavailable

## 6. Memory Copy

Allows users to copy Documents from one Project/Agent to another. Supports any direction. This is a one-time, user-initiated operation (snapshot), not continuous sync.

> **Design decision:** Live (continuous) sync was considered and rejected. Different agents operate in different contexts and may hold legitimately different views on the same topic. Automatically pushing Documents across Projects/Agents would introduce contradictions and make agent behavior unpredictable. Snapshot copy keeps the user in control.

### 6.1 ShareGrant Model

```
User creates a ShareGrant:
{
  "id": "grant_xxx",
  "from_owner": "agent_abc",
  "to_owner": "agent_def",
  "filter": {
    "types": ["memory", "context"],
    "tags": ["important"],
    "date_range": { "after": "2026-01-01" }
  },
  "created_at": "2026-03-31T10:00:00Z"
}
```

`from_owner` and `to_owner` can be Projects or Agents, supporting any direction.

### 6.2 Snapshot Copy

One-time copy of matching Documents from source to target.

- Documents are duplicated with `origin` field set for traceability
- No ongoing relationship after copy — source and target evolve independently
- Target can freely modify or delete copied Documents

### 6.3 Filter System

Users can control what gets copied:

| Filter | Description |
|--------|-------------|
| `types` | Document types (memory, context, conversation, skill) |
| `tags` | Only Documents with specific tags |
| `date_range` | Only Documents within a time window |
| `exclude_tags` | Exclude Documents with specific tags |

### 6.4 Traceability

Every copied Document carries an `origin` field:

```yaml
origin:
  owner_id: agent_abc
  file_path: role.md
  copied_at: 2026-03-31T12:00:00Z
  grant_id: grant_xxx
```

This enables:
- User can see where a Document came from
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
| SDK | TypeScript | JS/TS agent ecosystem integration |
| SDK | Python | Python agent ecosystem integration |
| CLI | Rust (cross-compile) | Project sync (Claude Code, etc.), terminal users, CI/CD |
| MCP Server | TypeScript | Optional integration for MCP-compatible agents |
| Skill/Plugin | Per agent platform | Install-and-use, zero code |

### 7.3 Storage

All data stored in PostgreSQL. No separate object storage required.

- **Metadata** (file_path, type, tags, version, timestamps): regular columns, indexed for filtering and sync queries
- **Encrypted content**: `BYTEA` column. PG TOAST handles compression and out-of-line storage automatically for values > ~2KB
- **Encrypted DEK, recovery-encrypted DEK**: `BYTEA` columns in user table
- **Query optimization**: list/filter queries should SELECT only metadata columns to avoid TOAST reads; pull content only when needed

Estimated scale: Document files are typically 1-10KB. 1000 users × 1000 Documents ≈ 1-5GB, well within PG comfort zone. If large file support is needed in the future (e.g., attachments), S3 can be introduced at that point.

### 7.4 Infrastructure

- Container deployment (Docker)
- PostgreSQL (managed or self-hosted)
- Optional: Redis for caching and pub/sub

## 8. API Overview

```
# Auth (Agent bind)
POST   /v1/bind/request          # Agent initiates bind
GET    /v1/bind/poll              # Agent polls for confirmation
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

# Documents (CRUD, primarily for dashboard)
GET    /v1/documents              # List Documents (metadata only). ?owner= to specify Project/Agent
POST   /v1/documents              # Create Document
GET    /v1/documents/*path        # Get single Document. ?owner= to specify Project/Agent
PATCH  /v1/documents/*path        # Update Document
DELETE /v1/documents/*path        # Delete Document

# Memory Copy
POST   /v1/copy                   # Copy Documents from source Project/Agent to target

# Sharing
POST   /v1/grants                 # Create share grant
GET    /v1/grants                 # List grants
DELETE /v1/grants/:id             # Revoke grant

# Projects
POST   /v1/projects               # Create Project
GET    /v1/projects               # List user's Projects
DELETE /v1/projects/:name         # Delete Project

# Agents
GET    /v1/agents                 # List user's Agents
PATCH  /v1/agents/:id            # Update Agent info
DELETE /v1/agents/:id             # Remove Agent
GET    /v1/agents/:id/status      # Agent sync status

# WebSocket
WS     /v1/ws                     # Real-time sync channel
```

## 9. Project Phases

See [ROADMAP.md](../ROADMAP.md). Core path: MVP (sync) → Phase 2 (memory copy + ecosystem) → Phase 3 (dashboard) → Phase 4 (MaaS API + enterprise).

## 10. Open Questions

- Multi-region deployment considerations?
- Multi-agent collaboration: messaging/events between agents, shared context, task delegation. Current architecture (WebSocket + auth model) is forward-compatible — design when use cases are clearer.

> Rate limiting and pricing details are defined separately in the pricing documentation.

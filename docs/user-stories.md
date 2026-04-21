# MemKeep User Stories & Flows

> Version: 0.2.0 | Date: 2026-04-21 | Status: Draft

## 1. Personas

### Primary User

A developer who uses multiple AI agents (OpenClaw, Claude Code, Codex, etc.) across different devices and environments. Core pain point: **knowledge is siloed** — what one agent learns, the others don't know.

Typical setup:
- Claude Code on laptop + desktop (same project, different machines)
- Codex for code review
- OpenClaw agent "Aria" (independent agent with persistent identity)

## 2. User Journeys

### 2.1 First-Time Setup (Registration + First Agent Binding)

The most critical flow. Must feel as simple as setting up a password manager.

**Steps:**

1. User tells their agent: "Connect to MemKeep, I'm voya"
2. Agent initiates bind request, receives a verification code
3. Agent displays: "Your code is MK-7829. Go to memkeep.ai to sign up and confirm."
4. User opens memkeep.ai:
   - Signs in with GitHub (one-click)
   - Sets a master password (prompted: "This protects all your agent memories")
   - Saves Recovery Key (prompted: "Save this somewhere safe — it's the only way to recover if you forget your password")
   - Enters verification code MK-7829 to confirm the bind
5. User returns to agent, provides master password when prompted
6. Agent derives DEK, starts syncing
7. Agent confirms: "Connected! I'll sync your Documents automatically from now on."

**Design principles:**
- Registration and first bind are one flow, not two separate steps
- Master password and Recovery Key setup happen during registration, not as afterthought
- User provides master password to agent only once — DEK is cached locally after that
- Total interaction: 1 OAuth login + 1 password + 1 verification code

### 2.2 Day-to-Day Sync (Invisible)

After setup, the user should not think about MemKeep at all.

**Trigger options (SDK configurable):**
- Timer-based: every N minutes (default: 5 min)
- Event-based: after agent writes a new Document
- Manual: user says "sync" or "sync now"

**Behavior:**
- Sync runs silently in the background
- No notifications, no interruptions on success
- On error (network issue, conflict), agent may briefly mention it: "Heads up: I couldn't sync, will retry later"
- Optionally, agent can mention new Documents at conversation start: "Synced 3 new Documents from cloud" (configurable, off by default)

### 2.3 Binding an Additional Agent

User sets up a new device or agent instance.

**Steps:**

1. User tells the new agent: "Connect to MemKeep, I'm voya"
2. Agent initiates bind, displays code: MK-4521
3. User receives a notification (dashboard / email / push):
   "New agent 'aria-desktop' is requesting access. Code: MK-4521. Approve?"
4. User taps [Approve]
5. User provides master password to the new agent
6. Agent obtains DEK, starts syncing
7. Done

**Difference from first-time setup:** No registration, no password setup. Just confirm code + provide master password.

### 2.4 Knowledge Migration (New Agent from Scratch)

A newly bound agent has zero Documents. User wants to bootstrap it with knowledge from an existing agent.

**Flow (in Dashboard):**

1. User navigates to Agents -> Echo -> "Copy Documents"
2. Dashboard shows:
   - Source selector (e.g., Agent "Aria" or Project "memkeep-server")
   - Type filter: checkboxes for memory, context, conversation, skill
   - Tag filter: optional include/exclude tags
3. User selects filters, clicks "Preview"
4. Dashboard shows: "43 Documents match. [Confirm Copy]"
5. User confirms -> server executes snapshot copy
6. New agent picks up the 43 Documents on next sync

**Design principles:**
- Done in dashboard, not in agent (user needs the full picture)
- Preview before confirm -- user knows exactly what will be copied
- Agent-side is zero effort -- just normal sync picks it up

### 2.5 Selective Cross-Agent Copy

The more common real-world scenario: Agent A and Agent B both have Documents, user wants to selectively copy some from A to B.

**Flow (Agent-initiated, conversational):**

1. User tells Agent B: "Copy the memory-type Documents from Aria"
2. Agent B (via SDK) fetches Aria's memory-type Document metadata from server
3. Agent B compares with its own local Documents:
   - **New** (Aria has, Agent B doesn't): candidates for copy
   - **Conflict** (same file_path, different content): needs user decision
   - **Same** (identical content): skip automatically
4. For conflicts, Agent B downloads Aria's encrypted content, decrypts locally, compares
5. Agent B reports to user:

   > "Found 15 memory Documents in Aria:
   > - 9 are new to me
   > - 4 I also have but with different content:
   >   - feedback_testing.md: Aria says 'use real DB', I say 'mock for unit tests, real DB for integration'
   >   - ...
   > - 2 are identical, skipping
   >
   > How should I handle these?"

6. User decides: "Take the 9 new ones. For conflicts, keep yours."
7. Agent B downloads and saves the 9 new Documents, syncs normally

**Design principles:**
- Agent-initiated and conversational -- user gives instructions in natural language
- Agent does the intelligence: it can understand content semantics, not just diff bytes
- Agent can even suggest merging conflicting Documents (AI advantage over traditional sync)
- Server only provides the data channel -- all comparison and decision-making happens client-side (E2E encrypted, server sees nothing)
- Aria does not need to be online -- its data is already on the server from previous syncs

**API support needed:**
- Document endpoints accept `owner` query parameter for cross-owner reads (e.g., `GET /v1/documents?owner=agent_abc`)
- Server verifies both owners belong to the same user

### 2.6 Document Management (Dashboard)

User wants to review, edit, or clean up Documents.

**Dashboard layout:**

```
Agents -> Aria -> Documents

Search: [________]  Type: [All v]  Tags: [All v]

file_path                type          version  updated
role.md                  memory        v3       10 min ago
work_context.md          context       v5       1 hour ago
project/infra.md         memory        v2       yesterday
project/api_design.md    memory        v1       3 days ago
--- copied ---
feedback_db.md           memory        v4       (from Echo)
```

**Flows:**

- **View**: Click a Document -> enter master password (once per session) -> content decrypted in browser -> displayed
- **Edit**: Modify content in browser -> encrypted before upload -> version incremented
- **Delete**: Confirm -> removed from server -> agent's next sync removes local file
- **Search**: Searches metadata (type, tags, file_path) -- content is encrypted, not searchable server-side
- **Copied Documents**: Clearly labeled with source, can be edited or deleted independently

### 2.7 Conflict Resolution (Dashboard)

Conflicts are rare but need a clear resolution path.

**When it happens:** An agent and the dashboard modify the same Document between syncs. Conflicts are within a single Project or Agent -- different agents have independent copies and cannot conflict with each other through normal sync.

**Example:** User edits `project/infra.md` in dashboard while Aria also modifies it locally. On next push, Aria's base_version doesn't match the server version -> conflict.

**Dashboard flow:**

```
Dashboard -> Agents -> Aria -> Conflicts (1)

project/infra.md -- conflicted at 2026-04-05 14:30

Server version (v5, edited in dashboard):      Agent version (v4, pushed by Aria):
+------------------------------+  +------------------------------+
| DB: PostgreSQL on AWS RDS    |  | DB: PostgreSQL, self-hosted  |
+------------------------------+  +------------------------------+

[Keep Server Version]  [Use Agent Version]  [Edit Manually]
```

**Design principles:**
- Conflicts don't block sync -- server auto-resolves (keeps one, stores the other as conflict copy)
- Dashboard shows both versions side-by-side, decrypted in browser
- User picks one, or manually edits a merged version
- Resolved conflicts are cleaned up automatically
- Conflicts are per-owner. To propagate a resolution to other agents, use cross-agent copy (Section 2.5)

### 2.8 Revoking an Agent

User loses a device or wants to remove an agent's access.

**Flow (in Dashboard):**

1. User navigates to Agents -> Aria -> "Remove Agent"
2. Dashboard warns: "This will revoke all access tokens for this agent. Its local data remains on the device but it can no longer sync."
3. User confirms
4. Server immediately invalidates the agent's access_token and refresh_token
5. Agent's next sync attempt fails with 401 -> agent notifies user: "MemKeep access was revoked. Re-bind if needed."

**If the device is compromised:**
- Revoking the agent stops sync access, but the DEK is cached locally on the device
- Changing the master password does NOT help -- agents cache the DEK itself, not the master password. The DEK stays the same after password change (only its encrypted wrapper changes)
- The compromised device can still decrypt any previously synced content with the cached DEK
- Full mitigation requires DEK rotation (generate new DEK, re-encrypt all content), which is expensive and not supported in Phase 1. Noted as a future security enhancement

### 2.9 Changing Master Password

User wants to change their master password.

**Flow (in Dashboard):**

1. User navigates to Settings -> "Change Master Password"
2. Enters current password + new password
3. Dashboard re-wraps DEK with new Master Key, uploads to server
4. Done

**Impact on agents:** None. Agents cache the DEK, not the master password. Since the DEK itself doesn't change (only its encrypted wrapper), all agents continue working without any action.

### 2.10 Claude Code Integration (Project Sync)

User uses Claude Code on multiple machines for the same project and wants Documents synced automatically.

**First machine setup:**

```
$ brew install memkeep

$ memkeep login
  Please open https://memkeep.ai/device and enter code: ABCD-1234
  Waiting for confirmation...
  Logged in as voya.
  Enter master password: ********
  Done.

$ cd ~/src/memkeep
$ memkeep init
  Detected Claude Code memory directory:
    ~/.claude/projects/-Users-voya-src-memkeep/memory/ (3 files)
  Project name: memkeep-server
  Created Project "memkeep-server".
  First sync complete. Uploaded 3 Documents.
```

**Second machine (same project, different path):**

```
$ memkeep login
  ...
$ cd ~/projects/memkeep
$ memkeep init
  Detected Claude Code memory directory:
    ~/.claude/projects/-Users-voya-projects-memkeep/memory/ (0 files)
  Project name: memkeep-server
  Found existing Project "memkeep-server" with 5 Documents.
  Sync complete. Pulled 5 Documents.
```

**Ongoing sync (via Claude Code hook):**

Configure in `~/.claude/settings.json`:
```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "command": "if [[ \"$CLAUDE_FILE_PATH\" == *\"/memory/\"* ]]; then memkeep sync --quiet; fi"
      }
    ]
  }
}
```

Effect: every time Claude Code writes a memory file, MemKeep syncs silently. User is completely unaware.

**Design principles:**
- Zero modification to Claude Code -- MemKeep works at the file system level
- `memkeep init` auto-detects Claude Code's memory directory from the current project path
- Project name is user-chosen, shared across machines by using the same name
- Sync trigger is flexible: manual CLI, hook, or future daemon mode

## 3. Account Lifecycle

### 3.1 Data Export

User wants to download all their Documents locally.

**Flow (Dashboard):**
1. Settings -> "Export Data"
2. Enter master password
3. Browser decrypts all Documents client-side
4. Downloads as a zip: one markdown file per Document, preserving directory structure

```
export_2026-04-21/
  agent_aria/
    role.md
    feedback_testing.md
    project/infra.md
  project_memkeep-server/
    MEMORY.md
    project_memkeep.md
```

Export is always **decrypted** (plaintext markdown) -- the whole point is portability. Optionally offer encrypted backup for migration to another MemKeep instance.

### 3.2 Account Deletion

**Flow (Dashboard):**
1. Settings -> "Delete Account"
2. Warning: "This will permanently delete all your data. This cannot be undone. Your agents will lose access immediately."
3. Enter master password to confirm
4. Server deletes: all encrypted content, encrypted DEK, recovery-encrypted DEK, metadata, account record
5. Confirmation email sent

**E2E encryption makes this clean:** since the server only stores ciphertext, deleting the encrypted DEK makes all content permanently unrecoverable. Deleting the ciphertext on top of that is defense-in-depth.

**GDPR compliance:** Encrypted data without its key is considered anonymized (GDPR Recital 26). We delete everything anyway (ciphertext + keys + metadata) within 30 days. Keep only an audit log entry (hashed user ID + deletion timestamp) for compliance.

## 4. Limits and Quotas

### 4.1 Default Resource Limits

Free tier defaults for the hosted service:

| Resource | Free | Pro ($9/month) |
|----------|------|-----------------|
| Projects | 5 | Unlimited |
| Agents | 3 | Unlimited |
| Documents per Project/Agent | Unlimited | Unlimited |
| Storage | 200 MB | 10 GB |
| Version history | 7 days | 90 days |

Self-hosted deployments can adjust these freely.

### 4.2 Rate Limits

| Endpoint type | Limit | Notes |
|---------------|-------|-------|
| Auth (bind, login, refresh) | 10 req/min per IP | Prevent brute force |
| Sync (push/pull) | 120 req/min per user | Higher limit for sync polling |
| CRUD (Documents, Projects, Agents) | 60 req/min per user | Standard API access |

Communication via standard headers:
- `X-RateLimit-Limit`: max requests in window
- `X-RateLimit-Remaining`: remaining requests
- `X-RateLimit-Reset`: window reset time (Unix timestamp)
- `429 Too Many Requests` with `Retry-After` header when exceeded

SDK handles 429 automatically with exponential backoff.

## 5. Error States and Graceful Degradation

### 5.1 Core Rule

**Local data is always accessible.** Server issues must never prevent the user from using their agent or reading their Documents. Sync is a background enhancement, not a dependency.

### 5.2 Error Scenarios

| Scenario | Agent behavior | Dashboard behavior |
|----------|---------------|-------------------|
| Server unreachable | Works normally with local Documents. Queues changes. Subtle status: "Sync pending" | Shows cached data if available, or "Server unavailable, retrying..." |
| Sync push fails | Retry with exponential backoff (1s, 2s, 4s... cap 5min). No user interruption | N/A |
| Sync pull fails | Use local data. Retry on next sync cycle | N/A |
| Auth token expired | SDK auto-refreshes. If refresh fails, agent notifies: "Please re-authenticate with MemKeep" | Redirect to login |
| Master password wrong | "Incorrect password. Please try again." (verified via Password Verify Hash before attempting DEK decryption) | Same, with retry limit (5 attempts, then cooldown) |
| Conflict detected | Server auto-resolves, stores conflict copy. Agent may mention: "1 Document conflict detected, resolve in dashboard" | Show in conflicts list |
| Storage quota exceeded | Push rejected with clear error. Agent: "MemKeep storage full. Clean up old Documents or upgrade." | Banner: "Storage 98% full" |

### 5.3 Retry Strategy

```
Sync failure retry:
  1st: wait 1s
  2nd: wait 2s
  3rd: wait 4s
  4th: wait 8s
  ...
  cap: wait 5 min
  
  After 1 hour of failures: stop retrying, notify user once
  Resume on: next manual sync, or next app/session start
```

## 6. Observability (Dashboard)

### 6.1 Sync Status

Each Agent and Project shows a sync health indicator:

```
Agents:
  Aria            [green]  Last sync: 2 min ago    56 Documents
  Echo            [green]  Last sync: 15 min ago   12 Documents

Projects:
  memkeep-server  [green]  Last sync: 5 min ago    5 Documents  (2 devices)
  my-side-project [yellow] Last sync: 3 days ago   8 Documents
```

Color coding:
- Green: synced within last hour
- Yellow: last sync > 24 hours ago
- Red: sync errors or auth issues

### 6.2 Activity Log

```
Dashboard -> Activity

2026-04-21 14:30  Aria              pushed 2 Documents
2026-04-21 14:25  memkeep-server    pulled 1 Document (from desktop)
2026-04-21 13:00  Echo              conflict detected: project/infra.md
2026-04-21 10:00  Aria (desktop)    bound successfully
```

Helps user understand what's happening across their agents without needing to check each one.

### 6.3 Storage Usage

```
Dashboard -> Settings -> Storage

Used: 4.2 MB / 200 MB (Free)

By owner:
  Aria              2.8 MB (56 Documents)
  Echo              0.6 MB (12 Documents)
  memkeep-server    0.3 MB (5 Documents)
  my-side-project   0.5 MB (8 Documents)
```

## 7. Experience Principles

1. **Invisible when working** -- sync happens silently, user only notices MemKeep when they need it
2. **User always decides** -- no automatic overwrites, no silent merges. Agent suggests, user approves
3. **Agent does the thinking** -- agents can understand content semantics, not just diff bytes. This is MemKeep's advantage over traditional sync tools
4. **Master password once** -- entered once per device/session, not on every operation
5. **Dashboard for overview, agent for action** -- dashboard gives the bird's eye view across all agents and projects; individual agents handle conversational operations
6. **Local-first** -- local data is always accessible, server is a sync enhancement not a dependency
7. **Graceful degradation** -- errors are handled silently with retries, user is only notified when action is needed

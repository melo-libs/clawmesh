# ClawMesh User Stories & Flows

> Version: 0.1.0 | Date: 2026-04-05 | Status: Draft

## 1. Personas

### Primary User

A developer who uses multiple AI bots (OpenClaw, Claude Code, etc.) across different devices and environments. Core pain point: **knowledge is siloed** — what one bot learns, the others don't know.

Typical setup:
- Bot A: OpenClaw on laptop (daily driver)
- Bot B: OpenClaw on cloud server (for deployments)
- Bot C: NanoClaw on phone (quick lookups)

## 2. User Journeys

### 2.1 First-Time Setup (Registration + First Bot Binding)

The most critical flow. Must feel as simple as setting up a password manager.

**Steps:**

1. User tells their bot: "Connect to ClawMesh, I'm voya"
2. Bot initiates bind request, receives a verification code
3. Bot displays: "Your code is MESH-7829. Go to clawmesh.dev to sign up and confirm."
4. User opens clawmesh.dev:
   - Signs in with GitHub (one-click)
   - Sets a master password (prompted: "This protects all your bot memories")
   - Saves Recovery Key (prompted: "Save this somewhere safe — it's the only way to recover if you forget your password")
   - Enters verification code MESH-7829 to confirm the bind
5. User returns to bot, provides master password when prompted
6. Bot derives DEK, starts syncing
7. Bot confirms: "Connected! I'll sync your memories automatically from now on."

**Design principles:**
- Registration and first bind are one flow, not two separate steps
- Master password and Recovery Key setup happen during registration, not as afterthought
- User provides master password to bot only once — DEK is cached locally after that
- Total interaction: 1 OAuth login + 1 password + 1 verification code

### 2.2 Day-to-Day Sync (Invisible)

After setup, the user should not think about ClawMesh at all.

**Trigger options (SDK configurable):**
- Timer-based: every N minutes (default: 5 min)
- Event-based: after bot writes a new memory
- Manual: user says "sync" or "sync now"

**Behavior:**
- Sync runs silently in the background
- No notifications, no interruptions on success
- On error (network issue, conflict), bot may briefly mention it: "Heads up: I couldn't sync, will retry later"
- Optionally, bot can mention new memories at conversation start: "Synced 3 new memories from cloud" (configurable, off by default)

### 2.3 Binding an Additional Bot

User sets up a new device or bot instance.

**Steps:**

1. User tells the new bot: "Connect to ClawMesh, I'm voya"
2. Bot initiates bind, displays code: MESH-4521
3. User receives a notification (dashboard / email / push):
   "New bot 'server-claw' is requesting access. Code: MESH-4521. Approve?"
4. User taps [Approve]
5. User provides master password to the new bot
6. Bot obtains DEK, starts syncing
7. Done

**Difference from first-time setup:** No registration, no password setup. Just confirm code + provide master password.

### 2.4 Knowledge Migration (New Bot from Scratch)

A newly bound bot has zero memories. User wants to bootstrap it with knowledge from an existing bot.

**Flow (in Dashboard):**

1. User navigates to Bots -> new-laptop-claw -> "Inherit Memories"
2. Dashboard shows:
   - Source bot selector (e.g., "laptop-claw")
   - Type filter: checkboxes for feedback, project, user, reference
   - Tag filter: optional include/exclude tags
3. User selects filters, clicks "Preview"
4. Dashboard shows: "43 memories match. [Confirm Inherit]"
5. User confirms -> server executes snapshot copy
6. New bot picks up the 43 memories on next sync

**Design principles:**
- Done in dashboard, not in bot (user needs the full picture of all bots)
- Preview before confirm -- user knows exactly what will be copied
- Bot-side is zero effort -- just normal sync picks it up

### 2.5 Selective Cross-Bot Sync (Both Bots Have Memories)

The more common real-world scenario: Bot A and Bot B both have memories, user wants to selectively copy some from A to B.

**Flow (Bot-initiated, conversational):**

1. User tells Bot B: "Sync the feedback memories from Bot A"
2. Bot B (via SDK) fetches Bot A's feedback memory metadata from server
3. Bot B compares with its own local memories:
   - **New** (Bot A has, Bot B doesn't): candidates for copy
   - **Conflict** (same file_path, different content): needs user decision
   - **Same** (identical content): skip automatically
4. For conflicts, Bot B downloads Bot A's encrypted content, decrypts locally, compares
5. Bot B reports to user:

   > "Found 15 feedback memories in Bot A:
   > - 9 are new to me
   > - 4 I also have but with different content:
   >   - feedback_testing.md: Bot A says 'use real DB', I say 'mock for unit tests, real DB for integration'
   >   - ...
   > - 2 are identical, skipping
   >
   > How should I handle these?"

6. User decides: "Take the 9 new ones. For conflicts, keep yours."
7. Bot B downloads and saves the 9 new memories, syncs normally

**Design principles:**
- Bot-initiated and conversational -- user gives instructions in natural language
- Bot does the intelligence: it can understand content semantics, not just file diff
- Bot can even suggest merging conflicting memories (AI advantage over traditional sync)
- Server only provides the data channel -- all comparison and decision-making happens client-side (E2E encrypted, server sees nothing)
- Bot A does not need to be online -- its data is already on the server from previous syncs

**API support needed:**
- Existing memory endpoints accept `bot_id` query parameter for cross-bot reads
- Server verifies both bots belong to the same user

### 2.6 Memory Management (Dashboard)

User wants to review, edit, or clean up bot memories.

**Dashboard layout:**

```
Bots -> laptop-claw -> Memories

Search: [________]  Type: [All v]  Tags: [All v]

file_path                type       version  updated
role.md                  user       v3       10 min ago
feedback_testing.md      feedback   v5       1 hour ago
project/infra.md         project    v2       yesterday
project/api_design.md    project    v1       3 days ago
--- inherited ---
feedback_db.md           feedback   v4       (from server-claw)
```

**Flows:**

- **View**: Click a memory -> enter master password (once per session) -> content decrypted in browser -> displayed
- **Edit**: Modify content in browser -> encrypted before upload -> version incremented
- **Delete**: Confirm -> removed from server -> bot's next sync removes local file
- **Search**: Searches metadata (type, tags, file_path) -- content is encrypted, not searchable server-side
- **Inherited memories**: Clearly labeled with source bot, can be edited or deleted independently

### 2.7 Conflict Resolution (Dashboard)

Conflicts are rare but need a clear resolution path.

**When it happens:** A bot and the dashboard modify the same memory between syncs. Conflicts are within a single bot's namespace -- different bots have independent copies and cannot conflict with each other through normal sync.

**Example:** User edits `project/infra.md` in dashboard while laptop-claw also modifies it locally. On next push, laptop-claw's base_version doesn't match the server version -> conflict.

**Dashboard flow:**

```
Dashboard -> laptop-claw -> Conflicts (1)

project/infra.md -- conflicted at 2026-04-05 14:30

Server version (v5, edited in dashboard):      Bot version (v4, pushed by laptop-claw):
+------------------------------+  +------------------------------+
| DB: PostgreSQL on AWS RDS    |  | DB: PostgreSQL, self-hosted  |
+------------------------------+  +------------------------------+

[Keep Server Version]  [Use Bot Version]  [Edit Manually]
```

**Design principles:**
- Conflicts don't block sync -- server auto-resolves (keeps one, stores the other as conflict copy)
- Dashboard shows both versions side-by-side, decrypted in browser
- User picks one, or manually edits a merged version
- Resolved conflicts are cleaned up automatically
- Conflicts are per-bot. To propagate a resolution to other bots, use cross-bot sync (Section 2.5)

### 2.8 Revoking a Bot

User loses a device or wants to remove a bot's access.

**Flow (in Dashboard):**

1. User navigates to Bots -> laptop-claw -> "Remove Bot"
2. Dashboard warns: "This will revoke all access tokens for this bot. Its local data remains on the device but it can no longer sync."
3. User confirms
4. Server immediately invalidates the bot's access_token and refresh_token
5. Bot's next sync attempt fails with 401 -> bot notifies user: "ClawMesh access was revoked. Re-bind if needed."

**If the device is compromised:**
- Revoking the bot stops sync access, but the DEK is cached locally on the device
- Changing the master password does NOT help -- bots cache the DEK itself, not the master password. The DEK stays the same after password change (only its encrypted wrapper changes)
- The compromised device can still decrypt any previously synced content with the cached DEK
- Full mitigation requires DEK rotation (generate new DEK, re-encrypt all content), which is expensive and not supported in Phase 1. Noted as a future security enhancement

### 2.9 Changing Master Password

User wants to change their master password.

**Flow (in Dashboard):**

1. User navigates to Settings -> "Change Master Password"
2. Enters current password + new password
3. Dashboard re-wraps DEK with new Master Key, uploads to server
4. Done

**Impact on bots:** None. Bots cache the DEK, not the master password. Since the DEK itself doesn't change (only its encrypted wrapper), all bots continue working without any action.

### 2.10 Claude Code Integration (Workspace Sync)

User uses Claude Code on multiple machines for the same project and wants memories synced automatically.

**First machine setup:**

```
$ brew install clawmesh

$ clawmesh login
  Please open https://clawmesh.dev/device and enter code: ABCD-1234
  Waiting for confirmation...
  Logged in as voya.
  Enter master password: ********
  Done.

$ cd ~/src/clawmesh
$ clawmesh init
  Detected Claude Code memory directory:
    ~/.claude/projects/-Users-voya-src-clawmesh/memory/ (3 files)
  Workspace name: clawmesh
  Created workspace "clawmesh".
  First sync complete. Uploaded 3 memories.
```

**Second machine (same project, different path):**

```
$ clawmesh login
  ...
$ cd ~/projects/clawmesh
$ clawmesh init
  Detected Claude Code memory directory:
    ~/.claude/projects/-Users-voya-projects-clawmesh/memory/ (0 files)
  Workspace name: clawmesh
  Found existing workspace "clawmesh" with 5 memories.
  Sync complete. Pulled 5 memories.
```

**Ongoing sync (via Claude Code hook):**

Configure in `~/.claude/settings.json`:
```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "command": "if [[ \"$CLAUDE_FILE_PATH\" == *\"/memory/\"* ]]; then clawmesh sync --quiet; fi"
      }
    ]
  }
}
```

Effect: every time Claude Code writes a memory file, ClawMesh syncs silently. User is completely unaware.

**Design principles:**
- Zero modification to Claude Code -- ClawMesh works at the file system level
- `clawmesh init` auto-detects Claude Code's memory directory from the current project path
- Workspace name is user-chosen, shared across machines by using the same name
- Sync trigger is flexible: manual CLI, hook, or future daemon mode

## 3. Experience Principles

1. **Invisible when working** -- sync happens silently, user only notices ClawMesh when they need it
2. **User always decides** -- no automatic overwrites, no silent merges. Bot suggests, user approves
3. **Bot does the thinking** -- bots can understand memory content semantically, not just diff bytes. This is ClawMesh's advantage over traditional sync tools
4. **Master password once** -- entered once per device/session, not on every operation
5. **Dashboard for overview, bot for action** -- dashboard gives the bird's eye view across all bots; individual bots handle conversational operations

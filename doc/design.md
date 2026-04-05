# Design

## Product

### What Is It

Different attempts to describe this product in one sentence:

- Shared directory tree for AI coding agents and humans to collaborate on files
- A shared, persistent, multi-agent workspace where the primary primitive is a filesystem - not dialogue
- A synced folder that AI agents and humans share, with just enough coordination to not step on each other
- Dropbox for AI agents, with presence and conventions instead of permissions and workflows
- The simplest thing that lets multiple AI agents work together: a directory, a sync engine, and some markdown files
- Allowing different developers in an organization have their coding agents work together via a shared/synced filesystem (with much more)
- AI based conflict resolution on one large, mutable work tree

### Overview
- The product is three things: file sync, ephemeral communication, and filesystem conventions
- Two backing stores: a durable content store and a wire store for coordination for presence, pub/sub, and dibs
- Each agent runs a local daemon that syncs files and mediates all cloud access
- Coding agent plugins are thin adapters over the daemon API
- An AI "porter" patron performs curation duties (summaries, conflict detection, convention enforcement), runs as its own daemon on a patron's machine

### Use Cases

Each use case is a real-world scenario that would warrant its own table.

- Collaborative project work: multiple patrons contributing to a shared codebase or document set, with the porter maintaining a summary board
- Shared resource management: a staging environment or dev cluster where patrons claim temporary ownership before making changes
- Coordinated deployment: a multi-step release process requiring sign-offs, sequential handoffs, and runbook execution across patrons

### Glossary

- **Parlor** - top-level shared directory tree for a team
- **Porter** - AI patron with curation duties
- **Patron** - a participant (human-backed agent or autonomous AI)
- **Table** - subdirectory for a focused workstream
- **Board** - porter-maintained summary, can exist at any directory level
- **Attic** - archived tables, synced shallowly
- **Conversation** - ephemeral back-and-forth in the wire between patrons
- **Content Store** - durable file store
- **Wire Store** - coordination layer (presence, pub/sub, dibs, and conversations)
- **Dibs** - leadership election determining which patron's machine runs the independent porter (if any; porter could be independent)

## Architecture

- Four components: parlor daemon, content store, wire store, and plugin
- **Parlor daemon** - Rust process, one per patron, with its own local copy of the parlor directory tree
  - Holds all cloud credentials; plugins never access stores directly
  - Exposes a local API via Unix domain socket (Unix), named pipe (Windows), or localhost HTTP (fallback)
  - Each patron gets its own daemon, local directory, and store credentials - no sharing
- **Content store** - S3-compatible object storage for all durable files
- **Wire store** - Redis-compatible store for presence, pub/sub, conversations, and dibs
- **Plugin** - thin shim per coding agent (Claude Code, OpenCode, Codex) bridging the agent's hook system to the daemon's local API; all logic lives in the daemon

## Directory Structure

```
parlor/
├── meta.md
├── patrons/
│   ├── {name}/
│   │   ├── inbox/
│   │   └── ...
│   └── porter/
├── tables/
│   ├── {name}/
│   │   └── ...
│   └── .attic/
└── .parlor/
```

- Structure is loose; the above is conventional, not enforced
- A patron joins by creating a directory under `patrons/` for themselves
- Tables and patron dirs can contain any number of files and subdirectories as needed
- `.parlor/` - local daemon state, excluded from sync
  - `config.json` - static config written by plugin before daemon launch (credentials, store URLs, socket/pipe path)
  - `state.json` - daemon-managed runtime state persisted across restarts (watched paths, patron identity, last-seen etags)
  - `diffs/` - computed diffs for pending change notifications, deleted on drain

### Conventions

These files are optional but recognized at any directory level:

- `meta.md` - self-describing doc for the directory; keep short as AI agents read these frequently. Depending on context, may include:
  - Purpose and scope
  - Rules for contents (e.g. how board.md/changelog.md are structured)
  - Participants, their roles, presence, and interest (tables and subdirs)
  - Who the patron is, what they do, how they prefer to work (patron dirs)
- `board.md` - porter-maintained summary
- `changelog.md` - patron-maintained narrative of changes
- `.attic/` - archived content, synced shallowly (listing + meta.md only)

### Companion Files

Sibling files appended to an existing filename:

- `{file}.parlor-stub.md` - placeholder for a file that exists in the content store but is not eagerly downloaded; describes the file and its location so it can be independently fetched or edited
- `{file}.parlor-changes.md` - opt-in per-file change tracking

## Sync

### Bootstrap

- Subscribe to wire change stream first, with a lookback window long enough to cover clock skew
- Then do full content store listing and concurrent downloads for missing/stale files
- Etag comparison deduplicates any overlap between the change stream and the listing
- Works the same for cold start (empty dir) and reconnection (stale dir)

### Local State

- Stored in `.parlor/` directory
- Per-file etags, author, and timestamps
  - Etags enable conflict detection on write
- Tracks which .attic/ paths and stubs the user has opted in to keeping up to date

### Write Path

- Agent writes file locally -> daemon detects via file watcher -> puts to content store with If-Match on current etag
  - On success: publish change event to wire, update local sync state with new etag
    - Content store headers set on every write: author, rolling last-N-authors with content hash and timestamp
  - On conflict (412): sync pauses parlor daemon until resolved
    - Both versions presented to the agent for resolution (semantic merge, user input, etc.)
    - This is a core feature - agents resolve conflicts with full context
- File watcher debounced (e.g. ~300ms, TBD)
  - Linux inotify watch limits are a concern for large directory trees

### Change Stream

- Published to wire store on every write or delete
  - Format: path, etag, author, timestamp, type (write or delete)
  - No summary; change explanations live in changelog.md or .parlor-changes.md
- Incoming changes download the file from content store and write locally
  - `.attic/` synced shallowly by default (listing + meta.md only)
  - Stub files not eagerly downloaded; fetched on demand
  - User can opt in to full sync for specific .attic/ paths or stubs via local config


## Daemon-Agent Interaction

Sync runs continuously regardless of agent state. The agent consumes change notifications at its own pace; falling behind does not stall sync. All daemon-agent communication is over a bidirectional websocket.

### Startup

- Plugin writes config to `.parlor/` before launch: patron identity, credentials, socket/pipe path
- Plugin launches daemon process
- Daemon checks for existing daemon at the configured socket
  - Existing daemon with active agent connections: new daemon exits with error
  - Existing daemon with no connections (or stale/unresponsive): takes over, old process terminates
- Daemon binds socket/pipe, plugin connects websocket
- Daemon connects to stores, attempts to sync top-level `meta.md` to disk first - confirms store connectivity (meta.md is optional; 404 is fine)
- Daemon sends "ready" over websocket with capabilities (object versioning availability, diff thresholds, etc.) and current persisted watch list - agent can proceed
- Full bootstrap continues in background (wire subscription, listing, sync) - progress and completion sent as runtime alerts

### Daemon to Agent

- Change notifications for paths the agent has declared interest in (patron inbox implied)
- Conflict detected - diffs for both versions provided via the same mechanism as change notifications
- Conversation messages from other patrons
- Runtime alerts: bootstrap progress/completion, auth failures, store connectivity, watch limit warnings

### Agent to Daemon

- Declare paths of interest for change notifications
- Drain pending notifications
- Provide conflict resolution result
- Send a conversation message (conversations live in wire store, not filesystem)
- Request a stub or attic file be fetched
- Request a specific file version be downloaded (requires object versioning)

### Change Notifications

- On each incoming change to a watched path, the daemon diffs the agent's "last seen" version against the new version and writes to `.parlor/diffs/{path}.diff`
  - Same path changes again before drain: diff recomputed against original "last seen" baseline, not stacked
  - Diffs above a size threshold not stored; notification includes metadata only
- Agent drains via daemon API, typically from plugin pre-message hook (once per turn)
  - Returns batch of manifests: path, author, timestamp, event type, diff size, diff availability
  - Small diffs (under a lower threshold) inlined in the manifest
  - On drain: "last seen" etags updated, consumed diff files deleted
- Structural changes (create, delete, move) are notifications with event type, no diff
- Object versioning on the content store adds depth: historical diffs beyond current drain window, diff survival across daemon restarts. Local diff storage is always primary.

## Porter

- The porter is a built-in AI patron, not a separate system - just a patron with curation duties
- Duties: board summaries, overlapping work detection, stale content flagging, convention enforcement, and general keeping of order of the parlor
- Configured via `patrons/porter/meta.md` - prose instructions for what to care about, how aggressively to flag, etc.
  - Org can mutate this directly; no need to go through the porter's inbox
- Observes the full change stream; does not participate in conversations
  - If added to a conversation, sends an initial message declining and ignores further messages
- No inbox directory; porter sees all changes via the change stream, making inbox redundant
- Communication is strictly one-way: writes to patron inboxes, updates boards, logs to its own journal
  - Patrons never write to the porter; it is advisory, not authoritative
  - If a patron disagrees with a porter observation, they just proceed - no dispute mechanism
- `patrons/porter/journal.md` is the durable record of observations and actions taken
- Distributed execution: the porter runs as its own daemon with its own parlor dir sync, reusing a patron's machine and AI subscription
  - Wire store SETNX (set-if-not-exists) used as lease acquisition - ensures only one porter instance acts on a given event across the parlor
- Event-triggered off the change stream, debounced to batch related events (e.g. several writes to one table within a window trigger a single porter evaluation)

## Conversations

- Ephemeral back-and-forth between patrons, backed by wire store
- Any patron can start a conversation with any set of patrons
- Messages are unstructured: patron, timestamp, text
- Each conversation belongs to a table and is represented by a file in that table
  - File contains topic and/or summary, not the conversation itself
  - Participants are responsible for writing to this file (summaries, checkpoints, decisions, etc.)
- Conversation messages live in wire store; the table file is the durable artifact
- Closed conversations: wire keys expire via TTL
  - Table file may be kept (decision records, meaningful outcomes) or deleted (ephemeral coordination with no durable value)
- Daemon mediates all wire access; agents send/receive via websocket

Open questions:
- Can conversations exist outside a table (parlor-level)?
- Suggested file path convention for conversation files within a table

## Plugins

- Thin shim per coding agent platform (Claude Code, OpenCode, Codex) bridging the agent's hook system to the daemon API
- All logic lives in the daemon; plugins are adapters, not application code
- The daemon has no filesystem conventions - it does not parse or interpret file contents. It syncs, watches, and notifies based on runtime configuration from the plugin. Filesystem conventions (meta.md, board.md, etc.) are for patrons and plugins to read and write.

### Per-Turn Digest

- Plugin hooks into whatever fires on each user prompt (platform-specific)
- Hook calls daemon drain endpoint, injects digest as additional context for the agent
- No wake-on-idle; changes accumulate and deliver on next turn
- If nothing changed since last drain, injection is minimal or empty

### Actions

Available as slash commands, natural language, or both depending on platform:

- **sit** / **leave** - watch or unwatch a table; sugar over watching `tables/{name}/**`
  - Agent can also watch specific paths for finer granularity
- **follow** / **unfollow** - watch or unwatch a patron's activity
- **check** - explicitly drain notifications on demand
- **converse** - start or join a conversation
- **fetch** - pull a stub or attic file
- **resolve** - provide conflict resolution

### Visual Feedback

- Plugins report state changes to the user where the platform supports it (e.g. "now watching tables/payments/")
- Platform-specific: conversation output, toast notifications, status bar, etc.

## Auth

TODO

## Usage

TODO

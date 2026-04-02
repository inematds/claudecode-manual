# Session Management

## Lesson 29: Session Management

How Claude Code persists, restores, and recovers conversations — from JSONL on disk to cloud hydration and interrupted-turn auto-resume.

### 01 Overview & Mental Model

Every conversation in Claude Code is a **session**. A session has a UUID, a JSONL transcript file, and a collection of metadata entries (title, agent config, worktree path, git branch, etc.). The session management layer spans four main concerns:

**Layer 1: Storage**
Append-only JSONL transcript files under `~/.claude/projects/`. Batched async writes via a per-file queue inside the `Project` singleton.

**Layer 2: State Tracking**
In-process tri-state machine (`idle` / `running` / `requires_action`) with listener hooks for the CCR bridge and SDK consumers.

**Layer 3: Restore / Resume**
Chain-walk of the parentUuid linked list on disk, deserialization filters, worktree re-entry, and agent-setting re-hydration.

**Layer 4: Recovery**
Interrupted-turn detection: mid-turn crashes become a synthetic "Continue from where you left off." message; the SDK can auto-resume without user input.

**Layer 5: Cloud Sync**
Dual remote-persistence paths: v1 Session Ingress (REST) and CCR v2 internal events. Hydration rewrites the local JSONL from remote data on reconnection.

**Layer 6: History API**
Paginated event fetch for remote session replay — used by the CCR web/mobile UI to scroll back through events in a live or completed session.

**Key Files**
- `utils/sessionStorage.ts` — core write path, Project class, hydration
- `utils/sessionState.ts` — idle/running/requires_action state machine
- `utils/sessionRestore.ts` — processResumedConversation orchestrator
- `utils/conversationRecovery.ts` — loadConversationForResume, interrupt detection
- `assistant/sessionHistory.ts` — remote paginated event fetch

### 02 Disk Layout

Sessions are stored under `~/.claude/projects/` (or `CLAUDE_CONFIG_DIR/projects/`). Each project directory is a sanitized, path-encoded representation of the working directory. Inside each project directory lives one JSONL file per session UUID, plus sidecar directories for subagents.

```
~/.claude/projects/
├── -Users-alice-myproject/           <- sanitizePath(cwd)
│   ├── 550e8400-e29b-41d4-a716-446655440000.jsonl  <- main transcript
│   ├── 550e8400-e29b-41d4-a716-446655440000/
│   │   ├── subagents/
│   │   │   ├── agent-<agentId>.jsonl          <- sidechain transcript
│   │   │   └── agent-<agentId>.meta.json      <- agentType, worktreePath
│   │   └── remote-agents/
│   │       └── remote-agent-<taskId>.meta.json
│   └── a1b2c3d4-....jsonl               <- another session
└── -Users-alice-otherproject/
    └── ...
```

The path derivation logic is inside `getProjectDir` (memoized):

```typescript
// utils/sessionStorage.ts
export const getProjectDir = memoize((projectDir: string): string => {
  return join(getProjectsDir(), sanitizePath(projectDir))
})

export function getTranscriptPath(): string {
  const projectDir = getSessionProjectDir() ?? getProjectDir(getOriginalCwd())
  return join(projectDir, `${getSessionId()}.jsonl`)
}
```

**Subtlety**

`getOriginalCwd()` is used — not `getCwd()` at module load time. This matters because bootstrap resolves symlinks via `realpathSync` after module import, so capturing too early produces a different sanitized path and makes sessions "invisible" to the resume picker.

### 03 The Project Singleton and Write Queue

All writes go through a single `Project` class instance obtained via `getProject()`. The class batches entries into per-file queues and drains them every 100 ms (or 10 ms when cloud persistence is active). This prevents thundering-herd file I/O during high-throughput turns.

```
Write Path
recordTranscript(messages)
│
▼
Project.insertMessageChain()        <- dedup via messageSet Set<UUID>
│
▼
Project.appendEntry(entry)          <- checks shouldSkipPersistence()
├─> [pending] pendingEntries[]      <- before sessionFile materialized
└─> enqueueWrite(filePath, entry)
    │
    ▼
    writeQueues: Map<path, Entry[]>
    │
    scheduleDrain() ─────────────── setTimeout(100ms)
    │
    ▼
    drainWriteQueue()
    ├─> fsAppendFile(path, batchedJsonl)    <- local disk
    └─> persistToRemote(sessionId, entry)   <- CCR v2 or v1 ingress
```

#### Lazy File Materialization

The session file is **not** created immediately. It is deferred until the first real user or assistant message arrives. This prevents polluting the resume list with metadata-only shells from sessions that never exchanged meaningful content.

```typescript
// utils/sessionStorage.ts — Project class
async insertMessageChain(messages: Transcript, ...) {
  // First user/assistant message materializes the session file.
  // Hook progress/attachment messages alone stay buffered.
  if (
    this.sessionFile === null &&
    messages.some(m => m.type === 'user' || m.type === 'assistant')
  ) {
    await this.materializeSessionFile()
  }
  // ... then write entries ...
}
```

#### Tail-Window Metadata Re-Append

Metadata like `customTitle`, `tag`, and `last-prompt` must stay within the last 64 KB of the file (the "tail window") because the resume picker uses a fast head+tail read instead of parsing the full JSONL. On every compaction and session exit, `reAppendSessionMetadata()` moves these entries back to EOF.

```typescript
// utils/sessionStorage.ts
reAppendSessionMetadata(skipTitleRefresh = false): void {
  if (!this.sessionFile) return
  // Sync tail read to absorb any SDK-written fresher title/tag
  const tail = readFileTailSync(this.sessionFile)
  // ... merge then re-append last-prompt, customTitle, tag,
  //     agent-name, agent-color, agent-setting, mode, worktree-state, pr-link ...
}
```

### 04 JSONL Entry Types

The JSONL file is not a flat list of messages. It is a superset — a mix of conversation entries and sidecar metadata entries. Here is the taxonomy:

| Entry type | Purpose | Participates in chain? |
|---|---|---|
| `user` | Human turn message | Yes (parentUuid) |
| `assistant` | Model response | Yes |
| `attachment` | Files, skills, invoked_skills | Yes |
| `system` | System/compact boundary messages | Yes |
| `custom-title` | User-renamed session name | No — metadata only |
| `last-prompt` | Last user text (for resume picker) | No |
| `tag` | Session tag label | No |
| `agent-name` | Standalone agent display name | No |
| `agent-setting` | Agent type used in session | No |
| `mode` | coordinator / normal mode | No |
| `worktree-state` | Git worktree path and original cwd | No |
| `pr-link` | PR number/URL for GitHub workflows | No |
| `file-history-snapshot` | File diff history for each turn | No |
| `content-replacement` | Large tool output substitution records | No |
| `marble-origami-commit` | Context-collapse commit (ant-only) | No |

**Important**

Progress entries (`bash_progress`, `mcp_progress`, etc.) are **never persisted** to JSONL. They are ephemeral UI state. Old transcripts may contain them — `loadTranscriptFile` bridges the chain across them during load.

### 05 The parentUuid Linked List

Messages form a linked list via `parentUuid`. This design allows branching (sidechains for subagents, forked sessions) while keeping the JSONL append-only. On load, Claude Code walks the chain from the newest leaf back to the root, then reverses it.

```
parentUuid chain (conceptually a singly-linked list, stored reversed)

ROOT                                                           LEAF (newest)
│                                                              │
▼                                                              ▼
[user-A] <── [asst-B] <── [user-C] <── [asst-D] <── [user-E]
uuid=A       uuid=B       uuid=C       uuid=D       uuid=E
parent=null  parent=A     parent=B     parent=C     parent=D

On disk (append order):          On load (buildConversationChain):
user-A                           Start at LEAF (E), walk parentUuids
asst-B                           E -> D -> C -> B -> A -> null
user-C                           .reverse() -> [A, B, C, D, E]
asst-D
user-E

Sidechains (subagent transcripts): isSidechain=true, separate file
Forks: fresh sessionId, copies source chain, re-stamps sessionId
Compact boundary: parentUuid=null (truncates chain at compact point)
```

```typescript
// utils/sessionStorage.ts
export function buildConversationChain(
  messages: Map<UUID, TranscriptMessage>,
  leafMessage: TranscriptMessage,
): TranscriptMessage[] {
  const transcript: TranscriptMessage[] = []
  const seen = new Set<UUID>()
  let currentMsg = leafMessage
  while (currentMsg) {
    if (seen.has(currentMsg.uuid)) {
      logError(new Error(`Cycle detected in parentUuid chain...`))
      break
    }
    seen.add(currentMsg.uuid)
    transcript.push(currentMsg)
    currentMsg = currentMsg.parentUuid
      ? messages.get(currentMsg.parentUuid)
      : undefined
  }
  transcript.reverse()
  return recoverOrphanedParallelToolResults(messages, transcript, seen)
}
```

### 06 Session State Machine

The `sessionState.ts` module provides a singleton tri-state machine for the current session. State transitions fire listener callbacks that wire into: the CCR bridge (for `requires_action` notifications), the SDK event stream (for VS Code / scmuxd), and the external metadata store (for the pending-action UI).

```
SessionState tri-state machine

user sends message
┌──────────────────────────────────────────────────────┐
│                                                      │
▼                 model starts responding              │
[ idle ] ──────────────────────────────────-> [ running ]
▲                                              │
│                    model finishes / error    │
└──────────────────────────────────────────────┘
       ▲
       │ tool needs approval
       │
   [ requires_action ]
       │
       │ user approves/denies
       └──────────────────────────
```

Side effects of each transition:
- `running` -> `stateListener(state)`
- `requires_action` -> `stateListener + metadataListener({pending_action: details})`
- `idle` -> `metadataListener({pending_action: null, task_summary: null}) + optional SDK system:session_state_changed event`

```typescript
// utils/sessionState.ts
export type SessionState = 'idle' | 'running' | 'requires_action'

export type RequiresActionDetails = {
  tool_name: string
  /** Human-readable summary, e.g. "Editing src/foo.ts" */
  action_description: string
  tool_use_id: string
  request_id: string
  input?: Record<string, unknown>
}

export function notifySessionStateChanged(
  state: SessionState,
  details?: RequiresActionDetails,
): void {
  currentState = state
  stateListener?.(state, details)
  if (state === 'requires_action' && details) {
    hasPendingAction = true
    metadataListener?.({ pending_action: details })
  } else if (hasPendingAction) {
    hasPendingAction = false
    metadataListener?.({ pending_action: null })
  }
  if (state === 'idle') {
    metadataListener?.({ task_summary: null })
  }
}
```

#### External Metadata

The `SessionExternalMetadata` type is a grab-bag of fields pushed to the CCR session record via RFC 7396 JSON Merge Patch. The `pending_action` field lets the CCR sidebar show _what_ the model is blocked on without a protobuf schema change — it is cleared (set to `null`) on every non-blocked transition.

```typescript
// utils/sessionState.ts
export type SessionExternalMetadata = {
  permission_mode?: string | null
  is_ultraplan_mode?: boolean | null
  model?: string | null
  pending_action?: RequiresActionDetails | null
  post_turn_summary?: unknown
  // Mid-turn progress every ~5 steps / 2min so long-running turns
  // show "what's happening now" before post_turn_summary arrives.
  task_summary?: string | null
}
```

### 07 Conversation Recovery — Interrupt Detection

When Claude Code loads a transcript for resume, it must determine whether the previous session was interrupted mid-turn. This matters because:

*   If interrupted mid-tool-use (tool_result is the last entry), a "Continue from where you left off." message is injected.
*   If interrupted before the model responded to a user prompt, the raw user message is returned as `interrupted_prompt` for the SDK to auto-retry.
*   If the turn completed normally, no synthetic message is added.

```
detectTurnInterruption() logic (conversationRecovery.ts)

After filterUnresolvedToolUses + filterOrphanedThinkingOnlyMessages:

lastRelevant = last message where type != 'system' && type != 'progress'
               and not an API error

lastRelevant.type == 'assistant'
  -> { kind: 'none' } (turn completed — stop_reason always null on disk)

lastRelevant.type == 'user' && isMeta
  -> { kind: 'none' } (synthetic tick message, not a real turn)

lastRelevant.type == 'user' && isToolUseResultMessage
  -> isTerminalToolResult? (SendUserFile, Brief tool)
    yes -> { kind: 'none' } (brief mode: turn ends on SendUserMessage)
    no  -> { kind: 'interrupted_turn' }

lastRelevant.type == 'user' (plain text)
  -> { kind: 'interrupted_prompt', message: lastRelevant }

lastRelevant.type == 'attachment'
  -> { kind: 'interrupted_turn' }

interrupted_turn is promoted to interrupted_prompt by appending:
createUserMessage({
  content: 'Continue from where you left off.',
  isMeta: true
})
```

```typescript
// utils/conversationRecovery.ts
export function deserializeMessagesWithInterruptDetection(
  serializedMessages: Message[],
): DeserializeResult {
  const migratedMessages = serializedMessages.map(migrateLegacyAttachmentTypes)
  // Strip invalid permissionMode values from disk (unvalidated JSON)
  const validModes = new Set<string>(PERMISSION_MODES)
  // ... strip invalid modes ...

  const filteredToolUses  = filterUnresolvedToolUses(migratedMessages)
  const filteredThinking  = filterOrphanedThinkingOnlyMessages(filteredToolUses)
  const filteredMessages  = filterWhitespaceOnlyAssistantMessages(filteredThinking)

  const internalState = detectTurnInterruption(filteredMessages)

  if (internalState.kind === 'interrupted_turn') {
    const [continuationMessage] = normalizeMessages([
      createUserMessage({
        content: 'Continue from where you left off.',
        isMeta: true,
      }),
    ])
    filteredMessages.push(continuationMessage)
    // turnInterruptionState becomes { kind: 'interrupted_prompt', ... }
  }
  // Append synthetic NO_RESPONSE_REQUESTED assistant sentinel after last user
  // so the conversation is API-valid if no resume action is taken.
}
```

### 08 loadConversationForResume — The Central Resume Function

`loadConversationForResume` in `conversationRecovery.ts` is the single entry point for all resume paths (CLI `--continue`, CLI `--resume <id>`, `--resume <path.jsonl>`, and the interactive `/resume` slash command). It handles four source shapes:

```
loadConversationForResume(source, sourceJsonlFile)

source === undefined
  -> most recent session (skip live background sessions)

source is string
  -> load by session UUID

source is LogOption
  -> already loaded log object

sourceJsonlFile set
  -> arbitrary .jsonl path (cross-directory resume)

In all cases:
1. resolveLog() / loadMessagesFromJsonlPath()
2. isLiteLog(log) ? loadFullLog(log) : use as-is
3. copyPlanForResume() <- copy plan files to resumed session
4. copyFileHistoryForResume()
5. checkResumeConsistency()
6. restoreSkillStateFromMessages() <- re-add invoked_skills to bootstrap state
7. deserializeMessagesWithInterruptDetection()
8. processSessionStartHooks('resume', { sessionId })
9. return { messages, turnInterruptionState, fileHistorySnapshots,
            attributionSnapshots, contentReplacements, contextCollapseCommits,
            ...session metadata... }
```

#### Lite vs Full Log

For performance, the session list uses "lite" reads: a fast head+tail read of each JSONL to extract metadata without parsing the whole file. If a lite log is selected for resume, `loadFullLog` re-reads the complete file and builds the full message map.

#### Skill State Restoration

After compaction, invoked skills are re-injected into bootstrap state from `invoked_skills` attachment entries in the transcript — ensuring that a second compaction cycle after resume still finds the skills list.

```typescript
// utils/conversationRecovery.ts
export function restoreSkillStateFromMessages(messages: Message[]): void {
  for (const message of messages) {
    if (message.type !== 'attachment') continue
    if (message.attachment.type === 'invoked_skills') {
      for (const skill of message.attachment.skills) {
        if (skill.name && skill.path && skill.content) {
          addInvokedSkill(skill.name, skill.path, skill.content, null)
        }
      }
    }
    if (message.attachment.type === 'skill_listing') {
      suppressNextSkillListing()  // avoid re-announcing ~600 tokens of skills
    }
  }
}
```

### 09 processResumedConversation — Full Restore Orchestrator

`processResumedConversation` in `sessionRestore.ts` is called after `loadConversationForResume` returns. It handles all the stateful side effects of switching into the resumed session:

```
processResumedConversation(result, opts, context)

1. Match coordinator/normal mode
   modeApi.matchSessionMode(result.mode) — emits warning if mismatch
   forkSession ? skip : switchSession(resumedId, transcriptDir)

2. Re-stamp session ID
   switchSession() — updates bootstrap STATE.sessionId atomically
   renameRecordingForSession() — renames asciicast file to match
   resetSessionFilePointer() — nulls Project.sessionFile (old path)
   restoreCostStateForSession(sid) — restore API cost counters

3. Restore session metadata cache
   restoreSessionMetadata(result) — repopulates Project's cache fields
                                   (customTitle, agentName, agentColor, mode,
                                    worktreeSession, prNumber...)

4. Restore worktree working directory
   restoreWorktreeForResume(result.worktreeSession)
   -> process.chdir(worktreePath) or override cache if dir gone

5. Adopt the resumed transcript file
   adoptResumedSessionFile() — points Project.sessionFile at existing JSONL
                              so reAppendSessionMetadata() can write to
                              the correct file on exit

6. Restore context-collapse commit log (ant-only, CONTEXT_COLLAPSE feature flag)

7. Restore agent setting
   restoreAgentFromSession() — re-applies agentType + model override

8. Persist current mode
   saveMode(coordinator/normal) for future resumes

9. Compute initial state before render
   computeRestoredAttributionState()
   computeStandaloneAgentContext()
   refreshAgentDefinitionsForModeSwitch()
   updateSessionName()

10. Return ProcessedResume { messages, fileHistorySnapshots,
                            contentReplacements, agentName, agentColor,
                            restoredAgentDef, initialState }
```

#### Worktree Resume

When a session was in a git worktree when it exited, Claude Code must `chdir` back into that directory. The implementation uses `process.chdir` as a TOCTOU-safe existence check — it throws `ENOENT` if the worktree was deleted between sessions, and the code gracefully falls back to recording "exited" in the cache.

```typescript
// utils/sessionRestore.ts
export function restoreWorktreeForResume(
  worktreeSession: PersistedWorktreeSession | null | undefined,
): void {
  const fresh = getCurrentWorktreeSession()
  if (fresh) { saveWorktreeState(fresh); return }
  if (!worktreeSession) return

  try {
    process.chdir(worktreeSession.worktreePath)
  } catch {
    // Directory is gone. Record "exited" so next reAppend doesn't persist stale path.
    saveWorktreeState(null)
    return
  }
  setCwd(worktreeSession.worktreePath)
  setOriginalCwd(getCwd())
  restoreWorktreeSession(worktreeSession)
  // Invalidate caches that reference old cwd
  clearMemoryFileCaches()
  clearSystemPromptSections()
  getPlansDirectory.cache.clear?.()
}
```

### 10 restoreSessionStateFromLog — AppState Hydration

In addition to the file/identity-level restore, the in-process `AppState` (React state) must be seeded from the transcript. This is handled by `restoreSessionStateFromLog` in `sessionRestore.ts` and is used by both interactive (REPL) and SDK resume paths.

```typescript
// utils/sessionRestore.ts
export function restoreSessionStateFromLog(
  result: ResumeResult,
  setAppState: (f: (prev: AppState) => AppState) => void,
): void {
  // Restore file history (per-turn file diff snapshots)
  if (result.fileHistorySnapshots?.length > 0) {
    fileHistoryRestoreStateFromLog(result.fileHistorySnapshots, newState => {
      setAppState(prev => ({ ...prev, fileHistory: newState }))
    })
  }
  // Restore attribution state (ant-only COMMIT_ATTRIBUTION feature)
  if (feature('COMMIT_ATTRIBUTION') && result.attributionSnapshots?.length > 0) {
    attributionRestoreStateFromLog(result.attributionSnapshots, newState => {
      setAppState(prev => ({ ...prev, attribution: newState }))
    })
  }
  // Restore context-collapse commits (resets store even if empty list)
  if (feature('CONTEXT_COLLAPSE')) {
    contextCollapsePersist.restoreFromEntries(
      result.contextCollapseCommits ?? [],
      result.contextCollapseSnapshot,
    )
  }
  // Restore todo list from transcript (SDK/non-interactive only)
  if (!isTodoV2Enabled() && result.messages?.length > 0) {
    const todos = extractTodosFromTranscript(result.messages)
    if (todos.length > 0) {
      setAppState(prev => ({
        ...prev,
        todos: { ...prev.todos, [getSessionId()]: todos },
      }))
    }
  }
}
```

### 11 Cloud Persistence — Dual Remote Paths

For Claude.ai sessions (CCR), transcripts are mirrored to the cloud. There are two distinct paths:

```
Remote Persistence Architecture

┌─────────────────────────────────────────────────────────┐
│ Local JSONL write                                       │
│ (always happens on-device)                              │
└──────────────────────┬──────────────────────────────────┘
                       │
                       ▼
            persistToRemote(sessionId, entry)
                       │
        ┌──────────────┴───────────────┐
        │                              │
        ▼                              ▼
  CCR v2 path              v1 Session Ingress
  (internalEventWriter)    (ENABLE_SESSION_PERSISTENCE=1)
        │                              │
        ▼                              ▼
  internalEventWriter(         sessionIngress.appendSessionLog()
  'transcript', entry,         -> REST POST to remoteIngressUrl
  { isCompaction, agentId }    -> gracefulShutdownSync(1) on failure
  )

  FLUSH_INTERVAL = 10ms        FLUSH_INTERVAL = 10ms
  (vs 100ms local)             (vs 100ms local)

On session reconnect (CCR v2 hydration):
hydrateFromCCRv2InternalEvents(sessionId)
  -> fetch foreground events via internalEventReader()
  -> fetch subagent events via internalSubagentEventReader()
  -> group subagent events by agent_id
  -> writeFile(sessionFile, content)        <- overwrites local JSONL
  -> writeFile(agentFile, agentContent)    per agent
```

```typescript
// utils/sessionStorage.ts
export async function hydrateFromCCRv2InternalEvents(
  sessionId: string,
): Promise<boolean> {
  const events = await reader()
  // Write foreground transcript
  const fgContent = events.map(e => jsonStringify(e.payload) + '\n').join('')
  await writeFile(sessionFile, fgContent, { encoding: 'utf8', mode: 0o600 })

  // Group subagent events by agent_id, write per-agent files
  const byAgent = new Map<string, Record<string, unknown>[]>()
  for (const e of subagentEvents) {
    const agentId = e.agent_id || ''
    if (!agentId) continue
    byAgent.set(agentId, [...(byAgent.get(agentId) ?? []), e.payload])
  }
  for (const [agentId, entries] of byAgent) {
    const agentFile = getAgentTranscriptPath(asAgentId(agentId))
    await writeFile(agentFile, entries.map(p => jsonStringify(p) + '\n').join(''))
  }
}
```

### 12 Remote Session History API

For the CCR web and mobile UI, `assistant/sessionHistory.ts` provides a paginated event-fetch client. This is used to render full conversation history by scrolling backward through events — oldest to newest. The API cursor is `before_id`, anchored to the latest event by default.

```typescript
// assistant/sessionHistory.ts
export const HISTORY_PAGE_SIZE = 100

export type HistoryPage = {
  events: SDKMessage[]    // chronological within page
  firstId: string | null  // cursor for next-older page
  hasMore: boolean        // true = older events exist
}

// Newest page: anchor_to_latest=true
export async function fetchLatestEvents(
  ctx: HistoryAuthCtx, limit = HISTORY_PAGE_SIZE,
): Promise<HistoryPage | null>

// Older page: before_id cursor
export async function fetchOlderEvents(
  ctx: HistoryAuthCtx, beforeId: string, limit = HISTORY_PAGE_SIZE,
): Promise<HistoryPage | null>
```

Auth is prepared once via `createHistoryAuthCtx` — it fetches the OAuth access token + org UUID and caches them in a `HistoryAuthCtx` object reused across all page fetches, with a 15-second timeout per request and the `ccr-byoc-2025-07-29` beta header.

### 13 Compaction and the Preserved Segment Relink

When context compaction runs (Lesson 15), it rewrites the conversation chain. The JSONL is append-only, so compacted messages are never deleted from disk — instead, a `system compact_boundary` message with `parentUuid=null` truncates the logical chain. Loading walks from the newest leaf, hits the boundary's null parent, and stops — producing only the post-compaction messages.

An advanced case is "preserved segments" — a suffix of pre-compaction messages the compactor wants to keep verbatim (useful for preserving recent context). On resume, `applyPreservedSegmentRelinks` patches the `parentUuid` pointers to splice this segment back into the chain correctly.

```
Preserved segment relink on resume

Before compaction (in JSONL, physical order):
[A] -> [B] -> [C] -> [D] -> [E(user)]

After compaction (logical, on disk):
[boundary(parentUuid=null)] -> [summary] -> [D'] -> [E(user)]
                                         <- preserved segment: [D', E]
                                           with anchored at [summary]

applyPreservedSegmentRelinks():
1. Find last compact_boundary with preservedSegment metadata
2. Walk tail->head to validate the segment is intact
3. Patch: head.parentUuid = anchorUuid (summary)
4. Patch: anchor's other children -> tail (splice in)
5. Zero stale usage tokens on preserved assistant messages
   (on-disk input_tokens reflect pre-compact context ~190K -> would trigger
    immediate autocompact loop on resume)
6. Prune everything before absolute-last-boundary that isn't preserved
```

### 14 Session Listing — Lite Reads

The resume picker (`--continue` / `/resume` UI) needs to show a list of sessions with title, last prompt, date, and size — without parsing full JSONL files that can be multiple GB. This is solved by `readSessionLite` in `sessionStoragePortable.ts`, which reads only the head (first few KB) and tail (last 64 KB) of each file:

```
Session list performance architecture (listSessionsImpl.ts)

For each .jsonl file in project dir:

┌──────────────────────────────────────────────────────┐
│ readHeadAndTail(path, LITE_READ_BUF_SIZE=65536)      │
│                                                      │
│ HEAD (first 4KB):                                    │
│ extractFirstPromptFromHead() <- SKIP_FIRST_PROMPT     │
│ extract gitBranch, cwd from first entry              │
│                                                      │
│ TAIL (last 64KB):                                    │
│ extractLastJsonStringField('customTitle')            │
│ extractLastJsonStringField('tag')                    │
│ extractLastJsonStringField('lastPrompt')             │
│                                                      │
│ stat() -> lastModified, fileSize                      │
└──────────────────────────────────────────────────────┘

SessionInfo returned: {
  sessionId, summary, lastModified, fileSize,
  customTitle, firstPrompt, gitBranch, cwd, tag, createdAt
}

Sessions sorted by lastModified desc, optionally filtered by dir.
Worktree paths included by default (includeWorktrees: true).
```

**Why this matters**

The tail-window strategy is why `reAppendSessionMetadata` moves title/tag/last-prompt to EOF on exit and compaction — it guarantees these fields fall within the 64 KB window that the listing reads. Without that, long sessions push metadata out of the window and the resume picker shows "No prompt" even when the session has a custom title.

### 15 Deduplication and UUID Collision Guard

Because `recordTranscript` can be called multiple times with the same messages (e.g., after compaction's `messagesToKeep`), the write path uses a `messageSet: Set<UUID>` per session to skip already-recorded entries.

```typescript
// utils/sessionStorage.ts — recordTranscript()
const messageSet = await getSessionMessages(sessionId)  // Set<UUID>
const newMessages: Message[] = []
let seenNewMessage = false

for (const m of cleanedMessages) {
  if (messageSet.has(m.uuid)) {
    // Only track prefix-skipped messages as parent
    if (!seenNewMessage && isChainParticipant(m)) {
      startingParentUuid = m.uuid
    }
  } else {
    newMessages.push(m)
    seenNewMessage = true
  }
}
// Only write truly new messages to JSONL
```

#### Sidechain Exception

Agent sidechain entries are **not** deduplicated against the main session's `messageSet`. They go to a separate file (`agent-<agentId>.jsonl`). Mixing the UUID sets would cause main-thread messages to chain from agent-file UUIDs that `buildConversationChain` can't find in the main JSONL — producing dangling parent references and orphaned transcript branches.

### 16 Complete Data Flow Diagram

```
Full Session Lifecycle

STARTUP                     RUNNING                      SHUTDOWN
│                           │                            │
├─ bootstrap/state.ts       ├─ query() called            ├─ cleanup handlers run
│ setSessionId(newUUID)     │                            │
│                           ├─ sessionState -> 'running'  ├─ Project.flush()
├─ sessionState -> 'idle'    │                            │
│                           ├─ recordTranscript(msgs)    ├─ reAppendSessionMetadata()
├─ --continue/--resume?     │ enqueueWrite(jsonl)        │ (title, tag, lastPrompt
│ loadConversationForResume()│ persistToRemote()          │ stay in tail window)
│ processResumedConversation│                            │
│ restoreSessionStateFromLog│ ├─ tool requires approval   │
│                           │ sessionState -> 'requires_action'
├─ hydrateFromCCRv2?        │ metadataListener({pending_action})
│ fetch remote events       │                            │
│ writeFile(local JSONL)    ├─ tool approved/denied      │
│                           │ sessionState -> 'running'   │
├─ materializeSessionFile() │                            │
│ (deferred until 1st msg)  ├─ model responds            │
│                           │ sessionState -> 'idle'      │
├─ sessionState -> 'idle'    │                            │
│                           │ metadataListener({task_summary: null})
```

### 17 Persistence Guards and Skip Conditions

Several conditions disable transcript persistence entirely. The `shouldSkipPersistence()` method centralizes these checks:

```typescript
// utils/sessionStorage.ts — Project class
private shouldSkipPersistence(): boolean {
  const allowTestPersistence = isEnvTruthy(
    process.env.TEST_ENABLE_SESSION_PERSISTENCE,
  )
  return (
    (getNodeEnv() === 'test' && !allowTestPersistence) ||
    getSettings_DEPRECATED()?.cleanupPeriodDays === 0 ||
    isSessionPersistenceDisabled() ||   // --no-session-persistence flag
    isEnvTruthy(process.env.CLAUDE_CODE_SKIP_PROMPT_HISTORY)
    // ^ set by tmuxSocket.ts for Tungsten test sessions
  )
}
```

#### Key Takeaways

*   Sessions are append-only JSONL files. Messages form a linked list via `parentUuid`. Reading walks from the newest leaf to root, then reverses.
*   The `Project` singleton batches all writes into per-file queues (100 ms local, 10 ms cloud) and deduplicates by UUID before writing.
*   The session file is lazily created on the first real user/assistant message — preventing metadata-only sessions from cluttering the resume list.
*   Metadata entries (title, tag, last-prompt) are re-appended to EOF on exit and compaction to stay within the 64 KB tail-window used by the fast listing read.
*   Interrupt detection classifies each resumed conversation into `none`, `interrupted_prompt`, or `interrupted_turn` — the latter two produce a synthetic continuation message so the SDK can auto-resume without user input.
*   CCR cloud sessions hydrate local JSONL from remote events on reconnect, overwriting the local file with the server-authoritative transcript.
*   Three state-machine values (`idle`, `running`, `requires_action`) drive the CCR sidebar UI, push notifications, and the SDK event stream — all wired through a single `notifySessionStateChanged` choke point.
*   Worktree resume uses `process.chdir` as a TOCTOU-safe existence check; if the directory is gone, the cache is immediately overridden to prevent stale paths from being re-persisted on exit.

### Deep Dive: Tombstone Removal of Orphaned Messages

When a streaming response fails mid-flight, an orphaned assistant message may land in the JSONL. Claude Code can remove it via `removeMessageByUuid` — a surgical in-place operation that avoids rewriting the full file:

1.  Read the last `LITE_READ_BUF_SIZE` (64 KB) bytes of the file.
2.  Search for `"uuid":"<targetUuid>"` in the buffer (not bare UUID, to avoid matching `parentUuid` of a child).
3.  Locate the containing line's boundaries via byte-scan (safe since `0x0a` never appears inside a UTF-8 multi-byte sequence).
4.  Truncate the file at the line start, then re-append any bytes that followed the line.
5.  If the UUID is not in the tail window (rare — requires many large entries written after the orphan), fall back to a full file rewrite — but bail if the file exceeds 50 MB to prevent OOM.

### Deep Dive: Fork Session Behavior

When `--fork-session` is used, Claude Code resumes the conversation content but assigns a **fresh session UUID**. This creates an independent copy that doesn't overwrite the original transcript. The tricky part: content-replacement records from the original session must be seeded into the new session's JSONL at startup, or the fork's first `loadConversationForResume` will fail to find replacement records and misclassify tool outputs as FROZEN (uncached), causing token overages.

```typescript
// utils/sessionRestore.ts
} else if (result.contentReplacements?.length) {
  // Fork keeps fresh startup sessionId. Seed content-replacement records
  // so loadTranscriptFile's keyed lookup matches the new session's ID.
  await recordContentReplacement(result.contentReplacements)
}
```

### Deep Dive: Agent Setting Restore

If a session was run with a custom agent type (e.g., a coding-specialist agent from the `~/.claude/agents/` directory), `restoreAgentFromSession` re-applies the agent type and model override on resume — unless the user explicitly specified a different `--agent` on the CLI, which takes precedence. If the agent no longer exists (deleted between sessions), a debug warning is logged and the default behavior is used.

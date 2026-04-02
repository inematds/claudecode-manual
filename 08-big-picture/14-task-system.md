# The Task System — Source Deep Dive

## Lesson 14

**Duration:** ~45 min | **Files:** Task.ts, tasks/, utils/task/, tools/TaskCreateTool/, tools/TaskUpdateTool/ | **Level:** Advanced

---

## What is a Task?

When Claude Code runs a bash command, spawns a subagent, launches a remote cloud session, or starts the "dream" memory-consolidation agent — each is tracked as a **task**. Tasks represent concurrent, observable work units running alongside the main conversation.

The core abstraction lives in `Task.ts`. Every task shares a common base state (`TaskStateBase`), a five-state lifecycle, and a single kill interface:

```typescript
// Task.ts — the minimal interface every task type implements
export type Task = {
  name: string
  type: TaskType
  kill(taskId: string, setAppState: SetAppState): Promise<void>
}

// Seven concrete types live in the codebase today
export type TaskType =
  | 'local_bash'        // LocalShellTask
  | 'local_agent'       // LocalAgentTask (+ LocalMainSessionTask)
  | 'remote_agent'      // RemoteAgentTask
  | 'in_process_teammate' // InProcessTeammateTask
  | 'local_workflow'    // LocalWorkflowTask
  | 'monitor_mcp'       // MonitorMcpTask
  | 'dream'             // DreamTask

// Shared fields every task state carries
export type TaskStateBase = {
  id: string          // e.g., "b3f7a2c1..." (prefix encodes type)
  type: TaskType
  status: TaskStatus   // pending | running | completed | failed | killed
  description: string
  toolUseId?: string   // links back to the tool_use that spawned this task
  startTime: number
  endTime?: number
  outputFile: string   // absolute path to the task output file on disk
  outputOffset: number // byte offset: how much output has been consumed
  notified: boolean    // prevents double-notification (atomically set)
}
```

### Task IDs Encode Their Type

Each task type has a one-letter prefix: `b`=bash, `a`=agent, `r`=remote, `t`=teammate, `w`=workflow, `m`=monitor, `d`=dream, `s`=main-session. The remaining 8 characters are cryptographically random (36^8 ~ 2.8 trillion combinations). This prevents symlink-based path-traversal attacks in sandboxed environments.

---

## The Lifecycle

Every task moves through exactly five statuses. Only three are terminal (no further transitions):

- **pending**
- **running**
- **completed** (terminal)
- **failed** (terminal)
- **killed** (terminal)

### State Diagram

```
[*] → pending : registerTask()
pending → running : task starts
running → completed : exit code 0 / agent success
running → failed : exit code != 0 / agent error
running → killed : kill() called
completed → [*] : evicted after notified=true
failed → [*] : evicted after notified=true
killed → [*] : evicted after notified=true
```

### The notified Flag: Preventing Double-Delivery

The `notified` field is a one-way latch. When a task completes (reaches any terminal status), the system atomically sets `notified: true` and enqueues a single XML notification to the model. Because this is guarded by a compare-and-set, even if two async paths race to completion, only one notification fires:

```typescript
// Pattern used by every task type (e.g. LocalAgentTask.ts)
let shouldEnqueue = false
updateTaskState<LocalAgentTaskState>(taskId, setAppState, task => {
  if (task.notified) {
    return task           // already notified — no-op
  }
  shouldEnqueue = true
  return { ...task, notified: true }
})
if (!shouldEnqueue) return   // another path already sent the notification
// ... enqueue XML notification ...
```

### Eviction: Freeing Memory After Notification

Once a task is both terminal and notified, it is eligible for eviction from `AppState.tasks`. Two mechanisms handle this: a lazy GC pass in `generateTaskAttachments()` (runs on each poll tick) and an eager path called `evictTerminalTask()` invoked by the task's own completion handler. The `local_agent` type adds a third: a configurable grace period (`evictAfter`) for tasks retained in the Coordinator Task Panel so the UI can display them briefly after they finish.

---

## The Seven Task Types

### LocalShellTask (local_bash)

A foreground or background bash command. Stdout/stderr go directly to a file via OS file descriptors — no JS buffering at all. Progress is extracted by polling the file tail every 1s.

### LocalAgentTask (local_agent)

A background Claude agent running in the same Node.js process. Uses `runAgent()`. Tracks token count, tool-use count, and recent activity for the panel UI. Also covers `LocalMainSessionTask` (main session backgrounded via Ctrl+B).

### RemoteAgentTask (remote_agent)

A session running in a cloud environment (teleport). Polled via the Teleport API. Subtypes include `ultraplan`, `ultrareview`, `autofix-pr`, and `background-pr`. Has a `ultraplanPhase` field for plan approval flow.

### InProcessTeammateTask (in_process_teammate)

A teammate agent running in-process with AsyncLocalStorage isolation. Has an identity (agentName@teamName), supports plan-mode approval, mailbox messaging, and idle/active lifecycle distinct from running/pending.

### LocalWorkflowTask (local_workflow)

A multi-step background workflow. Emits `task_progress` SDK events with structured `SdkWorkflowProgress[]` so external consumers can track per-step completion.

### MonitorMcpTask (monitor_mcp)

An MCP-sourced monitor process. Shown as a distinct pill in the footer ("N monitors") rather than folded into the shell count. Enabled by the `MONITOR_TOOL` feature flag.

### DreamTask (dream)

The auto-dream memory-consolidation subagent. UI-only — no model-facing notification path. Has a four-stage structure (orient/gather/consolidate/prune) expressed as two phases: `starting` and `updating`. On kill, rewinds the consolidation lock so the next session can retry.

### LocalMainSessionTask is Not a Separate TaskType

It reuses `local_agent` state with `agentType: 'main-session'`. Identification is by the `isMainSessionTask()` predicate. This lets the main session share the entire agent infrastructure without duplicating code.

---

## Deep Dives

### LocalShellTask: Foreground vs Background, Stall Watchdog

#### Two Spawn Paths: Background and Foreground

Most bash tasks are spawned directly as background tasks via `spawnShellTask()` with `isBackgrounded: true`. But when a command runs long enough to show the "Background Hint" in the TUI, it transitions via `registerForeground()` (creating the task with `isBackgrounded: false`) and later `backgroundTask()` (flipping the flag and installing the completion handler). This two-step is why you can press Ctrl+B to background a running command mid-flight.

```typescript
// isBackgrounded = false → task exists in AppState but is "owned" by
// the foreground UI. isBackgrounded = true → task is truly background.
export function isBackgroundTask(task: TaskState): task is BackgroundTaskState {
  if (task.status !== 'running' && task.status !== 'pending') return false
  // Foreground tasks (isBackgrounded === false) excluded from pill
  if ('isBackgrounded' in task && task.isBackgrounded === false) return false
  return true
}
```

#### The Stall Watchdog

Because bash stdout goes directly to a file (no JS stream), Claude Code can't listen for "no data received for 45s." Instead, `startStallWatchdog()` polls the output file's size every 5 seconds. If the size hasn't grown in 45 seconds, it reads the last 1KB and checks whether the tail matches any of 7 prompt-detection patterns (e.g., `(y/n)`, `Press Enter`, `Are you sure?`).

If a prompt is detected, a one-shot notification (no `<status>` tag — which would falsely close the task for SDK consumers) tells the model the command appears to be waiting for interactive input. If the tail does not look like a prompt, the watchdog resets its timer and waits another 45 seconds — so slow but progressing commands like `git log -S` never trigger false positives.

```typescript
const PROMPT_PATTERNS = [
  /\(y\/n\)/i,          // (Y/n), (y/N)
  /\[y\/n\]/i,          // [Y/n], [y/N]
  /\(yes\/no\)/i,
  /\b(?:Do you|Would you|Shall I|Are you sure)\b.*\? *$/i,
  /Press (any key|Enter)/i,
  /Continue\?/i,
  /Overwrite\?/i
]
```

#### Orphan Cleanup

Shell tasks track the `agentId` of the agent that spawned them. When an agent exits (its `runAgent()` finally block fires), `killShellTasksForAgent()` kills any still-running bash tasks with a matching `agentId`. This prevents "zombie" shell processes that outlive the agent that launched them — a real production problem that was observed as long-running background scripts continuing for days after the agent completed.

### LocalAgentTask: Progress Tracking, Retain/Evict, Background Summarization

#### Progress Tracking: Two Separate Counters, One TokenCount

The `ProgressTracker` inside `LocalAgentTask.tsx` tracks two things separately: **input tokens** (cumulative per-turn — always the latest API value) and **output tokens** (per-turn — summed across all turns). This is because the Claude API's `input_tokens` field is cumulative (includes all previous context) but `output_tokens` is per-turn. Summing input_tokens would double-count; averaging output_tokens would under-count. The `getTokenCountFromTracker()` function returns `latestInputTokens + cumulativeOutputTokens`.

```typescript
export function updateProgressFromMessage(
  tracker: ProgressTracker,
  message: Message,
): void {
  const usage = message.message.usage
  // input is cumulative: keep the latest, don't sum
  tracker.latestInputTokens = usage.input_tokens
    + (usage.cache_creation_input_tokens ?? 0)
    + (usage.cache_read_input_tokens ?? 0)
  // output is per-turn: sum across all turns
  tracker.cumulativeOutputTokens += usage.output_tokens
  // ... track tool uses and recent activities ...
}
```

#### The Retain/Evict Gate for the Panel

Agent tasks in the Coordinator Task Panel have an extra `retain` boolean and an `evictAfter` timestamp. When a user opens a task's transcript view, `retain` is set to `true` — this blocks eviction entirely (regardless of `evictAfter`). When the task completes and the user un-selects it, `evictAfter` is set to `Date.now() + PANEL_GRACE_MS` (30 seconds). The UI can show the completed task for 30 seconds, then it disappears. This prevents the panel from showing stale completed tasks forever while still giving the user time to see results.

#### Background Summarization

A periodic service calls `updateAgentSummary()` to store a 1-2 sentence progress summary on the task state. The key design decision: `updateAgentProgress()` deliberately preserves the existing `summary` field — so a background summarization result is never clobbered by a subsequent tool-use progress update. Progress and summary are written by different code paths and must not overwrite each other.

### RemoteAgentTask: Polling, Completion Checkers, Review Extraction

#### Five Remote Task Subtypes

The `remoteTaskType` field distinguishes: `remote-agent`, `ultraplan`, `ultrareview`, `autofix-pr`, and `background-pr`. Each subtype can register a `RemoteTaskCompletionChecker` — a function called on every poll tick that returns a non-null string to signal completion, or null to keep polling. This makes custom completion logic (e.g., checking if a PR was merged) pluggable without modifying the core poll loop.

```typescript
// Register a completion checker for a specific remote task type
export function registerCompletionChecker(
  remoteTaskType: RemoteTaskType,
  checker: RemoteTaskCompletionChecker,
): void {
  completionCheckers.set(remoteTaskType, checker)
}

// Called on every poll tick
type RemoteTaskCompletionChecker = (
  metadata: RemoteTaskMetadata | undefined
) => Promise<string | null>
// Return non-null string → task complete, string is the notification text
// Return null → keep polling
```

#### Metadata Persistence for --resume

Remote agent metadata (taskId, sessionId, remoteTaskType, remoteTaskMetadata) is written to a sidecar file on disk via `writeRemoteAgentMetadata()`. On session resume, this sidecar is read and live polling is re-established for any tasks that didn't finish. The metadata is deleted on task completion/kill so --resume doesn't resurrect finished tasks.

#### Review Content Extraction: Two Producers

The `ultrareview` path has two possible content sources: bughunter mode (run_hunt.sh is a SessionStart hook; its output lands as `{type:'system', subtype:'hook_progress'}`) and prompt mode (a real Claude assistant turn wrapping the review in `<remote-review>` tags). The extraction function scans both formats, with hook_progress taking priority. A concat-fallback handles large payloads that flush across two events due to pipe-buffer splits.

### InProcessTeammateTask: Identity, Idle Lifecycle, Message Cap

#### Identity: agentName@teamName

Every teammate has a structured identity stored on the task state — `agentId` (e.g., `"researcher@my-team"`), `agentName`, `teamName`, and an optional color for the TUI spinner display. The identity mirrors the `TeammateContext` stored in AsyncLocalStorage (which provides isolation between concurrent async chains) but is stored as plain data in AppState for UI access.

#### Idle vs Running: A Lifecycle Within Running

In-process teammates have an `isIdle` boolean that operates independently of the `status` field. A teammate with `status: 'running'` and `isIdle: true` is "running but waiting for work." This distinction matters for the UI spinner (`figures.ellipsis` vs `figures.play`) and for `describeTeammateActivity()` which returns `'idle'` rather than `'working'`. Idle callbacks (`onIdleCallbacks`) allow the leader agent to efficiently wait without polling.

#### The 50-Message UI Cap

Task state stores a `messages` array for the zoomed transcript view, but this is capped at 50 messages. Production analysis revealed sessions with 500+ turn agents holding a second full copy of every message, reaching ~20MB RSS per agent. A swarm session that launched 292 agents in 2 minutes hit 36.8GB of RSS. The `appendCappedMessage()` function drops oldest messages to stay within the cap:

```typescript
export const TEAMMATE_MESSAGES_UI_CAP = 50

export function appendCappedMessage<T>(
  prev: readonly T[] | undefined,
  item: T,
): T[] {
  if (prev && prev.length >= TEAMMATE_MESSAGES_UI_CAP) {
    // Slice oldest, append new — always returns a new array
    const next = prev.slice(-(TEAMMATE_MESSAGES_UI_CAP - 1))
    next.push(item)
    return next
  }
  return [...(prev ?? []), item]
}
```

### DreamTask: Memory Consolidation Phases, Kill-Then-Rewind Pattern

#### What Dream Does

The DreamTask wraps the auto-dream memory consolidation subagent — the background agent that periodically reviews past session transcripts and consolidates insights into long-term memory files. The task itself is pure UI surfacing: it makes the otherwise-invisible forked agent appear in the footer pill and the background task dialog.

#### Two Phases, No Phase Detection

DreamTask has a `phase` field with two values: `'starting'` and `'updating'`. The phase transitions from `starting` to `updating` the moment the first `Edit` or `Write` tool_use lands. The source comment explicitly notes: "we don't parse" the actual four-stage structure of the dream prompt (orient/gather/consolidate/prune). This is intentional — the UI only needs to communicate "hasn't started writing yet" vs "is now writing."

#### Kill-Then-Rewind

The consolidation lock prevents two dream processes from running simultaneously (a TOCTOU race that would corrupt memory files). On kill, `DreamTask.kill()` not only aborts the agent but also calls `rollbackConsolidationLock(priorMtime)` — rewinding the lock's mtime to what it was before this dream started. Without this, a killed dream would leave the lock in a "claimed" state, preventing the next session from ever dreaming. The same rollback path fires on fork failure.

```typescript
async kill(taskId, setAppState) {
  let priorMtime: number | undefined
  updateTaskState<DreamTaskState>(taskId, setAppState, task => {
    if (task.status !== 'running') return task
    task.abortController?.abort()
    priorMtime = task.priorMtime       // capture before state wipe
    return { ...task, status: 'killed', notified: true, ... }
  })
  if (priorMtime !== undefined) {
    await rollbackConsolidationLock(priorMtime)
  }
}
```

#### No Model-Facing Notification

DreamTask sets `notified: true` immediately in its complete/fail/kill handlers without enqueueing any XML notification. This is correct: dream is UI-only — it surfaces activity to the human in the TUI but the model doesn't need to know it happened. The inline `appendSystemMessage` completion note in the main session transcript is the only signal.

---

## The Notification System

When a task finishes, it must notify the model so it can process the result on its next turn. Notifications are XML messages enqueued into `messageQueueManager` with a specific format. Every task type builds the same XML envelope:

```xml
<task_notification>
<task_id>${taskId}</task_id>
<tool_use_id>${toolUseId}</tool_use_id>   <!-- links to spawning tool call -->
<output_file>${outputPath}</output_file>  <!-- model reads this file for full output -->
<status>${status}</status>              <!-- completed | failed | killed -->
<summary>${summary}</summary>          <!-- human-readable 1-liner -->
</task_notification>
```

### Priority: Next vs Later

Notifications are enqueued with a `priority` field: `'next'` delivers the notification at the start of the next turn (immediately visible), while `'later'` queues it after any pending normal messages. Shell tasks use `'later'` by default; monitor tasks (enabled by `MONITOR_TOOL` feature flag) use `'next'`. This prevents a flood of background shell completions from interrupting the conversation flow.

### The Stall Notification Has No <status> Tag — Deliberately

The stall watchdog notification (sent when a command appears to be waiting for input) intentionally omits the `<status>` tag. The reason is subtle: `print.ts` treats any `<status>` value as a terminal signal. Without a recognized status value, the notification is treated as a progress ping — the task stays open. With a `<status>` tag set to some placeholder value, `print.ts` would fall through to 'completed' and falsely close the task for SDK consumers.

### Agent Notifications Include Result and Usage

Unlike shell task notifications, agent task notifications carry richer payload: the final result text (via `<result>`), token usage statistics, duration, and optionally worktree path/branch if the agent ran in a git worktree:

```typescript
// LocalAgentTask enqueueAgentNotification — richer than shell
const resultSection   = finalMessage  ? `\n<result>${finalMessage}</result>` : ''
const usageSection    = usage ? `\n<usage><total_tokens>${total}</total_tokens>...` : ''
const worktreeSection = worktreePath  ? `\n<worktree><path>${worktreePath}</path>...` : ''
```

---

## Output Management

Every task has an output file on disk at `~/.claude/projects/<project>/tmp/<sessionId>/tasks/<taskId>.output`. The system uses a two-layer architecture:

| Layer | Class | Responsibility |
|-------|-------|-----------------|
| **DiskTaskOutput** | `utils/task/diskOutput.ts` | Write queue flushed to a single file handle per task. Chunks are GC'd immediately after each write. Enforces a 5GB cap, after which writes are dropped with a truncation notice. |
| **TaskOutput** | `utils/task/TaskOutput.ts` | Higher-level class used by hooks/pipe-mode. Buffers in memory (8MB default), spills to DiskTaskOutput on overflow. Also manages a shared progress poller for file-mode tasks. |

### File Mode vs Pipe Mode

Bash commands use **file mode**: stdout and stderr are redirected to the output file via OS-level file descriptors. No JS ever sees this data during execution. Progress is extracted by polling the file tail every 1 second using a shared static poller tied to React component mount/unmount lifecycle.

Hook executions and pipe-based outputs use **pipe mode**: data flows through `writeStdout()`/`writeStderr()` into an in-memory buffer. When the buffer exceeds 8MB, it spills to disk via `DiskTaskOutput`. In spilled state, `getStdout()` returns only the recent 5 lines with a truncation notice.

### Security: O_NOFOLLOW Prevents Symlink Attacks

All output file opens use the `O_NOFOLLOW` flag (where available — Unix only). Without this, an attacker in a sandboxed environment could create a symlink in the tasks directory pointing to an arbitrary file (e.g., `/etc/passwd`), causing Claude Code on the host to write task output there. The flag causes the open to fail with `ELOOP` if the path is a symlink, preventing the attack entirely.

### Agent Transcripts Use Symlinks

For `local_agent` tasks, the output file is initialized as a symlink to the agent's transcript JSONL file via `initTaskOutputAsSymlink()`. This means reading the task output file is equivalent to reading the agent's transcript — no data duplication. The delta-read system (`getTaskOutputDelta()`) uses byte offsets, so it can efficiently read only new content since the last poll.

### Truncation for API Consumption

Before task output is included in a notification to the model, it is formatted and truncated by `formatTaskOutput()`. The default limit is 32,000 characters (configurable via `TASK_MAX_OUTPUT_LENGTH`, capped at 160,000). Truncated output includes a header pointing to the full file path so the model can request the full output if needed:

```typescript
// outputFormatting.ts
export function formatTaskOutput(
  output: string,
  taskId: string,
): { content: string; wasTruncated: boolean } {
  const maxLen = getMaxTaskOutputLength()  // default 32_000
  if (output.length <= maxLen) return { content: output, wasTruncated: false }
  const filePath = getTaskOutputPath(taskId)
  const header = `[Truncated. Full output: ${filePath}]\n\n`
  const truncated = output.slice(-(maxLen - header.length))
  return { content: header + truncated, wasTruncated: true }
}
```

---

## TaskCreateTool and TaskUpdateTool

These two tools give the model the ability to manage a structured _task list_ — a separate concept from the background task system above. The task list is a lightweight project management system: subjects, descriptions, statuses, ownership, and dependency chains (blocks/blockedBy). It is enabled when `isTodoV2Enabled()` returns true.

### TaskCreateTool

Creates a task in the active task list. Beyond basic creation, it fires the `TaskCreated` hook pipeline (pluggable validation — if any hook returns a blocking error, the task is deleted and the error is thrown). On creation, it auto-expands the task list UI panel.

```typescript
// The input schema — minimal by design
z.strictObject({
  subject:     z.string(),            // "Fix authentication bug in login flow"
  description: z.string(),            // what needs to be done
  activeForm:  z.string().optional(), // "Fixing authentication bug" (spinner text)
  metadata:    z.record(...).optional(), // arbitrary key-value
})
```

### TaskUpdateTool: Status Workflow and Auto-Ownership

Updates any field of an existing task. Status workflow: `pending → in_progress → completed`, with `deleted` as a special action that removes the task file entirely. Key behaviors:

- **Auto-ownership**: When a teammate (agent swarms enabled) marks a task `in_progress` without specifying an owner, the tool automatically sets the owner to the calling agent's name. This keeps the task list synchronized with actual execution.
- **TaskCompleted hooks**: Marking a task `completed` fires the `TaskCompleted` hook pipeline. If any hook returns a blocking error (e.g., failing tests), the status update is rejected — the model cannot mark a task complete while a gate is failing.
- **Mailbox notification**: When ownership changes and agent swarms are enabled, a `task_assignment` mailbox message is written to the new owner's mailbox, enabling the teammate to discover assigned work without polling.
- **Verification nudge**: If a feature flag is enabled and the model just completed the last of 3+ tasks with no "verif*" task in the list, the tool result includes a nudge to spawn a verification agent.

```typescript
// The verification nudge — instructs the model to spawn a verifier
if (verificationNudgeNeeded) {
  resultContent += `\n\nNOTE: You just closed out 3+ tasks and none of them `
    + `was a verification step. Before writing your final summary, spawn `
    + `the verification agent (subagent_type="${VERIFICATION_AGENT_TYPE}"). `
    + `You cannot self-assign PARTIAL by listing caveats in your summary `
    + `— only the verifier issues a verdict.`
}
```

### Task List vs Background Tasks: Two Different Systems

**Two separate concepts share the word "task"**

The **background task system** (Task.ts, LocalShellTask, DreamTask, etc.) tracks concurrent execution units in AppState. The **task list** (TaskCreateTool, TaskUpdateTool, utils/tasks.ts) is the model's project-management checklist stored as YAML/JSON files on disk. They are completely separate — a background bash task is not a task-list task.

---

## UI: The Footer Pill

The footer pill (visible in the TUI as e.g., `3 shells / Shift+Down`) is generated by `getPillLabel()` in `tasks/pillLabel.ts`. It aggregates all _background_ tasks (those passing `isBackgroundTask()`) and renders a compact summary:

| Tasks in Background | Pill Label |
|---------------------|-----------|
| 1 bash task | `1 shell` |
| 3 bash + 2 monitor tasks | `3 shells, 2 monitors` |
| 1 ultraplan (needs_input) | `ultraplan needs your input` |
| 1 ultraplan (plan_ready) | `ultraplan ready` |
| 5 in-process teammates from 2 teams | `2 teams` |
| 1 DreamTask | `dreaming` |
| Mixed types | `N background tasks` |

The `pillNeedsCta()` function controls whether the `Shift+Down to view` call-to-action is appended — it fires only for single ultraplan tasks in an attention state (`needs_input` or `plan_ready`), not for plain running tasks.

In-process teammates are rendered in the spinner tree (one row per teammate) rather than the footer pill when the spinner tree is active. `shouldHideTasksFooter()` checks if every background task is an in-process teammate and hides the footer pill in that case to avoid duplication.

---

## Framework Internals: registerTask and updateTaskState

Two utility functions in `utils/task/framework.ts` handle all task state mutations. Understanding them is key to understanding how all seven task types stay consistent:

### registerTask

`registerTask(task, setAppState)` inserts a task into `AppState.tasks`. If a task with the same ID already exists (the "resume" path), it carries forward UI state — `retain`, `startTime`, `messages`, `diskLoaded`, `pendingMessages`. This allows a background agent to be resumed mid-conversation without losing the user's transcript view or scroll position. After registration, a `task_started` SDK event is emitted (skipped for replacements to avoid double-emit).

### updateTaskState

`updateTaskState<T>(taskId, setAppState, updater)` applies an updater function to a specific task. If the updater returns the same reference (early-exit no-op), the `setAppState` call is skipped — React subscribers do not re-render on unchanged state. This reference-equality check is a significant optimization for frequently-updated tasks like shell commands with active watchdogs.

```typescript
export function updateTaskState<T extends TaskState>(
  taskId: string,
  setAppState: SetAppState,
  updater: (task: T) => T,
): void {
  setAppState(prev => {
    const task = prev.tasks?.[taskId] as T | undefined
    if (!task) return prev
    const updated = updater(task)
    if (updated === task) return prev // reference-equal → no-op, no re-render
    return { ...prev, tasks: { ...prev.tasks, [taskId]: updated } }
  })
}
```

### The Poll Loop

`pollTasks()` runs on a 1-second interval. It calls `generateTaskAttachments()`, which reads output deltas for running tasks (byte-offset-based, never loads the whole file), then evicts terminal+notified tasks. The offset patches are applied to _fresh_ state (not the pre-await snapshot) to prevent clobbering concurrent transitions that may have happened during the async disk read.

---

## Type Comparison Reference

| Type | ID Prefix | Output File | Notification | Kill Mechanism |
|------|-----------|------------|--------------|-----------------|
| `local_bash` | `b` | Direct file via OS fd | XML with exit code, `'later'` priority | SIGKILL via shellCommand.kill() |
| `local_agent` | `a` | Symlink → transcript JSONL | XML with result + usage, no priority | abortController.abort() |
| `remote_agent` | `r` | Polled + written locally | XML via poll completion | archiveRemoteSession() API call |
| `in_process_teammate` | `t` | Symlink → transcript JSONL | None (team coordination via mailbox) | killInProcessTeammate() |
| `local_workflow` | `w` | Task output file | XML + SDK task_progress events | abortController.abort() |
| `monitor_mcp` | `m` | Task output file | XML, `'next'` priority | Process kill |
| `dream` | `d` | None | None (UI-only, notified=true immediately) | abortController.abort() + lock rewind |

---

## Key Takeaways

- **One interface, seven implementations.** All task types share `TaskStateBase` and the `Task.kill()` interface. Variation is in state shape, output strategy, and notification format — not in framework machinery.
- **The notified flag is an atomic one-way latch.** Every task type uses the same compare-and-set pattern inside `updateTaskState` to ensure exactly one notification reaches the model, regardless of how many async paths race to completion.
- **Bash output never touches JS during execution.** Stdout/stderr go directly to a file via OS file descriptors. Progress is extracted by polling the file tail. This eliminates back-pressure issues and JS heap pressure for long-running or output-heavy commands.
- **O_NOFOLLOW is a security, not performance, decision.** Opening task output files with this flag prevents an attacker in a sandboxed environment from creating symlinks that redirect Claude Code's writes to arbitrary host files.
- **DreamTask kills rewind the consolidation lock.** If the kill path skipped the lock rollback, a killed dream would permanently prevent the next session from running memory consolidation. The `priorMtime` field exists solely to enable this rollback.
- **The task list (TaskCreate/TaskUpdate) and background tasks are entirely separate systems.** One is a project-management checklist. The other is a runtime execution registry. They share the word "task" but nothing else.
- **The 50-message UI cap was a production necessity.** Real swarm sessions reached 36GB RSS because in-process teammate tasks held a second full copy of every API message. The cap is not a UI limitation — it's a memory safety valve.

---

## Quiz

### Question 1
You see task ID `t3a9bx2f` in the background task dialog. Without any other context, what task type is it?

- A. local bash command
- B. local background agent
- C. in-process teammate
- D. remote cloud session

**Answer: C** (The `t` prefix indicates `in_process_teammate`)

### Question 2
A bash command has been running for 60 seconds with no output growth. Its last line is `Continue? [y/N]`. What does Claude Code do?

- A. Automatically sends "y" and continues
- B. Kills the task immediately after the 45-second threshold
- C. Sends a stall notification (no <status> tag) telling the model the command needs input
- D. Sets the task status to 'failed' after the stall threshold

**Answer: C** (The stall watchdog detects prompt patterns and sends a special notification without a status tag to keep the task open)

### Question 3
Why does the `updateTaskState` helper check if the updater returned the same object reference before calling `setAppState`?

- A. To prevent infinite loops in the updater function
- B. To skip React re-renders when no state actually changed
- C. To ensure tasks are always evicted in order
- D. To batch multiple updates into a single AppState write

**Answer: B** (Reference-equality prevents unnecessary re-renders for unchanged state)

### Question 4
A DreamTask is killed mid-run. What happens to the consolidation lock, and why?

- A. The lock is left as-is — the next session will detect the stale lock and override it
- B. The lock file is deleted so the next session starts fresh
- C. The lock mtime is rewound to its pre-dream value so the next session can dream again
- D. The lock is held until the next session manually resets it via /dream reset

**Answer: C** (The `rollbackConsolidationLock(priorMtime)` rewinds the lock to prevent permanent blockage)

### Question 5
An in-process teammate has `status: 'running'` and `isIdle: true`. What is this teammate doing?

- A. Executing a long API call and will report back when done
- B. Waiting for new work — alive and registered but not currently processing any task
- C. Blocked waiting for another teammate to complete a dependency
- D. Shutting down — this state is transient before killed

**Answer: B** (The `isIdle` flag indicates the teammate is running but waiting for work, not actively processing)

### Question 6
Why does the LocalAgentTask notification system sum `output_tokens` across turns but always keep the _latest_ `input_tokens` instead of summing them?

- A. Input tokens are cheaper and don't need precise tracking
- B. The Claude API returns `input_tokens` as a cumulative value (includes all previous context), so summing would double-count; `output_tokens` is per-turn so summing is correct
- C. Output tokens are always larger and would overflow the counter if not summed
- D. The API only returns `input_tokens` on the first turn

**Answer: B** (API semantics: cumulative input vs per-turn output requires different accumulation strategies)

### Question 7
What is the primary purpose of the `O_NOFOLLOW` flag when opening task output files?

- A. Performance: following symlinks is slow for frequently-written files
- B. To ensure each task always gets a fresh file rather than appending to an existing one
- C. Security: prevents a sandboxed attacker from creating symlinks that redirect Claude Code's writes to arbitrary host files
- D. Compatibility: some filesystems don't support symlinks in temp directories

**Answer: C** (O_NOFOLLOW prevents symlink-based path-traversal attacks in sandboxed environments)

# mdENG — Lesson 44 — Error Handling and Recovery

## 01 Overview

Claude Code operates as a long-running networked process interacting with the Anthropic API, executing shell commands, reading/writing files, and managing sub-agents—all potential failure points. Rather than allowing arbitrary error propagation, the codebase implements a layered error architecture that classifies failures, determines retry strategy, and ensures meaningful user-facing messages.

**Source files covered:**
- `utils/errors.ts`
- `utils/toolErrors.ts`
- `utils/errorLogSink.ts`
- `services/api/withRetry.ts`
- `services/api/errors.ts`
- `ink/components/ErrorOverview.tsx`
- `utils/conversationRecovery.ts`
- `components/SentryErrorBoundary.ts`

**Four distinct error stack layers:**

| Layer | Component | Purpose |
|-------|-----------|---------|
| 1 | Typed Error Classes | Named exception vocabulary per failure domain |
| 2 | API Retry Engine | Exponential backoff, 529 fallback, auth refresh, context overflow |
| 3 | Terminal Error Overlay | Inline source excerpt with highlighted crash line |
| 4 | Conversation Recovery | Resume from interrupted/mid-turn sessions |

---

## 02 The Error Taxonomy (`utils/errors.ts`)

Each major failure mode receives its own named class. This approach enables `instanceof` checks that survive minification and class-name mangling in production builds.

| Class | Purpose | Key Fields |
|-------|---------|-----------|
| `ClaudeError` | Base class; sets name to subclass constructor | — |
| `AbortError` | User-initiated cancellation (Escape/Ctrl-C) | `name = 'AbortError'` |
| `MalformedCommandError` | Slash-command parse failures | — |
| `ConfigParseError` | Corrupt/unreadable config file | `filePath`, `defaultConfig` |
| `ShellError` | Shell command non-zero exit | `stdout`, `stderr`, `code`, `interrupted` |
| `TeleportOperationError` | Teleport SSH ops needing formatted message | `formattedMessage` |
| `TelemetrySafeError_I_VERIFIED_...` | Safe-for-telemetry errors | `telemetryMessage` |

### The `isAbortError` Three-Way Check

Abort signals originate from three sources, with names that mangle during production builds:

```typescript
export function isAbortError(e: unknown): boolean {
  return (
    e instanceof AbortError ||           // own class
    e instanceof APIUserAbortError ||    // SDK's class
    (e instanceof Error && e.name === 'AbortError')  // DOMException from AbortController
  )
}
```

The SDK's `APIUserAbortError` never sets `this.name`, and minified builds mangle constructor names to short strings like `'nJT'`. String matching fails silently in production.

### Utility Helpers — Catch-Site Boundaries

Rather than casting `unknown` to `Error` throughout, boundary normalization functions handle the conversion:

```typescript
// Normalize unknown → Error
export function toError(e: unknown): Error {
  return e instanceof Error ? e : new Error(String(e))
}

// Extract message string only
export function errorMessage(e: unknown): string {
  return e instanceof Error ? e.message : String(e)
}

// Trim stack to top-N frames for tool_result token efficiency
export function shortErrorStack(e: unknown, maxFrames = 5): string {
  if (!(e instanceof Error)) return String(e)
  if (!e.stack) return e.message
  const lines = e.stack.split('\n')
  const header = lines[0] ?? e.message
  const frames = lines.slice(1).filter(l => l.trim().startsWith('at '))
  if (frames.length <= maxFrames) return e.stack
  return [header, ...frames.slice(0, maxFrames)].join('\n')
}
```

**Context budget consideration:** `shortErrorStack` truncates full stacks (500–2000 characters of internal frames) to 5 frames, preserving the model's context window.

### Filesystem Error Helpers

Node.js filesystem errors carry `errno` codes typed as `any`. The codebase replaces unsafe casting with typed helpers:

```typescript
// Safe alternative to (e as NodeJS.ErrnoException).code
export function getErrnoCode(e: unknown): string | undefined

// Covers: ENOENT | EACCES | EPERM | ENOTDIR | ELOOP
export function isFsInaccessible(e: unknown): e is NodeJS.ErrnoException
```

`isFsInaccessible` covers five errno codes because multiple conditions trigger access failures—a file named `.claude` where a directory is expected produces `ENOTDIR`; circular symlinks generate `ELOOP`.

### The Telemetry Safety Discipline

`TelemetrySafeError_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` employs intentional verbosity as a code-review forcing function. The two-argument form separates user-facing and telemetry messages:

```typescript
throw new TelemetrySafeError_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS(
  `MCP tool timed out after ${ms}ms`,  // full message
  'MCP tool timed out'              // telemetry (no timing data)
)
```

---

## 03 Tool Error Formatting (`utils/toolErrors.ts`)

Tool failures require formatting for two consumers: the terminal (user) and the model (in `tool_result` blocks). The system enforces a hard 10,000-character cap with center-truncation to protect context budgets.

```typescript
export function formatError(error: unknown): string {
  if (error instanceof AbortError) {
    return error.message || INTERRUPT_MESSAGE_FOR_TOOL_USE
  }
  if (!(error instanceof Error)) return String(error)
  const parts = getErrorParts(error)
  const fullMessage = parts.filter(Boolean).join('\n').trim()
    || 'Command failed with no output'
  if (fullMessage.length <= 10000) return fullMessage

  // Center-truncate: keep head + tail of large outputs
  const halfLength = 5000
  return `${fullMessage.slice(0, halfLength)}\n\n...${fullMessage.length - 10000} characters truncated...\n\n${fullMessage.slice(-halfLength)}`
}
```

For `ShellError`, parts assemble in priority order: exit code first, then stderr, then stdout—mirroring developer expectations for diagnostic information.

### Zod Validation Errors → LLM-Friendly Messages

When models call tools with incorrect schemas, `ZodError` converts to structured English the model can understand and self-correct:

```
FileEditTool failed due to the following issues:
The required parameter `old_string` is missing
The parameter `new_string` type is expected as `string` but provided as `number`
```

**Design intent:** Generic Zod messages confuse LLMs. Formatting paths as `todos[0].activeForm` and specifying expected vs. received types enables model self-correction without human intervention.

---

## 04 The Error Log Sink (`utils/errorLogSink.ts`)

Error logging decouples from write implementation through a sink pattern. `log.ts` remains dependency-free, queuing events until a sink attaches. `errorLogSink.ts` implements heavy logic (file I/O, axios enrichment) initialized once at startup.

The sink writes JSONL (one JSON object per line) to date-stamped files, including timestamp, session ID, cwd, and version. For axios errors, it extracts request URL, HTTP status, and server error body—the three most useful API diagnostic fields.

```typescript
// Buffered JSONL writer — flushes every 1 second or after 50 entries
function createJsonlWriter(options: {
  writeFn: (content: string) => void
  flushIntervalMs?: number     // default: 1000
  maxBufferSize?: number        // default: 50
}): JsonlWriter
```

**Ant-only logging:** The `appendToLog` function guards on `process.env.USER_TYPE !== 'ant'`—error logs only write for internal Anthropic employees, preventing sensitive user data accumulation.

---

## 05 The API Retry Engine (`services/api/withRetry.ts`)

`withRetry` is an async generator wrapping every Anthropic API call. It implements sophisticated retry logic handling transient failures, rate limits, auth expiry, context overflow, and Claude-specific 529 overload status.

### Retry Delay Formula

Backoff uses exponential delay with ±25% jitter to prevent thundering-herd during rate limiting:

```typescript
export function getRetryDelay(
  attempt: number,
  retryAfterHeader?: string | null,
  maxDelayMs = 32000,
): number {
  if (retryAfterHeader) {
    const seconds = parseInt(retryAfterHeader, 10)
    if (!isNaN(seconds)) return seconds * 1000
  }
  const baseDelay = Math.min(
    500 * Math.pow(2, attempt - 1),   // 500ms, 1s, 2s, 4s... cap 32s
    maxDelayMs,
  )
  const jitter = Math.random() * 0.25 * baseDelay
  return baseDelay + jitter
}
```

### The 529 Overloaded Error — Special Handling

HTTP 529 (Claude-specific API overload status) receives dedicated logic:

- **Background query sources** (title generators, summarizers) bail immediately—retrying from concurrent clients amplifies capacity cascades
- **After 3 consecutive 529s on Opus**, the engine falls back to a configured fallback model via `FallbackTriggeredError`
- **SDK streaming bug workaround:** the SDK sometimes fails to set `status=529`; the fallback checks `error.message.includes('"type":"overloaded_error"')`

```typescript
export function is529Error(error: unknown): boolean {
  if (!(error instanceof APIError)) return false
  return (
    error.status === 529 ||
    (error.message?.includes('"type":"overloaded_error"') ?? false)
  )
}
```

### Context Overflow Auto-Adjustment

When requests are rejected with a 400 "input length and `max_tokens` exceed context limit" error, `withRetry` parses token counts and automatically reduces `maxTokens` for the next attempt:

```typescript
// Error message format:
// "input length and `max_tokens` exceed context limit: 188059 + 20000 > 200000"

const availableContext = Math.max(0, contextLimit - inputTokens - 1000)
retryContext.maxTokensOverride = Math.max(FLOOR_OUTPUT_TOKENS, availableContext, minRequired)
```

The floor is 3,000 tokens. If even that exceeds context limits, the error is re-thrown rather than attempting a call producing truncated output.

### Persistent Retry Mode (`CLAUDE_CODE_UNATTENDED_RETRY`)

For unattended/CI sessions, setting `CLAUDE_CODE_UNATTENDED_RETRY=1` enables indefinite retry on 429/529 with maximum 5-minute backoff and 6-hour total cap. Long waits chunk into 30-second heartbeat yields preventing idle marking.

---

## 06 User-Facing Error Message Constants (`services/api/errors.ts`)

All user-visible error strings are named constants in a single file, enabling searchability, testability, and preventing message drift:

```typescript
export const INVALID_API_KEY_ERROR_MESSAGE = 'Not logged in · Please run /login'
export const TOKEN_REVOKED_ERROR_MESSAGE    = 'OAuth token revoked · Please run /login'
export const REPEATED_529_ERROR_MESSAGE     = 'Repeated 529 Overloaded errors'
export const API_TIMEOUT_ERROR_MESSAGE      = 'Request timed out'
export const CREDIT_BALANCE_TOO_LOW_ERROR_MESSAGE = 'Credit balance is too low'
```

Interactive vs. non-interactive sessions receive different guidance for media errors:

```typescript
export function getImageTooLargeErrorMessage(): string {
  return getIsNonInteractiveSession()
    ? 'Image was too large. Try resizing the image or using a different approach.'
    : 'Image was too large. Double press esc to go back and try again with a smaller image.'
}
```

---

## 07 The Terminal Error Overlay (`ink/components/ErrorOverview.tsx`)

When unhandled exceptions reach the Ink render tree, `ErrorOverview` displays a formatted overlay with crash location and inline source context—not a raw stack dump. It uses `StackUtils` (V8 stack frame parsing) and `code-excerpt` (source display):

```typescript
// 1. Parse first stack frame for file + line + column
const stack = error.stack ? error.stack.split('\n').slice(1) : undefined
const origin = stack ? getStackUtils().parseLine(stack[0]!) : undefined

// 2. Read source file synchronously
const sourceCode = readFileSync(filePath, 'utf8')
excerpt = codeExcerpt(sourceCode, origin.line)

// 3. Render: crash line highlighted in red; surrounding lines dim
const isCrashLine = line_0 === origin.line
<Text
  backgroundColor={isCrashLine ? 'ansi:red' : undefined}
  color={isCrashLine ? 'ansi:white' : undefined}
  dim={!isCrashLine}
>
  {' ' + value}
</Text>
```

If the source file is unreadable, `readFileSync` wraps in a silent `try/catch`. The overlay degrades gracefully, still showing error message and full parsed stack.

**Sync file I/O rationale:** The component renders synchronously inside the Ink reconciler. Async would require suspense restructuring. Since this is the error path (process already broken), sync I/O is acceptable without REPL blocking.

### The `SentryErrorBoundary`

A React error boundary at `components/SentryErrorBoundary.ts` catches render errors from child components and renders `null` (silent failure):

```typescript
export class SentryErrorBoundary extends React.Component<Props, State> {
  static getDerivedStateFromError(): State {
    return { hasError: true }
  }

  render(): React.ReactNode {
    if (this.state.hasError) return null
    return this.props.children
  }
}
```

**Two recovery strategies:** `ErrorOverview` surfaces critical unhandled errors terminating the current render. `SentryErrorBoundary` wraps non-critical UI components that can silently drop without breaking the entire session.

---

## 08 Conversation Recovery (`utils/conversationRecovery.ts`)

When Claude Code crashes mid-turn or faces forceful termination, the transcript on disk may be in an inconsistent state. `conversationRecovery.ts` loads, cleans, and restores transcripts to a resumable state.

### The Four-Stage Deserialization Pipeline

`deserializeMessagesWithInterruptDetection` runs raw persisted messages through four filters:

**Stage 1: Legacy Migration**
Transform old attachment types (`new_file` → `file`, `new_directory` → `directory`) and backfill missing `displayPath` fields.

**Stage 2: Strip Bad Permission Modes**
Remove `permissionMode` values not in the current build's `PERMISSION_MODES` set, preventing stale config crashes.

**Stage 3: Filter Invalid Messages**
Remove unresolved tool_use pairs, orphaned thinking-only assistant messages, and whitespace-only assistant messages.

**Stage 4: Interrupt Detection**
Classify the transcript end as: none (completed) / interrupted_prompt (user sent message, AI never responded) / interrupted_turn (AI mid-tool-use).

### Interruption Classification

After filtering, the last "turn-relevant" message (skipping `system`, `progress`, and API error assistants) determines what happened:

```typescript
// Last message is an assistant → turn completed normally
if (lastMessage.type === 'assistant') return { kind: 'none' }

// Last message is a plain user prompt → CC hadn't started responding
if (lastMessage.type === 'user' && !isToolUseResultMessage(lastMessage))
  return { kind: 'interrupted_prompt', message: lastMessage }

// Last message is a tool_result → AI was mid-tool-use
if (isToolUseResultMessage(lastMessage)) {
  // Special case: brief mode ends on SendUserMessage tool_result legitimately
  if (isTerminalToolResult(lastMessage, messages, lastMessageIdx))
    return { kind: 'none' }
  return { kind: 'interrupted_turn' }
}
```

### Synthetic Continuation Messages

An `interrupted_turn` (process killed mid-tool) converts to `interrupted_prompt` by injecting a synthetic "Continue from where you left off." user message, unifying both interruption kinds:

```typescript
if (internalState.kind === 'interrupted_turn') {
  const [continuationMessage] = normalizeMessages([
    createUserMessage({ content: 'Continue from where you left off.', isMeta: true })
  ])
  filteredMessages.push(continuationMessage!)
  turnInterruptionState = { kind: 'interrupted_prompt', message: continuationMessage! }
}
```

### The API-Valid Sentinel

If the last relevant message after filtering is a user message, the Anthropic API would reject the conversation (which must end on an assistant turn for streaming). A synthetic `NO_RESPONSE_REQUESTED` assistant sentinel is spliced after the user message:

The sentinel inserts at `lastRelevantIdx + 1`, not the array end. This is intentional: `removeInterruptedMessage` calls `splice(idx, 2)` to remove the user message and sentinel as a pair. End insertion would break if trailing `system` or `progress` messages exist.

### Skill State Restoration

Before deserialization, `restoreSkillStateFromMessages` walks the transcript for `invoked_skills` attachments and re-registers those skills in process state:

```typescript
for (const message of messages) {
  if (message.attachment?.type === 'invoked_skills') {
    for (const skill of message.attachment.skills) {
      addInvokedSkill(skill.name, skill.path, skill.content, null)
    }
  }
  // Suppress re-sending skill listing if already in transcript
  if (message.attachment?.type === 'skill_listing') suppressNextSkillListing()
}
```

---

## 09 End-to-End Error Flow

**Runtime Errors Path:**
Tool execution fails → `formatError()` in toolErrors.ts → tool_result block to model + terminal display

**API Errors Path:**
Anthropic API returns error → `withRetry()` in withRetry.ts → (if retryable) Backoff + yield SystemAPIErrorMessage → (if exhausted) CannotRetryError wraps original → `logError()` in errorLogSink.ts → JSONL log file in ~/.cache/claude/errors/ + ErrorOverview.tsx terminal overlay

**Session Recovery Path:**
Process killed mid-turn → --continue / --resume → `deserializeMessages()` in conversationRecovery.ts → 4-stage filter pipeline → `detectTurnInterruption()` → (if interrupted_prompt) Auto-resumes with user message / (if interrupted_turn) Inject synthetic 'Continue...' message then auto-resume / (if none) Clean resume without action

---

## Key Takeaways

- Every failure domain has a dedicated typed error class (`ShellError`, `ConfigParseError`, `AbortError`, etc.) enabling `instanceof` checks surviving minification.

- `isAbortError` checks three different abort shapes because minification mangles `constructor.name` and the SDK never sets `this.name`.

- `withRetry` handles 10+ distinct failure modes: 529 model fallback, context overflow auto-adjustment, OAuth token refresh, Bedrock/Vertex auth cache clearing, and stale connection keep-alive disable.

- Background query sources bail immediately on 529 without retry—amplifying capacity events from dozens of concurrent clients would worsen the outage.

- Tool errors truncate to 10,000 characters (5k head + 5k tail) before sending to the model, preserving context window from large compiler outputs.

- The terminal `ErrorOverview` reads crash source files synchronously and highlights the exact line—acceptable on the error path where the REPL is already broken.

- Conversation recovery runs a 4-stage filter pipeline cleaning broken transcripts, classifies interruption type, and injects synthetic messages before resuming.

- `TelemetrySafeError_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` uses deliberate verbosity as a code-review forcing function—developers must consciously confirm no sensitive data exists in messages.

---

## Knowledge Check

**Q1. Why does `isAbortError` use `instanceof APIUserAbortError` instead of checking `e.name === 'APIUserAbortError'`?**

A. The SDK's class doesn't have a `name` property at all, and minified builds mangle constructor names
B. `instanceof` is faster than string comparison at runtime
C. The SDK deliberately uses a different name value
D. TypeScript requires `instanceof` for narrowing in catch blocks

**Answer: A** — The SDK's `APIUserAbortError` never sets `this.name`, and minified builds mangle constructor names to short strings. The comment in isAbortError explicitly calls this out.

---

**Q2. What does `withRetry` do when it receives a context overflow 400 error?**

A. Throws `CannotRetryError` immediately
B. Parses the token counts from the error message and sets `maxTokensOverride`, then retries
C. Triggers compaction to reduce context size
D. Falls back to a smaller model automatically

**Answer: B** — `parseMaxTokensContextOverflowError` parses "188059 + 20000 > 200000" from the message, computes availableContext, sets retryContext.maxTokensOverride, and continues the loop.

---

**Q3. Why do background query sources like title generators bail immediately on 529 without retrying?**

A. They use a different API endpoint that doesn't support retry
B. Retrying from many concurrent clients during a capacity event amplifies the cascade, and users never see these fail anyway
C. Background queries are not important enough to warrant retry
D. The retry loop doesn't support async generators for background tasks

**Answer: B** — Each retry from each client is 3–10× gateway amplification during capacity cascades. Background tasks are invisible to users, so bailing safely.

---

**Q4. When `deserializeMessages` finds an `interrupted_turn` (process killed mid-tool-use), what does it inject?**

A. An error assistant message explaining the interruption
B. A synthetic `NO_RESPONSE_REQUESTED` sentinel only
C. A synthetic user message "Continue from where you left off." (isMeta: true)
D. Nothing — it simply replays the existing messages

**Answer: C** — The synthetic isMeta:true user message unifies interrupted_turn into interrupted_prompt so the consumer handles one case.

---

**Q5. Why does `ErrorOverview.tsx` use synchronous `readFileSync` instead of an async read?**

A. Async file reads are not supported in the version of Bun used
B. The Ink render path is synchronous; going async would require suspense restructuring, and this is the error path where the REPL is already broken
C. It's a performance optimization — sync reads are faster for small files
D. The source file path may not be known until render time

**Answer: B** — The component renders synchronously inside the Ink reconciler. Async would require suspense restructuring. Since this is the error path (process already broken), sync I/O is acceptable.

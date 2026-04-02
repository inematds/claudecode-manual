# mdENG — Lesson 04 — Query Engine & LLM API — Source Deep Dive

## 1. The Big Picture

When a message is sent to Claude Code, it passes through **four distinct layers** before reaching the user's terminal:

1. **QueryEngine.submitMessage()** - Validates the prompt, builds the system prompt, resolves the model, records the transcript, then hands off to `query()`.

2. **query() → queryLoop()** - An `async function*` that loops until the model stops calling tools. Each iteration represents one model call.

3. **queryModel / callModel** - Calls the Anthropic API via the SDK's streaming interface, wrapped in `withRetry()`.

4. **Stop hooks & token budget** - After each turn, external hooks run; the token budget determines whether to inject a nudge and continue looping.

**Key insight:** "Every public surface of Claude Code — the REPL, the SDK, remote Claude Code — funnels through the same `query()` generator."

## 2. Sequence Diagram

The message flow for a single conversation turn involving tool calls shows the loop between `queryLoop` and `queryModel` as the core of agentic behavior:

- User sends message to QueryEngine via `submitMessage(prompt)`
- QueryEngine fetches system prompt parts and builds initial message
- Calls `query()` with messages and system prompt
- Loop iterates: applies tool result budget, microcompact, snip, autocompact
- Each iteration calls `queryModel` with messages and tools
- `queryModel` posts to Anthropic API (streaming, with retry)
- API streams back `content_block_delta` events
- `queryModel` yields AssistantMessage with text/tool_use blocks
- StreamingToolExecutor tracks tool_use blocks
- If tool_use blocks present: runs tools and appends tool_results to messages (continues loop)
- If no tool calls: handles stop hooks, checks token budget
- Returns Terminal response to user

## 3. QueryEngine — One Engine Per Conversation

`QueryEngine` is a **stateful class** instantiated once per conversation. It holds mutable message history, total token usage, permission denials, and the abort controller. Each `submitMessage()` call is one "turn" within that conversation.

```typescript
export class QueryEngine {
  private mutableMessages: Message[]
  private abortController: AbortController
  private totalUsage: NonNullableUsage
  private permissionDenials: SDKPermissionDenial[]

  // Turn-scoped: cleared at start of each submitMessage() call
  private discoveredSkillNames = new Set<string>()

  async *submitMessage(
    prompt: string | ContentBlockParam[],
    options?: { uuid?: string; isMeta?: boolean },
  ): AsyncGenerator<SDKMessage> {
    // 1. Build system prompt (fetchSystemPromptParts)
    // 2. processUserInput — handles slash commands
    // 3. recordTranscript — persists BEFORE the API call
    // 4. yield* query({ messages, ... })
    // 5. yield final result SDKMessage
  }
}
```

### Why Transcript is Written Before the API Call

The user's message is persisted to disk before `query()` is called. This makes a session resumable even if the process is killed before the model responds. The design principle states: "Writing now makes the transcript resumable from the point the user message was accepted, even if no API response ever arrives."

**SDK vs REPL:** QueryEngine is used by the SDK/headless path. The REPL uses its own wiring through `ask()` but calls the same `query()` function.

## 4. Inside queryLoop() — The While(true) Core

`queryLoop()` in `query.ts` is a `while(true)` loop carrying a typed `State` object between iterations:

```typescript
type State = {
  messages: Message[]
  toolUseContext: ToolUseContext
  autoCompactTracking: AutoCompactTrackingState | undefined
  maxOutputTokensRecoveryCount: number
  hasAttemptedReactiveCompact: boolean
  maxOutputTokensOverride: number | undefined
  pendingToolUseSummary: Promise<ToolUseSummaryMessage | null> | undefined
  stopHookActive: boolean | undefined
  turnCount: number
  transition: Continue | undefined   // WHY we looped again
}
```

### Continue Transitions — Seven Reasons to Loop

The `transition` field records why the loop continued:

| transition.reason | Meaning |
|---|---|
| `max_output_tokens_escalate` | First hit of 8k cap; retry at 64k max_tokens |
| `max_output_tokens_recovery` | Model hit output limit; inject recovery nudge (up to 3×) |
| `reactive_compact_retry` | Prompt-too-long → compacted history → retry |
| `collapse_drain_retry` | Prompt-too-long → drained context-collapse stages → retry |
| `stop_hook_blocking` | A stop hook returned a blocking error; re-query with error as user message |
| `token_budget_continuation` | Token budget says work isn't done; inject nudge and continue |
| _(needs follow-up)_ | Normal: model returned tool_use blocks → run tools → loop |

**Termination conditions:** The loop exits via a `Terminal` on: `completed`, `blocking_limit`, `model_error`, `prompt_too_long`, `aborted_streaming`, `stop_hook_prevented`, `image_error`.

## 5. Streaming & the API Layer

`queryModel` in `claude.ts` is an `async function*` that calls the Anthropic beta messages endpoint and re-yields each stream event as an internal `AssistantMessage` or `StreamEvent`.

```typescript
for await (const message of deps.callModel({
  messages: prependUserContext(messagesForQuery, userContext),
  systemPrompt: fullSystemPrompt,
  thinkingConfig: toolUseContext.options.thinkingConfig,
  tools: toolUseContext.options.tools,
  signal: toolUseContext.abortController.signal,
  options: { model: currentModel, fallbackModel, ... },
})) {
  if (message.type === 'assistant') {
    assistantMessages.push(message)
    // tool_use blocks trigger needsFollowUp = true
    const toolBlocks = message.message.content
      .filter(b => b.type === 'tool_use')
    if (toolBlocks.length > 0) needsFollowUp = true
  }
  yield yieldMessage // surfaces to SDK caller / REPL
}
```

### Streaming Tool Execution

When `config.gates.streamingToolExecution` is enabled, a `StreamingToolExecutor` fires tools while the stream is still open. Tools with available inputs start executing in parallel with the model still generating text, reducing latency on multi-tool turns.

**Tombstone messages:** If a streaming fallback is triggered mid-stream, partially-received `AssistantMessage` objects are tombstoned — yielded as `{ type: 'tombstone', message }` so the UI and transcript can remove them. This prevents "thinking blocks cannot be modified" API errors on retry.

### Deep Dive: withRetry() — Exponential Backoff, 529s, and OAuth Refresh

Every API call goes through `withRetry()` in `services/api/withRetry.ts`. The function is an `async function*` retrying up to `DEFAULT_MAX_RETRIES = 10` times, yielding a `SystemAPIErrorMessage` before each sleep so users see live status updates.

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
    BASE_DELAY_MS * Math.pow(2, attempt - 1),
    maxDelayMs,
  )
  const jitter = Math.random() * 0.25 * baseDelay
  return baseDelay + jitter
}
```

Key retry decision rules:

- **529 (overloaded):** Only foreground query sources retry (user is waiting). Background sources bail immediately to avoid amplifying capacity cascades.
- **Opus fallback:** After 3 consecutive 529s on a non-custom Opus model, throws `FallbackTriggeredError` which `queryLoop` catches and switches to `fallbackModel`.
- **OAuth 401:** Forces a token refresh via `handleOAuth401Error()` before the next attempt.
- **Context overflow 400:** Parses token counts from the error message and computes a new `maxTokensOverride`.
- **Persistent mode (UNATTENDED_RETRY):** Retries indefinitely with 30-min backoff cap, yielding heartbeat messages every 30s so the host doesn't kill the session for inactivity.
- **ECONNRESET/EPIPE:** Stale keep-alive socket detected; `disableKeepAlive()` is called before retry.

### Deep Dive: SSE Stream → AssistantMessage Reconstruction

The Anthropic streaming API sends Server-Sent Events in this sequence: `message_start` → one or more `content_block_start` / `content_block_delta` / `content_block_stop` pairs → `message_delta` (with final usage + stop_reason) → `message_stop`.

`queryModel` reconstructs a complete `AssistantMessage` object per content block and yields it. Usage is mutated in-place on the last message once `message_delta` arrives — final stop_reason and token counts are not available until the stream ends.

```typescript
// From QueryEngine.ts — usage tracking
if (message.event.type === 'message_start') {
  currentMessageUsage = updateUsage(EMPTY_USAGE, message.event.message.usage)
}
if (message.event.type === 'message_delta') {
  currentMessageUsage = updateUsage(currentMessageUsage, message.event.usage)
  if (message.event.delta.stop_reason != null) {
    lastStopReason = message.event.delta.stop_reason
  }
}
```

One subtlety: `tool_use` blocks include their JSON `input` via deltas. If a tool's `backfillObservableInput` method adds fields to the input (e.g., expanding a file path), only a clone of the message is yielded to observers — the original stays byte-for-byte identical for prompt caching.

## 6. Context Management & Autocompact

Before each API call, `queryLoop` runs a pipeline of context reduction strategies in fixed priority order:

1. **applyToolResultBudget()** - Caps the byte size of individual tool results. Large results are stored externally and replaced with a reference stub.

2. **snipCompact (HISTORY_SNIP feature)** - Removes old messages from the middle of history when provably unneeded, freeing tokens without full summarization.

3. **microcompact / cached microcompact** - Merges consecutive tool-result/user message pairs into condensed summaries. Cached variant uses API-side cache edits to avoid retransmitting deleted blocks.

4. **contextCollapse (CONTEXT_COLLAPSE feature)** - A read-time projection over the REPL's full history. Staged collapses are committed on each entry; the model sees a collapsed view while the UI retains full history for scrollback.

5. **autoCompact** - When context approaches the blocking limit, triggers full summarization via a forked agent. If it fires, the loop continues immediately with post-compact messages.

### Deep Dive: Autocompact — Thresholds, Circuit Breakers, and task_budget

The blocking limit check happens _after_ all compaction strategies have run. If context is still over the limit, a synthetic `PROMPT_TOO_LONG_ERROR_MESSAGE` is yielded and the loop exits with reason `blocking_limit` — the user must manually run `/compact`.

Reactive compact is a fallback path triggered by a real 413 from the API (prompt-too-long). The engine withholds the error message during streaming, then attempts one reactive compaction. If that fails, the error is surfaced and stop hooks are skipped to prevent a death spiral.

The **task_budget** feature tracks total context tokens consumed across compact boundaries. When the server summarizes history, it would normally under-count the pre-compact spend; `taskBudgetRemaining` carries the correct cumulative spend across boundaries.

```typescript
// task_budget carryover across compaction (query.ts ~508)
if (params.taskBudget) {
  const preCompactContext =
    finalContextTokensFromLastResponse(messagesForQuery)
  taskBudgetRemaining = Math.max(
    0,
    (taskBudgetRemaining ?? params.taskBudget.total) - preCompactContext,
  )
}
```

## 7. Stop Hooks — Post-Turn Lifecycle

After the model finishes (no tool calls, no recovery needed), the engine calls `handleStopHooks()` in `query/stopHooks.ts`. Stop hooks are external shell scripts or commands configured by the user. They run after every turn and can:

- **Produce blocking errors** — injected as user messages, triggering another loop iteration
- **Prevent continuation** — the engine returns `{ reason: 'stop_hook_prevented' }`
- **Fire background tasks** — prompt suggestions, memory extraction, auto-dream — all fire-and-forget

### Deep Dive: Stop Hooks, TeammateIdle, TaskCompleted, and Fire-and-Forget Side Effects

`handleStopHooks()` runs three categories of hooks in order:

#### 1. Stop Hooks (always)

Registered via `settings.json` hooks configuration. Run in parallel; each result is collected as a `hook_success`, `hook_non_blocking_error`, or `hook_error_during_execution` attachment. A blocking error is any hook exit-code failure where the hook explicitly signals it should block.

#### 2. TaskCompleted Hooks (teammate mode only)

In teammate mode (multi-agent setups), hooks fire for each `in_progress` task owned by this agent. These mirror stop hook semantics (can block, can prevent continuation).

#### 3. TeammateIdle Hooks (teammate mode only)

Fire when this teammate transitions to idle. Can also block or prevent continuation.

#### 4. Fire-and-Forget Background Tasks

Skipped in bare mode (`-p` flag). Fired without `await` in interactive mode:

- `executePromptSuggestion` — generates _btw..._ suggestions
- `executeExtractMemories` — extracts facts to MEMORY.md
- `executeAutoDream` — autonomous background exploration

```typescript
// --bare / SIMPLE: skip background bookkeeping
// Scripted -p calls don't want auto-memory or forked agents
// contending for resources during shutdown.
if (!isBareMode()) {
  void executePromptSuggestion(stopHookContext)
  if (feature('EXTRACT_MEMORIES') && isExtractModeActive()) {
    void extractMemoriesModule!.executeExtractMemories(...)
  }
  if (!toolUseContext.agentId) {
    void executeAutoDream(...)
  }
}
```

## 8. Token Budget — The Auto-Continue Feature

`query/tokenBudget.ts` implements an auto-continue feature for the SDK path. When a per-turn token budget is configured, the engine checks after each clean model stop whether the model has "used up" enough of its budget. If not, it injects a nudge message and loops again.

### Deep Dive: BudgetTracker, Thresholds, and Diminishing-Returns Detection

```typescript
const COMPLETION_THRESHOLD = 0.9   // 90% used = done
const DIMINISHING_THRESHOLD = 500  // <500 new tokens = no progress

export function checkTokenBudget(
  tracker: BudgetTracker,
  agentId: string | undefined,
  budget: number | null,
  globalTurnTokens: number,
): TokenBudgetDecision {
  if (agentId || budget === null || budget <= 0) {
    return { action: 'stop', completionEvent: null }
  }
  const pct = Math.round((globalTurnTokens / budget) * 100)
  const isDiminishing =
    tracker.continuationCount >= 3 &&
    deltaSinceLastCheck < DIMINISHING_THRESHOLD &&
    tracker.lastDeltaTokens < DIMINISHING_THRESHOLD
  // Continue if under 90% AND not diminishing
  if (!isDiminishing && turnTokens < budget * COMPLETION_THRESHOLD) {
    return { action: 'continue', nudgeMessage: ... }
  }
  return { action: 'stop', ... }
}
```

The decision logic has two early-stop conditions:

- **Budget exhausted:** turn tokens >= 90% of budget → stop
- **Diminishing returns:** after 3+ continuations, if both the current delta and previous delta are under 500 tokens → stop (the model is spinning)

The nudge message is injected as an _isMeta_ user message so it doesn't appear in the REPL transcript, and the loop continues with `transition.reason = 'token_budget_continuation'`.

## 9. Key Takeaways

#### One Loop, Many Exit Reasons

The `while(true)` in `queryLoop` exits via a typed `Terminal` value. "Every possible stopping condition — completion, errors, abort, stop hooks, budget — has a named reason."

#### Generators All the Way Down

`submitMessage`, `query`, `queryLoop`, `queryModel`, `withRetry`, `handleStopHooks` — all are `async function*`. This lets the entire stack compose cleanly with `yield*` and backpressure flows naturally.

#### Transcript-First Reliability

The user message is written to disk before the API is called. "Even a process kill between send and response leaves a resumable session."

#### Feature-Gated Dead Code Elimination

`feature('HISTORY_SNIP')`, `feature('TOKEN_BUDGET')`, `feature('CONTEXT_COLLAPSE')` etc. are evaluated at bundle time by Bun, eliminating unreachable code from external builds and preventing string leakage.

#### Background Effects Are Fire-and-Forget

Memory extraction, prompt suggestions, auto-dream — all are `void` promises. "They must not block the response stream and must not run in bare (`-p`) mode where resource contention on shutdown matters."

#### Retry Is Smarter Than Exponential Backoff

"Foreground vs background source routing, fast-mode cooldowns, OAuth refresh, persistent keep-alive, Opus→fallback after 3×529, context-overflow token recalculation — the retry layer is a small state machine, not just a sleep loop."

## 10. Quiz

Five questions to check understanding:

#### Question 1: Why does QueryEngine.submitMessage() write the transcript to disk _before_ calling query()?

**Correct Answer:** So the session can be resumed even if the process is killed before any API response arrives

#### Question 2: After exactly 3 consecutive 529 (overloaded) errors on a non-custom Opus model, withRetry() throws which error?

**Correct Answer:** `FallbackTriggeredError`

#### Question 3: What does the transition.reason field on the State object represent?

**Correct Answer:** The reason the _previous_ iteration decided to continue instead of exit

#### Question 4: In the token budget feature, when does checkTokenBudget() trigger a _diminishing returns_ early stop?

**Correct Answer:** After 3+ continuations AND both the current and previous token deltas are below 500 tokens

#### Question 5: Why does the stop-hooks logic skip running hooks when the last assistant message is an API error (e.g., rate limit or prompt-too-long)?

**Correct Answer:** To prevent a death spiral where a blocking stop hook causes a retry that produces another error, which triggers the hook again indefinitely

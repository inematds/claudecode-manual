# Context Compaction

## Lesson 15: Context Compaction

**How Claude Code manages a finite context window across long sessions — from instant microcompact to full LLM summarization.**

### 01 Overview

Every LLM has a finite context window. For Claude Code running a long coding session — reading files, running shell commands, iterating on bugs — that window fills up faster than you might expect. **Context compaction** is the set of strategies Claude Code uses to keep the conversation alive and useful as token pressure builds.

**Source files covered:**
- `services/compact/microCompact.ts`
- `services/compact/compact.ts`
- `services/compact/autoCompact.ts`
- `services/compact/sessionMemoryCompact.ts`
- `services/compact/prompt.ts`
- `services/compact/postCompactCleanup.ts`
- `services/compact/timeBasedMCConfig.ts`
- `services/compact/apiMicrocompact.ts`
- `commands/compact/compact.ts`
- `commands/context/context.tsx`

**Four major strategies**, each with a different cost/fidelity trade-off:

**Strategy 1: Microcompact**
Silently clear old tool-result content from the in-memory message array. Zero API calls, instant.

**Strategy 2: Session Memory Compact**
Replace old messages with a pre-built session-memory file. Zero summarization API call.

**Strategy 3: Full LLM Compact**
Fork a sub-agent to write a 9-section structured summary. One extra API call, highest fidelity.

**Strategy 4: Reactive Compact**
Triggered by a 413 prompt-too-long API error. Peels API rounds from the tail until the request fits.

**Mental model:**
Think of these strategies as a cost ladder. Claude Code always tries the cheapest option first. Microcompact is free. Session memory is nearly free. Full LLM compact costs one extra API call. Reactive compact is the emergency escape hatch when everything else fails.

---

### 02 Thresholds & Token Warning States

Compaction is threshold-gated. Before every API call, `autoCompact.ts` computes an _effective context window_ by subtracting output headroom from the raw context size, then derives four distinct alert levels.

```typescript
// autoCompact.ts — effective window calculation
const MAX_OUTPUT_TOKENS_FOR_SUMMARY = 20_000  // p99.99 of compact output

export function getEffectiveContextWindowSize(model: string): number {
  const reservedTokensForSummary = Math.min(
    getMaxOutputTokensForModel(model),
    MAX_OUTPUT_TOKENS_FOR_SUMMARY,
  )
  const contextWindow = getContextWindowForModel(model, getSdkBetas())
  return contextWindow - reservedTokensForSummary
}
```

| State | Buffer below effective window | Effect | Constant |
|-------|------|--------|----------|
| Normal | > 20,000 tokens left | No action | -- |
| Warning | <= 20,000 tokens left | UI shows yellow indicator | `WARNING_THRESHOLD_BUFFER_TOKENS = 20_000` |
| Error | <= 20,000 tokens left (same level) | UI shows red indicator | `ERROR_THRESHOLD_BUFFER_TOKENS = 20_000` |
| Auto-Compact | <= 13,000 tokens left | Triggers automatic compaction | `AUTOCOMPACT_BUFFER_TOKENS = 13_000` |
| Blocking | <= 3,000 tokens left | Blocks new user input | `MANUAL_COMPACT_BUFFER_TOKENS = 3_000` |

**Why 20k tokens reserved?**

The compact summary agent itself generates output. The team measured p99.99 of compact output as 17,387 tokens and rounded up to 20k. If the effective window didn't leave this headroom, the compaction request could itself hit prompt-too-long — a nasty failure to debug.

**Deep dive: the circuit breaker**

Auto-compact can fail (network timeout, prompt-too-long on the compaction request itself). Without a guard, each subsequent turn would retry compaction, hammering the API with doomed attempts. The code tracks `consecutiveFailures` and stops after 3:

```typescript
// autoCompact.ts
const MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3

// BQ 2026-03-10: 1,279 sessions had 50+ consecutive failures (up to 3,272)
// in a single session, wasting ~250K API calls/day globally.
if (tracking?.consecutiveFailures >= MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES) {
  return { wasCompacted: false }
}
```

The comment is unusually candid: before this circuit breaker existed, one failure mode was burning a quarter-million API calls per day globally. The fix is three lines.

---

### 03 Microcompact — Instant Tool-Result Pruning

Microcompact is a pre-API-call pass that clears the content of old tool results directly in the in-memory message array. It does _not_ call the LLM and does _not_ write anything to disk. The goal: shrink the prompt before it is sent, paying nothing.

**Which tools are eligible?**

```typescript
// microCompact.ts — only results from these tools can be cleared
const COMPACTABLE_TOOLS = new Set<string>([
  FILE_READ_TOOL_NAME,
  ...SHELL_TOOL_NAMES,     // Bash, etc.
  GREP_TOOL_NAME,
  GLOB_TOOL_NAME,
  WEB_SEARCH_TOOL_NAME,
  WEB_FETCH_TOOL_NAME,
  FILE_EDIT_TOOL_NAME,
  FILE_WRITE_TOOL_NAME,
])
```

Only read/search/shell tool results qualify. The results of these tools are large, often stale, and unlikely to be needed verbatim after a few turns. Tool results from custom MCP tools, agent spawns, or user-visible actions are left alone.

**Three microcompact paths:**

```
flowchart TD
A[microcompactMessages called] --> B{Time-based trigger?}
B -- Yes, gap > threshold --> C[Time-Based MC\nContent-clear old tool results\nmutate messages directly]
B -- No --> D{feature CACHED_MC\n+ supported model?}
D -- Yes --> E[Cached MC Path\nQueue cache_edits block\ndo NOT mutate messages]
D -- No --> F[Return messages unchanged\nautocompact handles pressure]
C --> G[Reset cachedMCState\nreturn mutated messages]
E --> H[Return messages + pendingCacheEdits\nAPI layer inserts edits]
```

**Path 1: Time-based microcompact**

If the gap since the last assistant message exceeds a threshold (default: 60 minutes), the server-side prompt cache has almost certainly expired. Rewriting the prompt is unavoidable — so content-clearing old tool results _before_ the request shrinks what gets rewritten. The logic is purely client-side and mutates messages in place.

```typescript
// timeBasedMCConfig.ts — GrowthBook-controlled config
const TIME_BASED_MC_CONFIG_DEFAULTS: TimeBasedMCConfig = {
  enabled: false,
  gapThresholdMinutes: 60,  // server 1h cache TTL
  keepRecent: 5,            // always keep the last 5 tool results
}

// microCompact.ts — content-clearing loop
const keepSet = new Set(compactableIds.slice(-keepRecent))
const clearSet = new Set(compactableIds.filter(id => !keepSet.has(id)))

// Replace each cleared block's content with a sentinel string
return { ...block, content: TIME_BASED_MC_CLEARED_MESSAGE }
// TIME_BASED_MC_CLEARED_MESSAGE = '[Old tool result content cleared]'
```

**Path 2: Cached microcompact (experimental)**

The regular time-based path mutates message content, which breaks the server-side prompt cache (the prefix has changed). Cached MC solves this differently: instead of rewriting message content, it queues a `cache_edits` block for the _API layer_ to apply server-side, leaving the cached prefix intact.

```typescript
// microCompact.ts — cached MC result shape
return {
  messages,  // UNCHANGED — messages are not mutated
  compactionInfo: {
    pendingCacheEdits: {
      trigger: 'auto',
      deletedToolIds: toolsToDelete,
      baselineCacheDeletedTokens: baseline,
    },
  },
}
```

**Cache-editing insight:**

The key design insight: the API lets you tell it "delete these tool results from the cached prefix" without resending the whole prompt. This means the cache stays warm even after pruning. Regular microcompact (content-clear) necessarily invalidates the cache because the prompt text changes.

**Deep dive: token estimation for microcompact decisions**

Microcompact needs to estimate how many tokens a tool result contains so it can decide what to clear. It uses a rough character-based heuristic, padded by 4/3:

```typescript
// microCompact.ts
export function estimateMessageTokens(messages: Message[]): number {
  // ... walk all blocks ...
  // Pad estimate by 4/3 to be conservative since we're approximating
  return Math.ceil(totalTokens * (4 / 3))
}
```

Images and documents are always counted as 2,000 tokens regardless of format (`IMAGE_MAX_TOKEN_SIZE = 2000`). The 4/3 padding compensates for the character-to-token ratio being higher than 1:1.

---

### 04 Session Memory Compaction

Session memory compaction is an experimental path that avoids the cost of a full LLM summarization call entirely. Instead of asking Claude to summarize the conversation, it uses a continuously-updated _session memory file_ written in the background as context for the compacted session.

**When does it activate?**

Both `autoCompactIfNeeded` and the `/compact` command try session memory compaction first, before falling back to full LLM compact:

```typescript
// autoCompact.ts — session memory is always tried first
const sessionMemoryResult = await trySessionMemoryCompaction(
  messages,
  toolUseContext.agentId,
  recompactionInfo.autoCompactThreshold,
)
if (sessionMemoryResult) {
  setLastSummarizedMessageId(undefined)
  runPostCompactCleanup(querySource)
  return { wasCompacted: true, compactionResult: sessionMemoryResult }
}
```

**What messages are kept?**

The key function `calculateMessagesToKeepIndex` finds the boundary between what has already been summarized into session memory and what is recent enough to keep verbatim. It expands backwards from the last-summarized message until it satisfies configurable minimums:

```typescript
// sessionMemoryCompact.ts — default config (can be overridden by GrowthBook)
export const DEFAULT_SM_COMPACT_CONFIG: SessionMemoryCompactConfig = {
  minTokens: 10_000,          // keep at least 10k tokens of recent context
  minTextBlockMessages: 5,    // keep at least 5 messages with text content
  maxTokens: 40_000,          // hard cap: don't keep more than 40k tokens
}
```

**The tool-pair invariant:**

A subtle correctness requirement: the API rejects conversations where a `tool_result` block references a `tool_use` block that doesn't appear earlier in the message list. When we slice to keep only recent messages, we might accidentally include a user message with `tool_result` blocks but exclude the preceding assistant message that had the corresponding `tool_use`. The function `adjustIndexToPreserveAPIInvariants` walks backwards to find and include any orphaned tool-use pairs.

```
flowchart LR
A["Messages: [... old ... | SUMMARIZED | NEW ]"] --> B["findLastSummarizedIndex()"]
B --> C["calculateMessagesToKeepIndex()\n(start from summarized+1,\nexpand backward to meet minimums)"]
C --> D["adjustIndexToPreserveAPIInvariants()\n(pull in orphaned tool_use pairs\n+ thinking block partners)"]
D --> E["messagesToKeep = messages.slice(startIndex)\n.filter(!isCompactBoundary)"]
E --> F["Build CompactionResult\nwith session memory as summary\nno LLM call needed"]
```

**Deep dive: two compaction scenarios handled**

**Scenario 1 -- Normal case:** `lastSummarizedMessageId` is set. The session memory extraction ran at least once and we know exactly which messages it covered. We keep only messages after that ID (expanded to meet minimums).

**Scenario 2 -- Resumed session:** Session memory has content but `lastSummarizedMessageId` is unset (e.g. the session was resumed from a previous transcript). We treat this as "everything is summarized" and set `lastSummarizedIndex` to `messages.length - 1`. The expansion loop may keep some recent messages anyway to meet minimums.

```typescript
// sessionMemoryCompact.ts
if (!lastSummarizedMessageId) {
  // Resumed session: session memory has content but we don't know the boundary
  lastSummarizedIndex = messages.length - 1
  logEvent('tengu_sm_compact_resumed_session', {})
}
```

---

### 05 Full LLM Compaction — The 9-Section Summary Prompt

When neither microcompact nor session memory compaction is available, Claude Code forks a sub-agent and asks it to write a structured summary of the entire conversation. This is the most expensive path — one extra API call — but produces the most faithful summary.

**The compaction flow:**

```
sequenceDiagram
participant Q as Query Loop
participant C as compactConversation()
participant H as Pre-Compact Hooks
participant F as Forked Agent
participant P as Post-Compact
Q->>C: messages, toolUseContext
C->>H: executePreCompactHooks()
H-->>C: customInstructions (merged)
C->>F: streamCompactSummary() with 9-section prompt
F-->>C: summary text (analysis + summary XML)
Note over C: formatCompactSummary() strips <analysis> block
C->>P: createCompactBoundaryMessage()
C->>P: createPostCompactFileAttachments() — up to 5 files
C->>P: processSessionStartHooks()
C-->>Q: CompactionResult{boundaryMarker, summaryMessages, attachments}
```

**The 9-section summary prompt:**

The compaction prompt instructs the model to produce a structured summary with exactly these nine sections. This structure is intentional: it ensures every subsequent session can understand what was happening even with no other context.

1. **Primary Request and Intent** — All of the user's explicit requests and intents in detail.
2. **Key Technical Concepts** — Important technologies, frameworks, and design patterns discussed.
3. **Files and Code Sections** — Specific files examined/modified/created, with full code snippets where applicable.
4. **Errors and Fixes** — Every error encountered and how it was resolved. User feedback is highlighted.
5. **Problem Solving** — Solved problems and ongoing troubleshooting efforts.
6. **All User Messages** — Every non-tool-result user message listed verbatim. Critical for tracking intent drift.
7. **Pending Tasks** — Tasks explicitly assigned that have not yet been completed.
8. **Current Work** — Precisely what was happening immediately before the compact, with file names and snippets.
9. **Optional Next Step** — Only if directly in line with the most recent user request. Must include verbatim quotes.

**Why section 6 matters:**

"All user messages" is the only section that cannot be inferred from tool use alone. Without it, the summary would capture what Claude _did_ but lose the user's _intent shifts_. Including verbatim user messages prevents the model from hallucinating a coherent narrative that doesn't match what actually happened.

**The analysis scratchpad pattern:**

The prompt asks the model to wrap its reasoning in `<analysis>` tags before producing the `<summary>`. The analysis section is a drafting scratchpad — it is stripped before the summary reaches the context:

```typescript
// prompt.ts — formatCompactSummary strips analysis block
export function formatCompactSummary(summary: string): string {
  let formatted = summary

  // Strip analysis section — drafting scratchpad, no informational value
  formatted = formatted.replace(
    /<analysis>[\s\S]*?<\/analysis>/,
    '',
  )

  // Extract and format <summary> section
  const match = formatted.match(/<summary>([\s\S]*?)<\/summary>/)
  if (match) {
    formatted = formatted.replace(
      /<summary>[\s\S]*?<\/summary>/,
      `Summary:\n${match[1]!.trim()}`,
    )
  }
  return formatted.trim()
}
```

**No-tools preamble:**

Because the compact request forks the main conversation's tool set (for cache-key match), the model might attempt a tool call despite being asked to summarize. A tool call wastes the only turn and produces no summary. The prompt starts with an aggressive preamble:

```typescript
// prompt.ts
const NO_TOOLS_PREAMBLE = `CRITICAL: Respond with TEXT ONLY. Do NOT call any tools.

- Do NOT use Read, Bash, Grep, Glob, Edit, Write, or ANY other tool.
- You already have all the context you need in the conversation above.
- Tool calls will be REJECTED and will waste your only turn — you will fail the task.
- Your entire response must be plain text: an <analysis> block followed by a <summary> block.

`
```

**Prompt-too-long on compaction itself:**

The compact request can itself hit a 413 if the conversation is very long. The code handles this with up to 3 retries: each retry drops the oldest API-round groups until the request fits or nothing can be dropped. This is tracked as `tengu_compact_ptl_retry` events.

**Deep dive: partial compact prompts:**

There are actually three compact prompt variants, not one:

- **BASE_COMPACT_PROMPT** — full conversation summary, sections 1-9 including "Current Work"
- **PARTIAL_COMPACT_PROMPT** (`direction: 'from'`) — summary of only the _recent_ portion; earlier messages are kept intact
- **PARTIAL_COMPACT_UP_TO_PROMPT** (`direction: 'up_to'`) — summary placed at the _start_ of a continuing session; section 9 becomes "Context for Continuing Work" instead of "Optional Next Step"

The reactive compact path uses partial prompts to summarize only the portion of the conversation that needs to be dropped.

---

### 06 Reactive Compact — Context Collapse

Proactive auto-compact fires _before_ the context window is full. But what if the window is already over limit when the session starts — for instance, a resumed session with a large transcript, or a session where auto-compact was disabled? The reactive compact path handles this case.

**When does reactive compact fire?**

Reactive compact activates in two modes:

- **Reactive-only mode** (`tengu_cobalt_raccoon` feature flag): proactive auto-compact is suppressed entirely; the 413 error from the API is the only trigger.
- **Emergency fallback**: the API returns a 413 `prompt_too_long` error during a normal request. The code peels API-round groups from the oldest end and retries.

```typescript
// autoCompact.ts — reactive-only mode short-circuit
if (feature('REACTIVE_COMPACT')) {
  if (getFeatureValue_CACHED_MAY_BE_STALE('tengu_cobalt_raccoon', false)) {
    return false  // suppress proactive autocompact
  }
}
```

**Context collapse:**

There is also a separate _context collapse_ feature (`CONTEXT_COLLAPSE`) that suppresses auto-compact entirely when enabled. Context collapse is its own context management system that operates at 90% (commit) and 95% (blocking) thresholds, more granular than compaction. Auto-compact sitting at ~93% would race it:

```typescript
// autoCompact.ts — suppress autocompact when context collapse is active
if (feature('CONTEXT_COLLAPSE')) {
  if (isContextCollapseEnabled()) {
    return false  // let collapse manage the headroom problem
  }
}
```

**Recursion guards:**

Auto-compact must not fire for `querySource === 'session_memory'` or `querySource === 'compact'` — these are forked agents that would deadlock if they tried to compact themselves. The guard is checked before any compaction logic runs.

---

### 07 Message Grouping by API Round

Both the compact-request prompt-too-long retry and the reactive compact path need to drop messages in safe units. The unit is an _API round_: the set of messages from one complete request-response pair.

```typescript
// grouping.ts — group at assistant message.id boundaries
export function groupMessagesByApiRound(messages: Message[]): Message[][] {
  const groups: Message[][] = []
  let current: Message[] = []
  let lastAssistantId: string | undefined

  for (const msg of messages) {
    if (
      msg.type === 'assistant' &&
      msg.message.id !== lastAssistantId &&
      current.length > 0
    ) {
      groups.push(current)
      current = [msg]
    } else {
      current.push(msg)
    }
    if (msg.type === 'assistant') lastAssistantId = msg.message.id
  }
  if (current.length > 0) groups.push(current)
  return groups
}
```

The boundary signal is the **assistant message ID** changing. Streaming sends one `AssistantMessage` per content block (thinking, tool_use, text) all sharing the same `message.id`. A new ID means a genuinely new API round-trip. This lets the code safely split at round boundaries without breaking tool_use/tool_result pairs that belong to the same round.

---

### 08 Post-Compact Cleanup

After any successful compaction — microcompact, session memory, or full LLM — a cleanup function runs to invalidate caches and state that would be wrong in the new context window.

```typescript
// postCompactCleanup.ts — called by ALL compaction paths
export function runPostCompactCleanup(querySource?: QuerySource): void {
  const isMainThread =
    querySource === undefined ||
    querySource.startsWith('repl_main_thread') ||
    querySource === 'sdk'

  resetMicrocompactState()           // always

  if (feature('CONTEXT_COLLAPSE') && isMainThread) {
    resetContextCollapse()           // main thread only
  }
  if (isMainThread) {
    getUserContext.cache.clear?.()   // re-read CLAUDE.md on next turn
    resetGetMemoryFilesCache()       // arm InstructionsLoaded hook
  }
  clearSystemPromptSections()
  clearClassifierApprovals()
  clearSpeculativeChecks()
  clearBetaTracingState()
  clearSessionMessagesCache()
}
```

**Subagent safety:**

Subagents run in the same OS process as the main thread and share module-level state. If a subagent's compaction cleared `getUserContext`, it would corrupt the main thread's memory-file cache. The `isMainThread` guard prevents this. The guard uses `startsWith('repl_main_thread')` because output-style variants produce sources like `'repl_main_thread:outputStyle:custom'`.

**What is intentionally NOT cleared:**

The cleanup deliberately does _not_ reset `sentSkillNames`. Re-injecting the full skill listing (~4k tokens) on every compact would be pure cache invalidation with minimal benefit — the model still has the SkillTool schema and the invoked_skills attachment preserves used skill content. This is a deliberate performance trade-off documented in comments.

**Post-compact file attachments:**

After a full LLM compact, the system re-injects files the model had previously read, so it doesn't need to re-read them in the new session:

```typescript
// compact.ts — constants for post-compact file restoration
export const POST_COMPACT_MAX_FILES_TO_RESTORE = 5
export const POST_COMPACT_TOKEN_BUDGET = 50_000
export const POST_COMPACT_MAX_TOKENS_PER_FILE = 5_000
export const POST_COMPACT_MAX_TOKENS_PER_SKILL = 5_000
export const POST_COMPACT_SKILLS_TOKEN_BUDGET = 25_000
```

Skills are per-skill-capped rather than dropped entirely: skill files can be 18-20 KB each, and previously they were re-injected unbounded, costing 5-10k tokens per compact. Per-skill truncation keeps the most important instructions (at the top of the file) while bounding the total cost.

---

### 09 The /context Command

The `/context` command shows the user how full their context window is. The important design detail: it applies the same pre-API transforms that `query.ts` applies, so what the user sees reflects what the model actually receives — not the raw REPL scroll-back history.

```typescript
// context.tsx — mirrors the query.ts pre-API transform pipeline
function toApiView(messages: Message[]): Message[] {
  // 1. Slice to only messages after the last compact boundary
  let view = getMessagesAfterCompactBoundary(messages)
  // 2. Apply context-collapse projection if enabled
  if (feature('CONTEXT_COLLAPSE')) {
    view = projectView(view)
  }
  return view
}

// Then apply microcompact to get accurate token count
const { messages: compacted } = await microcompactMessages(apiView)
```

Without this pipeline, the token count would overcount by however much context collapse had saved — the user would see "180k, 3 spans collapsed" when the API only sees 120k. Applying the same transforms as the real API call path makes the display accurate.

---

## Key Takeaways

- Compaction is a cost ladder: microcompact is free, session memory is nearly free, full LLM compact costs one extra API call, reactive compact is the escape hatch.
- The effective context window is the raw window minus 20k tokens reserved for the compact summary output itself — a p99.99-based constant.
- Auto-compact triggers at 13k tokens before the effective window limit (`AUTOCOMPACT_BUFFER_TOKENS`); blocking triggers at 3k.
- Microcompact never calls the API — it content-clears old tool results in the local message array, or (with cached MC) queues a server-side cache_edit that doesn't break the prompt cache.
- Session memory compaction avoids the summarization API call entirely by using a continuously-updated memory file. It is gated on two feature flags and has a configurable min/max token budget for how many recent messages to preserve verbatim.
- The 9-section summary prompt is a deliberate structure: section 6 (all user messages) captures intent shifts that tool-use history alone would miss.
- The `<analysis>` block in compact output is a scratchpad — it is always stripped before the summary enters the context window.
- Post-compact cleanup is centralized in `runPostCompactCleanup` and guarded for subagents, which share module-level state with the main thread.
- The `/context` command applies the same pre-API transforms as the query loop to show accurate token counts, not the raw REPL history.

---

## Knowledge Check

**Q1. What is the primary reason the effective context window is smaller than the model's raw context window?**

A) The system prompt occupies a fixed 20k tokens
B) 20k tokens are reserved for the compact summary output so the compaction request itself doesn't fail
C) Images are estimated at 2000 tokens each and add overhead
D) The AUTOCOMPACT_BUFFER_TOKENS constant is subtracted at model-load time

**Q2. Cached microcompact differs from regular (time-based) microcompact primarily because it:**

A) Makes an extra API call to compress the tool results
B) Stores the compressed results to disk for faster startup next session
C) Queues a server-side cache_edits block rather than mutating local message content, preserving the prompt cache
D) Only clears tool results older than 24 hours

**Q3. Why does the compact summary prompt include section 6 "All user messages" when the tool-use history already captures what was done?**

A) Regulatory compliance requires preserving user input verbatim
B) Tool-use history captures what Claude did but not the user's shifting intent; verbatim messages prevent the summary from hallucinating a coherent narrative
C) User messages are smaller than tool results and cheaper to include
D) The summarizer model is weaker and needs explicit repetition

**Q4. The `adjustIndexToPreserveAPIInvariants` function expands the session-memory compact "keep" boundary backwards. What two invariants does it protect?**

A) Tool-use/tool-result pairs, and thinking blocks that share a message.id with kept assistant messages
B) Minimum token count and maximum token count
C) The compact boundary marker and the session memory file path
D) File attachments and MCP tool schemas

**Q5. Why does `runPostCompactCleanup` only clear `getUserContext` and `getMemoryFilesCache` for main-thread compactions, not subagent compactions?**

A) Subagents do not use CLAUDE.md files
B) Subagents run in a separate process with their own module state
C) Subagents run in the same process and share module-level state; resetting it from a subagent compact would corrupt the main thread's memory-file cache
D) Subagents have a shorter context window and don't need memory file reloads

**Q6. The `/context` command applies microcompact before displaying token usage. Why?**

A) To reduce the context window so the display fits on screen
B) To show the token count as the API will actually see it — after all pre-API transforms — rather than the raw REPL history
C) So the /context command itself doesn't count toward the token limit
D) To trigger auto-compact before showing the user the token count

# Message Processing Pipeline — Source Deep Dive

## Lesson 38

## 01 Overview

Every prompt travels through a layered pipeline before reaching the Anthropic API. The system handles images, slash commands, bash mode, queuing, hooks, attachments, and multi-block normalization across three source files totaling approximately 1,500 lines of TypeScript.

**Source files covered:**
- `utils/handlePromptSubmit.ts`
- `utils/processUserInput/processUserInput.ts`
- `utils/processUserInput/processTextPrompt.ts`
- `utils/messages.ts`

**Four conceptual stages:**

**Stage 1: Submit & Route**
`handlePromptSubmit` handles validation, exit-word handling, queuing versus immediate execution

**Stage 2: Input Classification**
`processUserInput` manages image resize, slash/bash/text branching, hook execution

**Stage 3: Message Construction**
`processTextPrompt` and `createUserMessage` build typed `UserMessage` objects

**Stage 4: API Normalization**
`normalizeMessagesForAPI` flattens, deduplicates, strips virtual messages

## 02 End-to-End Pipeline Diagram

The complete code path flows from React `onSubmit` handler through `handlePromptSubmit()`, conditional routing based on queue status and command type, image processing, attachment loading, hook execution, message creation, and final API normalization.

## 03 Stage 1 — handlePromptSubmit

`handlePromptSubmit.ts` serves as the React-facing entry point, receiving raw text input alongside UI context including the current message list, abort controllers, IDE selection state, and queued commands.

### The Two Execution Paths

**Path A — queue processor (commands already validated):**
```typescript
if (queuedCommands?.length) {
  startQueryProfile()
  await executeUserInput({ queuedCommands, ... })
  return
}
```

**Path B — direct user input:**
```typescript
const finalInput = expandPastedTextRefs(input, pastedContents)
// ... validation, queuing checks ...
const cmd: QueuedCommand = { value: finalInput, mode, pastedContents, ... }
await executeUserInput({ queuedCommands: [cmd], ... })
```

The queue processor path deliberately skips validation—those commands were validated when originally enqueued. Direct input passes through reference expansion, exit word checking, immediate command detection, and the `queryGuard` busy-check.

### Paste Reference Expansion

When files or text are pasted into input, Claude stores them under numeric IDs and inserts placeholder tokens like `[Pasted text #N]`. Before further processing, `expandPastedTextRefs` replaces these tokens with actual content. Images receive similar treatment but only include those whose `[Image #N]` pill remains present—orphaned image entries filter out early:

```typescript
const referencedIds = new Set(parseReferences(input).map(r => r.id))
const pastedContents = Object.fromEntries(
  Object.entries(rawPastedContents).filter(
    ([, c]) => c.type !== 'image' || referencedIds.has(c.id),
  ),
)
```

### The Queue Guard

When `queryGuard.isActive` is true (a query running), new input enters the module-level `commandQueue` rather than executing immediately. This plain array integrates with React components via `useSyncExternalStore`, enabling queue reactivity without additional state management libraries.

**Concurrency detail:** The queue guard reserves before `processUserInput` starts, not after. This prevents race conditions where a second submit arrives during awaits inside `processBashCommand` or `getMessagesForSlashCommand`.

### Immediate Commands

Commands flagged `immediate: true` in the command registry (like `/config` and `/doctor`) can execute while a query is in progress. They are detected via `startsWith('/')` check and dispatched through `setToolJSX`—rendering directly into the Ink component tree rather than being sent to the API:

```typescript
const immediateCommand = commands.find(
  cmd => cmd.immediate && isCommandEnabled(cmd) && cmd.name === commandName
)
if (immediateCommand && immediateCommand.type === 'local-jsx') {
  const impl = await immediateCommand.load()
  const jsx = await impl.call(onDone, context, commandArgs)
  setToolJSX({ jsx, isLocalJSXCommand: true, isImmediate: true })
}
```

## 04 Stage 2 — processUserInput

`processUserInput.ts` functions as the main classification engine. It receives a `QueuedCommand` value and routes it to one of four handlers based on mode and content shape.

### Image Pre-Processing

Array-form input (used when the SDK or VS Code sends multimodal content) normalizes first. Every image block passes through `maybeResizeAndDownsampleImageBlock` before string extraction:

```typescript
for (const block of input) {
  if (block.type === 'image') {
    const resized = await maybeResizeAndDownsampleImageBlock(block)
    processedBlocks.push(resized.block)
  } else {
    processedBlocks.push(block)
  }
}
normalizedInput = processedBlocks
```

Pasted images from the UI clipboard path resize in parallel to avoid serializing on slow images:

```typescript
const imageProcessingResults = await Promise.all(
  imageContents.map(async pastedImage => {
    const resized = await maybeResizeAndDownsampleImageBlock(imageBlock)
    return { resized, originalDimensions, sourcePath }
  })
)
```

### The Branching Switch

After images normalize and attachments load, a three-way branch sends input to the correct processor:

```typescript
// 1. Bash mode
if (mode === 'bash') {
  return processBashCommand(inputString, ...)
}

// 2. Slash command (not from bridge/CCR)
if (!effectiveSkipSlash && inputString.startsWith('/')) {
  return processSlashCommand(inputString, ...)
}

// 3. Regular text prompt (default)
return processTextPrompt(normalizedInput, imageContentBlocks, ...)
```

### The Ultraplan Keyword Path

A hidden fourth branch exists: if the `ULTRAPLAN` feature flag is enabled and the input (before paste expansion) contains the ultraplan keyword, it silently rewrites to `/ultraplan <input>` and routes through the slash command path. Detection runs against pre-expansion input specifically to prevent pasted content from accidentally triggering it.

### Bridge / Remote Control Safety

Messages from iOS or web clients arrive with `skipSlashCommands: true` to prevent accidental slash command execution. The `bridgeOrigin` flag creates a nuanced override: if the command is flagged `isBridgeSafeCommand()`, the skip clears and the command executes normally. If it is a known-but-unsafe command (local-jsx or terminal-only), a helpful error returns immediately rather than letting raw text like `/config` confuse the model.

### UserPromptSubmit Hooks

After `processUserInputBase` returns with `shouldQuery: true`, registered `UserPromptSubmit` hooks execute. A hook can:

- **Block** the prompt entirely—returning a system warning message instead
- **Prevent continuation**—keeping the prompt in context but stopping the query
- **Inject additional context**—appending `AttachmentMessage` objects before the API call

```typescript
for await (const hookResult of executeUserPromptSubmitHooks(...)) {
  if (hookResult.blockingError) {
    return { messages: [createSystemMessage(blockingMessage)], shouldQuery: false }
  }
  if (hookResult.preventContinuation) {
    result.messages.push(createUserMessage({ content: 'Operation stopped by hook' }))
    result.shouldQuery = false
    return result
  }
  if (hookResult.additionalContexts) {
    result.messages.push(createAttachmentMessage({ type: 'hook_additional_context', ... }))
  }
}
```

**Hook output safety:** Hook output caps at `10,000` characters via `applyTruncation()`. Any output beyond that replaces with a truncation notice, preventing runaway hook stdout from inflating the context window.

## 05 Stage 3 — Message Construction

The final "normal prompt" path lands in `processTextPrompt.ts`, where raw strings become typed `UserMessage` objects that the rest of the system understands.

### processTextPrompt — What It Does

This function spans approximately 100 lines with three core responsibilities:

1. Assign a fresh `promptId` via `setPromptId(randomUUID())` and start the OpenTelemetry interaction span
2. Emit analytics for negative keyword detection (`matchesNegativeKeyword`) and keep-going phrases
3. Assemble the content array—text first, then pasted image blocks below

```typescript
// Pasted images: text first, images after
if (imageContentBlocks.length > 0) {
  const textContent = typeof input === 'string'
    ? [{ type: 'text', text: input }]
    : input
  const userMessage = createUserMessage({
    content: [...textContent, ...imageContentBlocks],
    uuid, imagePasteIds, permissionMode,
  })
  return { messages: [userMessage, ...attachmentMessages], shouldQuery: true }
}
```

### createUserMessage — The Message Factory

`createUserMessage` in `utils/messages.ts` functions as the canonical factory for all user-side messages. Every field it stamps carries meaning:

**uuid:** Stable identity used for rewind, file history snapshots, and tool result pairing (either `randomUUID()` or caller-supplied)

**isMeta:** When true, the message is model-visible but never shown in the transcript. Used for image metadata, scheduled tasks, and system-generated prompts.

**origin:** Tracks message provenance—`undefined` indicates human keyboard input, `task-notification` indicates scheduled task, etc.

**permissionMode:** Safety snapshot storing the permission mode at message creation time, allowing rewind to restore the exact safety level.

**toolUseResult:** For `tool_result` messages, carries the structured output from the tool call.

**imagePasteIds:** Ordered list of paste IDs for images in the content array—used when splitting messages during normalization.

### Image Metadata isMeta Messages

Whenever images process, a separate `isMeta` user message appends containing dimension and source path information. This gives the model spatial context (e.g., `"[Image source: /tmp/screenshot.png, 1920x1080]"`) without polluting the visible transcript:

```typescript
function addImageMetadataMessage(result, imageMetadataTexts) {
  if (imageMetadataTexts.length > 0) {
    result.messages.push(
      createUserMessage({
        content: imageMetadataTexts.map(text => ({ type: 'text', text })),
        isMeta: true,
      })
    )
  }
  return result
}
```

### Synthetic Messages

`utils/messages.ts` exports constant strings used to inject synthetic signals into the conversation. These never show directly to users but influence model behavior at critical moments:

```typescript
export const INTERRUPT_MESSAGE = '[Request interrupted by user]'
export const CANCEL_MESSAGE =
  "The user doesn't want to take this action right now. STOP what you are doing..."
export const REJECT_MESSAGE =
  "The user doesn't want to proceed with this tool use..."
export const SYNTHETIC_TOOL_RESULT_PLACEHOLDER =
  '[Tool result missing due to internal error]'
```

**Training data protection:** `SYNTHETIC_TOOL_RESULT_PLACEHOLDER` exports specifically so the HFI (human feedback interface) submission path can detect it and _reject_ any payload containing it. Fake tool results would poison training data if submitted—this is an explicit safety valve.

## 06 Stage 4 — normalizeMessagesForAPI

Before the final message list reaches `api.ts`, it passes through `normalizeMessagesForAPI`—a multi-pass transformation ensuring payload validity for the Anthropic API.

### What normalizeMessages Does First

`normalizeMessages` (distinct from the API variant) splits every multi-content-block message into individual single-block messages. It uses a deterministic UUID derivation scheme so identical input always produces identical output UUIDs:

```typescript
// Derive stable UUID from parent UUID + block index
export function deriveUUID(parentUUID: UUID, index: number): UUID {
  const hex = index.toString(16).padStart(12, '0')
  return `${parentUUID.slice(0, 24)}${hex}` as UUID
}
```

### normalizeMessagesForAPI Passes

The API normalization function performs several distinct passes:

1. **Reorder attachments**—bubbles attachment messages up until they hit a tool result or assistant message boundary
2. **Strip virtual messages**—display-only messages (`isVirtual: true`) must never reach the API
3. **Build error-to-strip map**—maps API error types (PDF too large, image too large, request too large) to the block types they should strip from the preceding message
4. **Filter system/attachment messages**—these are UI-only and excluded from the API payload
5. **Merge consecutive same-role messages**—the Anthropic API requires strict alternating user/assistant turns
6. **Ensure tool result pairing**—every `tool_use` block must have a matching `tool_result`; orphaned uses get the `SYNTHETIC_TOOL_RESULT_PLACEHOLDER`

**Why consecutive merging matters:** Because hooks, attachments, and user messages can all add `user`-role content in a single pipeline run, it is entirely possible to end up with two consecutive user messages. The merge pass collapses them into a single user message with concatenated content blocks—satisfying the API's strict alternating-turn requirement.

**Deep dive — The short message ID scheme:**

Claude Code uses 6-character base36 "short IDs" derived from full UUIDs for the snip tool referencing system. These IDs inject into API-bound messages as `[id:abc123]` tags so the model can reference specific past messages by a short token rather than the full 36-character UUID.

```typescript
export function deriveShortMessageId(uuid: string): string {
  // First 10 hex chars of UUID (dashes removed)
  const hex = uuid.replace(/-/g, '').slice(0, 10)
  // Convert to base36, take 6 chars
  return parseInt(hex, 16).toString(36).slice(0, 6)
}
```

The derivation is **deterministic**: the same UUID always yields the same short ID. This is critical—the system injects these tags on every API call, so random derivation would produce different tags each turn, breaking any model references to prior messages.

## 07 Message Taxonomy

The internal message type system has significantly more variants than just "user" and "assistant". Understanding the full taxonomy explains why the pipeline needs so many transformation passes:

**UserMessage**
Role `"user"`. Carries content (string or block array), plus isMeta, origin, permissionMode, imagePasteIds, toolUseResult.

**AssistantMessage**
Role `"assistant"`. Contains content blocks including text, tool_use, and thinking blocks. Carries full usage metrics.

**AttachmentMessage**
Not sent to the API directly. Carries IDE selection, hook outputs, agent mentions, memory headers. Reordered and merged before API dispatch.

**SystemMessage**
UI-only signals with subtypes: `api_error`, `compact_boundary`, `turn_duration`, `informational`. Filtered entirely from API payloads.

**ProgressMessage**
Ephemeral—live updates for tool execution progress. Never stored long-term; filtered before API calls.

**TombstoneMessage**
Deletion marker indicating a message as deleted from the conversation. Processed during compact/rewind operations.

## 08 The Memory Correction Hint Pattern

One subtle but instructive example of how the pipeline injects model guidance is the memory correction hint. When auto-memory is enabled and the GrowthBook feature gate `tengu_amber_prism` is active, rejection and cancellation messages get a postscript appended:

```typescript
const MEMORY_CORRECTION_HINT =
  "\n\nNote: The user's next message may contain a correction or preference. "
  + "Pay close attention — if they explain what went wrong or how they'd "
  + "prefer you to work, consider saving that to memory for future sessions."

export function withMemoryCorrectionHint(message: string): string {
  if (isAutoMemoryEnabled() && getFeatureValue_CACHED('tengu_amber_prism', false)) {
    return message + MEMORY_CORRECTION_HINT
  }
  return message
}
```

This pattern demonstrates how the pipeline uses message content itself as a mechanism to steer model behavior—no special API field needed. The hint only reaches the model via normal text content of a tool result block.

## 09 Query Profiling Checkpoints

Throughout the pipeline are calls to `queryCheckpoint(label)`. These performance breadcrumbs record wall-clock timestamps at key moments. The labels provide an exact map of where time is spent:

- query_process_user_input_start
- query_process_user_input_base_start
- query_image_processing_start / _end
- query_pasted_image_processing_start / _end
- query_attachment_loading_start / _end
- query_hooks_start / _end
- query_process_user_input_base_end
- query_file_history_snapshot_start / _end
- query_process_user_input_end

**Performance insight:** The checkpoint names reveal the three biggest potential latency sources: image processing (resize + base64 encode), attachment loading (IDE selection + todo list + git diff), and hook execution (external process calls). All three run async but not all parallelize—pasted images use `Promise.all` but attachment loading is sequential within `getAttachmentMessages`.

## Key Takeaways

- Every submission normalizes through a single `QueuedCommand` shape so both the direct and queue-processor paths share identical execution logic.
- The pipeline has four distinct mode branches: bash, slash command, ultraplan keyword (hidden), and plain text prompt.
- Images resize _twice_—once during inline-block normalization and again when processing pasted UI clipboard content—both times in parallel via `Promise.all`.
- `UserPromptSubmit` hooks run after input classification but before the API call, giving external processes the ability to block, annotate, or stop any prompt.
- `createUserMessage` is the single factory for all user-role messages; its `isMeta` flag is the mechanism that keeps system-generated context out of the visible transcript.
- `normalizeMessagesForAPI` is the final gatekeeper—it strips virtual messages, merges consecutive same-role turns, and ensures every `tool_use` has a paired `tool_result`.
- Synthetic message constants (`INTERRUPT_MESSAGE`, `CANCEL_MESSAGE`, etc.) double as training data guards—the HFI system rejects payloads containing `SYNTHETIC_TOOL_RESULT_PLACEHOLDER`.

## Knowledge Check

**1. What does `handlePromptSubmit` do when `queryGuard.isActive` is `true` and a user submits a normal prompt?**

A) Executes the prompt immediately, aborting the running query
B) Calls `enqueue()` to push the command into the `commandQueue`
C) Silently drops the input
D) Saves it to `fileHistory` and waits for the guard to clear

**Answer: B** — When the guard is active, input enters the commandQueue via enqueue(). The queue processor picks it up when the current turn completes.

**2. Why does the ultraplan keyword detection run against `preExpansionInput` rather than the final expanded input?**

A) `preExpansionInput` is shorter and faster to search
B) Expanded inputs may contain binary image data that breaks regex
C) To prevent pasted content from accidentally triggering the ultraplan flow
D) Because `finalInput` is not yet available at detection time

**Answer: C** — The reason is security/correctness: pasted text could accidentally contain the keyword, which would silently route the prompt to ultraplan.

**3. What is the purpose of `SYNTHETIC_TOOL_RESULT_PLACEHOLDER` being exported from `messages.ts`?**

A) So the HFI submission path can detect and reject payloads with fake tool results that would poison training data
B) So the UI can display a special indicator when a tool result is missing
C) To satisfy TypeScript's strict mode requirement for exhaustive switch cases
D) As a sentinel value for the compact/summarization system to detect old turns

**Answer: A** — The export exists specifically as a training data safety valve for the HFI (human feedback interface) submission pipeline.

**4. At what point in the pipeline are `UserPromptSubmit` hooks executed?**

A) Before any input validation, in `handlePromptSubmit`
B) Inside `processTextPrompt`, after `createUserMessage`
C) After `normalizeMessagesForAPI`, just before the SDK call
D) After `processUserInputBase` returns with `shouldQuery: true`, inside `processUserInput`

**Answer: D** — Hooks run in processUserInput, after processUserInputBase has built the messages but before the function returns to executeUserInput.

**5. What does `isMeta: true` on a `UserMessage` mean?**

A) The message is only shown in the UI, never sent to the API
B) The message is sent to the model but hidden from the visible transcript
C) The message came from a remote bridge client
D) The message was generated by a tool call rather than the user

**Answer: B** — isMeta messages are sent to the model (they appear in the API payload) but are filtered from the UI transcript. Used for image metadata, scheduled task prompts, etc.

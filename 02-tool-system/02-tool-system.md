# mdENG — Lesson 02 — The Tool System | Complete Content

## Overview

Every capability Claude Code exposes to the model — reading files, running bash commands, searching the web, calling MCP servers — is a **Tool**. The tool system is the bridge between the AI's natural-language reasoning and concrete side effects on your machine.

This lesson dissects that bridge: how a tool is defined, how it is registered and filtered, how the orchestration layer decides concurrency, how permissions gate execution, and how results stream back to the model.

**Files covered:** `Tool.ts`, `tools.ts`, `tools/utils.ts`, `services/tools/toolOrchestration.ts`, `services/tools/toolExecution.ts`, `services/tools/StreamingToolExecutor.ts`

## Architecture: From Interface to Result

The full lifecycle of a single tool invocation follows this path:

1. **Tool Interface** (Tool.ts) — name, inputSchema, call, checkPermissions, isConcurrencySafe, isReadOnly, isDestructive
2. **Registration** (tools.ts) — getAllBaseTools() → getTools() → assembleToolPool() with feature flags, env vars, deny-rule filtering
3. **Routing / Dispatch** (toolExecution.ts: runToolUse) — findToolByName(), alias fallback, abort-before-start check
4. **Input Validation** (toolExecution.ts) — Zod safeParse → schema errors, tool.validateInput() → semantic errors, backfillObservableInput cloning
5. **Permission Gate** (toolExecution.ts: checkPermissionsAndCallTool) — PreToolUse hooks → hookPermissionResult, canUseTool(), alwaysAllow / alwaysDeny rules, Interactive prompt / auto-classifier
6. **Execution** (tool.call) — async generator, onProgress callback, ToolResult<Output> + contextModifier, PostToolUse hooks
7. **Result Processing** (toolExecution.ts) — mapToolResultToToolResultBlockParam, processToolResultBlock (size budget), yield MessageUpdateLazy
8. **Orchestration** (toolOrchestration.ts / StreamingToolExecutor) — partitionToolCalls, concurrent vs serial, sibling-abort on Bash error, in-order result emission

## 1. The Tool Interface (Tool.ts)

The `Tool<Input, Output, P>` type is a **protocol contract** — a TypeScript structural type every tool must satisfy. It is generic over three parameters: the Zod input schema, the output type, and the progress event shape.

### Core required members

```typescript
export type Tool<
  Input extends AnyObject,
  Output,
  P extends ToolProgressData
> = {
  name: string                 // primary identifier the model uses
  aliases?: string[]          // legacy names for backward compat
  inputSchema: Input          // Zod schema — source of truth for validation
  maxResultSizeChars: number  // overflow → persist to disk

  call(
    args: z.infer<Input>,
    context: ToolUseContext,
    canUseTool: CanUseToolFn,
    parentMessage: AssistantMessage,
    onProgress?: ToolCallProgress<P>
  ): Promise<ToolResult<Output>>

  checkPermissions(
    input: z.infer<Input>,
    context: ToolUseContext
  ): Promise<PermissionResult>

  isConcurrencySafe(input: z.infer<Input>): boolean
  isReadOnly(input: z.infer<Input>): boolean
  isDestructive?(input: z.infer<Input>): boolean
}
```

### buildTool() — the factory function

Rather than making authors supply every method, `buildTool()` merges a `ToolDef` (partial) with safe defaults. The defaults are **fail-closed**: assume writes, assume not concurrency safe, deny nothing by default (let the general permission system decide).

```typescript
// Defaults — applied when a ToolDef omits the key
const TOOL_DEFAULTS = {
  isEnabled:         () => true,
  isConcurrencySafe: () => false,   // conservative: assume state mutation
  isReadOnly:        () => false,
  isDestructive:     () => false,
  // Defer to the general permission system by default
  checkPermissions:  (input) =>
    Promise.resolve({ behavior: 'allow', updatedInput: input }),
  toAutoClassifierInput: () => '',  // skip security classifier
  userFacingName:    () => '',
}

export function buildTool<D extends AnyToolDef>(def: D): BuiltTool<D> {
  return {
    ...TOOL_DEFAULTS,
    userFacingName: () => def.name, // sensible fallback
    ...def,
  } as BuiltTool<D>
}
```

**Key insight:** The TypeScript magic in `BuiltTool<D>` mirrors the runtime spread at the type level, so the return type is as narrow as possible — preserving literal types from the definition.

### ToolResult and contextModifier

A tool's `call()` returns a `ToolResult<T>`:

```typescript
export type ToolResult<T> = {
  data: T
  newMessages?: (UserMessage | AssistantMessage | ...)[]
  // Only honored for non-concurrency-safe tools
  contextModifier?: (context: ToolUseContext) => ToolUseContext
}
```

The `contextModifier` is how a tool (e.g., `EnterPlanMode`) mutates shared state without reaching into global variables. It is applied serially after the tool completes. Concurrent tools _cannot_ use `contextModifier` — the comment in `StreamingToolExecutor` explicitly notes this as a known limitation.

### Notable optional methods: UI & classifier

The interface carries a large surface area for rendering and security integration:

*   `renderToolUseMessage()` — React node shown while streaming tool input
*   `renderToolResultMessage()` — React node for the result in transcript
*   `renderGroupedToolUse()` — batch rendering when multiple same-type tools run together
*   `toAutoClassifierInput()` — compact representation for the security classifier; return `''` to skip
*   `extractSearchText()` — for transcript search indexing; must match what renders or you get phantom hits
*   `interruptBehavior()` — `'cancel'` or `'block'`: what happens when the user types while this tool runs
*   `shouldDefer` / `alwaysLoad` — ToolSearch deferred loading flags

## 2. Registration (tools.ts)

`tools.ts` is the **source of truth** for which tools exist. It implements a three-tier assembly pipeline.

### getAllBaseTools() — the exhaustive catalog

Returns every tool that _could_ be available in the current build. Feature flags and environment variables gate conditional tools at module load time using Bun's `feature()` dead-code elimination:

```typescript
const REPLTool = process.env.USER_TYPE === 'ant'
  ? require('./tools/REPLTool/REPLTool.js').REPLTool
  : null

const SleepTool = feature('PROACTIVE') || feature('KAIROS')
  ? require('./tools/SleepTool/SleepTool.js').SleepTool
  : null

export function getAllBaseTools(): Tools {
  return [
    AgentTool, TaskOutputTool, BashTool,
    ...(hasEmbeddedSearchTools() ? [] : [GlobTool, GrepTool]),
    FileReadTool, FileEditTool, FileWriteTool,
    WebFetchTool, TodoWriteTool, WebSearchTool,
    // ... 30+ more tools, conditionally included
    ...(isToolSearchEnabledOptimistic() ? [ToolSearchTool] : []),
  ]
}
```

**Cache stability note:** This list is kept in sync with a Statsig dynamic config so that the system prompt tool list is stable across users — enabling server-side prompt caching.

### getTools() — filtered by context

Applies mode-specific filtering on top of `getAllBaseTools()`:

1.  **Simple mode** (`CLAUDE_CODE_SIMPLE`) — only Bash, Read, Edit
2.  **REPL mode** — hides primitive tools; they live inside the REPL VM
3.  **Deny rules** — filters tools matching `alwaysDenyRules` in the permission context
4.  **`isEnabled()`** — each tool can veto itself

### assembleToolPool() — built-ins + MCP, sorted for cache stability

```typescript
export function assembleToolPool(
  permissionContext: ToolPermissionContext,
  mcpTools: Tools,
): Tools {
  const builtInTools = getTools(permissionContext)
  const allowedMcpTools = filterToolsByDenyRules(mcpTools, permissionContext)

  // Built-ins sorted alphabetically as a prefix, then MCP tools sorted alphabetically.
  // Keeps a stable cache breakpoint between the two groups.
  const byName = (a, b) => a.name.localeCompare(b.name)
  return uniqBy(
    [...builtInTools].sort(byName).concat(allowedMcpTools.sort(byName)),
    'name',
  )
}
```

Sorting built-ins and MCP tools as two separate alphabetical groups preserves the server's cache breakpoint. Interleaving them would bust the cache whenever an MCP tool sorts between built-ins.

## 3. Orchestration (toolOrchestration.ts)

When a model response contains multiple `tool_use` blocks, the orchestrator decides **which run concurrently and which run serially** using `partitionToolCalls()`.

### Partitioning algorithm

The rule is simple but powerful:

*   Consecutive tools where `isConcurrencySafe(input) === true` are batched together and run in parallel.
*   Any non-safe tool breaks the batch and runs alone serially.
*   A try/catch wraps `isConcurrencySafe()` — parse failures default to `false` (conservative).

```typescript
// Simplified from partitionToolCalls()
for (const toolUse of toolUseMessages) {
  const safe = isConcurrencySafe(toolUse)
  if (safe && lastBatch?.isConcurrencySafe) {
    lastBatch.blocks.push(toolUse)       // extend parallel group
  } else {
    acc.push({ isConcurrencySafe: safe, blocks: [toolUse] })
  }
}
```

Concurrent batches use the `all()` async-generator combinator with a concurrency ceiling from `CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY` (default 10).

### Context mutation: the contextModifier dance

Non-safe tools may return a `contextModifier` to mutate `ToolUseContext` (e.g., change the permission mode). Serial tools apply the modifier immediately, before the next tool runs. Concurrent tools queue their modifiers and apply them all after the batch completes:

```typescript
// Serial: apply immediately so next tool sees updated context
if (update.contextModifier) {
  currentContext = update.contextModifier.modifyContext(currentContext)
}

// Concurrent: queue, apply after batch
queuedContextModifiers[toolUseID].push(modifyContext)
// ... after all concurrent tools complete:
for (const modifier of modifiers) {
  currentContext = modifier(currentContext)
}
```

## 4. Streaming Execution (StreamingToolExecutor.ts)

`StreamingToolExecutor` is the _real-time_ variant: it starts executing tools as their blocks stream in from the API, before the model's full response has finished. This is the class used in production streaming mode.

### Tool lifecycle states

Each tool tracks a `ToolStatus`:

```typescript
type ToolStatus = 'queued' | 'executing' | 'completed' | 'yielded'
```

*   **queued** — block received, waiting for concurrency slot
*   **executing** — `runToolUse()` generator is being consumed
*   **completed** — results collected, not yet emitted to caller
*   **yielded** — emitted in order, done

Progress messages (`type: 'progress'`) are stored in `pendingProgress` and emitted _immediately_ out-of-order — they don't need to wait for the result.

### Concurrency guard: canExecuteTool()

```typescript
private canExecuteTool(isConcurrencySafe: boolean): boolean {
  const executing = this.tools.filter(t => t.status === 'executing')
  return (
    executing.length === 0 ||
    (isConcurrencySafe && executing.every(t => t.isConcurrencySafe))
  )
}
```

A non-safe tool must wait for all executing tools to finish. A safe tool can only join if all currently executing tools are also safe.

### Sibling abort: Bash errors cascade

The executor holds a `siblingAbortController` — a child of the main abort controller. When a **Bash tool** produces an error result, it aborts siblings:

```typescript
if (isErrorResult && tool.block.name === BASH_TOOL_NAME) {
  this.hasErrored = true
  this.erroredToolDescription = getToolDescription(tool)
  this.siblingAbortController.abort('sibling_error')
}
```

Only Bash errors cascade. Read/WebFetch/etc are treated as independent — one failure does not nuke parallel reads.

The per-tool `toolAbortController` bubbles non-sibling aborts up to the main query controller — critical for ExitPlanMode's "clear context + auto" flow.

### In-order result emission with getRemainingResults()

Even though tools execute concurrently, _results must be emitted in the order the model requested them_ (the model's tool\_result messages are paired by ID). The executor achieves this by iterating `this.tools` in insertion order and yielding only when the head tool is `completed`:

```typescript
for (const tool of this.tools) {
  // Progress always goes through immediately
  while (tool.pendingProgress.length > 0) {
    yield { message: tool.pendingProgress.shift()! }
  }
  if (tool.status === 'completed' && tool.results) {
    tool.status = 'yielded'
    for (const msg of tool.results) yield { message: msg }
  } else if (tool.status === 'executing' && !tool.isConcurrencySafe) {
    break  // head is a non-safe executing tool — must wait
  }
}
```

## 5. Tool Execution Pipeline (toolExecution.ts)

`runToolUse()` is the central dispatch function. It handles unknown tools, pre-abort checks, and delegates to `streamedCheckPermissionsAndCallTool()` which wraps the async permission+execution flow in a `Stream` to multiplex progress and final results.

### The full checkPermissionsAndCallTool() flow

1.  **Zod validation** — `inputSchema.safeParse(input)`. Failure returns an `InputValidationError` tool result immediately. Includes a hint for deferred tools whose schema wasn't sent.
2.  **Semantic validation** — `tool.validateInput()`. Custom per-tool checks (path traversal, file size limits, etc.).
3.  **Speculative classifier** — Bash commands speculatively start the allow-classifier _before_ hooks run, so the classifier result is ready by the time the permission dialog might need it.
4.  **backfillObservableInput** — creates a shallow clone and adds legacy/derived fields. The clone is what hooks and canUseTool see; the original (API-bound) input is never mutated.
5.  **PreToolUse hooks** — async generators; can yield progress, update input, set a permission result, or stop execution entirely.
6.  **canUseTool()** — the main permission gate. Checks always-allow/always-deny rules, the auto-classifier, and optionally shows the interactive permission prompt.
7.  **tool.call()** — actual execution with the progress callback.
8.  **PostToolUse hooks** — run after the tool completes.
9.  **Result serialization** — `mapToolResultToToolResultBlockParam()` + size-budget processing.

### Input mutation safety: backfillObservableInput

```typescript
// A shallow clone is made for hooks/canUseTool to observe
const backfilledClone =
  tool.backfillObservableInput && processedInput !== null
    ? ({ ...processedInput } as typeof processedInput)
    : null
if (backfilledClone) {
  tool.backfillObservableInput!(backfilledClone as Record<string, unknown>)
  processedInput = backfilledClone
}
```

The original `parsedInput.data` goes to `tool.call()`. Mutation of the original would alter transcript serialization and break VCR fixture hashes in tests.

### Defense-in-depth: \_simulatedSedEdit stripping

The Bash tool has an internal `_simulatedSedEdit` field used by the permission system after user approval. If the model somehow supplies this in its output, the code strips it before execution:

```typescript
if (tool.name === BASH_TOOL_NAME && '_simulatedSedEdit' in processedInput) {
  const { _simulatedSedEdit: _, ...rest } = processedInput
  processedInput = rest  // field stripped, execution proceeds safely
}
```

This is a defense-in-depth measure even though Zod's `strictObject` should already reject the field.

### Progress + result multiplexing via Stream

`streamedCheckPermissionsAndCallTool()` bridges the callback-based progress API and the generator-based result API into a single `AsyncIterable<MessageUpdateLazy>`:

```typescript
const stream = new Stream<MessageUpdateLazy>()

checkPermissionsAndCallTool(..., progress => {
  // Progress callback → enqueue progress message to stream
  stream.enqueue({ message: createProgressMessage(...) })
})
  .then(results => {
    for (const r of results) stream.enqueue(r)
  })
  .finally(() => stream.done())

return stream  // AsyncIterable that yields progress then results
```

## 6. The Permission Context

The `ToolPermissionContext` (wrapped in `DeepImmutable`) flows through the entire system. It drives both registration-time filtering and runtime permission checks.

### Structure

```typescript
export type ToolPermissionContext = DeepImmutable<{
  mode: PermissionMode             // 'default' | 'plan' | 'bypassPermissions' | ...
  additionalWorkingDirectories: Map<string, AdditionalWorkingDirectory>
  alwaysAllowRules: ToolPermissionRulesBySource
  alwaysDenyRules:  ToolPermissionRulesBySource
  alwaysAskRules:   ToolPermissionRulesBySource
  isBypassPermissionsModeAvailable: boolean
  shouldAvoidPermissionPrompts?: boolean  // background agents: auto-deny
  awaitAutomatedChecksBeforeDialog?: boolean
}>
```

`DeepImmutable` prevents any code path from accidentally mutating permissions in-place. The only way to change the context is via a `contextModifier` returned from `ToolResult`.

### filterToolsByDenyRules()

Applied both at registration time (tool list visible to model) and at MCP tool assembly time. Uses `getDenyRuleForTool()` which understands MCP server-prefix rules like `mcp__server` that blanket-deny all tools from a server:

```typescript
export function filterToolsByDenyRules(
  tools: readonly T[],
  permissionContext: ToolPermissionContext
): T[] {
  return tools.filter(tool => !getDenyRuleForTool(permissionContext, tool))
}
```

## Key Takeaways

1. **Protocol, not class hierarchy.** `Tool` is a TypeScript structural type. Any object satisfying the interface is a tool — built-in, MCP, or dynamically generated. `buildTool()` fills in safe fail-closed defaults so authors only override what they need.

2. **Three-tier assembly.** `getAllBaseTools()` → `getTools()` → `assembleToolPool()`. Feature flags gate tools at load time; deny rules filter them before the model sees them; `isEnabled()` is the final veto. MCP tools are sorted into a separate alphabetical suffix to preserve prompt-cache stability.

3. **Concurrency is data-driven.** `isConcurrencySafe(input)` is called per-tool-call, not per-tool-type. A Bash tool running `ls` could be safe while one running `rm -rf` is not. The orchestrator partitions tool blocks into concurrent/serial batches at runtime.

4. **Immutable context, functional mutations.** `ToolPermissionContext` is `DeepImmutable`. State changes happen via `contextModifier` functions returned from `ToolResult`, applied after tool completion. This prevents accidental cross-tool state contamination.

5. **Input mutation is carefully controlled.** The code maintains three distinct copies of input: the API-bound original (for cache), the backfilled observable clone (for hooks/canUseTool), and the potentially-hook-updated call input. Each boundary is intentional and documented.

6. **Bash is special.** Only Bash errors cascade to sibling tools via `siblingAbortController`. Bash speculatively starts the security classifier before hooks run. Bash has the `_simulatedSedEdit` internal field that is defense-stripped. Bash's implicit dependency chains justify treating it differently from purely-read tools.

## Quiz — 5 Questions

### Question 1: What does `buildTool()` default `isConcurrencySafe` to, and why?

- `true` — most tools just read data
- `false` — assume state mutation (fail-closed)
- `true` — parallel execution is faster
- It throws an error if not provided

**Correct Answer:** `false` — assume state mutation (fail-closed)

**Explanation:** Correct. The default is `false` to assume state mutation and fail closed. Parallel execution requires explicit opt-in.

---

### Question 2: In `assembleToolPool()`, why are built-in tools and MCP tools sorted as two separate alphabetical groups rather than a single flat sort?

- Built-in tools must always come first alphabetically
- MCP tools are slower to load
- Interleaving would invalidate the server-side prompt cache breakpoint between the two groups
- MCP tools aren't stable enough to sort reliably

**Correct Answer:** Interleaving would invalidate the server-side prompt cache breakpoint between the two groups

**Explanation:** Correct. The server places a cache breakpoint after the last built-in tool. A flat sort would interleave MCP tools between built-ins and bust all downstream cache keys whenever MCP tools are added or renamed.

---

### Question 3: In `StreamingToolExecutor`, only Bash tool errors abort sibling tools. Which code comment explains the reasoning?

- Bash is the only tool that can throw exceptions
- Bash commands often have implicit dependency chains; Read/WebFetch/etc are independent — one failure shouldn't nuke the rest
- Bash is always run last in the batch
- Sibling abort is only safe for synchronous tools

**Correct Answer:** Bash commands often have implicit dependency chains; Read/WebFetch/etc are independent — one failure shouldn't nuke the rest

**Explanation:** Correct. From the source comment: "Bash commands often have implicit dependency chains (e.g. mkdir fails → subsequent commands pointless). Read/WebFetch/etc are independent — one failure shouldn't nuke the rest."

---

### Question 4: What is the purpose of `backfillObservableInput()` working on a _clone_ rather than mutating `parsedInput.data` directly?

- Cloning is required by the Zod library
- To avoid type errors from the TypeScript compiler
- Mutating the original alters the serialized transcript and breaks VCR fixture hashes in tests
- The original is needed for the PostToolUse hook

**Correct Answer:** Mutating the original alters the serialized transcript and breaks VCR fixture hashes in tests

**Explanation:** Correct. The comment in `toolExecution.ts` explicitly states: "tool results embed the input path verbatim... changing it alters the serialized transcript and VCR fixture hashes."

---

### Question 5: A tool returns a `contextModifier` in its `ToolResult`. When is this modifier applied if the tool ran as part of a _concurrent_ batch?

- Immediately when the tool's `call()` resolves
- Only at the start of the next query turn
- After all concurrent tools in the batch have completed
- Concurrent tools cannot use contextModifier — it is ignored

**Correct Answer:** After all concurrent tools in the batch have completed

**Explanation:** Correct. In `toolOrchestration.ts`, modifiers from a concurrent batch are queued in `queuedContextModifiers` and applied in insertion order only after all blocks in the batch have completed.

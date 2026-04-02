# State Management — Lesson 12: Complete Content

## Overview

Claude Code implements its own store in 35 lines of TypeScript without relying on Redux, Zustand, or Context. The architecture spans three layers: a primitive framework-agnostic store, domain-specific AppState and factory, and React integration through hooks and providers.

## Source Files Architecture

Six files comprise the state system:
- `state/store.ts` — Generic 35-line store
- `state/AppStateStore.ts` — Full state shape (400+ fields) and factory
- `state/AppState.tsx` — React provider and hooks
- `state/onChangeAppState.ts` — Diff-observer for side effects
- `state/selectors.ts` — Pure functions over AppState slices
- `state/teammateViewHelpers.ts` — Stateful feature updaters

## The createStore Pattern

The store primitive provides three operations:

```typescript
type Listener = () => void
type OnChange<T> = (args: { newState: T; oldState: T }) => void

export type Store<T> = {
  getState: () => T
  setState: (updater: (prev: T) => T) => void
  subscribe: (listener: Listener) => () => void
}

export function createStore<T>(
  initialState: T,
  onChange?: OnChange<T>,
): Store<T> {
  let state = initialState
  const listeners = new Set<Listener>()

  return {
    getState: () => state,

    setState: (updater) => {
      const prev = state
      const next = updater(prev)
      if (Object.is(next, prev)) return   // bail if no change
      state = next
      onChange?.({ newState: next, oldState: prev })  // side-effect hook
      for (const listener of listeners) listener()  // notify React
    },

    subscribe: (listener) => {
      listeners.add(listener)
      return () => listeners.delete(listener)  // unsubscribe
    },
  }
}
```

### Why Not useState / useReducer?

React hooks bind state lifetime to component trees. Claude Code needs state accessible from non-React code: headless mode, SDK print layer, per-process teammate sessions, and the `onChangeAppState` side-effect chain. A plain JavaScript object with a `Set` of listeners avoids this constraint.

### Why Not Zustand / Jotai?

Zero bundle dependency. The needed interface—`getState`, `setState`, `subscribe`—maps exactly to `useSyncExternalStore` requirements with nothing left to add.

### Key Invariant

`setState` takes an updater function `(prev) => next`, never a partial object. This enforces immutability at the call site: callers must spread previous state and return a new reference. `Object.is` equality checking means re-renders only fire when the reference changes.

### useSyncExternalStore Integration

React 18's `useSyncExternalStore` accepts three arguments: `subscribe`, `getSnapshot`, and optional `getServerSnapshot`. The contract:

- **subscribe** — register a callback, return an unsubscribe function. Matches `store.subscribe` exactly.
- **getSnapshot** — return the current value synchronously. This wraps `store.getState()` with a selector.

React calls `getSnapshot` during render to read the current value, and invokes the subscribed callback whenever the store updates—triggering re-render only if the snapshot changed.

```typescript
// From AppState.tsx — the full useAppState implementation:
export function useAppState(selector) {
  const store = useAppStore()
  const get = () => selector(store.getState())
  return useSyncExternalStore(store.subscribe, get, get)
}
```

The `get` closure recreates on every render if `selector` changes identity. The compiled output (React Compiler's `_c` memo cache) caches it by `[selector, store]`. Inline arrow selectors defeat the cache on every render.

## The AppState Shape

`AppState` defined in `state/AppStateStore.ts` contains over 90 distinct fields with clear internal structure. The type is `DeepImmutable<{...}>` for the serializable portion, with certain fields escaping the immutability wrapper—function types, `Map`, `Set`—explicitly listed after the `&`.

### Escape Hatch

`tasks`, `agentNameRegistry`, `sessionHooks`, `activeOverlays`, and `replContext` are not wrapped in `DeepImmutable` because they contain function types that TypeScript's recursive readonly transform doesn't handle. Treat them as logically immutable (always spread before mutating) even though the type permits mutation.

### Six Logical Categories

**Session Core**
- Model & Settings: settings, verbose, mainLoopModel, mainLoopModelForSession, thinkingEnabled, effortValue, fastMode, kairosEnabled, agent, authVersion

**UI State**
- View & Navigation: expandedView, isBriefOnly, footerSelection, spinnerTip, activeOverlays, statusLineText, viewSelectionMode, coordinatorTaskIndex

**Permissions**
- Tool & Denial: toolPermissionContext (mode, bypass flags), denialTracking, initialMessage (mode override), pendingPlanVerification

**Agent & Tasks**
- Concurrency: tasks (keyed by taskId), agentNameRegistry (name → AgentId), foregroundedTaskId, viewingAgentTaskId, teamContext, standaloneAgentContext

**Remote & Bridge**
- Connectivity: remoteSessionUrl, remoteConnectionStatus, replBridgeEnabled/Connected/Active, ultraplanSessionUrl, isUltraplanMode, workerSandboxPermissions

**Subsystem State**
- Features: mcp (clients, tools, commands, resources), plugins (enabled, disabled, installationStatus), speculation, promptSuggestion, notifications, elicitation, todos, inbox, tungstenActive* (tmux panel), bagel* (browser), computerUseMcpState, replContext, fileHistory

### Selected Field Reference

| Field | Type | Purpose |
|-------|------|---------|
| settings | SettingsJson | Full settings.json contents — read by nearly every subsystem |
| mainLoopModel | ModelSetting | Active model override; null = use default. Written to settings on change via onChangeAppState |
| toolPermissionContext | ToolPermissionContext | Current permission mode (default/plan/auto/yolo) plus bypass availability flags |
| tasks | { [taskId]: TaskState } | Live state for all in-flight agent tasks (local_agent, in_process_teammate, etc.) |
| agentNameRegistry | Map<string, AgentId> | Name → AgentId routing table populated by Agent tool; latest wins on collision |
| speculation | SpeculationState | Idle or active predictive completion with boundary, abort, and pipelined suggestion state |
| expandedView | 'none' \| 'tasks' \| 'teammates' | Controls which panel is expanded; persisted to globalConfig via onChangeAppState |
| notifications | { current, queue } | Priority-queued notification system; useNotifications manages transitions |
| replBridgeEnabled | boolean | Desired state of always-on bridge (controlled by /config or footer toggle) |
| mcp.pluginReconnectKey | number | Monotonically incremented by /reload-plugins; effects watch this as a dependency trigger |
| initialMessage | { message, mode, ... } \| null | Set to trigger a REPL query programmatically (CLI args, plan mode exit) |
| activeOverlays | ReadonlySet<string> | Registry of open Select dialogs — Escape key checks this before acting |
| fileHistory | FileHistoryState | Snapshots + tracked files for undo/rewind support |
| attribution | AttributionState | Commit authorship tracking for git operations |
| tungstenActiveSession | { sessionName, socketName, target } \| undef | Active tmux integration session (ant-only) |
| computerUseMcpState | { allowedApps, grantFlags, ... } \| undef | Per-session computer-use allowlist and display state (chicago MCP) |

### DeepImmutable and the & Escape Hatch

The `AppState` type declaration is structured as:

```typescript
export type AppState = DeepImmutable<{
  settings: SettingsJson
  verbose: boolean
  // ... ~60 serializable fields ...
}> & {
  // Excluded from DeepImmutable — contain function types
  tasks: { [taskId: string]: TaskState }
  agentNameRegistry: Map<string, AgentId>
  mcp: { clients: MCPServerConnection[]; /* ... */ }
  // ...
}
```

`DeepImmutable` is a recursive conditional type that wraps every nested object in `Readonly<>`. The intersection (`&`) appends the mutable-typed fields back without the immutability wrapper. The TypeScript compiler accepts this because intersection merges the property sets: the `Readonly` version of each field from the first half is effectively overridden by the raw type from the second half when the field names collide — but the intent is to list them separately so they're structurally visible.

In practice, nothing stops a caller from mutating `tasks[id].someField = x` at the JavaScript runtime level. Discipline comes from team convention and the updater pattern in `setState`.

### SpeculationState — the Most Complex Field

`speculation` is a discriminated union with two arms:

```typescript
type SpeculationState =
  | { status: 'idle' }
  | {
      status: 'active'
      id: string
      abort: () => void
      startTime: number
      messagesRef: { current: Message[] }     // mutable ref — no array copy per msg
      writtenPathsRef: { current: Set<string> } // relative paths in overlay
      boundary: CompletionBoundary | null
      suggestionLength: number
      toolUseCount: number
      isPipelined: boolean
      contextRef: { current: REPLHookContext }
      pipelinedSuggestion?: { text: string; promptId: ...; generationRequestId: ... } | null
    }
```

The `messagesRef` and `writtenPathsRef` fields are intentionally mutable—they escape the immutability contract so that speculation can append messages to an in-progress prediction without triggering a full store update (and a re-render) for every token. The ref is mutated directly; only state transitions (idle ↔ active) go through `setState`.

## onChangeAppState — the Side-Effect Chokepoint

The second argument to `createStore` is an optional `onChange` callback. Claude Code passes `onChangeAppState` here—a single function that fires on every state transition and diffs relevant fields to drive side effects.

### Why This Matters

Before this pattern was introduced, permission-mode changes were relayed to the remote dashboard (CCR) by only 2 of 8+ mutation paths. The other 6 paths mutated AppState silently, leaving the web UI stale. Centralizing the diff here means any `setState` call that changes the mode automatically syncs—with zero changes to individual call sites.

### The Current Diff Blocks

```typescript
export function onChangeAppState({ newState, oldState }) {

  // 1. Permission mode — sync to CCR external_metadata + SDK status stream
  const prevMode = oldState.toolPermissionContext.mode
  const newMode = newState.toolPermissionContext.mode
  if (prevMode !== newMode) {
    const prevExternal = toExternalPermissionMode(prevMode)
    const newExternal  = toExternalPermissionMode(newMode)
    if (prevExternal !== newExternal) {
      // Guard: internal-only modes (bubble, ungated-auto) don't pollute CCR.
      // is_ultraplan_mode set null per RFC 7396 (removes key from JSON patch).
      notifySessionMetadataChanged({ permission_mode: newExternal, ... })
    }
    notifyPermissionModeChanged(newMode)  // SDK channel — passes raw mode
  }

  // 2. mainLoopModel — persist to settings + bootstrap override
  if (newState.mainLoopModel !== oldState.mainLoopModel) {
    if (newState.mainLoopModel === null) {
      updateSettingsForSource('userSettings', { model: undefined })
      setMainLoopModelOverride(null)
    } else {
      updateSettingsForSource('userSettings', { model: newState.mainLoopModel })
      setMainLoopModelOverride(newState.mainLoopModel)
    }
  }

  // 3. expandedView — persist to globalConfig (showExpandedTodos / showSpinnerTree)
  if (newState.expandedView !== oldState.expandedView) {
    saveGlobalConfig(current => ({
      ...current,
      showExpandedTodos: newState.expandedView === 'tasks',
      showSpinnerTree:   newState.expandedView === 'teammates',
    }))
  }

  // 4. verbose — persist to globalConfig
  if (newState.verbose !== oldState.verbose) {
    saveGlobalConfig(current => ({ ...current, verbose: newState.verbose }))
  }

  // 5. tungstenPanelVisible — ant-only, persist to globalConfig
  if (process.env.USER_TYPE === 'ant' && newState.tungstenPanelVisible !== oldState.tungstenPanelVisible) {
    saveGlobalConfig(current => ({ ...current, tungstenPanelVisible: newState.tungstenPanelVisible }))
  }

  // 6. settings — clear auth caches + re-apply env vars
  if (newState.settings !== oldState.settings) {
    clearApiKeyHelperCache()
    clearAwsCredentialsCache()
    clearGcpCredentialsCache()
    if (newState.settings.env !== oldState.settings.env) {
      applyConfigEnvironmentVariables()
    }
  }
}
```

### The Externalization Guard

Not all internal permission modes have external equivalents. The `toExternalPermissionMode` call collapses internal-only names like `'bubble'` and `'ungated-auto'` to `'default'` before sending to CCR. Without this, the remote dashboard would receive meaningless mode names and potentially cycle on internal transitions (`default → bubble → default`) that are invisible from the outside.

### externalMetadataToAppState — the Inverse

The file also exports `externalMetadataToAppState`, which is the inverse of the permission-mode push: when a worker process restarts and pulls `SessionExternalMetadata` from the CCR session store, it calls this to hydrate `AppState` with the persisted mode.

```typescript
export function externalMetadataToAppState(
  metadata: SessionExternalMetadata
): (prev: AppState) => AppState {
  return prev => ({
    ...prev,
    ...(typeof metadata.permission_mode === 'string'
      ? { toolPermissionContext: {
            ...prev.toolPermissionContext,
            mode: permissionModeFromString(metadata.permission_mode),
          }}
      : {}),
    ...(typeof metadata.is_ultraplan_mode === 'boolean'
      ? { isUltraplanMode: metadata.is_ultraplan_mode }
      : {}),
  })
}
```

Notice it returns an updater function `(prev) => AppState`—it's designed to be passed directly to `store.setState()`. This is the conventional shape for all AppState mutations.

## The React Hooks Layer

`state/AppState.tsx` is the React face of the store. It exports three hooks and one provider:

```typescript
// Read a slice — re-renders only when the selected value changes
const verbose = useAppState(s => s.verbose)
const model   = useAppState(s => s.mainLoopModel)

// Write without subscribing — stable reference, never causes re-renders
const setAppState = useSetAppState()
setAppState(prev => ({ ...prev, verbose: true }))

// Get the raw store — for passing to non-React helpers
const store = useAppStateStore()
doSomethingOutsideReact(store.getState, store.setState)
```

### Selector Rule

Do NOT return new objects or arrays from the selector. `useSyncExternalStore` compares snapshots with `Object.is`. An inline `s => ({ a: s.a, b: s.b })` creates a new object on every render, which triggers an infinite re-render loop. Return a single sub-object reference or a primitive:

```typescript
// Good — returns existing reference
const { text, promptId } = useAppState(s => s.promptSuggestion)

// Bad — new object every render
const { text, promptId } = useAppState(s => ({ text: s.promptSuggestion.text, promptId: s.promptSuggestion.promptId }))
```

### AppStateProvider Setup

`AppStateProvider` creates the store exactly once (via `useState` lazy initializer), wires `onChangeAppState` into it, then applies a one-time fixup if remote settings were loaded before the component mounted:

```typescript
const [store] = useState(() =>
  createStore(
    initialState ?? getDefaultAppState(),
    onChangeAppState,        // side-effect hook wired here
  )
)

// One-time remote-settings fixup on mount
useEffect(() => {
  const { toolPermissionContext } = store.getState()
  if (toolPermissionContext.isBypassPermissionsModeAvailable
      && isBypassPermissionsModeDisabled()) {
    store.setState(prev => ({
      ...prev,
      toolPermissionContext: createDisabledBypassPermissionsContext(prev.toolPermissionContext)
    }))
  }
}, [])
```

## Selectors and Transition Helpers

### selectors.ts — Pure Derivations

`state/selectors.ts` contains functions that derive computed values from `AppState` slices without accessing the store or producing side effects. They accept a `Pick<AppState, ...>` (not the full state) so callers can test them in isolation.

```typescript
/**
 * Get the currently viewed teammate task, if any.
 * Takes a Pick — not the full AppState — so tests don't need the whole object.
 */
export function getViewedTeammateTask(
  appState: Pick<AppState, 'viewingAgentTaskId' | 'tasks'>
): InProcessTeammateTaskState | undefined { ... }

/**
 * Discriminated union — tells input routing exactly where to send a message.
 */
export type ActiveAgentForInput =
  | { type: 'leader' }
  | { type: 'viewed';     task: InProcessTeammateTaskState }
  | { type: 'named_agent'; task: LocalAgentTaskState       }

export function getActiveAgentForInput(appState: AppState): ActiveAgentForInput { ... }
```

### teammateViewHelpers.ts — Colocated State Transitions

More complex state transitions for the teammate-view feature live in `state/teammateViewHelpers.ts`. These are not React hooks—they take `setAppState` as an argument, making them testable and usable from any context.

```typescript
// Enter a teammate's transcript view — retain: true blocks eviction, loads from disk
export function enterTeammateView(
  taskId: string,
  setAppState: (updater: (prev: AppState) => AppState) => void,
): void

// Exit back to leader's view — releases retain, schedules eviction if terminal
export function exitTeammateView(
  setAppState: (updater: (prev: AppState) => AppState) => void,
): void

// Context-sensitive x button: abort if running, dismiss if terminal
export function stopOrDismissAgent(
  taskId: string,
  setAppState: (updater: (prev: AppState) => AppState) => void,
): void
```

### Design Pattern

The `release(task)` helper inside `teammateViewHelpers.ts` is a good example of local helper hygiene: it's not exported (no one outside the file needs it), it's pure (takes a task, returns a task), and it encodes a policy decision in one place — "releasing" a task means `retain: false`, `messages: undefined`, and setting `evictAfter` if the task is in a terminal state.

### Retain/Evict Lifecycle for Agent Tasks

Agent task rows in the background panel follow a retain/evict lifecycle:

- **Stub form**: `retain: false`, `messages: undefined`. Row shows in panel but no transcript is loaded.
- **Retained form**: `retain: true`, messages loaded. Triggered by `enterTeammateView`. Blocks eviction and enables streaming.
- **Eviction pending**: task is terminal, `evictAfter = Date.now() + 30_000`. Row lingers for 30s (PANEL_GRACE_MS) so the user sees it complete, then the filter drops it.
- **Immediate dismiss**: `evictAfter = 0`. Filter hides row immediately. Triggered by the x button on a terminal task.

The `PANEL_GRACE_MS = 30_000` constant is inlined in both `framework.ts` and `teammateViewHelpers.ts` with a comment instructing them to be kept in sync—a deliberate choice to avoid importing across a module boundary that would create a circular dependency through `BackgroundTasksDialog`.

## Full Data Flow Diagram

How a state update moves from a component through the system:

```
Component calls useSetAppState()
  |
  v
returns store.setState
  |
  v
store.setState(updater)
  |
  v
Object.is(next, prev)?
+-- same --> Return (no-op)
+-- changed --> state = next
  |
  v
  onChangeAppState({ newState, oldState })
  +-- permission mode changed? --> notifySessionMetadataChanged, notifyPermissionModeChanged
  +-- mainLoopModel changed? --> updateSettingsForSource, setMainLoopModelOverride
  +-- expandedView changed? --> saveGlobalConfig
  +-- settings changed? --> clearAuthCaches, applyConfigEnvironmentVariables
  +-- for listener of listeners: listener()
    |
    v
    useSyncExternalStore triggers re-render
    |
    v
    selector(store.getState()) compares with Object.is
    +-- changed --> Component re-renders
    +-- same --> Render skipped
```

## Context vs State: Where Context Lives

Not everything in `context/` is React Context in the traditional sense. The directory contains a mix of patterns:

**React Context (thin)**
- modalContext, overlayContext, promptOverlayContext: True React Context — values shared down the component tree, not stored in AppState.

**Hooks over AppState**
- notifications.tsx: Reads/writes `AppState.notifications` via `useAppState` + `useSetAppState`. No local state.

**External Store Pattern**
- fpsMetrics, stats: Own their data outside AppState (perf metrics don't need to be part of the main diff). May use their own store or module-level state.

**Side-Effect Manager**
- mailbox, voice, QueuedMessage: Manage WebSocket or IPC connections. May read AppState but primarily drive effects.

### context.ts is Different

The top-level `context.ts` (not `context/`) is entirely unrelated to React Context. It builds the system prompt injected into each Claude API call: `getSystemContext()` (git status, cache breaker) and `getUserContext()` (CLAUDE.md files, current date). Both are `memoize()`d for the lifetime of the conversation.

## Key Takeaways

- `createStore<T>` is 35 lines of TypeScript that implements exactly the interface `useSyncExternalStore` needs—no library required.
- `setState` takes an updater function, not a partial. Callers always spread the previous state. The `Object.is` bail-out prevents spurious re-renders.
- `AppState` is enormous on purpose—it is the single source of truth for the entire session. The alternative (scattered module singletons) would be harder to test and reset.
- The `AppState` type uses `DeepImmutable<...> & { mutables }` to make the serializable core read-only while keeping function-typed fields usable.
- `onChangeAppState` is the architectural key: centralizing all side effects of state changes in a single diff observer means new mutation paths automatically get the right behavior for free.
- Selectors take a `Pick<AppState, ...>` not the full state—making them testable without constructing a complete default state.
- Transition helpers like `enterTeammateView` receive `setAppState` as an argument—keeping them framework-agnostic and testable outside React.
- The retain/evict lifecycle for agent task rows is entirely managed through AppState fields (`retain`, `evictAfter`) rather than component-local state.

## Quiz

**Question 1**: What does `store.setState` do when the updater returns the exact same reference it received?

A. It still notifies listeners to be safe
B. It returns early — no state update, no side effects, no listeners called
C. It calls onChange but skips the listeners
D. It throws a warning in development mode

**Answer**: B

---

**Question 2**: Why does `AppStateStore.ts` live in a `.ts` file instead of a `.tsx` file?

A. It was originally a plain JavaScript file and never renamed
B. .ts files compile faster than .tsx files
C. Non-React consumers (headless mode, SDK, teammates) can import it without pulling in React as a dependency
D. TypeScript requires store types to be in .ts files

**Answer**: C

---

**Question 3**: What is the danger of writing `useAppState(s => ({ a: s.a, b: s.b }))`?

A. The selector will throw a runtime error
B. useSyncExternalStore will call Object.is on the returned object — a new object is always not-equal, causing the component to re-render on every state update regardless of whether a or b changed
C. It will cause a memory leak because the object is not cleaned up
D. TypeScript will flag it as a compile error

**Answer**: B

---

**Question 4**: Before `onChangeAppState` was introduced, what was the primary problem with permission-mode syncing?

A. Permission mode changes were synchronous instead of async
B. Only 2 of 8+ mutation paths relayed the change to CCR; the rest mutated AppState silently, leaving the web UI out of sync
C. Permission mode was stored in globalConfig instead of AppState
D. The SDK did not expose a permission-mode event

**Answer**: B

---

**Question 5**: What happens in `onChangeAppState` when the permission mode changes from `'default'` to `'bubble'` (an internal-only mode)?

A. Both CCR and the SDK status stream are notified of the mode change
B. Neither CCR nor the SDK are notified — bubble is not an external mode
C. Only the SDK is notified; CCR is skipped because toExternalPermissionMode maps bubble to default (no external change)
D. An error is thrown — bubble is not a valid mode

**Answer**: C

---

**Question 6**: Why does `teammateViewHelpers.ts` inline `PANEL_GRACE_MS = 30_000` instead of importing it from `framework.ts`?

A. framework.ts uses a different unit (seconds) that would need conversion
B. Importing framework.ts creates a circular dependency through BackgroundTasksDialog
C. framework.ts does not export the constant
D. The values are intentionally different to allow separate configuration

**Answer**: B

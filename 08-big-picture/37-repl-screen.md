# The REPL Screen: Main Interaction Loop — Source Deep Dive

## Lesson 37

## 01 What REPL.tsx Actually Does

`screens/REPL.tsx` is the largest and most consequential file in the entire codebase. It is the React component that _is_ the Claude Code session. Everything visible on screen — the conversation history, the spinner, the permission prompts, the input box — is orchestrated here.

The file is compiled from roughly 5,000 lines and exports one function: `REPL(props)`. Despite its size, the file has a clear internal structure that can be read in layers:

**Layer 1 — lines ~526-700: Props & Env Guards**

Type definition for `Props`, feature-flag memos, and mount-time `useEffect` logging.

**Layer 2 — lines ~700-1200: State Declarations**

Every `useState` and `useRef`: messages, abort controller, loading flags, dialog queues, streaming text, input value, screen mode.

**Layer 3 — lines ~1200-2700: Core Callbacks**

`setMessages`, `onCancel`, `getToolUseContext`, `onQueryEvent`, `onQueryImpl`, `onQuery`, `onSubmit`.

**Layer 4 — lines ~2700-4100: Effects & Side Systems**

Session resume, queue processor, notification hooks, keyboard handlers, idle detection, teammate inbox.

**Layer 5 — lines ~4100-4490: Transcript Mode**

The `screen === 'transcript'` early return, virtual-scroll layout, search bar, dump mode.

**Layer 6 — lines ~4490-5005: Main Render**

The primary JSX tree: `FullscreenLayout`, `Messages`, spinner, dialogs, `PromptInput`, keybinding handlers.

## 02 The Turn Lifecycle

Every user interaction flows through a chain of three functions. Understanding the chain is the master key to the entire file.

### onSubmit — The Entry Point

`onSubmit` is where keystrokes become API calls. The function signature reveals its responsibilities:

```javascript
const onSubmit = useCallback(async (
  input: string,
  helpers: PromptInputHelpers,
  speculationAccept?: ActiveSpeculationState,
  options?: { fromKeybinding?: boolean }
) => {
  repinScroll();          // snap to bottom on any submit

  // Fast path: immediate local-jsx commands run NOW even while Claude is busy
  if (shouldTreatAsImmediate) {
    void executeImmediateCommand();
    return;
  }

  // Idle-return gate: if user has been gone 75+ min, show dialog first
  // Add to shell/history, restore stashed prompt, clear input field
  // Route remote mode through WebSocket, not local query

  await awaitPendingHooks();   // block until SessionStart hooks resolve
  await handlePromptSubmit(...); // shell mode, command routing, onQuery
}, [/* ~25 deps */]);
```

**Why the large dep array?**

The comments in the source are explicit: `messages` is intentionally read via `messagesRef.current` inside callbacks (not via the closure) to keep `onSubmit` stable across the ~30 `setMessages` calls per turn. Without this discipline, each streaming delta would recreate `onSubmit`, pinning stale REPL render scopes in memory. A heap analysis found ~9 leaked REPL scopes per turn before this was fixed.

### onQuery — The Concurrency Guard

`onQuery` wraps `onQueryImpl` with a critical state machine: `QueryGuard`. Unlike a simple boolean flag, the guard uses a generation counter so stale `finally` blocks from cancelled queries don't incorrectly update state.

```javascript
const thisGeneration = queryGuard.tryStart();
if (thisGeneration === null) {
  // Already running — extract user text and enqueue it
  newMessages.filter(isUserMessage).forEach(msg => enqueue({ value, mode: 'prompt' }));
  return;
}
try {
  await onQueryImpl(messagesRef.current, newMessages, abortController, ...);
} finally {
  if (queryGuard.end(thisGeneration)) {
    // Only the latest generation cleans up
    resetLoadingState();
    await mrOnTurnComplete(messagesRef.current, aborted);
  }
  // Auto-restore runs OUTSIDE the generation check
  // (forceEnd bumps generation; end() returns false for Esc path)
  if (abortController.signal.reason === 'user-cancel' && !queryGuard.isActive ...) {
    restoreMessageSync(lastUserMsg);
  }
}
```

### onQueryImpl — The API Call

`onQueryImpl` does the actual work: builds the system prompt, calls `query()`, and streams the response through `onQueryEvent`. Key steps:

```javascript
// 1. Haiku title extraction (one-shot, first real user message only)
if (!titleDisabled && !haikuTitleAttemptedRef.current) {
  void generateSessionTitle(text, signal).then(t => setHaikuTitle(t));
}

// 2. Write skill-scoped allowedTools to store BEFORE the API call
store.setState(prev => ({ ...prev, toolPermissionContext: { ...prev.toolPermissionContext,
  alwaysAllowRules: { ...prev.toolPermissionContext.alwaysAllowRules, command: additionalAllowedTools }
}}));

// 3. Build full context — all reads from store.getState() not render closure
const toolUseContext = getToolUseContext(messages, newMessages, abortController, model);

// 4. Parallel async: system prompt + user context + killswitch checks
const [,, defaultSystemPrompt, userContext, systemContext] = await Promise.all([
  checkAndDisableBypassPermissionsIfNeeded(...),
  getSystemPrompt(freshTools, model, workingDirs, mcpClients),
  getUserContext(),
  getSystemContext()
]);

// 5. Stream the query
for await (const event of query({ messages, systemPrompt, canUseTool, toolUseContext, ... })) {
  onQueryEvent(event);
}
```

## 03 Loading State: Three Sources of Truth

One of the subtler design decisions in REPL.tsx is how it tracks "is Claude currently working?" There are three independent sources that can all make the spinner appear:

| Source | Mechanism | When it fires |
|--------|-----------|---------------|
| `isQueryActive` | `useSyncExternalStore(queryGuard.subscribe, queryGuard.getSnapshot)` | Local `onQuery` is running |
| `isExternalLoading` | `useState` + `setIsExternalLoading` | Remote session / SSH / foregrounded background task |
| `hasRunningTeammates` | `useMemo` over `tasks` AppState | Swarm worker agents still executing |

```javascript
const isLoading = isQueryActive || isExternalLoading;
const showSpinner = (!toolJSX || toolJSX.showSpinner === true)
  && toolUseConfirmQueue.length === 0
  && promptQueue.length === 0
  && (isLoading || userInputOnProcessing || hasRunningTeammates || getCommandQueueLength() > 0)
  && !pendingWorkerRequest
  && !onlySleepToolActive
  && (!visibleStreamingText || isBriefOnly);
```

### The Timing Ref Pattern

Elapsed time in the spinner is computed from `loadingStartTimeRef`, not state — so the animation frame can read it without triggering a re-render. The ref is reset inline on the first render where `isQueryActive` becomes true, not inside a `useEffect`. The comment explains why: there was a race where the effect fired after the first spinner render, causing it to show "56 years elapsed" (`Date.now() - 0`).

## 04 The Dialog Priority Queue

When multiple things need the user's attention simultaneously — a permission prompt, an idle-return hint, an IDE onboarding dialog — REPL.tsx resolves conflicts through a single `getFocusedInputDialog()` function that returns a string union of all possible dialog types:

```javascript
function getFocusedInputDialog():
  'message-selector' | 'sandbox-permission' | 'tool-permission' |
  'prompt' | 'worker-sandbox-permission' | 'elicitation' |
  'cost' | 'idle-return' | 'ide-onboarding' | ... | undefined

  // Priority order (highest to lowest):
  if (isMessageSelectorVisible) return 'message-selector';    // always
  if (isPromptInputActive) return undefined;                   // suppress while typing
  if (sandboxPermissionRequestQueue[0]) return 'sandbox-permission';
  // ... permission/interactive dialogs ...
  // ... onboarding dialogs ...
  // ... callouts (effort, remote, LSP rec) ...
  return undefined;
```

The `isPromptInputActive` guard is particularly notable: interrupt dialogs are _suppressed_ while the user is typing. A 1.5-second debounce (`PROMPT_SUPPRESSION_MS = 1500`) resets the flag after the last keystroke. This prevents accidental permission-dismiss when the user is mid-sentence.

### Ordering Constraint

`ScrollKeybindingHandler` must be rendered _before_ `CancelRequestHandler` in the JSX tree. The comment explains: `ctrl+c` with a text selection should copy, not cancel the active task. The scroll handler's `useInput` only stops propagation when a selection exists — without a selection, `ctrl+c` falls through to the cancel handler naturally.

## 05 The Messages Array: Source of Truth

The conversation is stored in a `messages: MessageType[]` state array, but it is _not_ managed with plain `useState`. The wrapper pattern used is the same as Zustand: a ref holds the live value, React state is a render projection:

```javascript
const [messages, rawSetMessages] = useState<MessageType[]>(initialMessages ?? []);
const messagesRef = useRef(messages);

const setMessages = useCallback((action) => {
  const prev = messagesRef.current;
  const next = typeof action === 'function' ? action(messagesRef.current) : action;
  messagesRef.current = next;             // sync update — no await needed
  if (next.length > prev.length && userMessagePendingRef.current) {
    // Track whether the submitted user message has landed yet
    // to control the placeholder text visibility
  }
  rawSetMessages(next);
}, []);
```

Three related mechanisms keep the messages array consistent:

1. **Ephemeral progress replacement** — Sleep and Bash emit progress ticks every second. Rather than appending (which bloats the array to 13,000+ entries), REPL.tsx replaces the previous tick for the same tool use ID in-place.

2. **Compact boundary handling** — When `query()` emits a compact boundary message, the messages array is replaced with just the post-compact messages. In fullscreen mode, the pre-compact messages are kept for scrollback but capped at one compact interval.

3. **Deferred rendering** — `useDeferredValue(messages)` produces `deferredMessages`, which the `Messages` component renders at transition priority. This keeps the input box responsive during streaming. The deferred path is bypassed when streaming text is visible (so the final message appears in the same frame the streaming text clears).

## 06 The toolJSX Overlay System

Tools and slash commands can render custom UI by calling `setToolJSX()`. The REPL tracks two independent overlay slots:

```javascript
const [toolJSX, setToolJSXInternal] = useState<{
  jsx: React.ReactNode | null;
  shouldHidePromptInput: boolean;
  shouldContinueAnimation?: true;
  showSpinner?: boolean;
  isLocalJSXCommand?: boolean;
  isImmediate?: boolean;
} | null>(null);

const localJSXCommandRef = useRef(...); // preserves /btw and similar while Claude streams
```

The `setToolJSX` wrapper enforces an important invariant: **local JSX commands cannot be overwritten by tool updates**. If a user runs `/btw` (which shows an overlay while Claude keeps processing), subsequent tool updates are silently ignored until the user explicitly dismisses it with `clearLocalJSX: true`.

In fullscreen mode, local JSX commands are rendered in a _modal slot_ (absolute-positioned, bottom-anchored) rather than inline in the scrollable area. This prevents the dialog from jiggling as new messages arrive.

## 07 Two Render Paths: Prompt vs Transcript

REPL has two screens, toggled by `screen: 'prompt' | 'transcript'` state:

The transcript mode early return (around line 4392) exists for a critical performance reason: without virtual scrolling, rendering all messages in a `ScrollBox` would allocate ~250 MB for long sessions. Transcript mode enables the `VirtualMessageList` path that only renders visible rows.

Transcript mode also enables a less-style search experience with `/` to open a search bar, `n`/`N` for navigation, `v` to open in `$VISUAL/$EDITOR`, and `[` to dump to terminal scrollback.

## 08 Session Resume Flow

The `resume()` callback handles the `/resume` command. It is one of the most involved operations in the file, coordinating a long sequence of state resets:

```
1. Deserialize messages (clean up unresolved tool uses)
2. Fire SessionEnd hooks for the current session
3. Fire SessionStart hooks for the resumed session
4. Copy or reuse the plan slug (fork vs resume)
5. Restore file history snapshots
6. Restore agent definition (name, color, type)
7. Restore standalone agent context
8. Save current session costs before switchSession()
9. Reset cost state, then restore target session costs
10. Atomically switch sessionId + project dir
11. Rename asciicast recording to match new session ID
12. Clear then restore session metadata (ordering matters)
13. Exit current worktree, enter resumed session's worktree
14. Reconstruct contentReplacementState for the new session
15. setMessages → setToolJSX(null) → setInputValue('')
```

**Why clearSessionMetadata before restoreSessionMetadata?**

`restoreSessionMetadata` only sets fields that are truthy in the log. Without the clear, a resumed session without an agent name would inherit the _previous_ session's cached name — and write that stale name to the wrong transcript on the first message.

## 09 Auto-Restore on Interrupt

When the user presses Escape to interrupt Claude and the query produced no meaningful response, REPL.tsx automatically rewinds the conversation and restores their input. This feature has several guards:

```javascript
// Inside the onQuery finally block:
if (
  abortController.signal.reason === 'user-cancel'  // Esc, not background/interrupt
  && !queryGuard.isActive                             // no newer query racing in
  && inputValueRef.current === ''                   // user hasn't typed anything
  && getCommandQueueLength() === 0                   // no queued commands (don't undo B while A was loading)
  && !store.getState().viewingAgentTaskId             // not viewing a teammate's transcript
) {
  const lastUserMsg = msgs.findLast(selectableUserMessagesFilter);
  if (lastUserMsg && messagesAfterAreOnlySynthetic(msgs, idx)) {
    removeLastFromHistory();  // undo the history entry too
    restoreMessageSync(lastUserMsg);
  }
}
```

This runs _outside_ the `queryGuard.end()` check because `onCancel` calls `queryGuard.forceEnd()`, which bumps the generation counter. `end(thisGeneration)` returns `false` for the Escape path — but the auto-restore must still run.

## 10 Main Render Tree Anatomy

The final JSX tree assembles all the pieces. Simplified structure:

```jsx
<AlternateScreen mouseTracking>
  <KeybindingSetup>                        // provides keybinding context
    <AnimatedTerminalTitle />               // 960ms tick, isolated leaf
    <GlobalKeybindingHandlers />            // ctrl+o transcript toggle, etc.
    <ScrollKeybindingHandler />             // PgUp/PgDn/g/G — BEFORE CancelRequest
    <CancelRequestHandler />               // Esc / ctrl+c
    <MCPConnectionManager>                  // manages MCP server lifecycle
      <FullscreenLayout
        scrollRef={scrollRef}              // shared with ScrollKeybindingHandler
        overlay={toolPermissionOverlay}    // PermissionRequest floats above messages
        modal={centeredModal}              // local-jsx commands in fullscreen
        scrollable={<>
          <TeammateViewHeader />
          <Messages messages={displayedMessages} />
          {placeholderText && <UserTextMessage param={placeholderText} />}
          {toolJSX && <Box>{toolJSX.jsx}</Box>}
          {showSpinner && <SpinnerWithVerb />}
          <PromptInputQueuedCommands />
        </>}
        bottom={<Box>
          {permissionStickyFooter}
          {focusedInputDialog === 'sandbox-permission' && <SandboxPermissionRequest />}
          {focusedInputDialog === 'tool-permission' && <PermissionRequest />}
          // ... other dialogs keyed to focusedInputDialog ...
          <FeedbackSurvey />
          <PromptInput onSubmit={onSubmit} />
          <SessionBackgroundHint />
          {cursor && <MessageActionsBar />}
          {focusedInputDialog === 'message-selector' && <MessageSelector />}
        </Box>}
      />
    </MCPConnectionManager>
  </KeybindingSetup>
</AlternateScreen>
```

## Key Takeaways

- REPL.tsx is 5,000 lines because it has genuine complexity — it handles concurrency, permission queues, remote sessions, swarm workers, two render modes, session resume, and keyboard navigation all in one component.

- The `QueryGuard` state machine replaces a simple boolean and prevents desync between synchronous cancellation and React's async batching. Generation numbers mean stale `finally` blocks do not corrupt state.

- Reading state via refs inside callbacks (messages, inputValue, stream mode) keeps `onSubmit` stable across 30+ `setMessages` calls per turn, preventing cascading closure capture and memory leaks.

- The dialog system is a pure function: `getFocusedInputDialog()` returns exactly one winner from a deterministic priority list. All rendering is conditional on this value — no ad-hoc boolean soup.

- The auto-restore on interrupt runs outside the generation guard by design: `forceEnd()` bumps the generation before the finally block runs, so auto-restore must be gated on `signal.reason` and `!queryGuard.isActive` instead.

- Fullscreen mode and scrollback mode produce structurally identical output — the difference is whether `<AlternateScreen>` wraps the tree and whether the virtual-scroll `ScrollBox` is mounted.

## Deep Dive: The setMessages Ref Pattern

The standard React pattern for reading state inside a callback is to add the state to the `useCallback` dep array. REPL.tsx deliberately breaks this rule for `messages`:

> "messages is read via messagesRef.current inside the callback to keep onSubmit stable across message updates... Without this, each setMessages call (~30x per turn) recreates onSubmit, pinning the REPL render scope (1776B) + that render's messages array in downstream closures... Heap analysis showed ~9 REPL scopes and ~15 messages array versions accumulating... all traced to this dep."

The trade-off is that the ref must be kept in sync on every render — which the `setMessages` wrapper does synchronously. Any code that needs the latest messages inside an async callback reads `messagesRef.current`, not the closed-over `messages`. This pattern recurs throughout the file: `inputValueRef`, `streamModeRef`, `abortControllerRef`, and `onSubmitRef` are all kept in sync for the same reason.

## Deep Dive: AnimatedTerminalTitle Isolation

The terminal tab title animates with a spinner glyph while Claude is working, cycling every 960ms. A naive implementation would put this in REPL state — but a 960ms `setInterval` re-rendering REPL would drag PromptInput, Messages, and every other child along for every tick.

The solution is to extract the animation into a separate leaf component that returns `null` (pure side-effect via `useTerminalTitle`). Only this tiny component re-renders on each tick.

```javascript
function AnimatedTerminalTitle({ isAnimating, title, disabled, noPrefix }) {
  const [frame, setFrame] = useState(0);
  useEffect(() => {
    if (!isAnimating) return;
    const interval = setInterval(() => setFrame(f => (f + 1) % frames.length), 960);
    return () => clearInterval(interval);
  }, [isAnimating]);
  useTerminalTitle(disabled ? null : `${prefix} ${title}`);
  return null;  // zero render cost
}
```

## Deep Dive: Immediate vs Non-Immediate Local JSX Commands

Slash commands that render custom UI (`type: 'local-jsx'`) fall into two placement categories:

| Category | Where rendered | Why |
|----------|---|---|
| Immediate (`/btw`, `/sandbox`) | `bottom` slot, outside ScrollBox | Stays mounted while main loop streams. If placed inside ScrollBox, new message appends would jiggle the dialog position. |
| Non-immediate (`/diff`, `/status`, `/theme`) | `scrollable` slot, inside ScrollBox | Main loop is paused while these run, so no jiggle. Their tall content (DiffDetailView up to 400 lines) needs the outer ScrollBox for scrollability. |
| Fullscreen modal (`/config`, `/model`) | `modal` slot, absolute-positioned | In fullscreen mode all local-jsx commands use the centered modal slot for consistent visual treatment. |

## Deep Dive: The Unseen Messages Divider

When the user scrolls up while Claude is responding, new messages accumulate below the viewport. REPL.tsx tracks how many unseen messages there are and shows a "jump to new" pill.

The key insight: `dividerIndex` changes only _twice_ per scroll session (once when the user scrolls away, once when they re-pin). This means `useUnseenDivider` triggers very few re-renders even as dozens of messages stream in. The pill visibility and sticky-prompt state are managed inside `FullscreenLayout`, which subscribes directly to the `ScrollBox` — so per-frame scroll never re-renders REPL.

## Knowledge Check

**Q1. Why does REPL.tsx read `messages` via `messagesRef.current` inside callbacks rather than adding `messages` to the `useCallback` dep array?**

A. Refs are faster than state reads
B. Adding messages to deps would recreate onSubmit ~30x per turn, causing memory leaks via pinned closure chains
C. React rules prohibit using state inside useCallback
D. The messages array is too large to serialize

**Q2. What is the purpose of `QueryGuard`'s generation counter?**

A. It tracks how many API calls have been made in the session
B. It prevents stale finally blocks from a cancelled query from corrupting state when a newer query has started
C. It limits concurrent tool calls to one at a time
D. It counts how many times the user has submitted input

**Q3. Why must `ScrollKeybindingHandler` be rendered before `CancelRequestHandler` in the JSX tree?**

A. ScrollKeybindingHandler must initialize its ref before CancelRequestHandler reads it
B. ctrl+c with a text selection should copy the selection, not cancel the active task; the scroll handler stops propagation when a selection exists
C. The cancel handler has higher z-index and would intercept all keystrokes
D. React renders hooks in tree order so scroll bindings must register first

**Q4. The auto-restore on interrupt runs _outside_ the `queryGuard.end(thisGeneration)` block. Why?**

A. Auto-restore is a UI effect and must run after all loading state is cleared
B. onCancel calls forceEnd(), which bumps the generation counter, so end(thisGeneration) returns false for the Escape path — but auto-restore still needs to run
C. Auto-restore uses different state and has no dependency on the guard
D. It is a bug that will be fixed in a future version

**Q5. When does `isLoading` return `true` even though no local `onQuery` is running?**

A. Never — isLoading is always false when onQuery is not running
B. When the streaming text buffer is non-empty
C. When a remote session, SSH connection, or foregrounded background task is active (isExternalLoading)
D. When the user is viewing a teammate's transcript

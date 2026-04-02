# mdENG Lesson 42: Fullscreen and Alternate Screen Modes

## Overview

Claude Code's rich interactive UI renders on the **alternate screen buffer** — the same DEC private mode used by `vim`, `less`, and `htop`. When you exit, shell history remains untouched. Mouse wheel events scroll the message list rather than terminal history.

**Source files covered:**
- `utils/fullscreen.ts`
- `ink/termio/dec.ts`
- `ink/components/AlternateScreen.tsx`
- `components/FullscreenLayout.tsx`
- `components/OffscreenFreeze.tsx`

**Three nested layers:**

1. **Detection** — Env vars, tmux probe, interactive flag
2. **DEC Sequences** — Raw escape codes for alternate screen and mouse tracking
3. **React Integration** — AlternateScreen component, FullscreenLayout slots, OffscreenFreeze optimization

---

## 02 The DEC Private Mode Sequences

Everything starts in `ink/termio/dec.ts`, which encodes DEC private mode numbers and generates escape sequences.

```typescript
// ink/termio/dec.ts — the complete DEC mode table
export const DEC = {
  CURSOR_VISIBLE:      25,
  ALT_SCREEN:         47,    // older; does NOT save/restore cursor
  ALT_SCREEN_CLEAR:   1049,  // modern: save cursor + switch + clear
  MOUSE_NORMAL:       1000,  // button press/release + wheel
  MOUSE_BUTTON:       1002,  // adds drag (button-motion)
  MOUSE_ANY:          1003,  // adds all-motion (hover)
  MOUSE_SGR:          1006,  // SGR format: CSI < btn;col;row M/m
  FOCUS_EVENTS:       1004,
  BRACKETED_PASTE:    2004,
  SYNCHRONIZED_UPDATE:2026,
} as const
```

Helper functions `decset(mode)` and `decreset(mode)` wrap each number into standard `CSI ? N h` (set) / `CSI ? N l` (reset) format. All concrete sequences are pre-generated as module-level constants to avoid string formatting at render time:

```typescript
export const ENTER_ALT_SCREEN = decset(DEC.ALT_SCREEN_CLEAR)   // \x1b[?1049h
export const EXIT_ALT_SCREEN  = decreset(DEC.ALT_SCREEN_CLEAR)  // \x1b[?1049l

// All four mouse modes stacked — DEC 1000 + 1002 + 1003 + 1006
export const ENABLE_MOUSE_TRACKING =
  decset(DEC.MOUSE_NORMAL) +
  decset(DEC.MOUSE_BUTTON) +
  decset(DEC.MOUSE_ANY)    +
  decset(DEC.MOUSE_SGR)
export const DISABLE_MOUSE_TRACKING =
  decreset(DEC.MOUSE_SGR)    +  // reversed order — disable outer modes first
  decreset(DEC.MOUSE_ANY)    +
  decreset(DEC.MOUSE_BUTTON) +
  decreset(DEC.MOUSE_NORMAL)
```

### Why 1049 not 47?

DEC mode 1049 saves cursor position before switching and restores it on exit. The older mode 47 switches buffers without cursor preservation. Claude Code uses 1049 exclusively to preserve the user's shell cursor position when fullscreen exits.

### Why are the mouse modes disabled in reverse order?

The modes form a superset hierarchy: 1003 (any-motion) includes everything 1002 (button-motion) does, which includes everything 1000 (normal) does. Setting them in order expands the tracking envelope. Disabling in reverse order (outer -> inner, 1006 -> 1003 -> 1002 -> 1000) ensures clean teardown rather than leaving a partial mode active if the process crashes between two `decreset` calls. The SGR format (1006) disables first because it modifies the reporting format rather than the tracking mode — disabling it before tracking modes means spurious events during teardown arrive in legacy X10 format, which the parser safely ignores.

---

## 03 Fullscreen Detection Logic

`utils/fullscreen.ts` is a pure decision module answering "should fullscreen activate right now?" via four exported predicates. None touch the terminal directly — they only inspect environment state.

**Detection flow:**

- `isFullscreenActive()` checks `getIsInteractive()`
- If false (headless/SDK/--print), return false
- If true, check `isFullscreenEnvEnabled()`
- Check if `CLAUDE_CODE_NO_FLICKER` is explicitly falsy (=0) -> return false
- Check if `CLAUDE_CODE_NO_FLICKER` is explicitly truthy (=1) -> return true
- Check `isTmuxControlMode()` — if yes (iTerm2 -CC), return false
- Check if `USER_TYPE === 'ant'` — if yes, return true
- Otherwise return false

### The tmux -CC probe: why it's synchronous

The most interesting detection code is `probeTmuxControlModeSync()`. It calls `spawnSync('tmux', ['display-message', '-p', '#{client_control_mode}'])` — a synchronous subprocess deliberately blocking the event loop for ~5ms.

```
// Sync (spawnSync) because the answer gates whether we enter fullscreen —
// an async probe raced against React render and lost: coder-tmux
// (ssh -> tmux -CC on a remote box) doesn't propagate TERM_PROGRAM, so
// the env heuristic missed, and by the time the async probe resolved
// we'd already entered alt-screen with mouse tracking enabled.
// Mouse wheel is dead in iTerm2's -CC integration, so users couldn't scroll at all.
```

This is a deliberate correctness/performance trade-off: pay 5ms once at startup to avoid an unrecoverable UX bug (dead mouse wheel for iTerm2 tmux -CC users). The cost is bounded — it only fires when `$TMUX` is set AND `$TERM_PROGRAM` is absent (SSH-into-tmux case). Direct iTerm2 and non-tmux paths skip the subprocess entirely via fast heuristic.

### The caching trick

`tmuxControlModeProbed` is a module-level `boolean | undefined`. It is seeded with the env heuristic result _before_ spawning, so even if the spawn throws or returns non-zero, the cache is already populated. Without this, every subsequent call to `isTmuxControlMode()` (firing 15+ times per render frame) would re-enter the probe function and potentially re-spawn a subprocess.

### Mouse knobs: two orthogonal controls

Even when fullscreen is on, mouse behavior has two independent kill-switches:

| Env var | What it disables | What still works |
|---------|-----------------|-----------------|
| `CLAUDE_CODE_NO_FLICKER=0` | Alt-screen entirely + all mouse tracking | Normal terminal scrollback, no virtualized scroll |
| `CLAUDE_CODE_DISABLE_MOUSE=1` | Mouse capture (wheel + click/drag) | Alt-screen stays; keyboard PgUp/PgDn/Ctrl+Home/End still work |
| `CLAUDE_CODE_DISABLE_MOUSE_CLICKS=1` | Click and drag events only | Alt-screen + wheel scroll still work |

`CLAUDE_CODE_DISABLE_MOUSE` exists specifically for users who want alt-screen (no flicker) but also need tmux/kitty copy-on-select to work. Those terminal multiplexers intercept mouse events when the application has capture enabled; disabling capture restores native terminal text selection while preserving the fullscreen layout.

---

## 04 The AlternateScreen React Component

`ink/components/AlternateScreen.tsx` is the React boundary that actually writes the DEC sequences to the terminal. It wraps the entire REPL tree with one critical constraint: the escape sequences must reach the terminal _before_ the first rendered frame — otherwise the first frame paints on the main screen, then the alt-screen switch happens and that frame is preserved as a broken view when you exit.

### Why useInsertionEffect instead of useLayoutEffect

```typescript
// useInsertionEffect (not useLayoutEffect): react-reconciler calls
// resetAfterCommit between the mutation and layout commit phases, and
// Ink's resetAfterCommit triggers onRender. With useLayoutEffect, that
// first onRender fires BEFORE this effect — writing a full frame to the
// main screen with altScreen=false. That frame is preserved when we
// enter alt screen and revealed on exit as a broken view. Insertion
// effects fire during the mutation phase, before resetAfterCommit, so
// ENTER_ALT_SCREEN reaches the terminal before the first frame does.
useInsertionEffect(() => {
  const ink = instances.get(process.stdout)
  if (!writeRaw) return

  writeRaw(
    ENTER_ALT_SCREEN
    + '\x1b[2J\x1b[H'           // clear screen + home cursor
    + (mouseTracking ? ENABLE_MOUSE_TRACKING : '')
  )
  ink?.setAltScreenActive(true, mouseTracking)

  return () => {
    ink?.setAltScreenActive(false)
    ink?.clearTextSelection()
    writeRaw((mouseTracking ? DISABLE_MOUSE_TRACKING : '') + EXIT_ALT_SCREEN)
  }
}, [writeRaw, mouseTracking])
```

The effect also calls `ink.setAltScreenActive(true, mouseTracking)`. This notifies the Ink renderer to constrain the cursor inside the viewport on every paint — preventing the cursor-restore newline from scrolling the alt-screen content up. It also registers a signal-exit cleanup handler so the alt-screen exits cleanly even if React unmount never runs (e.g. SIGKILL).

### Height constraint

```typescript
// Constrain height to terminal rows — no native scrollback in alt-screen
return (
  <Box
    flexDirection="column"
    height={size?.rows ?? 24}  // from TerminalSizeContext
    width="100%"
    flexShrink={0}
  >
    {children}
  </Box>
)
```

Because the alternate screen has no scrollback, any content taller than the terminal would vanish past the bottom. AlternateScreen pins its own height to `size.rows` (from `TerminalSizeContext`), which forces all overflow to be handled by Ink's `overflow: scroll` / flexbox layout — not by the terminal.

---

## 05 FullscreenLayout: Slot-Based Composition

`components/FullscreenLayout.tsx` is the highest-level layout component with two completely different rendering paths depending on `isFullscreenEnvEnabled()`:

**Fullscreen ON:**
- Slot-based layout: ScrollBox (grows), sticky bottom strip (shrinks), absolute modal overlay
- Everything is viewport-constrained via AlternateScreen

**Fullscreen OFF:**
- Sequential render: `<>{scrollable}{bottom}{overlay}{modal}</>`
- Content stacks vertically into normal scrollback

In fullscreen mode the layout has five named slots:

```typescript
type Props = {
  scrollable:    ReactNode  // message list — lives in a ScrollBox with stickyScroll
  bottom:        ReactNode  // pinned bottom strip: prompt input, spinner, permissions
  overlay?:     ReactNode  // rendered inside ScrollBox after messages (PermissionRequest)
  bottomFloat?: ReactNode  // absolute bottom-right of scroll area (companion speech bubble)
  modal?:       ReactNode  // slash-command dialog: absolute bottom-anchored, fullscreen only
}
```

### The modal pane sizing calculation

```typescript
// MODAL_TRANSCRIPT_PEEK = 2 — rows of transcript visible above the modal divider
modal != null && (
  <ModalContext value={{
    rows:    terminalRows - MODAL_TRANSCRIPT_PEEK - 1,
    columns: columns - 4,
  }}>
    <Box
      position="absolute"
      bottom={0} left={0} right={0}
      maxHeight={terminalRows - MODAL_TRANSCRIPT_PEEK}
    >
      /* divider line, then modal content with paddingX=2 */
    </Box>
  </ModalContext>
)
```

The modal occupies almost the full viewport height, always leaving exactly 2 transcript rows visible above the divider line. The `ModalContext` carries the computed interior dimensions (`rows - 3`, `columns - 4`) so child scroll boxes inside the modal know how tall they can grow.

### The "N new messages" pill

When the user scrolls up, a pill floats at the bottom of the scroll area showing how many new messages arrived. The visibility state is computed via `useSyncExternalStore` subscribing to the `ScrollBox`'s scroll position — so the pill appears and disappears without triggering any re-render of the parent REPL component.

```typescript
// pillVisible subscribes directly to ScrollBox — no REPL re-render per scroll frame
const pillVisible = useSyncExternalStore(subscribe, () => {
  const s = scrollRef?.current
  const dividerY = dividerYRef?.current
  if (!s || dividerY == null) return false
  return s.getScrollTop() + s.getPendingDelta() + s.getViewportHeight() < dividerY
})
```

---

## 06 OffscreenFreeze: Eliminating Offscreen Redraws

`components/OffscreenFreeze.tsx` solves a specific performance problem in the non-fullscreen (main-screen) rendering path. When content has scrolled above the terminal viewport into the scrollback buffer, any change to that content forces Ink's renderer into a full terminal reset — it cannot partially update rows that have already scrolled out. For spinner components or elapsed-time counters that update every tick, this produces a visible reset per animation frame.

```typescript
export function OffscreenFreeze({ children }: Props): React.ReactNode {
  'use no memo'  // React Compiler opt-out — the freeze IS the memo mechanism

  const inVirtualList = useContext(InVirtualListContext)
  const [ref, { isVisible }] = useTerminalViewport()
  const cached = useRef(children)

  if (isVisible || inVirtualList) {
    cached.current = children  // update cache only while visible
  }
  // while offscreen: return stale ref — React reconciler bails, zero diff
  return <Box ref={ref}>{cached.current}</Box>
}
```

The mechanism works because React bails out of reconciliation when the returned element has the same object identity as the previous render. By returning the cached ref unchanged while offscreen, the entire subtree produces zero diff and zero terminal output.

### Virtual list exemption

OffscreenFreeze explicitly skips the optimization when `inVirtualList` is true. The ScrollBox's virtual list clips all content inside the viewport — there is no terminal scrollback to worry about. More importantly, freezing inside a virtual list would break click-to-expand, because `useTerminalViewport`'s visibility calculation can disagree with the ScrollBox's virtual scroll position.

### React Compiler interaction

The component uses `'use no memo'` — an explicit opt-out from React Compiler's automatic memoization. If the compiler memoized the component, it would cache the returned element itself, which would defeat the freeze mechanism (the freeze works by intentionally returning a stale cached ref, not by memoizing the component's output).

---

## 07 Lifecycle: Enter to Exit

**Sequence:**

1. **Startup** -> Check `isFullscreenActive()`
2. **fullscreen.ts** -> Call `getIsInteractive()`, `isFullscreenEnvEnabled()`, `isTmuxControlMode()`, check `USER_TYPE`
3. **AlternateScreen mounts** (useInsertionEffect)
4. **Terminal receives:** `ENTER_ALT_SCREEN` + clear + `ENABLE_MOUSE_TRACKING`
5. **Ink notified:** `setAltScreenActive(true, mouseTracking)` — renderer clamps cursor to viewport
6. **Each render frame:** Ink outputs diff within alt-screen bounds
7. **AlternateScreen unmounts** (cleanup)
8. **Ink notified:** `setAltScreenActive(false)`, `clearTextSelection()`
9. **Terminal receives:** `DISABLE_MOUSE_TRACKING` + `EXIT_ALT_SCREEN`
10. **Result:** Main screen + cursor position restored

### tmux and the mouse scroll hint

When running inside tmux (but NOT in tmux -CC mode), mouse wheel events are forwarded to the application only when tmux's `mouse` option is enabled. Claude Code does not programmatically set `tmux set mouse on` — a deliberate choice made after a previous implementation leaked tmux mouse state to sibling panes (vim, less, shell). Instead, `maybeGetTmuxMouseHint()` fires once at startup and returns a hint string if the tmux mouse option is currently off:

```
"tmux detected - scroll with PgUp/PgDn - or add 'set -g mouse on' to ~/.tmux.conf for wheel scroll"
```

---

## Key Takeaways

- The alternate screen is DEC mode 1049 (`\x1b[?1049h`), not the older mode 47 — 1049 saves and restores cursor position.
- `useInsertionEffect` was chosen over `useLayoutEffect` to ensure `ENTER_ALT_SCREEN` reaches the terminal before Ink's first rendered frame — a subtle timing requirement that `useLayoutEffect` gets wrong.
- The tmux -CC detection probe is synchronous by design: an async probe raced against the React render cycle and caused an unrecoverable broken state (dead mouse wheel) for SSH+tmux users.
- Fullscreen defaults **on** for Anthropic internal users (`USER_TYPE=ant`) and **off** for external users — the env var `CLAUDE_CODE_NO_FLICKER=1` opts in from outside.
- `OffscreenFreeze` exploits React object-identity bail-out to produce zero diff output for content that has scrolled into terminal scrollback, eliminating per-tick full resets.
- Mouse tracking has two orthogonal kill-switches: `CLAUDE_CODE_DISABLE_MOUSE` (kill capture, keep alt-screen) and `CLAUDE_CODE_NO_FLICKER=0` (kill everything including alt-screen).

---

## Check Your Understanding

**Question 1:** What does DEC private mode 1049 do that mode 47 does not?

A) Enables mouse tracking in addition to switching the screen buffer
B) Saves the cursor position before switching and restores it on exit
C) Clears the alternate screen buffer automatically on entry
D) Enables synchronized update mode to prevent partial frames

**Answer: B**

**Question 2:** Why does AlternateScreen use `useInsertionEffect` instead of `useLayoutEffect`?

A) `useInsertionEffect` is newer and always preferred for terminal I/O
B) `useLayoutEffect` causes a memory leak in the Ink reconciler
C) Insertion effects fire during the mutation phase, before Ink's onRender — ensuring `ENTER_ALT_SCREEN` reaches the terminal before the first frame
D) `useLayoutEffect` does not support cleanup functions in the current React version

**Answer: C**

**Question 3:** The tmux control-mode probe (`probeTmuxControlModeSync`) is synchronous. What would happen if it were async?

A) The probe would never complete because the event loop is blocked by React rendering
B) The probe would race against React render; if fullscreen activated before the probe returned, SSH+tmux -CC users would end up with a broken alt-screen and a dead mouse wheel
C) The async probe would work correctly for local users but fail on Windows
D) The cache would not be populated in time, causing 15+ extra subprocess spawns per render frame

**Answer: B**

**Question 4:** What is the purpose of `OffscreenFreeze`'s `'use no memo'` directive?

A) It prevents React from batching state updates inside the component
B) It opts the component out of React Compiler's automatic memoization, because memoizing would defeat the freeze (the freeze works by returning a stale ref, not by memoizing output)
C) It tells the React reconciler to skip this component's children entirely
D) It disables React Concurrent Mode scheduling for this subtree

**Answer: B**

**Question 5:** A user sets `CLAUDE_CODE_DISABLE_MOUSE=1`. Which behavior best describes the result?

A) Fullscreen is disabled; the normal terminal scrollback is used
B) The alt-screen is still active; mouse capture is disabled but keyboard scrolling (PgUp/PgDn) still works
C) Only click and drag are disabled; mouse wheel still works
D) Mouse events are queued and replayed after the model response arrives

**Answer: B**

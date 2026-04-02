# mdENG — Lesson 31 — Vim Mode Implementation — Source Deep Dive

## Vim Mode Implementation

How Claude Code implements a full vim keybinding engine inside a terminal UI — state machines, pure functions, and zero compromise on correctness.

### 01 Why Vim Mode in a Terminal App?

Claude Code's input box is a rich terminal widget, not a browser `<input>`. That means standard OS keybindings are all that's available by default. To offer a full vim experience — motions, operators, text objects, dot-repeat, registers — the team had to implement the entire vim input grammar from scratch in TypeScript.

The result lives in five files under `src/vim/`, totalling roughly 700 lines. It's one of the most carefully engineered subsystems in the codebase: pure functions throughout, an explicit state machine for every mode, and TypeScript's discriminated unions used as living documentation.

#### types.ts — State Machine Types

All modes, states, and recorded changes as TypeScript unions. Reading this file tells you everything the engine can do.

#### transitions.ts — Transition Table

One function per state. Each receives a keypress and returns the next state and/or a side-effect to run.

#### motions.ts — Pure Cursor Math

Stateless functions that resolve a motion key + count to a target Cursor. No mutations.

#### operators.ts — Text Mutations

delete / change / yank applied to character ranges, linewise ranges, or text objects.

#### textObjects.ts — Structural Selection

iw, aw, i(, a", etc. — boundary finders that work on raw text offsets with grapheme safety.

### 02 The Type System as Specification

The first thing to read in `types.ts` is the block comment at the top — it contains an ASCII state diagram. But the real spec is the types themselves. `VimState` is a discriminated union with exactly two members:

```typescript
/**
 * Complete vim state. Mode determines what data is tracked.
 *
 * INSERT mode: Track text being typed (for dot-repeat)
 * NORMAL mode: Track command being parsed (state machine)
 */
export type VimState =
  | { mode: 'INSERT'; insertedText: string }
  | { mode: 'NORMAL'; command: CommandState }
```

INSERT mode carries only `insertedText` — the characters typed since entering insert mode, which will be replayed by dot-repeat. NORMAL mode delegates all complexity to `CommandState`:

```typescript
export type CommandState =
  | { type: 'idle' }
  | { type: 'count'; digits: string }
  | { type: 'operator'; op: Operator; count: number }
  | { type: 'operatorCount'; op: Operator; count: number; digits: string }
  | { type: 'operatorFind'; op: Operator; count: number; find: FindType }
  | { type: 'operatorTextObj'; op: Operator; count: number; scope: TextObjScope }
  | { type: 'find'; find: FindType; count: number }
  | { type: 'g'; count: number }
  | { type: 'operatorG'; op: Operator; count: number }
  | { type: 'replace'; count: number }
  | { type: 'indent'; dir: '>' | '<'; count: number }
```

Each variant encodes exactly what the parser needs to remember mid-sequence. `operatorCount` exists because vim allows `2d3w` — a count on the operator _and_ a count on the motion, and the effective count is their product (`2 * 3 = 6` words).

#### Design Insight

TypeScript's exhaustive switch-checking means that if a new state is added to `CommandState`, every switch statement that handles it will produce a compile error until a handler is added. The type system enforces completeness at zero runtime cost.

#### Persistent State — The Editor's Memory

Beyond the current command being parsed, there's state that must survive across commands:

```typescript
export type PersistentState = {
  lastChange: RecordedChange | null   // dot-repeat target
  lastFind:   { type: FindType; char: string } | null  // ; / , repeat
  register:   string                     // yank/delete clipboard
  registerIsLinewise: boolean           // affects paste behavior
}
```

`RecordedChange` is its own discriminated union capturing everything needed to replay a command: the operator, the motion, the count, or — for text-object operations — the scope and object type. This is what makes `.` work correctly after `ci"` or `3dw`.

### 03 The Transition Table

`transitions.ts` is the heart of the engine. It exports a single `transition()` function that dispatches based on `state.type`:

```typescript
export function transition(
  state: CommandState,
  input: string,
  ctx: TransitionContext,
): TransitionResult {
  switch (state.type) {
    case 'idle':         return fromIdle(input, ctx)
    case 'count':        return fromCount(state, input, ctx)
    case 'operator':     return fromOperator(state, input, ctx)
    case 'operatorCount': return fromOperatorCount(state, input, ctx)
    case 'operatorFind':  return fromOperatorFind(state, input, ctx)
    case 'operatorTextObj':return fromOperatorTextObj(state, input, ctx)
    case 'find':          return fromFind(state, input, ctx)
    case 'g':             return fromG(state, input, ctx)
    case 'operatorG':     return fromOperatorG(state, input, ctx)
    case 'replace':       return fromReplace(state, input, ctx)
    case 'indent':        return fromIndent(state, input, ctx)
  }
}
```

The return type is `TransitionResult`: optionally a **next state** and optionally an **execute callback**. Callers call `execute()` if present, then update the command state to `next` (or reset to `idle` if `next` is absent after an execute). This separation means the transition logic never directly mutates editor state.

#### State Diagram

```
stateDiagram-v2
[*] --> idle
idle --> count : digit 1-9
idle --> operator : d / c / y
idle --> find : f F t T
idle --> g : g
idle --> replace : r
idle --> indent : > or <
idle --> idle : execute (motion / action)
count --> count : digit 0-9
count --> operator : d / c / y
count --> idle : execute (motion / action)
operator --> operatorCount : digit
operator --> operatorTextObj : i / a
operator --> operatorFind : f F t T
operator --> operatorG : g
operator --> idle : execute (dd cc yy or motion)
operatorCount --> operatorCount : digit
operatorCount --> idle : execute
operatorFind --> idle : execute (char received)
operatorTextObj --> idle : execute (obj type received)
find --> idle : execute (char received)
g --> idle : execute (gg / gj / gk)
operatorG --> idle : execute or cancel
replace --> idle : execute (char received)
indent --> idle : execute (>> or <<)
```

#### Shared Input Handling

Both `fromIdle` and `fromCount` accept the same set of inputs after accounting for their count. Rather than duplicating logic, both delegate to a shared helper:

```typescript
function handleNormalInput(
  input: string,
  count: number,
  ctx: TransitionContext,
): TransitionResult | null {
  if (isOperatorKey(input)) {
    return { next: { type: 'operator', op: OPERATORS[input], count } }
  }
  if (SIMPLE_MOTIONS.has(input)) {
    return {
      execute: () => {
        const target = resolveMotion(input, ctx.cursor, count)
        ctx.setOffset(target.offset)
      },
    }
  }
  // ... fFtT, g, r, >, <, ~, x, J, p, P, D, C, Y, G, ., ;, ,, u, i/I/a/A, o/O
  return null  // unrecognized input
}
```

#### Key Pattern

Returning `null` from `handleNormalInput` means "this input is not recognized in this context." Callers can fall through to reset to `idle`. This cleanly handles invalid keystrokes without throwing errors.

### 04 Motions — Pure Cursor Math

`motions.ts` is deliberately the simplest file. It exports three pure functions:

```typescript
/** Resolve a motion to a target cursor position. Pure calculation. */
export function resolveMotion(key: string, cursor: Cursor, count: number): Cursor {
  let result = cursor
  for (let i = 0; i < count; i++) {
    const next = applySingleMotion(key, result)
    if (next.equals(result)) break   // at boundary, stop early
    result = next
  }
  return result
}

export function isInclusiveMotion(key: string): boolean {
  return 'eE$'.includes(key)
}

export function isLinewiseMotion(key: string): boolean {
  return 'jkG'.includes(key) || key === 'gg'
}
```

The loop in `resolveMotion` applies one step at a time, breaking early when the cursor stops moving. This correctly handles hitting the start/end of the buffer mid-count.

Motion inclusivity matters for operators: `dw` excludes the character under the target, but `de` includes it. `isInclusiveMotion` and `isLinewiseMotion` are the flags that `getOperatorRange` in `operators.ts` uses to compute the exact byte range to operate on.

#### Motion Categories

| Category | Keys | Cursor Method |
|----------|------|---------------|
| Character | h l | `cursor.left()` / `cursor.right()` |
| Line (logical) | j k | `cursor.downLogicalLine()` / `cursor.upLogicalLine()` |
| Line (visual) | gj gk | `cursor.down()` / `cursor.up()` |
| Word (small-w) | w b e | `nextVimWord()` / `prevVimWord()` / `endOfVimWord()` |
| Word (WORD) | W B E | `nextWORD()` / `prevWORD()` / `endOfWORD()` |
| Line positions | 0 ^ $ | `startOfLogicalLine()` / `firstNonBlank…()` / `endOfLogicalLine()` |
| File | G gg | `startOfLastLine()` / `startOfFirstLine()` |

### 05 Operators — Text Mutations

Operators (`d`, `c`, `y`) need a _range_ before they can act. `operators.ts` provides one entry point per combination of operator + range-source:

#### executeOperatorMotion

dw, c$, y^, ...

Calls `resolveMotion`, converts to a byte range respecting inclusive/linewise flags, then applies the operator.

#### executeOperatorFind

dfx, ctY, ...

Uses `cursor.findCharacter()` to locate the target, then computes an inclusive range to that offset.

#### executeOperatorTextObj

diw, ca(, yi", ...

Delegates to `findTextObject()` from `textObjects.ts` and applies the operator to the returned `{start, end}`.

#### executeLineOp

dd, cc, yy

Operator doubled — computes full logical lines from current position, handles edge case of last line in buffer.

All four share a private `applyOperator()` helper:

```typescript
function applyOperator(
  op: Operator,
  from: number,
  to: number,
  ctx: OperatorContext,
  linewise: boolean = false,
): void {
  let content = ctx.text.slice(from, to)
  if (linewise && !content.endsWith('\n')) content = content + '\n'
  ctx.setRegister(content, linewise)

  if (op === 'yank') {
    ctx.setOffset(from)                          // cursor to yank start, no text change
  } else if (op === 'delete') {
    const newText = ctx.text.slice(0, from) + ctx.text.slice(to)
    ctx.setText(newText)
    ctx.setOffset(Math.min(from, newText.length - 1))
  } else if (op === 'change') {
    const newText = ctx.text.slice(0, from) + ctx.text.slice(to)
    ctx.setText(newText)
    ctx.enterInsert(from)                        // change always enters INSERT mode
  }
}
```

#### The cw Special Case

Real vim treats `cw` differently from `dw`: it changes to the _end of the word_, not the start of the next word. `getOperatorRange()` handles this explicitly:

```typescript
// Special case: cw/cW changes to end of word, not start of next word
if (op === 'change' && (motion === 'w' || motion === 'W')) {
  let wordCursor = cursor
  for (let i = 0; i < count - 1; i++) {
    wordCursor = motion === 'w' ? wordCursor.nextVimWord() : wordCursor.nextWORD()
  }
  const wordEnd = motion === 'w'
    ? wordCursor.endOfVimWord()
    : wordCursor.endOfWORD()
  to = cursor.measuredText.nextOffset(wordEnd.offset)
}
```

#### Edge Case

There's also a guard for image ref "chips" in the text buffer: `cursor.snapOutOfImageRef()` is called on both ends of every operator range. This ensures that word or motion operations never leave a partial `[Image #N]` placeholder in the buffer — a concern unique to Claude Code's multimodal input widget.

### 06 Text Objects — Structural Selection

Text objects are one of vim's killer features: `ci"` changes inside quotes without needing to move the cursor there first. `textObjects.ts` implements them as a single exported function:

```typescript
export function findTextObject(
  text: string,
  offset: number,
  objectType: string,
  isInner: boolean,
): TextObjectRange {
  if (objectType === 'w') return findWordObject(text, offset, isInner, isVimWordChar)
  if (objectType === 'W') return findWordObject(text, offset, isInner, ch => !isVimWhitespace(ch))

  const pair = PAIRS[objectType]
  if (pair) {
    const [open, close] = pair
    return open === close
      ? findQuoteObject(text, offset, open, isInner)
      : findBracketObject(text, offset, open, close, isInner)
  }
  return null
}
```

The delimiter pair table covers all standard vim text objects:

```typescript
const PAIRS: Record<string, [string, string]> = {
  '(': ['(', ')'],  ')': ['(', ')'],  b: ['(', ')'],
  '[': ['[', ']'],  ']': ['[', ']'],
  '{': ['{', '}'],  '}': ['{', '}'],  B: ['{', '}'],
  '<': ['<', '>'],  '>': ['<', '>'],
  '"': ['"', '"'],  "'": ["'", "'"],  '`': ['`', '`'],
}
```

#### Bracket vs Quote Algorithms

Bracket pairs (`()`, `[]`, `{}`, `<>`) use a depth-counting bidirectional scan — walk left to find the matching open bracket, then walk right to find the close, properly handling nesting. Quote pairs use a different approach because quotes are symmetric — there's no "nesting." Instead, they find all quote positions on the line and pair them as 0-1, 2-3, 4-5.

```typescript
// bracket scan
for (let i = offset; i >= 0; i--) {
  if (text[i] === close && i !== offset) depth++
  else if (text[i] === open) {
    if (depth === 0) { start = i; break }
    depth--
  }
}
```

The word object finder pre-segments the text into graphemes using `Intl.Segmenter` before scanning. This guarantees correctness with multi-byte unicode characters — a detail most vim implementations get wrong.

### 07 The Context Interface — Dependency Inversion

None of the vim files import React, Ink, or any UI framework. All side effects flow through two context types:

```typescript
export type OperatorContext = {
  cursor:      Cursor
  text:        string
  setText:     (text: string) => void
  setOffset:   (offset: number) => void
  enterInsert: (offset: number) => void
  getRegister: () => string
  setRegister: (content: string, linewise: boolean) => void
  getLastFind: () => { type: FindType; char: string } | null
  setLastFind: (type: FindType, char: string) => void
  recordChange:(change: RecordedChange) => void
}
```

```typescript
export type TransitionContext = OperatorContext & {
  onUndo?:      () => void
  onDotRepeat?: () => void
}
```

The real editor creates these context objects by closing over React state setters. The vim engine just calls the callbacks. This means the entire engine can be tested against any object that satisfies the interface — no mocking required.

#### Architecture Win

The vim engine has zero knowledge of how text is stored, rendered, or transmitted to the API. It operates purely through the `OperatorContext` + `TransitionContext` interfaces. This is textbook dependency inversion, and it's why the engine is fully unit-testable without spinning up a terminal.

### 08 Special Commands Deep-Dive

#### Dot Repeat (.)

When `.` is pressed, the transition calls `ctx.onDotRepeat?.()`. The caller (the editor component) holds the `PersistentState.lastChange` and replays it by constructing a new context and re-running the appropriate execute function. The `RecordedChange` union ensures every command type carries exactly the data needed to re-run it.

#### Find / Till (f F t T) and Repeat (; ,)

`fromFind` calls `cursor.findCharacter(char, findType, count)` and stores the result in `PersistentState.lastFind` via `ctx.setLastFind()`. The `;` and `,` handlers in `handleNormalInput` read that stored find, flip direction if `,`, then repeat the search — a three-line implementation of a subtle vim feature.

```typescript
function executeRepeatFind(reverse: boolean, count: number, ctx: TransitionContext): void {
  const lastFind = ctx.getLastFind()
  if (!lastFind) return
  let findType = lastFind.type
  if (reverse) {
    const flipMap: Record<FindType, FindType> = { f: 'F', F: 'f', t: 'T', T: 't' }
    findType = flipMap[findType]
  }
  const result = ctx.cursor.findCharacter(lastFind.char, findType, count)
  if (result !== null) ctx.setOffset(result)
}
```

#### Replace Character (r)

`fromReplace` handles one edge case that's easy to miss: if the input is an empty string, it means Backspace/Delete was pressed. In vim, `r<BS>` cancels the replace rather than deleting the character. The guard `if (input === '') return { next: { type: 'idle' } }` implements this correctly.

#### G and gg Navigation

`G` with no count goes to the last line. With a count (e.g., `42G`), it goes to line 42. The implementation checks `count === 1` as a sentinel for "no count given" — because pressing `G` alone produces count=1 from the idle state. The same pattern applies to `gg`.

### 09 Count Multiplication: 2d3w

Real vim supports compound counts: `2d3w` deletes 6 words (2 x 3). This requires two separate count accumulators — one on the operator, one on the motion — and the `operatorCount` state exists solely for this purpose.

```typescript
function fromOperatorCount(state, input, ctx) {
  if (/[0-9]/.test(input)) {
    const newDigits = state.digits + input
    const parsedDigits = Math.min(parseInt(newDigits, 10), MAX_VIM_COUNT)
    return { next: { ...state, digits: String(parsedDigits) } }
  }
  const motionCount = parseInt(state.digits, 10)
  const effectiveCount = state.count * motionCount  // <-- multiplication
  const result = handleOperatorInput(state.op, effectiveCount, input, ctx)
  if (result) return result
  return { next: { type: 'idle' } }
}
```

The `MAX_VIM_COUNT = 10000` constant prevents a user from accidentally triggering `999999j` and causing a performance catastrophe.

### 10 No Magic Strings — Named Key Constants

Every key group is a named constant, not an inline literal:

```typescript
export const OPERATORS = {
  d: 'delete',
  c: 'change',
  y: 'yank',
} as const satisfies Record<string, Operator>

export const SIMPLE_MOTIONS = new Set([
  'h', 'l', 'j', 'k',       // Basic movement
  'w', 'b', 'e', 'W', 'B', 'E', // Word motions
  '0', '^', '$',             // Line positions
])

export const FIND_KEYS      = new Set(['f', 'F', 't', 'T'])
export const TEXT_OBJ_TYPES = new Set(['w', 'W', '"', "'", '`', '(', ...])
```

Using `as const satisfies Record<string, Operator>` on `OPERATORS` gives both the narrowed literal types (so `OPERATORS['d']` is typed as `'delete'`, not just `string`) and the compile-time check that all values are valid `Operator` members.

## Key Takeaways

* The vim engine is a pure TypeScript state machine — no UI imports, all side effects through context callbacks.
* `CommandState` is a discriminated union that doubles as documentation; TypeScript's exhaustive checking enforces that every state has a handler.
* Motions are pure functions that return a new `Cursor`; operators compose on top by converting the cursor delta into a byte range.
* Text object finders use different algorithms for symmetric pairs (quotes: linear scan + pairing) vs asymmetric pairs (brackets: depth-counting bidirectional scan).
* Compound counts (`2d3w`) are handled by a dedicated `operatorCount` state that multiplies the two counts before executing.
* Dot-repeat works because every mutation records a `RecordedChange` capturing the operation type, count, and all parameters needed to replay it.
* Grapheme segmentation via `Intl.Segmenter` makes the engine unicode-safe — important for an AI tool that may edit text in any language.

## Check Your Understanding

### 1. When pressing `d` in NORMAL mode, what does the transition table return?

**A** An execute callback that deletes the current character
**B** A next state of `{ type: 'operator', op: 'delete', count: 1 }`
**C** A next state of `{ type: 'count', digits: '1' }`
**D** Nothing — the key is ignored until a motion follows

**Answer: B** — Pressing an operator key transitions to the `operator` state, waiting for a motion or text object. No text is changed yet.

### 2. What is the effective count when the user types `3d2w`?

**A** 3 — the operator count wins
**B** 2 — the motion count wins
**C** 6 — the counts are multiplied
**D** 5 — the counts are added

**Answer: C** — The `operatorCount` state multiplies `state.count * motionCount` before calling `handleOperatorInput`.

### 3. Why does `textObjects.ts` use a different algorithm for `i"` vs `i(`?

**A** Quotes are ASCII, brackets are unicode
**B** Quotes are symmetric (no nesting), so a depth-counting scan would fail
**C** Brackets support the `around` scope, quotes only support `inner`
**D** It is the same algorithm with a different delimiter

**Answer: B** — Because `"hello "world"` has no nesting semantics, a depth counter is meaningless. Quotes are paired linearly: positions 0-1, 2-3, etc.

### 4. What guarantees that adding a new `CommandState` variant won't silently break the transition table?

**A** A runtime assertion at startup
**B** TypeScript's exhaustive switch checking on the discriminated union
**C** A unit test that iterates all state types
**D** ESLint rule that enforces switch completeness

**Answer: B** — The `switch (state.type)` in `transition()` covers a discriminated union. TypeScript raises a compile error if any variant is unhandled.

### 5. Which state is entered when the user presses `d` then `3` in NORMAL mode?

**A** `count`
**B** `operator`
**C** `operatorCount`
**D** `idle`

**Answer: C** — After `d` the state is `operator`. A digit in `fromOperator` goes to `operatorCount` — a separate state from the pre-operator `count` state.

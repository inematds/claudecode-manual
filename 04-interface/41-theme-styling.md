# Theme and Visual Styling System

## Lesson 41: Theme and Visual Styling System
### Source Deep Dive

How Claude Code transforms a `ThemeName` string into colored terminal output — from the six-palette type system to chalk level clamping to the `/color` command.

---

## S1 The Big Picture

Terminal styling in Claude Code is not a thin wrapper around chalk. It is a deliberately layered system that must survive at least three hostile environments: VS Code's embedded terminal (which lies about its color support), tmux (which silently drops truecolor backgrounds), and Apple Terminal (which can't handle 24-bit SGR sequences at all). Each layer has a single job, and they compose cleanly.

### Layer 1 — utils/theme.ts
**Semantic Color Palette**

6 named themes, each mapping ~70 semantic tokens to raw color values (RGB, hex, or ANSI). This is the source of truth for every color decision.

### Layer 2 — ink/colorize.ts
**Chalk Normalization**

Detects terminal environment at module load time, boosts or clamps chalk's color level, then routes color strings to the right chalk method.

### Layer 3 — ink/styles.ts
**Layout + Style Types**

Defines the `Styles` and `TextStyles` TypeScript types that Ink `Box`/`Text` components accept — the CSS-like API for terminal layout.

### Layer 4 — design-system/color.ts
**Theme-Aware Colorizer**

Curried helper that accepts either a theme key (like `"claude"`) or a raw color, resolves theme keys at call time, then delegates to `colorize()`.

### Commands — /theme, /color
**User Controls**

`/theme` opens an interactive picker. `/color` sets the prompt-bar color for sub-agent sessions — forbidden for swarm teammates.

### AgentColorManager
**Sub-agent Identity**

Maps agent type strings to one of 8 theme color slots (`_FOR_SUBAGENTS_ONLY`) for visual differentiation in multi-agent sessions.

---

## S2 The Theme Type System

The core data structure lives in `utils/theme.ts`. The `Theme` type is a flat record of ~70 named string slots. Every slot holds a raw color value — no nesting, no tokens-inside-tokens. This flatness is intentional: any component anywhere can look up a color in O(1) with zero risk of infinite resolution loops.

```typescript
export type Theme = {
  // Brand identity
  claude: string          // rgb(215,119,87) — "Claude orange"
  claudeShimmer: string  // Lighter version for shimmer animation

  // UI surface roles
  promptBorder: string
  userMessageBackground: string
  selectionBg: string    // Alt-screen text selection highlight

  // Semantic roles
  success: string
  error: string
  warning: string

  // Diff colors (4 variants per operation)
  diffAdded: string
  diffAddedDimmed: string
  diffAddedWord: string

  // Agent colors — named to discourage general use
  red_FOR_SUBAGENTS_ONLY: string
  blue_FOR_SUBAGENTS_ONLY: string
  // ... 6 more

  // Rainbow colors for ultrathink keyword highlighting
  rainbow_red: string
  rainbow_red_shimmer: string
  // ... 12 more rainbow slots
}
```

The naming conventions tell a story. `_FOR_SUBAGENTS_ONLY` suffixes act as lint-time guardrails — you can visually grep for misuse. The `Shimmer` suffix signals that a color exists purely as the lighter step in a pulse animation, never for static text. The `_FOR_SYSTEM_SPINNER` suffix isolates the blue used by Claude's own spinner from user-visible permission prompts.

### The Six Themes

There are exactly six concrete theme objects, mapped by the `ThemeName` union:

```typescript
export const THEME_NAMES = [
  'dark',
  'light',
  'light-daltonized',
  'dark-daltonized',
  'light-ansi',
  'dark-ansi',
] as const

export const THEME_SETTINGS = ['auto', ...THEME_NAMES] as const
// ThemeSetting is stored in config; ThemeName is resolved at runtime
```

The `auto` setting is only valid in config storage — it is resolved to either `dark` or `light` by following the system's dark/light mode at runtime, then replaced by a concrete `ThemeName` before any component touches a color token.

**Design note**

The ANSI themes (`light-ansi`, `dark-ansi`) use only the 16 standard ANSI color names like `ansi:redBright`. This is important for terminals that don't support 256-color or truecolor — it means every color respects the user's own terminal palette customization. The RGB themes use explicit `rgb(r,g,b)` strings precisely to _avoid_ being affected by custom terminal palettes.

---

## S3 Dark vs. Light vs. Daltonized — What Actually Changes

The three families (dark, light, daltonized) differ in more than just brightness. The daltonized variants systematically replace green-red distinctions with blue-red ones, because deuteranopia (the most common form of color blindness) affects the green channel. Here's how a few key tokens change across the three dark variants:

| Token | dark | dark-daltonized | dark-ansi |
|-------|------|-----------------|-----------|
| `claude` | rgb(215,119,87) | rgb(255,153,51) | ansi:redBright |
| `success` | rgb(78,186,101) | rgb(51,153,255) — blue! | ansi:greenBright |
| `diffAdded` | rgb(34,92,43) | rgb(0,68,102) — dark blue! | ansi:green |
| `error` | rgb(255,107,128) | rgb(255,102,102) | ansi:redBright |
| `selectionBg` | rgb(38,79,120) | rgb(38,79,120) | ansi:blue |
| `autoAccept` | rgb(175,135,255) | rgb(175,135,255) | ansi:magentaBright |

Notice that `success` in `dark-daltonized` is bright blue, not green. Someone with deuteranopia cannot rely on green to mean "good" — so the daltonized theme substitutes blue, which is on a completely different channel. The `diffAdded` token shifts from dark green to dark blue for the same reason.

**Careful**

The daltonized themes share the exact same `selectionBg` and `autoAccept` values as the regular dark theme — only the tokens that depend on green/red discrimination are swapped. This means you cannot just diff the two theme objects to understand which colors are "accessibility-critical": you have to reason about which tokens are used for semantic distinction vs. pure decoration.

---

## S4 colorize.ts — The Terminal Environment Problem

This is where the engineering gets interesting. The file opens with two long block comments explaining two separate terminal environment bugs, and then fixes both at module load time — before any color is ever rendered.

### Problem 1: VS Code Lies About Its Color Support

```typescript
function boostChalkLevelForXtermJs(): boolean {
  // xterm.js has supported truecolor since 2017, but code-server/Coder
  // containers often don't set COLORTERM=truecolor. chalk's supports-color
  // doesn't recognize TERM_PROGRAM=vscode (it only knows iTerm.app/
  // Apple_Terminal), so it falls through to the -256color regex -> level 2.
  // At level 2, chalk.rgb() downgrades to the nearest 6x6x6 cube color:
  // rgb(215,119,87) (Claude orange) -> idx 174 rgb(215,135,135) — washed-out salmon.
  if (process.env.TERM_PROGRAM === 'vscode' && chalk.level === 2) {
    chalk.level = 3
    return true
  }
  return false
}

export const CHALK_BOOSTED_FOR_XTERMJS = boostChalkLevelForXtermJs()
```

The comment is worth reading closely. Claude's brand orange `rgb(215,119,87)` becomes a washed-out salmon `rgb(215,135,135)` in 256-color mode because the cube quantization rounds in the wrong direction. Rather than accept brand-color corruption in VS Code, the code manually bumps chalk to level 3 (truecolor) when it detects `TERM_PROGRAM=vscode`.

### Problem 2: tmux Drops Truecolor Backgrounds

```typescript
function clampChalkLevelForTmux(): boolean {
  // tmux parses truecolor SGR (\e[48;2;r;g;bm) into its cell buffer correctly,
  // but its client-side emitter only re-emits truecolor to the outer terminal
  // if the outer terminal advertises Tc/RGB capability. Default tmux config
  // doesn't set this. Without it, backgrounds are simply dropped — bg=default
  // -> black on dark profiles.
  // Clamping to level 2 makes chalk emit 256-color (\e[48;5;Nm),
  // which tmux passes through cleanly. grey93 (255) is visually identical.
  if (process.env.CLAUDE_CODE_TMUX_TRUECOLOR) return false
  if (process.env.TMUX && chalk.level > 2) {
    chalk.level = 2
    return true
  }
  return false
}

export const CHALK_CLAMPED_FOR_TMUX = clampChalkLevelForTmux()
```

The ordering of these two calls matters: `boostChalkLevelForXtermJs` runs first. If someone is running Claude Code inside VS Code's terminal inside tmux (common in remote dev setups), the boost happens first and the clamp re-clamps it back to 2. The clamp wins over the boost for tmux, because tmux's passthrough limitation is a hard constraint that can't be worked around without reconfiguring tmux itself. The escape hatch is `CLAUDE_CODE_TMUX_TRUECOLOR=1`, which skips the clamp for users who have correctly configured `terminal-overrides ,*:Tc` in their tmux config.

**Implementation detail**

Both exports (`CHALK_BOOSTED_FOR_XTERMJS` and `CHALK_CLAMPED_FOR_TMUX`) are marked as exported for debugging. The comment says "tree-shaken if unused" — they exist so an engineer can `import { CHALK_CLAMPED_FOR_TMUX } from './colorize'` in a diagnostic and know whether the clamp fired, without adding any runtime cost in production builds where nothing imports them.

### The colorize() Dispatch Table

With the chalk level set correctly, the actual color dispatch is a straightforward parser:

```typescript
export const colorize = (
  str: string,
  color: string | undefined,
  type: ColorType,  // 'foreground' | 'background'
): string => {
  if (color.startsWith('ansi:')) {
    // Routes to chalk.red / chalk.bgRed etc.
    return type === 'foreground' ? chalk.red(str) : chalk.bgRed(str)
  }
  if (color.startsWith('#')) {
    return type === 'foreground'
      ? chalk.hex(color)(str)
      : chalk.bgHex(color)(str)
  }
  if (color.startsWith('ansi256')) {
    // Parses ansi256(N) -> chalk.ansi256(N)
  }
  if (color.startsWith('rgb')) {
    // Parses rgb(r,g,b) -> chalk.rgb(r,g,b)
  }
}
```

The string-prefix dispatch means the color format is self-describing. Any component that has resolved a theme token gets back a string that tells `colorize` exactly what kind of color it is — no separate type tag needed.

---

## S5 styles.ts — Terminal Layout as TypeScript Types

The `Styles` type in `ink/styles.ts` is Claude Code's equivalent of a CSS properties object, but for terminal rendering. It covers layout (flexbox via Yoga), dimensions, borders, overflow, text wrapping, and color — all as readonly TypeScript properties.

```typescript
export type TextStyles = {
  readonly color?: Color            // Raw color value, not a theme key
  readonly backgroundColor?: Color
  readonly dim?: boolean
  readonly bold?: boolean
  readonly italic?: boolean
  readonly underline?: boolean
  readonly strikethrough?: boolean
  readonly inverse?: boolean
}

// Color is a discriminated union of all supported formats:
export type Color = RGBColor | HexColor | Ansi256Color | AnsiColor
// where RGBColor = `rgb(${number},${number},${number})`
// and   AnsiColor = 'ansi:black' | 'ansi:red' | ... (16 ANSI names)
```

The key architectural decision here: `TextStyles.color` is always a raw `Color` value, never a theme key. The comment in the source is explicit: _"Colors are raw values — theme resolution happens at the component layer."_ This means `styles.ts` and `colorize.ts` are completely unaware of themes. They are pure mechanics. Only the component layer (via `design-system/color.ts`) bridges from theme tokens to raw colors.

### The Styles -> Yoga Mapping

The default export of `styles.ts` is a function that applies a `Styles` object onto a `LayoutNode` (Yoga layout engine). This is what Ink calls when you write `<Box flexDirection="row" padding={2}>`:

```typescript
const styles = (
  node: LayoutNode,
  style: Styles = {},
  resolvedStyle?: Styles,  // Full current style, for diff application
): void => {
  applyPositionStyles(node, style)
  applyOverflowStyles(node, style)
  applyMarginStyles(node, style)
  applyPaddingStyles(node, style)
  applyFlexStyles(node, style)
  applyDimensionStyles(node, style)
  applyDisplayStyles(node, style)
  applyBorderStyles(node, style, resolvedStyle)
  applyGapStyles(node, style)
}
```

The `resolvedStyle` parameter exists specifically for `applyBorderStyles`. When a style update is applied as a diff (only changed properties), `borderStyle` might be in the diff but `borderTop` might not be — because it didn't change. The resolved style carries the previous full value so the function can correctly set all four border edges even when the diff is partial.

One notable property: `noSelect`. This controls whether a box's cells are excluded from text selection in fullscreen mode. The `'from-left-edge'` variant extends the exclusion from column 0 to the box's right edge for every row — specifically designed so that clicking and dragging over a diff panel doesn't accidentally copy line-number prefixes and diff sigils into the clipboard.

---

## S6 design-system/color.ts — Bridging Themes to Raw Colors

This file is tiny but it is the architectural glue. It is the only place where theme key strings are resolved to raw color values:

```typescript
export function color(
  c: keyof Theme | Color | undefined,
  theme: ThemeName,
  type: ColorType = 'foreground',
): (text: string) => string {
  return text => {
    if (!c) return text

    // Raw color values bypass theme lookup entirely
    if (
      c.startsWith('rgb(') || c.startsWith('#') ||
      c.startsWith('ansi256(') || c.startsWith('ansi:')
    ) {
      return colorize(text, c, type)
    }

    // Theme key -> raw color -> chalk output
    return colorize(text, getTheme(theme)[c as keyof Theme], type)
  }
}
```

The return value is a curried function, not a string. This means components can create colorizers once (e.g., at the top of a render function) and reuse them across multiple text strings, avoiding repeated theme lookups. The check for raw color prefixes means you can pass either a theme key like `"claude"` or a raw color like `"rgb(215,119,87)"` — both work transparently.

---

## S7 The /theme and /color Commands

### The /theme Command

The `/theme` command renders a full interactive `ThemePicker` component inside a `Pane` (wrapped with `color="permission"` — which is the blue/purple permission-request color, making the picker visually distinct from normal output):

```typescript
// commands/theme/theme.tsx
export const call: LocalJSXCommandCall = async (onDone, _context) => {
  return <ThemePickerCommand onDone={onDone} />
}

function ThemePickerCommand({ onDone }: Props) {
  const [, setTheme] = useTheme()

  return (
    <Pane color="permission">
      <ThemePicker
        onThemeSelect={setting => {
          setTheme(setting)
          onDone(`Theme set to ${setting}`)
        }}
        onCancel={() => onDone('Theme picker dismissed', { display: 'system' })}
        skipExitHandling={true}
      />
    </Pane>
  )
}
```

The `ThemePicker` component itself (in `components/ThemePicker.tsx`) provides live preview — you can arrow through themes and see the UI re-render before committing. This works via a `usePreviewTheme()` hook that sets a temporary theme state distinct from the saved setting. Pressing Escape cancels and restores the previous theme; pressing Enter saves it.

### The /color Command

The `/color` command is for sub-agent sessions only — it sets the color of the prompt bar for the current session, creating visual differentiation when multiple Claude Code agents are running simultaneously:

```typescript
// commands/color/color.ts
export async function call(onDone, context, args) {
  // Teammates cannot set their own color — only the team leader assigns them
  if (isTeammate()) {
    onDone('Cannot set color: This session is a swarm teammate...', { display: 'system' })
    return null
  }

  const colorArg = args.trim().toLowerCase()

  // 'default', 'reset', 'none', 'gray', 'grey' all reset to gray
  if (RESET_ALIASES.includes(colorArg)) {
    await saveAgentColor(sessionId, 'default', fullPath)
    // Updates AppState for immediate effect
    context.setAppState(prev => ({
      ...prev,
      standaloneAgentContext: { ...prev.standaloneAgentContext, color: undefined }
    }))
    return null
  }

  // Valid colors: AGENT_COLORS = ['red','blue','green','yellow','purple','orange','pink','cyan']
  await saveAgentColor(sessionId, colorArg, fullPath)
  context.setAppState(prev => ({
    ...prev,
    standaloneAgentContext: { ...prev.standaloneAgentContext, color: colorArg }
  }))
}
```

**Key detail**

The color is saved to the transcript file (`saveAgentColor(sessionId, colorArg, fullPath)`) for persistence across session restarts, and also applied immediately via `setAppState`. The "default" sentinel is not an empty string — it uses the literal string `"default"` so that the truthiness guard in `sessionStorage.ts` still persists the reset. An empty string would be falsy and might not be written.

---

## S8 AgentColorManager — Sub-agent Visual Identity

In multi-agent (swarm) sessions, Claude Code needs to visually distinguish agents from each other. The `agentColorManager.ts` handles the mapping from agent type strings to theme color slots:

```typescript
export const AGENT_COLORS: readonly AgentColorName[] = [
  'red', 'blue', 'green', 'yellow',
  'purple', 'orange', 'pink', 'cyan'
]

export const AGENT_COLOR_TO_THEME_COLOR = {
  red:    'red_FOR_SUBAGENTS_ONLY',
  blue:   'blue_FOR_SUBAGENTS_ONLY',
  // ... maps human-readable name -> Theme key
} as const satisfies Record<AgentColorName, keyof Theme>
```

The `satisfies` constraint is the clever part — it ensures at compile time that every entry in the map points to a valid key of the `Theme` type, without widening the type of the constant to `Record<AgentColorName, keyof Theme>`. If someone adds a new agent color but forgets to add a corresponding `_FOR_SUBAGENTS_ONLY` slot to the `Theme` type, the build fails.

The `general-purpose` agent type returns `undefined` from `getAgentColor` — it intentionally gets no color, because a general-purpose session is not visually differentiated. Only specialized agent types (code-review, testing, etc.) get assigned colors from the pool.

---

## S9 Full Color Resolution Data Flow

```
flowchart TD
A[User selects theme via /theme] --> B[useTheme setTheme called]
B --> C[ThemeSetting saved to settings.json]
C --> D[ThemeName resolved at runtime auto -> dark or light]
D --> E[getTheme ThemeName returns Theme object]
E --> F[Component calls color fn from design-system/color.ts]
F --> G{Is c a raw color? starts with rgb/hash/ansi}
G -- Yes --> H[colorize directly]
G -- No --> I[getTheme theme key lookup raw value]
I --> H[colorize raw color string, type]
H --> J{chalk.level}
J -- 3 truecolor --> K[chalk.rgb / chalk.hex]
J -- 2 256-color --> L[chalk.ansi256]
J -- 0/1 no color --> M[plain string]
K --> N[ANSI escape sequence to terminal]
L --> N
M --> N
subgraph Module Load Time
O[boostChalkLevelForXtermJs TERM_PROGRAM=vscode AND level=2 -> level=3]
P[clampChalkLevelForTmux TMUX AND level gt 2 -> level=2]
O --> P
end
```

---

## S10 Deep Dives

### Why RGB strings instead of TypeScript color objects?

The color system stores all color values as strings (`"rgb(215,119,87)"`, `"#d77757"`, `"ansi:red"`) rather than structured objects. This might look like a code smell but it has real advantages:

*   The `Theme` type is just `{ [key: string]: string }` — trivially serializable to JSON for config storage.
*   The `Color` TypeScript type uses template literal types (`` `rgb(${number},${number},${number})` ``) to get compile-time validation without runtime parsing.
*   The string prefix (`rgb(`, `#`, `ansi:`) is a self-describing discriminator — code can branch on it without a separate type tag.
*   Chalk already expects strings (hex, rgb, ansi color names) — wrapping in objects would add an unwrapping step with no benefit.

The one cost: regex parsing in `colorize()` for every color application. But since color application happens in a render loop that's already doing terminal I/O, the regex cost is negligible compared to the I/O.

### The shimmer animation pattern

Many theme tokens come in pairs: `claude` / `claudeShimmer`, `inactive` / `inactiveShimmer`, and so on. The shimmer variants are always slightly lighter (in dark mode) or slightly more saturated — tuned so that when a component oscillates between the two values, the transition reads as a "breathing" or "pulse" animation rather than a harsh blink.

For example: `claude = rgb(215,119,87)` (Claude orange) and `claudeShimmer = rgb(235,159,127)` — 20 units brighter on each channel, enough to be visually distinct but not enough to look like a different color.

The rainbow colors for the `ultrathink` keyword follow the same pattern: seven hues x two weights (base + shimmer) = 14 theme slots. When Claude Code detects the word "ultrathink" in a message, it cycles through rainbow colors. The shimmer variants allow the cycling to include alternating intensity, making the effect more visually dynamic.

### Apple Terminal and the 256-color fallback

While `colorize.ts` handles the VS Code boost and tmux clamp, there's a separate Apple Terminal handling in `utils/theme.ts` itself:

```typescript
// Create a chalk instance with 256-color level for Apple Terminal
// Apple Terminal doesn't handle 24-bit color escape sequences well
const chalkForChart =
  env.terminal === 'Apple_Terminal'
    ? new Chalk({ level: 2 }) // 256 colors
    : chalk

export function themeColorToAnsi(themeColor: string): string {
  const rgbMatch = themeColor.match(/rgb\(\s?(\d+),\s?(\d+),\s?(\d+)\s?\)/)
  if (rgbMatch) {
    const colored = chalkForChart.rgb(r, g, b)('X')
    return colored.slice(0, colored.indexOf('X'))
  }
}
```

This function is used specifically for `asciichart` rendering (cost/token usage graphs in the UI). Rather than relying on chalk's global level detection, it creates a separate `Chalk` instance locked to level 2 for Apple Terminal. The "extract escape sequence" trick — rendering a single character 'X' and slicing off everything before it — is a clever way to get the opening SGR sequence without chalk exposing that as a public API.

### Why the /color command is forbidden for swarm teammates

In a swarm (multi-agent) session, there is a "team leader" Claude Code instance and one or more "teammate" instances. The team leader assigns colors to teammates via the `AgentColorManager` — this is how the UI can color-code which agent is speaking in a multi-agent transcript.

If a teammate could call `/color` itself, it could conflict with or override the team leader's color assignment, breaking the visual consistency of the swarm display. The guard in `color.ts`:

```typescript
if (isTeammate()) {
  onDone('Cannot set color: This session is a swarm teammate.
Teammate colors are assigned by the team leader.', { display: 'system' })
  return null
}
```

The `isTeammate()` check reads from `bootstrap/state.ts` — the session knows at startup whether it was launched as a teammate. This is set before any user interaction, so the guard is reliable even if the agent tries to `/color` as its first action.

---

## Key Takeaways

*   The theme system has three distinct phases: **palette definition** (six named `Theme` objects in `theme.ts`), **environment normalization** (chalk level clamping/boosting in `colorize.ts` at module load), and **rendering** (theme key -> raw color -> chalk output at component render time).
*   The two chalk level adjustments in `colorize.ts` — VS Code boost and tmux clamp — fire once at import time and affect all subsequent color output. Their ordering is deliberate: boost first so tmux-inside-vscode re-clamps correctly.
*   Theme resolution is separated from color rendering: `styles.ts` and `colorize.ts` are completely unaware of themes. Only `design-system/color.ts` bridges theme keys to raw values.
*   The daltonized themes substitute blue for green in all semantic success/diff-added tokens — a systematic accessibility decision, not just individual color tweaks.
*   The `_FOR_SUBAGENTS_ONLY` and `Shimmer` naming conventions in the `Theme` type are intentional guardrails, enforced by naming discipline rather than runtime checks.
*   The `/color` command saves to the transcript file using the sentinel string `"default"` (not empty string) to ensure the reset persists across session restarts.

---

## Check Your Understanding

**Q1. What problem does `boostChalkLevelForXtermJs()` solve?**

A) VS Code terminals don't support any ANSI escape codes at all
B) chalk auto-detects VS Code as level 2 (256-color) causing brand colors to be cube-quantized to wrong shades
C) VS Code terminals show colors inverted compared to other terminals
D) chalk crashes when TERM_PROGRAM is set to "vscode"

**Q2. Why does the tmux clamp run _after_ the VS Code boost?**

A) It doesn't matter — both functions read environment variables independently
B) So that a user running Claude Code inside VS Code inside tmux gets the tmux clamp (level 2) as the final state, because tmux's passthrough limitation is a harder constraint than VS Code's preference for truecolor
C) The VS Code boost is only applied when chalk.level is 0, so it must run first
D) tmux clamp requires knowing if VS Code boosted so it can skip itself

**Q3. In the daltonized themes, why is the `success` color changed to _blue_ instead of just a different shade of green?**

A) Blue is more visible on dark backgrounds than green
B) Deuteranopia (green-weakness) makes it impossible to distinguish any green from background, regardless of shade
C) Deuteranopia affects the green channel, so two colors that differ only in green amount are indistinguishable — switching to blue (a separate channel) ensures the semantic distinction is perceivable
D) The terminal's color palette doesn't include enough green shades for fine-grained differentiation

**Q4. Why does `design-system/color.ts` return a _curried function_ rather than a colored string directly?**

A) To allow lazy evaluation — the color only resolves when the terminal is ready
B) So components can create the colorizer once (with theme lookup) and apply it to multiple strings, avoiding repeated theme lookups per string
C) TypeScript requires currying for functions that accept union types
D) Because chalk itself is curried and the interface must match

**Q5. When `/color reset` is called, the code saves `"default"` to the transcript instead of an empty string. Why?**

A) An empty string would crash JSON serialization
B) The word "default" is a valid CSS color name
C) A truthiness guard in sessionStorage.ts would skip writing an empty string, so the reset would not persist across session restarts
D) `saveAgentColor` only accepts strings with at least 4 characters

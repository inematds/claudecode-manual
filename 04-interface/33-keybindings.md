# The Keybindings System — Lesson 33

## Overview

Claude Code processes keypresses through a five-stage pipeline: terminal decoding converts escape sequences to structured key objects, binding configuration merges defaults with user overrides, key matching normalizes modifiers, chord resolution handles multi-step sequences, and React dispatch invokes handlers through hooks.

## Stage 1: Terminal Byte Decode

The `parse-keypress.ts` module converts terminal escape sequences into `ParsedKey` objects containing:
- Key name (`'enter'`, `'escape'`, `'up'`, etc.)
- Boolean flags for modifiers: `ctrl`, `meta`, `shift`, `option`, `super`
- Function key indicator
- Paste detection flag

Three keyboard protocols are supported:

**Legacy VT sequences** handle arrow and function keys with modifier information encoded as numeric parameters (e.g., `\x1b[1;5D` = Ctrl+Left).

**CSI u (Kitty protocol)** uses the format "ESC [ codepoint ; modifier u" and enables previously-impossible combinations like Shift+Enter and Ctrl+Space.

**xterm modifyOtherKeys** uses "ESC [ 27 ; modifier ; keycode ~" and requires explicit parsing ordering to avoid partial matches.

Modifier decoding follows XTerm's bitmask standard where modifier value = 1 + (shift?1:0) + (alt?2:0) + (ctrl?4:0) + (super?8:0).

Mouse events use a separate `ParsedMouse` type, though scroll wheel events remain as `ParsedKey` to work through the binding resolver.

## Stage 2: Default Binding Config

`defaultBindings.ts` exports `DEFAULT_BINDINGS` as an array of `KeybindingBlock` objects, each grouping bindings by UI context.

**Supported contexts** (18 total): Global, Chat, Autocomplete, Confirmation, Help, Transcript, HistorySearch, Task, ThemePicker, Settings, Tabs, Attachments, Footer, MessageSelector, DiffDialog, ModelPicker, Select, Plugin.

In Chat context, notable bindings include:
- `enter` -> `chat:submit`
- `escape` -> `chat:cancel`
- `ctrl+x ctrl+k` -> `chat:killAgents` (chord)
- `ctrl+s` -> `chat:stash`
- `ctrl+v` / `alt+v` -> `chat:imagePaste` (platform-specific)

**Platform-aware dynamic keys** are computed at module load. For example, Windows uses `alt+v` for image paste because Ctrl+V is system paste. Shift+Tab support depends on terminal VT mode, which Node.js enabled in version 24.2.0+ (or 22.17.0+); older Windows Terminal falls back to `meta+m`.

Feature flags conditionally include bindings like `KAIROS`, `QUICK_SEARCH`, and `VOICE_MODE` — if disabled, the binding doesn't appear in defaults.

## Stage 2b: Parsing Key Strings

`parser.ts` converts config strings like `"ctrl+shift+k"` into `ParsedKeystroke` structures containing canonical key name and boolean modifier flags.

**Modifier aliases** normalize user input:
- `ctrl`, `control` -> ctrl
- `alt`, `opt`, `option` -> alt
- `cmd`, `command`, `win` -> super
- Special aliases: `esc` -> escape, `return` -> enter, `space` -> (space character), unicode arrows work

**Chord parsing** splits sequences on whitespace. The edge case of a lone space character `" "` correctly maps to the space key, not an empty chord.

## Stage 3: Key Matching

`match.ts` bridges Ink's boolean flag interface to `ParsedKeystroke` format.

**Alt/meta unification**: Legacy terminals cannot distinguish Alt from Meta — both set `key.meta = true`. The matcher accepts if either `alt` OR `meta` is required in the target binding.

**Escape special case**: Pressing Escape sends `\x1b`, which Ink interprets as setting `key.meta = true` (a legacy artefact). To avoid requiring the meta modifier for plain escape bindings, the code explicitly strips meta when matching escape keys.

## Stage 4: Chord Resolution

`resolver.ts` returns one of five outcomes:
- `match` — binding fired, action ID provided
- `none` — no match, event propagates
- `unbound` — key explicitly null-unbound, event swallowed
- `chord_started` — first keystroke of multi-key sequence, state stored
- `chord_cancelled` — chord aborted (Escape or dead-end path)

**Last-wins override model**: User bindings are appended after defaults, so identical key+context pairs naturally supersede defaults.

**Chord prefix detection** checks whether longer chords in the active context use the current keystroke as a prefix. If so, the resolver enters `chord_started` state rather than firing a single-key binding — even if one exists. Only when no longer chord is possible does it fall back to exact matching.

The `chordWinners` map tracks longer chords and their actions (null for unbound overrides), ensuring null-unbinding a chord doesn't leave the prefix in chord-wait state.

## Stage 5: React Hooks & Context

Components register interest through `useKeybinding()` for single actions or `useKeybindings()` for action maps. Both use the same resolution path.

**False return convention**: A handler can return `false` to signal "not consumed — propagate further." `ScrollKeybindingHandler` uses this to allow child components to handle wheel events when content fits on screen.

**KeybindingContext** is the shared bus holding:
- `resolve()` function for chord resolution
- `setPendingChord()` for state management
- `getDisplayText()` for help UI formatting
- Parsed bindings array
- Active contexts set (components register/unregister on mount/unmount)
- `registerHandler()` returning cleanup function
- `invokeAction()` for direct action firing

## User Config & Hot-Reload

`~/.claude/keybindings.json` holds user customization:

```json
{
  "$schema": "https://www.schemastore.org/claude-code-keybindings.json",
  "$docs": "https://code.claude.com/docs/en/keybindings",
  "bindings": [
    {
      "context": "Chat",
      "bindings": {
        "ctrl+y": "chat:submit",
        "enter": null,
        "ctrl+shift+p": "command:compact"
      }
    }
  ]
}
```

**Merge strategy**: User bindings append after defaults. Linear scanning with last-match-wins means user entries supersede defaults naturally.

**Hot-reload pipeline**: `chokidar` watches the file with 500ms write-stabilise delay. File changes trigger `loadUserBindings()`, which updates the binding cache and emits `keybindingsChanged` signal. React UI re-renders with new bindings. File deletion resets to defaults.

The feature gate `isKeybindingCustomizationEnabled()` checks GrowthBook flag `tengu_keybinding_customization_release`; when false, only defaults are used.

Two loading functions exist: `loadKeybindingsSync()` for React's synchronous `useState` initializer, and `loadKeybindings()` for async chokidar callbacks.

## Validation & Reserved Shortcuts

`validate.ts` produces typed `KeybindingWarning` objects with severity (`'error'` or `'warning'`) and optional suggestion text.

**Five warning types**:
- `parse_error` — missing context field, malformed key strings
- `duplicate` — same key listed twice in one context block
- `reserved` — attempting to bind protected keys
- `invalid_context` — unknown context name
- `invalid_action` — malformed command string or wrong context

**Duplicate key detection** parses raw JSON strings with regex to catch silently-overwritten duplicates before they're lost to `JSON.parse()`.

**Reserved shortcuts** protect three categories:

Non-rebindable (error): `ctrl+c`, `ctrl+d`, `ctrl+m` are hardcoded. `ctrl+m` equals Enter in all terminals.

Terminal reserved (warn/error): `ctrl+z`, `ctrl+\` are intercepted by Unix kernel (SIGTSTP, SIGQUIT).

macOS only (error): `cmd+c`, `cmd+v`, `cmd+x`, `cmd+q`, `cmd+w`, `cmd+tab`, `cmd+space` intercepted by OS before terminal app.

**Intentional omission**: `ctrl+s` is NOT reserved because flow control is disabled on modern terminals, and Claude Code uses it for stashing.

## Display Formatting & Template Generation

`parser.ts` exports `keystrokeToDisplayString()` for platform-aware display:
- macOS shows `"opt"` for alt modifier
- Linux/Windows show `"alt"`

`getBindingDisplayText()` searches bindings in reverse so user overrides display instead of defaults in help UI.

The `/keybindings` command generates `~/.claude/keybindings.json` via `generateKeybindingsTemplate()`, filtering out `NON_REBINDABLE` keys and wrapping results in the schema envelope.

## Full Pipeline

Raw bytes from terminal -> parse-keypress.ts decodes escape sequences -> match.ts normalizes Ink flags -> resolver.ts applies chord logic and context matching -> React hooks invoke handlers -> stopImmediatePropagation() stops event bubbling.

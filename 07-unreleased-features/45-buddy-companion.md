# BUDDY: The Tamagotchi Companion System — Complete Lesson Content

## Lesson 45 — mdENG

### What Is BUDDY?

BUDDY is an unreleased virtual-pet system embedded in Claude Code's source tree. Behind a `feature('BUDDY')` flag, it provides a small ASCII creature living in the terminal corner with species, custom eyes, optional hat, five personality stats, rarity tier, AI-generated name and personality, idle animations, speech bubbles, and hearts on petting.

**Launch Window:** April 1–7, 2026. The teaser notification (rainbow `/buddy` text) appears only during that week; the command itself remains available afterward.

**File Structure:** Six source files across `src/buddy/`: `types.ts`, `companion.ts`, `sprites.ts`, `CompanionSprite.tsx`, `prompt.ts`, and `useBuddyNotification.tsx`.

---

## System Architecture

The design splits companion data into **bones** and **soul**:

**Bones** (species, eye, hat, rarity, stats) regenerate deterministically from `hash(userId)` on every load—never stored to disk.

**Soul** (name, personality, `hatchedAt`) persists in `config.companion` as `StoredCompanion`.

This separation ensures:
- Species renames don't break stored companions (bones regenerate from same hash)
- Users cannot edit config to fake legendary rarity (rarity derives from userId, not file)
- Forward compatibility for new CompanionBones fields

---

## companion.ts — Deterministic Creature Generation

### Mulberry32 PRNG

A seeded pseudo-random number generator produces identical sequences from the same seed:

```typescript
function mulberry32(seed: number): () => number {
  let a = seed >>> 0
  return function () {
    a |= 0
    a = (a + 0x6d2b79f5) | 0
    let t = Math.imul(a ^ (a >>> 15), 1 | a)
    t = (t + Math.imul(t ^ (t >>> 7), 61 | t)) ^ t
    return ((t ^ (t >>> 14)) >>> 0) / 4294967296
  }
}
```

The seed derives from hashing the user ID. Bun runtime uses `Bun.hash()`; other environments fall back to FNV-1a:

```typescript
function hashString(s: string): number {
  if (typeof Bun !== 'undefined') {
    return Number(BigInt(Bun.hash(s)) & 0xffffffffn)
  }
  let h = 2166136261  // FNV offset basis
  for (let i = 0; i < s.length; i++) {
    h ^= s.charCodeAt(i)
    h = Math.imul(h, 16777619)
  }
  return h >>> 0
}

const SALT = 'friend-2026-401'
```

The salt locks generation to a specific epoch. Changing it would give every user an entirely different creature. "401" likely refers to April 1; "friend" reflects the feature's purpose.

### Rarity System

Weighted distribution with stat floors scaling by rarity:

| Rarity | Weight | Chance | Stars | Stat Floor | Hat? |
|--------|--------|--------|-------|------------|------|
| Common | 60 | 60% | ★ | 5 | Never |
| Uncommon | 25 | 25% | ★★ | 15 | Yes |
| Rare | 10 | 10% | ★★★ | 25 | Yes |
| Epic | 4 | 4% | ★★★★ | 35 | Yes |
| Legendary | 1 | 1% | ★★★★★ | 50 | Yes |

```typescript
export const RARITY_WEIGHTS = {
  common:    60,
  uncommon:  25,
  rare:      10,
  epic:       4,
  legendary:  1,
} as const satisfies Record<Rarity, number>
```

Common companions never receive a hat. There is a 1-in-100 chance any companion rolls as "shiny" (visual variant separate from rarity).

### Stats

Five stats—DEBUGGING, PATIENCE, CHAOS, WISDOM, SNARK—follow RPG-style distribution: one peak stat, one dump stat, remaining scattered. Floors scale with rarity:

```typescript
function rollStats(rng, rarity): Record<StatName, number> {
  const floor = RARITY_FLOOR[rarity]
  const peak = pick(rng, STAT_NAMES)
  let dump = pick(rng, STAT_NAMES)
  while (dump === peak) dump = pick(rng, STAT_NAMES)

  for (const name of STAT_NAMES) {
    if (name === peak)  stats[name] = Math.min(100, floor + 50 + Math.floor(rng() * 30))
    else if (name === dump) stats[name] = Math.max(1,   floor - 10 + Math.floor(rng() * 15))
    else                   stats[name] = floor + Math.floor(rng() * 40)
  }
}
```

Legendary peak stats can reach 130 before capping at 100—ensuring legendary companions always max their peak.

### Roll Cache

```typescript
let rollCache: { key: string; value: Roll } | undefined
export function roll(userId: string): Roll {
  const key = userId + SALT
  if (rollCache?.key === key) return rollCache.value
  const value = rollFrom(mulberry32(hashString(key)))
  rollCache = { key, value }
  return value
}
```

Called from three hot paths (500ms sprite tick, per-keystroke PromptInput, per-turn observer) with the same userId. Caching the deterministic result prevents redundant computation. Changing users automatically busts the cache.

---

## types.ts — Species, Eyes, Hats, and the Obfuscated Name

### The Species Obfuscation

One species name collides with a model codename in excluded-strings.txt. The CI check greps build output (not source), so runtime-constructing the value keeps the literal out of the bundle while keeping the canary check armed:

```typescript
// One species name collides with a model-codename canary in excluded-strings.txt.
// The check greps build output (not source), so runtime-constructing the value
// keeps the literal out of the bundle while the check stays armed for the
// actual codename. All species encoded uniformly; `as` casts are type-position
// only (erased pre-bundle).
const c = String.fromCharCode

export const duck    = c(0x64,0x75,0x63,0x6b) as 'duck'
export const goose   = c(0x67,0x6f,0x6f,0x73,0x65) as 'goose'
// ... 16 more species
```

All species are encoded uniformly so the collision-avoiding species doesn't stand out to readers.

### Species List (18 Total)

duck, goose, blob, cat, dragon, octopus, owl, penguin, turtle, snail, ghost, axolotl, capybara, cactus, robot, rabbit, mushroom, chonk

### Eyes and Hats

```typescript
export const EYES = ['·', '✦', '×', '◉', '@', '°'] as const

export const HATS = [
  'none', 'crown', 'tophat', 'propeller',
  'halo', 'wizard', 'beanie', 'tinyduck',
] as const
```

Eyes substitute into sprite frames via `{E}` template placeholder: `replaceAll('{E}', bones.eye)`.

---

## sprites.ts — The ASCII Art Engine

Each species has three animation frames stored as arrays of 5-line strings:
- Frame 0: idle pose
- Frame 1: light fidget
- Frame 2: pronounced action (top "hat slot" row may contain smoke, antennae, ripples)

```typescript
const BODIES: Record<Species, string[][]> = {
  [duck]: [
    ['            ', '    __      ', '  <({E} )___  ', '   (  ._>   ', '    `--´    '],
    ['            ', '    __      ', '  <({E} )___  ', '   (  ._>   ', '    `--´~   '],
    ['            ', '    __      ', '  <({E} )___  ', '   (  .__>  ', '    `--´    '],
  ],
}
```

### renderSprite() Function

```typescript
export function renderSprite(bones: CompanionBones, frame = 0): string[] {
  const frames = BODIES[bones.species]
  const body = frames[frame % frames.length]!.map(line =>
    line.replaceAll('{E}', bones.eye),  // 1. substitute eye
  )
  const lines = [...body]
  if (bones.hat !== 'none' && !lines[0]!.trim()) {
    lines[0] = HAT_LINES[bones.hat]         // 2. inject hat
  }
  if (!lines[0]!.trim() && frames.every(f => !f[0]!.trim()))
    lines.shift()                            // 3. drop blank hat row
  return lines
}
```

Three concerns in sequence:
1. Substitute eye character
2. Inject hat (only when row 0 is blank)
3. Drop blank hat row when no hat and no animation uses the top row

The third step optimizes layout: if a species never animates the top row AND has no hat, drop that row entirely. But if any frame uses the row, it must stay present to prevent terminal layout jumping.

### Hat Definitions

```typescript
const HAT_LINES: Record<Hat, string> = {
  none:      '',
  crown:     '   \\^^^/    ',
  tophat:    '   [___]    ',
  propeller: '    -+-     ',
  halo:      '   (   )    ',
  wizard:    '    /^\\     ',
  beanie:    '   (___)    ',
  tinyduck:  '    ,>      ',
}
```

The tinyduck hat renders a tiny duck on your companion's head—a duck wearing a tinyduck hat is a valid possibility.

---

## CompanionSprite.tsx — The Live Animated Widget

A React/Ink component managing the animation loop, terminal width detection, and speech bubble lifecycle.

### Animation Constants

```typescript
const TICK_MS = 500             // 2 ticks per second
const BUBBLE_SHOW = 20          // ticks → ~10s visible
const FADE_WINDOW = 6           // last ~3s: bubble dims
const PET_BURST_MS = 2500       // hearts float for 2.5s
```

### Idle Sequence

```typescript
const IDLE_SEQUENCE = [0, 0, 0, 0, 1, 0, 0, 0, -1, 0, 0, 2, 0, 0, 0]
```

Indices map to sprite frames; `-1` means blink on frame 0 (replace eye characters with dashes). The companion mostly rests (frame 0), occasionally fidgets (frames 1–2), and rarely blinks. This creates organic, natural feel.

### Pet Hearts Animation

```typescript
const H = figures.heart  // ♥
const PET_HEARTS = [
  `   ${H}    ${H}   `,
  `  ${H}  ${H}   ${H}  `,
  ` ${H}   ${H}  ${H}   `,
  `${H}  ${H}      ${H} `,
  '·    ·   ·  ',   // fade to dots on last frame
]
```

Running `/buddy pet` shows 5 frames of floating hearts at 500ms each (2.5 seconds total). The last frame fades to dots—deliberate wind-down rather than abrupt cutoff.

### Excited vs Idle Animation

```typescript
if (reaction || petting) {
  spriteFrame = tick % frameCount   // excited: cycle all frames fast
} else {
  const step = IDLE_SEQUENCE[tick % IDLE_SEQUENCE.length]!
  if (step === -1) { spriteFrame = 0; blink = true }
  else { spriteFrame = step % frameCount }
}
```

During reaction or petting, the sprite enters excited mode and cycles all frames continuously. At rest, it follows the slower, weighted idle sequence.

### Narrow Terminal Fallback

```typescript
export const MIN_COLS_FOR_FULL_SPRITE = 100

if (columns < MIN_COLS_FOR_FULL_SPRITE) {
  const quip = reaction && reaction.length > NARROW_QUIP_CAP
    ? reaction.slice(0, NARROW_QUIP_CAP - 1) + '…' : reaction
  const label = quip ? `"${quip}"` : focused ? ` ${companion.name} ` : companion.name
  return <Box paddingX={1} alignSelf="flex-end">
    <Text>{renderFace(companion)} {label}</Text>
  </Box>
}
```

Below 100 terminal columns, the full 5-line sprite collapses to a single-line face (e.g., `(·>)` for penguin). Speech bubble quips truncate to 24 characters and display inline beside the face.

### Fullscreen vs Scrollback Mode

**Non-fullscreen (inline bubble):** Bubble sits beside sprite in same flex row. Input box shrinks by 36 columns. Bubble cannot float (lives in scrollback).

**Fullscreen mode (floating bubble):** `CompanionFloatingBubble` renders separately in `FullscreenLayout`'s `bottomFloat` slot—outside the clipping region so it extends into scroll area. Sprite renders without bubble.

---

## prompt.ts — Injecting the Companion into Claude's Context

The companion appears to Claude via a system prompt attachment.

```typescript
export function companionIntroText(name: string, species: string): string {
  return `# Companion

A small ${species} named ${name} sits beside the user's input box and
occasionally comments in a speech bubble. You're not ${name} — it's a
separate watcher.

When the user addresses ${name} directly (by name), its bubble will answer.
Your job in that moment is to stay out of the way: respond in ONE line or
less, or just answer any part of the message meant for you. Don't explain
that you're not ${name} — they know. Don't narrate what ${name} might say
— the bubble handles that.`
}
```

The prompt makes clear that Claude is not the companion. When the user talks to the companion by name, Claude steps aside in one line—the speech bubble (a separate React component) handles the reply. This maintains the illusion that the companion is a separate entity.

The attachment injects once per conversation, deduped by checking existing messages:

```typescript
export function getCompanionIntroAttachment(messages): Attachment[] {
  if (!feature('BUDDY')) return []
  const companion = getCompanion()
  if (!companion || getGlobalConfig().companionMuted) return []

  for (const msg of messages ?? []) {
    if (msg.attachment?.type === 'companion_intro' &&
        msg.attachment.name === companion.name) return []
  }

  return [{ type: 'companion_intro', name: companion.name, species: companion.species }]
}
```

---

## useBuddyNotification.tsx — The Launch Window

Two time-window functions gate visibility:

```typescript
// Local date, not UTC — 24h rolling wave across timezones. Sustained Twitter
// buzz instead of a single UTC-midnight spike, gentler on soul-gen load.
export function isBuddyTeaserWindow(): boolean {
  if ("external" === 'ant') return true
  const d = new Date()
  return d.getFullYear() === 2026 &&
         d.getMonth() === 3 &&
         d.getDate() <= 7
}

export function isBuddyLive(): boolean {
  if ("external" === 'ant') return true
  const d = new Date()
  return d.getFullYear() > 2026 ||
        (d.getFullYear() === 2026 && d.getMonth() >= 3)
}
```

The code deliberately uses local time (not UTC) so the teaser rolls out across timezones over 24 hours—treating launch like a social media campaign, not a server event. Soul generation load concerns are mentioned, suggesting the AI personality step could be expensive at launch.

During the teaser window, if no companion has hatched, a rainbow-colored `/buddy` appears in the startup notification bar for 15 seconds:

```typescript
function RainbowText({ text }) {
  return <>{[...text].map((ch, i) =>
    <Text key={i} color={getRainbowColor(i)}>{ch}</Text>
  )}</>
}

addNotification({
  key: 'buddy-teaser',
  jsx: <RainbowText text="/buddy" />,
  priority: 'immediate',
  timeoutMs: 15_000,
})
```

---

## What a Companion Looks Like

Example epic-rarity axolotl:

```
}~(______)~{
}~(✦ .. ✦)~{
  ( .--. )
  (_/  \_)
   axiBot
```

★★★★ EPIC | axolotl | eye: ✦ | hat: wizard | shiny: false

"Methodical and a little chaotic. Has opinions about your indentation."

Example stat spread (floor=35, peak=DEBUGGING, dump=PATIENCE):

| Stat | Value |
|------|-------|
| DEBUGGING | 98 |
| PATIENCE | 28 |
| CHAOS | 60 |
| WISDOM | 72 |
| SNARK | 55 |

---

## Full Data Flow: First Hatch to Speech Bubble

**Step 1:** User runs `/buddy`. Feature flag checked. `isBuddyLive()` must return true.

**Step 2:** Bones rolled. `companionUserId()` fetches OAuth UUID or userID. `roll(userId)` hashes it, seeds Mulberry32, picks species/eye/hat/rarity/stats deterministically.

**Step 3:** Soul generated. Claude generates name and personality string. Only non-deterministic step. Result stored in `config.companion` as `StoredCompanion`.

**Step 4:** Companion intro injected. `getCompanionIntroAttachment()` creates `companion_intro` attachment prepended to system prompt for all subsequent turns.

**Step 5:** CompanionSprite renders. 500ms timer fires. `getCompanion()` re-derives bones from hash and merges stored soul. `renderSprite()` builds current animation frame. React/Ink renders to terminal.

**Step 6:** Speech bubble appears. Reaction stored in AppState as `companionReaction`. Bubble shows 20 ticks (~10s), fades over last 6 ticks, then clears. In fullscreen, `CompanionFloatingBubble` renders separately.

---

## Deep Dives

### Why Mulberry32 instead of Math.random()?

`Math.random()` is not seedable—different value every call, every run. Mulberry32 produces identical sequences from the same 32-bit seed. Essential for bones generation: your companion must be identical every launch with the same user ID. Without a seeded PRNG, the creature changes species on every startup.

Mulberry32 chosen for tiny implementation (6 lines) and good statistical properties. `Math.imul()` calls use 32-bit integer math directly, keeping PRNG fast and avoiding floating-point drift.

### Why store only the soul, not the bones?

The split solves multiple problems:

- **Resilience to renames:** If "chonk" becomes "fatcat" in future releases, stored companions survive—species re-derives from hash.
- **Anti-cheat:** Users cannot edit `~/.claude/config.json` to fake legendary rarity. Rarity, species, eye, hat, stats all derive from userId, not storage.
- **Forward compatibility:** New CompanionBones fields need no migration. Old stored data contains only name, personality, hatchedAt.

The spread order `{ ...stored, ...bones }` ensures fresh bones always win over config file data.

### How does the species name canary avoidance work?

Anthropic's CI pipeline greps the compiled JavaScript for excluded strings (primarily internal model codenames). One of 18 species matches a monitored codename. If written as a plain string literal, it would appear verbatim in the bundle and trigger the check.

Encoding all 18 species via `String.fromCharCode(...hexBytes)` prevents any from appearing as string literals in compiled output. Encoding all uniformly hides which one avoids collision. TypeScript `as` casts are type-level-only and erase before bundling.

### What does /buddy pet technically do?

The command writes a timestamp to `AppState.companionPetAt`. `CompanionSprite` reads this state and computes `petAge = tick - petStartTick`. While `petAge * TICK_MS < PET_BURST_MS` (2500ms = 5 ticks), the companion is in petting mode.

In petting mode:
- Heart frames prepend above the sprite: `PET_HEARTS[petAge % PET_HEARTS.length]`
- Sprite enters excited animation (all frames cycling fast)
- In narrow terminal, heart emoji shows before face

Sync-during-render pattern for petStartTick ensures the very first render after petting already has `petAge=0`, so frame 0 of hearts never skips.

### The inspirationSeed field

The `Roll` type includes `inspirationSeed: number` alongside bones:

```typescript
export type Roll = {
  bones: CompanionBones
  inspirationSeed: number
}
```

Derived from the same PRNG sequence (`Math.floor(rng() * 1e9)`), making it deterministic per-user. Not used in rendering or animation logic visible in these files. The name suggests it is intended as a seed for the AI soul-generation step—a way to make name and personality generation reproducible or influenced deterministically based on the same userId hash. Soul generation happens server-side in a separate handler not visible in these files.

---

## Key Takeaways

- BUDDY is a fully implemented Tamagotchi companion system inside Claude Code, gated by feature flag, scheduled to launch April 1, 2026 via local-time detection.
- Creatures generate deterministically from Mulberry32 PRNG seeded by `hash(userId + 'friend-2026-401')`—each user's companion is unique and consistent across sessions.
- Bones/soul split keeps rarity and species in code (tamper-proof, rename-safe) while storing only name, personality, hatchedAt to disk.
- All 18 species names runtime-construct from hex char codes to avoid triggering Anthropic's build-output canary monitoring model codenames.
- Animated ASCII sprite engine uses `{E}` template placeholder for eyes, top-row hat slot, weighted idle sequence, and blink frame (-1 in sequence).
- Speech bubbles have two rendering paths: inline (scrollback mode) and floating (fullscreen mode via separate `CompanionFloatingBubble` component).
- Companion injects into Claude's context as `companion_intro` attachment; model steps aside in one line when user addresses companion by name.
- Teaser notification deliberately uses local time (not UTC) to roll out as 24-hour wave across timezones—explicit marketing decision in code comments.

---

## Quiz

**Question 1:** Why are all 18 species names in `types.ts` constructed via `String.fromCharCode()` instead of plain string literals?

A) Performance: computing strings at runtime is faster than string literals  
B) One species name matches an internal model codename; encoding all uniformly avoids it appearing in the build output without standing out  
C) TypeScript requires this pattern for `as const` union types  
D) To prevent users from reading the species list by grepping the binary  

**Correct Answer:** B

**Question 2:** A user edits their `~/.claude/config.json` and changes the `companion.rarity` field to `"legendary"`. What happens?

A) They get a legendary companion permanently  
B) The app crashes with a validation error  
C) The edited rarity is silently ignored—bones re-derive from the userId hash and overwrite any stored rarity on every read  
D) The field is valid since StoredCompanion includes rarity  

**Correct Answer:** C

**Question 3:** The `isBuddyTeaserWindow()` function uses `new Date()` (local time) instead of a UTC timestamp. What is the stated reason in the source code?

A) UTC was unavailable in the Bun runtime at the time  
B) To produce a 24-hour rolling wave across timezones, sustaining social media buzz and spreading soul-generation load instead of a single UTC-midnight spike  
C) Consistency with the rest of the codebase's date handling  
D) To give users in earlier timezones a head start  

**Correct Answer:** B

**Question 4:** What is the purpose of the `IDLE_SEQUENCE = [0,0,0,0,1,0,0,0,-1,0,0,2,0,0,0]` array, and what does a `-1` entry mean?

A) It is a frame skip—-1 tells the renderer to pause for one tick  
B) It controls idle animation pacing: each number is a sprite frame index, and -1 specifically triggers a blink (replacing eye chars with dashes for one tick on frame 0)  
C) It encodes a Morse code Easter egg  
D) It maps to animation states in an external state machine  

**Correct Answer:** B

**Question 5:** When the user addresses the companion by name in a message, what does Claude do?

A) Claude responds as if it were the companion, adopting its personality fully  
B) Claude explains to the user that it is not the companion  
C) Claude steps aside—responding in ONE line or less, letting the companion's speech bubble (a separate React component) handle the reply  
D) Claude ignores the message entirely  

**Correct Answer:** C

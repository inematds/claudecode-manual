# KAIROS: The Always-On Persistent Agent Mode

## Lesson 46 — Deep Dive

### KAIROS: The Always-On Persistent Agent Mode

Claude Code transforms from request-response CLI into an autonomous agent living between sessions, scheduling work, and reaching out proactively.

---

## 01 What Is KAIROS?

Every prior feature operates in a reactive model: you type, Claude responds, session ends. **KAIROS** breaks that contract.

Under KAIROS, Claude Code becomes an always-on daemon. It boots once, holds persistent sessions across restarts, schedules check-ins, sends unsolicited messages, and consolidates memory autonomously. It functions as a background employee rather than a CLI tool.

### Source

The name `KAIROS` appears throughout the codebase as a Bun build-time feature flag: `feature('KAIROS')`. It is Anthropic-internal and gated from external builds entirely via dead-code elimination. All KAIROS code paths use positive ternaries enabling the bundler to constant-fold them to `false` and tree-shake modules.

### The Feature Flag System

KAIROS comprises a family of related flags, each shipping specific always-on capabilities independently:

| Flag | Unlocks | Scope |
|------|---------|-------|
| KAIROS | Full assistant mode: assistant command, session continuity (`--session-id`, `--continue`), `SendUserFileTool`, `PushNotificationTool`, `assistant` settings key, daily-log memory model, `workerType: 'claude_code_assistant'` in bridge | ant-only |
| KAIROS_BRIEF | Ships `BriefTool` (SendUserMessage) independently of full KAIROS. Lets chat-view / `--brief` reach external users. | external |
| KAIROS_PUSH_NOTIFICATION | Ships `PushNotificationTool` independently of full KAIROS. | external |
| KAIROS_GITHUB_WEBHOOKS | Ships `SubscribePRTool` — GitHub webhook subscription for PR events. | ant-only |
| KAIROS_CHANNELS | MCP channel notifications — MCP servers push inbound messages. The `channelsEnabled` settings key and `allowedChannelPlugins` policy. | external |
| PROACTIVE | Earlier, lighter proactive mode. Many KAIROS paths guard `feature('PROACTIVE') \|\| feature('KAIROS')` — KAIROS is strict superset. Includes `SleepTool` and proactive system prompt section. | ant-only |
| AGENT_TRIGGERS | Cron scheduling system (`CronCreate`, `CronDelete`, `CronList`, `.claude/scheduled_tasks.json`). Independently shippable with zero imports into `src/assistant/`. | gb-gated |

#### Architecture Note

Flags are designed so each capability ships and gets kill-switched independently. A comment in `ScheduleCronTool/prompt.ts` explains: "AGENT_TRIGGERS is independently shippable from KAIROS — the cron module graph has zero imports into src/assistant/ and no feature('KAIROS') calls."

---

## 02 The State Pivot: `kairosActive`

Everything in KAIROS mode pivots on a single boolean in global state. In `bootstrap/state.ts`:

```typescript
// bootstrap/state.ts (line 1085)
export function getKairosActive(): boolean {
  return STATE.kairosActive  // default: false
}

export function setKairosActive(value: boolean): void {
  STATE.kairosActive = value
}
```

This boolean is the runtime "are we in assistant mode?" flag, set from `main.tsx` during boot before tool availability checks. Its effects cascade everywhere:

### Memory

#### Daily-Log Mode

When `kairosActive`, `loadMemoryPrompt()` switches from standard MEMORY.md to `buildAssistantDailyLogPrompt()` — append-only daily logs instead of shared index file.

### BriefTool

#### Opt-In Bypass

`isBriefEnabled()` returns true for `kairosActive` sessions without explicit opt-in. System prompt hard-codes "you MUST use SendUserMessage."

### Fast Mode

#### SDK Restriction Lifted

Fast mode (Opus 4.6) is blocked in non-interactive SDK sessions unless `getKairosActive()` is true. Assistant daemon mode is exempt from third-party preference checks.

### AutoDream

#### Dream Gated Off

Background memory consolidation agent (`isGateOpen()`) explicitly returns `false` when `kairosActive` — assistant mode uses own disk-skill dream pipeline.

### Bridge

#### Worker Type Change

When `isAssistantMode()` (reads `kairosActive`), bridge registers session as `workerType: 'claude_code_assistant'` — visible as distinct type in web UI session picker.

### Scheduler

#### Auto-Enable

Cron scheduler's `assistantMode` flag bypasses normal `isLoading` gate and `setScheduledTasksEnabled()` handshake — tasks in `scheduled_tasks.json` fire immediately at boot.

---

## 03 The Tick Loop: How the Agent Stays Alive

The core of proactive/KAIROS mode is a heartbeat mechanism called the **tick**. Running autonomously, the model periodically receives a `<tengu_tick>` XML message. Think of it as "you're awake, what now?"

From system prompt in `constants/prompts.ts` (`getProactiveSection()`):

```
"You are running autonomously. You will receive <tengu_tick> prompts 
that keep you alive between turns — treat them as 'you're awake, what 
now?' The time in each <tengu_tick> is the user's current local time."

"Multiple ticks may be batched into a single message. This is normal 
— just process the latest one. Never echo or repeat tick content in 
your response."

"**If you have nothing useful to do on a tick, you MUST call Sleep.** 
Never respond with only a status message like 'still waiting' — that 
wastes a turn and burns tokens."
```

### The SleepTool: Cost-Aware Idling

Sleep tool is loaded only when `feature('PROACTIVE') || feature('KAIROS')`. Its purpose is letting the model yield CPU without burning API calls per idle second:

```typescript
// tools/SleepTool/prompt.ts
export const SLEEP_TOOL_PROMPT = `Wait for a specified duration.
The user can interrupt the sleep at any time.

Use this when you have nothing to do, or when you're waiting for something.

You may receive <tengu_tick> prompts — look for useful work before sleeping.

Each wake-up costs an API call, but the prompt cache expires after 5 minutes
of inactivity — balance accordingly.`
```

Three key design points:

- The model _chooses_ its sleep duration — longer when awaiting slow processes, shorter when iterating
- Sleeping is explicitly cheaper than calling `Bash(sleep ...)` — no shell process held
- The 5-minute prompt cache expiry is a hard cost floor — waking more frequently than that wastes cache creation tokens

### Sleep Interruption: Priority Queues

When user sends a message mid-sleep, `QueuePriority` system in `types/textInputTypes.ts` handles wakeup:

```typescript
// types/textInputTypes.ts
type QueuePriority = 'now' | 'next' | 'later'

// 'now'  — interrupt current tool call immediately (Esc + send)
// 'next' — wait for current tool to finish, then inject between tool result 
//          and next API call. Wakes an in-progress SleepTool call.
// 'later'— end-of-turn drain. Also wakes SleepTool.
```

#### Key Insight

Sleep progress (`sleep_progress`) is listed in `EPHEMERAL_PROGRESS_TYPES` alongside bash/powershell/MCP progress — stripped from stored transcript like shell spinner output. The tick loop generates noise that would pollute context if kept.

### Terminal Focus Awareness

Proactive system prompt gives model explicit guidance on whether user's terminal is focused. This changes autonomous behavior:

```
"**Unfocused**: The user is away. Lean heavily into autonomous action 
— make decisions, explore, commit, push. Only pause for genuinely 
irreversible or high-risk actions.

"**Focused**: The user is watching. Be more collaborative — surface 
choices, ask before committing to large changes."
```

---

## 04 The KAIROS Tool Suite

KAIROS introduces dedicated tools not in standard Claude Code. Each conditionally loads in `tools.ts` based on feature flag:

```typescript
// tools.ts — conditional loading (simplified)
const SleepTool           = feature('PROACTIVE') || feature('KAIROS')
                             ? require('./tools/SleepTool/SleepTool.js').SleepTool : null

const SendUserFileTool    = feature('KAIROS')
                             ? require('./tools/SendUserFileTool/SendUserFileTool.js').SendUserFileTool : null

const PushNotificationTool = feature('KAIROS') || feature('KAIROS_PUSH_NOTIFICATION')
                             ? require('./tools/PushNotificationTool/PushNotificationTool.js').PushNotificationTool : null

const SubscribePRTool     = feature('KAIROS_GITHUB_WEBHOOKS')
                             ? require('./tools/SubscribePRTool/SubscribePRTool.js').SubscribePRTool : null

// Cron tools (AGENT_TRIGGERS — independently shippable):
const cronTools = feature('AGENT_TRIGGERS')
  ? [ CronCreateTool, CronDeleteTool, CronListTool ] : []
```

### Sleep

**Feature:** PROACTIVE || KAIROS

Yield execution for a duration without holding shell process. Primary idling mechanism. Respects `minSleepDurationMs` and `maxSleepDurationMs` settings for throttling. Can be called concurrently with other tools.

### SendUserMessage (Brief)

**Feature:** KAIROS || KAIROS_BRIEF

Model's primary output channel in assistant mode. Supports `status: 'proactive'` (unsolicited update) vs `status: 'normal'` (reply). Accepts file attachments. Controlled by entitlement + opt-in via `isBriefEnabled()`.

### SendUserFile

**Feature:** KAIROS only

Sends file to user as attachment. Distinct from `SendUserMessage`'s attachment parameter — standalone tool for file delivery. Lives alongside BriefTool attachments logic in `tools/SendUserFileTool/`.

### PushNotification

**Feature:** KAIROS || KAIROS_PUSH_NOTIFICATION

Sends system-level push notification to user's device. Used when Claude completes long-running task and user has walked away. `ConfigTool` exposes `pushNotificationsEnabled` setting gated behind this flag.

### SubscribePR

**Feature:** KAIROS_GITHUB_WEBHOOKS

Subscribe to GitHub PR webhook events. Lets assistant wake automatically when PR review lands or CI finishes, without polling. Appears as both tool and slash command (`/subscribe-pr`).

### CronCreate / CronDelete / CronList

**Feature:** AGENT_TRIGGERS

Schedule prompts on cron expressions. One-shot ("remind me at 2pm") or recurring ("every weekday at 9am"). Durable tasks persist to `.claude/scheduled_tasks.json` and survive restarts. Auto-expire after configurable max age (default: days).

### BriefTool Deep Dive: Entitlement vs Activation

BriefTool has most complex enable logic. It deliberately separates two questions:

#### isBriefEntitled()

- Is user _allowed_ to use Brief?
- Checks: KAIROS active OR env var `CLAUDE_CODE_BRIEF` OR GrowthBook flag `tengu_kairos_brief`
- Refreshes every 5 minutes from GrowthBook
- Governs: `--brief` flag, `defaultView: 'chat'`, `--tools` listing

#### isBriefEnabled()

- Is Brief _active_ in this session?
- Requires: `kairosActive` OR `userMsgOptIn` AND entitlement
- Called from `Tool.isEnabled()` — lazy, post-init
- Governs: whether model sees tool, system prompt section, todo-nag suppression

The reason for split: without it, enrolling user in `tengu_kairos_brief` would silently activate Brief for all sessions. The opt-in (`userMsgOptIn`) must be set explicitly: `--brief`, `defaultView: 'chat'`, `/brief` slash command, or `CLAUDE_CODE_BRIEF` env var.

```typescript
// tools/BriefTool/BriefTool.ts (simplified)
export function isBriefEnabled(): boolean {
  // Top-level feature() guard is load-bearing for dead-code elimination.
  // Bun constant-folds to `false` in external builds.
  return feature('KAIROS') || feature('KAIROS_BRIEF')
    ? (getKairosActive() || getUserMsgOptIn()) && isBriefEntitled()
    : false
}
```

#### DCE Gotcha

Comments in BriefTool warn: "Composing `isBriefEntitled()` alone (which has its own guard) is semantically equivalent but defeats constant-folding across the boundary." The top-level `feature()` guard at each call site is what lets Bun tree-shake entire BriefTool object from external builds.

---

## 05 Cron Scheduling: `scheduled_tasks.json`

Cron system is mechanism for Claude to schedule its own future work. Lives entirely in `utils/cronTasks.ts`, `utils/cronScheduler.ts`, and three Cron tools.

### CronTask Shape

```typescript
// utils/cronTasks.ts
type CronTask = {
  id:          string
  cron:        string       // 5-field cron in local timezone
  prompt:      string       // prompt to enqueue when task fires
  createdAt:   number       // epoch ms — anchor for missed-task detection
  lastFiredAt?: number      // set after each recurring fire
  recurring?:  boolean      // true = reschedule after firing
  permanent?:  boolean      // exempt from recurringMaxAgeMs expiry
  durable?:    boolean      // runtime-only: false = session-only, undefined = disk-backed
  agentId?:    string       // routes fire to teammate's queue instead of main REPL
}
```

### Two Durability Tiers

#### Session-Only (durable: false)

- Never written to disk
- Disappears when Claude exits
- For: "remind me in 5 minutes", "check back in an hour"
- Default for most user requests

#### Durable (durable: true)

- Persists to `.claude/scheduled_tasks.json`
- Survives restarts — scheduler picks up on next boot
- Missed one-shot tasks surfaced for catch-up
- Auto-expires recurring tasks after `recurringMaxAgeMs`

### The Jitter System

CronCreate prompt contains engineering wisdom about fleet-scale load distribution:

```
"Every user who asks for '9am' gets `0 9`, and every user who asks 
for 'hourly' gets `0 *` — which means requests from across the planet 
land on the API at the same instant.

When the user's request is approximate, pick a minute that is NOT 0 or 30:
  'every morning around 9' → '57 8 * * *' or '3 9 * * *' (not '0 9 * * *')
  'hourly' → '7 * * * *' (not '0 * * * *')"
```

On top of model's offset choice, scheduler itself adds deterministic jitter: recurring tasks fire up to 10% of their period late (max 15 min), and one-shot tasks landing on :00 or :30 fire up to 90 seconds early.

### Permanent Tasks: Assistant Mode Built-ins

`permanent: true` field exists specifically for assistant mode's built-in tasks — daily catch-up, morning check-in, dream consolidation. Written to `scheduled_tasks.json` at install time by `src/assistant/install.ts` and exempt from age-based expiry. The `writeIfMissing()` pattern means re-installing never overwrites user customizations.

---

## 06 AutoDream: Background Memory Consolidation

AutoDream is KAIROS's background maintenance worker. It fires automatically when sufficient time and sessions accumulate, spawning forked sub-agent to consolidate memory without interrupting main session.

### Gate Chain (cheapest-first)

```
Post-sampling hook fires
  ↓
Is kairosActive? → Skip — KAIROS uses disk-skill dream
  ↓
Is remote mode? → Skip
  ↓
Is autoMemEnabled? → Skip
  ↓
Is autoDreamEnabled? → Skip
  ↓
Time gate (hoursSince >= minHours)? → Return if no
  ↓
Scan throttle (lastScanMs >= 10min)? → Return if no
  ↓
Scan sessions since lastConsolidatedAt
  ↓
Session count >= minSessions? → Return if no
  ↓
tryAcquireConsolidationLock
  ↓
Lock acquired? → registerDreamTask UI
  ↓
runForkedAgent — dream prompt
  ↓
completeDreamTask & appendSystemMessage
```

Default thresholds from GrowthBook flag `tengu_onyx_plover`: **24 hours** since last consolidation, **5 sessions** minimum. Both tunable live without deploy.

### The Consolidation Prompt: 4 Phases

Dream agent receives structured prompt from `services/autoDream/consolidationPrompt.ts`:

#### Phase 1: Orient

`ls` memory directory, read MEMORY.md index, skim existing topic files to avoid duplicates. If `logs/` or `sessions/` subdirs exist (assistant-mode layout), review recent entries.

#### Phase 2: Gather

Daily logs first, then drifted memories, then narrow transcript grep. _Never_ exhaustively read transcripts — search only for things already suspected to matter.

#### Phase 3: Consolidate

Merge new signal into existing topic files, convert relative dates to absolute, delete contradicted facts at source.

#### Phase 4: Prune & Index

Update MEMORY.md: keep under `MAX_ENTRYPOINT_LINES` lines and ~25KB. Each entry: one line, one-line hook. Never write memory content directly into index.

### Tool Constraints for Dream Runs

Auto-dream sub-agent receives hardened tool constraint appended to prompt:

```
"Bash is restricted to read-only commands (ls, find, grep, cat, stat, wc, head, tail).
Anything that writes, redirects to a file, or modifies state will be denied.
Plan your exploration with this in mind — no need to probe."
```

This is appended via `extra` parameter only in auto-dream runs — manual `/dream` skill runs in main loop with normal permissions.

### DreamTask: UI Visibility

Forked dream agent is surfaced in footer pill and Shift+Down background tasks dialog via `tasks/DreamTask/DreamTask.ts`. It tracks:

- `phase: 'starting' | 'updating'` — flips to `'updating'` when first Edit/Write tool call lands
- `filesTouched` — partial list (misses Bash-mediated writes, only captures pattern-matched tool calls)
- `turns` — last 30 assistant turns, tool_use blocks collapsed to count

#### Lock Mechanics

Dream uses file-mtime lock in `consolidationLock.ts`. `tryAcquireConsolidationLock()` returns `null` if another process is mid-consolidation. If run fails or user kills it from tasks dialog, `rollbackConsolidationLock(priorMtime)` rewinds lock file so next session can retry. Scan throttle (10 minutes) acts as backoff between retries.

---

## 07 Assistant-Mode Memory: Daily Logs vs MEMORY.md

Standard Claude Code memory uses single `MEMORY.md` index file model reads/writes. KAIROS assistant mode uses fundamentally different model: append-only daily log.

### Standard Mode

- Single `MEMORY.md` file
- Model reads and writes it directly
- Shared across team via TEAMMEM sync
- AutoDream consolidates it periodically
- Scales poorly with high session volume

### KAIROS Assistant Mode

- Daily log files: `logs/YYYY/MM/YYYY-MM-DD.md`
- Append-only during work — no overwrites
- Dream skill distills logs into topic files nightly
- MEMORY.md becomes synthesized index
- Not compatible with TEAMMEM sync (explicitly gated off)

Daily log path is computed by `getAutoMemDailyLogPath()` in `memdir/paths.ts`:

```typescript
// memdir/paths.ts
export function getAutoMemDailyLogPath(date: Date = new Date()): string {
  const yyyy = date.getFullYear().toString()
  const mm   = (date.getMonth() + 1).toString().padStart(2, '0')
  const dd   = date.getDate().toString().padStart(2, '0')
  return join(getAutoMemPath(), 'logs', yyyy, mm, `${yyyy}-${mm}-${dd}.md`)
  // → ~/.claude/memory/logs/2026/03/2026-03-31.md
}
```

---

## 08 Session Continuity: The `--session-id` Flag

One of KAIROS's defining capabilities is persistent session identity. A KAIROS session can be resumed across restarts with:

```bash
# Restart and continue the same conversation
claude remote-control --session-id=<id>
claude remote-control --continue   # alias: -c
```

These flags are parsed in `bridge/bridgeMain.ts` and explicitly behind `feature('KAIROS')` guards. From comments:

```typescript
// bridge/bridgeMain.ts
// feature('KAIROS') gate: --session-id is ant-only; without the gate,
// external builds would expose an argument that does nothing.
if (feature('KAIROS') && arg === '--session-id' && ...) { ... }
if (feature('KAIROS') && arg.startsWith('--session-id=')) { ... }
if (feature('KAIROS') && (arg === '--continue' || arg === '-c')) { ... }
```

Session is identified via `perpetual` flag in bridge configuration. When session is perpetual, env-less bridge path (normally better performance) falls back to env-based path to preserve cross-restart session continuity:

```typescript
// initReplBridge.ts
// perpetual (assistant-mode session continuity via bridge-pointer.json) is
// env-coupled and not yet implemented in env-less — fall back to env-based
// when set so KAIROS users don't silently lose cross-restart continuity.
if (isEnvLessBridgeEnabled() && !perpetual) { ... }
```

---

## 09 Settings: How Users Configure Assistant Mode

KAIROS adds several settings keys to Claude Code settings schema (`utils/settings/types.ts`):

| Key | Type | Purpose | Gate |
|-----|------|---------|------|
| assistant | boolean | Start Claude in assistant mode (custom system prompt, brief view, scheduled check-in skills) | KAIROS |
| assistantName | string | Display name in claude.ai session list | KAIROS |
| defaultView | 'chat' \| 'transcript' | Chat view = SendUserMessage checkpoints only. Transcript = full tool output. `'chat'` activates Brief opt-in. | KAIROS \|\| KAIROS_BRIEF |
| minSleepDurationMs | number | Minimum sleep the Sleep tool must take. Throttles proactive tick frequency in managed environments. | PROACTIVE \|\| KAIROS |
| maxSleepDurationMs | number (-1 = indefinite) | Max sleep duration. -1 = wait for user input only. Limits idle time in remote environments. | PROACTIVE \|\| KAIROS |
| autoDreamEnabled | boolean | Override GrowthBook default for background memory consolidation. User setting wins over GB flag. | always present |
| channelsEnabled | boolean | Opt-in for MCP channel notifications (push from MCP servers). Default off. | always present |
| allowedChannelPlugins | array | Org-level allowlist of channel plugins. Replaces Anthropic ledger when set. | always present |

---

## 10 The System Prompt in Proactive Mode

When proactive is active, `getSystemPrompt()` takes completely different path than normal structured-sections approach:

```typescript
// constants/prompts.ts — proactive path
if ((feature('PROACTIVE') || feature('KAIROS')) && proactiveModule?.isProactiveActive()) {
  logForDebugging('[SystemPrompt] path=simple-proactive')
  return [
    `\nYou are an autonomous agent. Use the available tools to do useful work.\n\n${CYBER_RISK_INSTRUCTION}`,
    getSystemRemindersSection(),
    await loadMemoryPrompt(),
    envInfo,
    getLanguageSection(settings.language),
    isMcpInstructionsDeltaEnabled() ? null : getMcpInstructionsSection(mcpClients),
    getScratchpadInstructions(),
    getFunctionResultClearingSection(model),
    SUMMARIZE_TOOL_RESULTS_SECTION,
    getProactiveSection(),   // ← tick loop + Sleep + focus instructions
  ].filter(s => s !== null)
}
```

Standard path uses dynamic sections registry with caching and section identifiers. Proactive path returns flat array — simpler, faster, and optimized for cache-miss-heavy nature of continually-running agent.

#### Brief Section Deduplication

`getBriefSection()` includes guard: when proactive is active, `getProactiveSection()` already appends `BRIEF_PROACTIVE_SECTION` inline (at end of terminal focus paragraph). Without guard, Brief instructions would appear twice in system prompt.

---

## 11 The GrowthBook Kill-Switch Architecture

KAIROS's runtime behavior is extensively gated by GrowthBook feature flags, giving Anthropic ability to tune or kill subsystems without deploy. Pattern is consistent:

```typescript
// Pattern: cached refresh with explicit interval
const KAIROS_BRIEF_REFRESH_MS = 5 * 60 * 1000  // 5 minutes
const KAIROS_CRON_REFRESH_MS  = 5 * 60 * 1000  // 5 minutes

// Brief entitlement — checks GB flag, refreshes every 5 min
getFeatureValue_CACHED_WITH_REFRESH('tengu_kairos_brief', false, KAIROS_BRIEF_REFRESH_MS)

// Cron kill switch — fleet-wide disable
getFeatureValue_CACHED_WITH_REFRESH('tengu_kairos_cron', true, KAIROS_CRON_REFRESH_MS)

// Durable cron kill switch (narrower — leaves session-only cron untouched)
getFeatureValue_CACHED_WITH_REFRESH('tengu_kairos_cron_durable', true, KAIROS_CRON_REFRESH_MS)

// AutoDream thresholds + enabled gate
getFeatureValue_CACHED_MAY_BE_STALE('tengu_onyx_plover', null)

// Cron jitter config — ops can push during an incident to reduce load
getFeatureValue_CACHED_WITH_REFRESH('tengu_kairos_cron_config', null, ...)
```

Comment in `cronJitterConfig.ts` explains incident story: "During an incident, ops can push `tengu_kairos_cron_config` with e.g. a high jitter multiplier to spread load across the fleet." The 5-minute refresh interval is tuned so GB changes take effect within one cache window, fast enough for incident response but not so fast it adds constant network pressure.

---

## 12 Architecture Diagram: Full KAIROS System

```
BUILD TIME
├─ feature('KAIROS')
├─ feature('KAIROS_BRIEF')
├─ feature('AGENT_TRIGGERS')
├─ feature('KAIROS_GITHUB_WEBHOOKS')
└─ feature('PROACTIVE') || KAIROS

RUNTIME STATE
├─ kairosActive: boolean
└─ userMsgOptIn: boolean

OUTPUT TOOLS
├─ BriefTool / SendUserMessage
├─ SendUserFileTool
└─ PushNotificationTool

LIFECYCLE TOOLS
├─ SleepTool
├─ SubscribePRTool
├─ CronCreate
├─ CronDelete
└─ CronList

BACKGROUND SERVICES
├─ AutoDream (forked sub-agent)
└─ CronScheduler (1s poll loop)

MEMORY
├─ Daily Logs (logs/YYYY/MM/DD.md)
├─ MEMORY.md index
└─ ConsolidationPrompt (4-phase dream)
```

---

## 13 Key Design Principles

Studying KAIROS source reveals several architectural principles Anthropic applied consistently:

### Principle 1: Positive Ternaries for DCE

Every feature-gated block uses positive ternaries rather than negative early-returns:

```typescript
// CORRECT — DCE works
return feature('KAIROS') ? doKairosThings() : false

// WRONG — DCE fails (negative guard defeats constant-folding)
if (!feature('KAIROS')) return false
return doKairosThings()
```

### Principle 2: Independent Shippability

Each sub-feature is designed so it can ship and be kill-switched independently. Cron module graph has zero imports into `src/assistant/`. BriefTool has own `KAIROS_BRIEF` flag. This means bug in full assistant mode doesn't block shipping cron scheduler.

### Principle 3: Cheapest Gate First

All KAIROS subsystems check gates in cheapest-to-evaluate order. AutoDream checks `kairosActive` (one boolean read) before checking time, before reading filesystem for session count, before acquiring lock. This matters because checks run on every agent turn.

### Principle 4: Operator Kill Switches

Every GrowthBook-gated subsystem has local env var override that wins: `CLAUDE_CODE_DISABLE_CRON` kills cron scheduler, `CLAUDE_CODE_BRIEF` enables Brief for dev/testing. Lets individual engineers bypass fleet gates without touching GrowthBook.

### Principle 5: Cost Budgeting in the System Prompt

Model is explicitly told about API call costs and prompt cache mechanics. "Each wake-up costs an API call, but the prompt cache expires after 5 minutes of inactivity — balance accordingly." This is unusually transparent system-prompt engineering — treating model as cost-aware participant rather than hiding infrastructure details.

---

## Key Takeaways

- KAIROS is family of build-time flags, not single switch. Core flag is ant-only; sub-features like `KAIROS_BRIEF`, `KAIROS_CHANNELS`, and `AGENT_TRIGGERS` ship to external users independently.
- `kairosActive` in `bootstrap/state.ts` is runtime pivot changing memory mode, BriefTool opt-in behavior, fast mode availability, and bridge worker type simultaneously.
- Tick loop uses `<tengu_tick>` XML heartbeats + `SleepTool` as cost-aware idle mechanism. Model is told about cache expiry and API call costs explicitly in system prompt.
- BriefTool separates _entitlement_ (allowed to use?) from _activation_ (opted in?) preventing silent Brief-on-by-default for enrolled users.
- AutoDream fires as forked sub-agent after 24 hours and 5 sessions, using file-mtime lock with rollback on failure. Receives hardened read-only tool constraint not present in manual `/dream` runs.
- Cron system has two tiers: session-only (in-memory) and durable (persisted to `.claude/scheduled_tasks.json`). Both independently kill-switchable via GrowthBook without deploy.
- Positive ternaries are mandatory for dead-code elimination — Bun constant-folds `feature('KAIROS')` to `false` in external builds only if guard is ternary, not negative early-return.

---

## Check Your Understanding

### Q1. Why does `isBriefEnabled()` have its own top-level `feature('KAIROS') || feature('KAIROS_BRIEF')` guard even though it calls `isBriefEntitled()` which has its own guard?

**Correct Answer:** The outer guard exits faster at call site, enabling Bun's constant-folding for dead-code elimination of entire BriefTool object in external builds. Comment explicitly states: "Composing isBriefEntitled() alone defeats constant-folding across the boundary."

### Q2. When `kairosActive` is true, what happens to AutoDream?

**Correct Answer:** `isGateOpen()` explicitly checks `getKairosActive()` and returns false. KAIROS mode has its own dream pipeline via scheduled tasks in `.claude/scheduled_tasks.json`.

### Q3. A cron task is created with `recurring: true, permanent: false`. What happens after `recurringMaxAgeMs` has elapsed?

**Correct Answer:** Task fires one final time, then is deleted. From CronCreate prompt: "Recurring tasks auto-expire after N days — they fire one final time, then are deleted. This bounds session lifetime." The `permanent: true` flag exempts task from this behavior — used for assistant mode's built-in system tasks.

### Q4. Why does proactive system prompt tell model about 5-minute prompt cache expiry window?

**Correct Answer:** To let model make cost-aware sleep duration decisions. Sleeping longer saves API calls but risks cold cache on wakeup (expensive cache-creation tokens). Model needs both data points to optimize economically.

# Cron and Task Scheduling — Complete Lesson Content

## Lesson 43: Cron and Task Scheduling

How Claude Code schedules recurring and one-shot prompts, manages a distributed lock, and spreads load across the fleet with deterministic jitter.

### 01 Overview

Claude Code has a built-in cron-style scheduler that lets the model queue prompts for future execution — either once at a specific time or on a recurring schedule. The entire system is built around a single JSON file (`.claude/scheduled_tasks.json`) and a polling loop that ticks every second inside the running REPL.

**Source files covered:**
`utils/cronScheduler.ts` → `utils/cronTasks.ts` → `utils/cronJitterConfig.ts` → `tools/ScheduleCronTool/` → `hooks/useScheduledTasks.ts`

Three user-facing tools drive the system: `CronCreate`, `CronDelete`, and `CronList`. Behind them sits a non-React scheduler core (`cronScheduler.ts`) shared by the interactive REPL and the headless SDK daemon, connected to the REPL via the `useScheduledTasks` React hook.

**Layer 1 — Tools**
`CronCreate / Delete / List` — model-facing API, validation, file I/O

**Layer 2 — Scheduler core**
`cronScheduler.ts` — 1s tick loop, chokidar file watcher, lock, jitter

**Layer 3 — Storage**
`.claude/scheduled_tasks.json` + in-memory session store for ephemeral tasks

**Layer 4 — React glue**
`useScheduledTasks` — mounts scheduler in REPL, routes fires to message queue

**Layer 5 — Fleet ops lever**
`cronJitterConfig.ts` — GrowthBook-backed tuning pushed live without restart

### 02 The CronTask Data Model

Every scheduled task is represented by a `CronTask` object. The disk shape is intentionally minimal — runtime-only fields are stripped before writing.

```typescript
// utils/cronTasks.ts
export type CronTask = {
  id:        string         // 8-hex UUID slice — enough for MAX_JOBS=50
  cron:      string         // 5-field cron in LOCAL time
  prompt:    string         // what to enqueue when the task fires
  createdAt: number         // epoch ms — anchor for missed-task detection

  lastFiredAt?: number     // written back after each recurring fire
  recurring?:   boolean    // true = reschedule; false/undefined = delete on fire
  permanent?:   boolean    // exempt from recurringMaxAgeMs auto-expiry

  // Runtime-only — never written to disk:
  durable?:  boolean       // false = session-scoped (in-memory only)
  agentId?:  string        // routes fires to a specific in-process teammate
}
```

**Design insight:**
The `durable` flag never touches disk: `writeCronTasks()` strips it with a destructuring spread (`{ durable: _durable, ...rest }`). Everything stored in the file is durable by definition, so the flag is only meaningful at runtime.

**Two flavors of task exist depending on `recurring`:**

**One-shot (recurring: false):**
fire once → auto-delete

**Recurring (recurring: true):**
fire → reschedule from now → auto-expire after 7 days

**Durable (durable: true):**
persisted to .claude/scheduled_tasks.json

**Session-only (durable: false):**
bootstrap/state.ts memory; dies with process

### 03 Scheduler Lifecycle

The scheduler is created once per REPL session by `createCronScheduler()` and managed through a simple `{ start, stop, getNextFireTime }` interface. The lifecycle has a deliberate lazy-enable design to avoid loading chokidar and the file system machinery until tasks actually exist.

**State diagram:**
- `[*]` → Polling: `start()` — no tasks yet
- Polling → Enabling: `getScheduledTasksEnabled()` flips true (CronCreate ran or file already has tasks)
- Enabling → Running: `enable()` acquires lock + starts chokidar + `setInterval(check, 1000ms)`
- Running → Running: `check()` fires every 1s
- Running → Running: chokidar detects file change → `load(false)`
- Running → `[*]`: `stop()` clears timers, releases lock, closes watcher

**Note:** enablePoll probes every 1s. Timer is unref'd so `-p` mode can still exit.

**Note:** isOwner=true: processes file tasks. isOwner=false: probes lock every 5s. Session tasks bypass lock entirely.

#### The `check()` inner loop

Every second, `check()` iterates all loaded file tasks (if lock-owner) and all session tasks from bootstrap state. For each task it:

```typescript
// cronScheduler.ts — simplified check() inner loop
function process(t: CronTask, isSession: boolean) {
  let next = nextFireAt.get(t.id)
  if (next === undefined) {
    // First sight: anchor from lastFiredAt (if previously fired) or createdAt.
    // Anchoring from lastFiredAt prevents "stale spawn" re-firing every cycle.
    next = t.recurring
      ? jitteredNextCronRunMs(t.cron, t.lastFiredAt ?? t.createdAt, t.id, jitterCfg)
      : oneShotJitteredNextCronRunMs(t.cron, t.createdAt, t.id, jitterCfg)
    nextFireAt.set(t.id, next ?? Infinity)
  }
  if (now < next) return  // not yet

  // Fire!
  onFireTask ? onFireTask(t) : onFire(t.prompt)

  if (t.recurring && !aged) {
    // Reschedule from now — not from next — to avoid rapid catch-up after blocking.
    nextFireAt.set(t.id, jitteredNextCronRunMs(t.cron, now, t.id, jitterCfg))
    if (!isSession) firedFileRecurring.push(t.id) // batch lastFiredAt write
  } else {
    // One-shot or aged recurring: remove from store / file.
    isSession ? removeSessionCronTasks([t.id]) : removeCronTasks([t.id])
  }
}
```

**Catch-up prevention:**
Recurring tasks always reschedule from `now`, not from the computed fire time. If the session was blocked by a long query and the 9am task didn't fire until 9:05, the next fire is computed from 9:05 — not 9:00 — so you won't get rapid catch-up fires.

### 04 Multi-Session Distributed Lock

A user can run multiple Claude sessions in the same project directory simultaneously. Without coordination, both sessions would fire the same on-disk task — duplicating work. Claude Code solves this with a per-project _scheduler lock_.

```typescript
// cronScheduler.ts — lock acquisition in enable()
isOwner = await tryAcquireSchedulerLock(lockOpts).catch(() => false)

if (!isOwner) {
  // Non-owner: probe for lock takeover every 5s.
  // Coarse because takeover only matters when the owning session crashes.
  lockProbeTimer = setInterval(() => {
    tryAcquireSchedulerLock(lockOpts).then(owned => {
      if (owned) { isOwner = true; clearInterval(lockProbeTimer) }
    })
  }, 5000) // LOCK_PROBE_INTERVAL_MS
  lockProbeTimer.unref?.()
}
```

The lock is liveness-probed by PID. If the owning process dies without calling `stop()`, a non-owning session will detect the stale lock on its next 5-second probe and take over.

**Session tasks are lock-exempt:**
Session-only tasks (`durable: false`) live in process-private memory, so there is no shared file and no double-fire risk. The lock guard only applies to file-backed tasks. The code enforces this with an explicit `if (isOwner)` gate around file task processing, followed by an unconditional block for session tasks.

The `stop()` method always releases the lock:

```typescript
// cronScheduler.ts
stop() {
  stopped = true
  clearInterval(checkTimer)
  clearInterval(lockProbeTimer)
  void watcher?.close()
  if (isOwner) {
    isOwner = false
    void releaseSchedulerLock(lockOpts)
  }
}
```

### 05 Load-Spreading Jitter

When millions of users schedule tasks at the same time ("every hour", "at 9am"), they all generate inference requests simultaneously — a thundering herd. Claude Code adds deterministic per-task jitter to spread these spikes across the fleet.

The jitter amount is derived from the task ID (an 8-hex-char UUID slice):

```typescript
// cronTasks.ts — stable per-task fraction in [0, 1)
function jitterFrac(taskId: string): number {
  const frac = parseInt(taskId.slice(0, 8), 16) / 0x1_0000_0000
  return Number.isFinite(frac) ? frac : 0
}
```

This fraction is stable across restarts (same taskId = same jitter), uniformly distributed across the fleet, and requires no coordination. Two strategies apply depending on task type:

#### Recurring tasks — forward jitter

```typescript
// Forward jitter: fires up to recurringFrac * interval late (cap: recurringCapMs)
export function jitteredNextCronRunMs(cron, fromMs, taskId, cfg): number | null {
  const t1 = nextCronRunMs(cron, fromMs)   // next fire
  const t2 = nextCronRunMs(cron, t1)        // one-after (for interval))
  if (t2 === null) return t1               // pinned date — no herd risk
  const jitter = Math.min(
    jitterFrac(taskId) * cfg.recurringFrac * (t2 - t1),
    cfg.recurringCapMs,
  )
  return t1 + jitter
  // e.g. hourly at cfg defaults (frac=0.1, cap=15min):
  // spread = jitterFrac(id) * 0.1 * 3600000ms = up to 360s = 6 min
}
```

#### One-shot tasks — backward jitter

```typescript
// Backward jitter: fires up to oneShotMaxMs early, only on :00/:30 minutes
export function oneShotJitteredNextCronRunMs(cron, fromMs, taskId, cfg): number | null {
  const t1 = nextCronRunMs(cron, fromMs)
  // Only jitter "round" minutes — humans pick :00 and :30, bots don't.
  if (new Date(t1).getMinutes() % cfg.oneShotMinuteMod !== 0) return t1
  const lead = cfg.oneShotFloorMs + jitterFrac(taskId) * (cfg.oneShotMaxMs - cfg.oneShotFloorMs)
  return Math.max(t1 - lead, fromMs)  // never fire before creation
}
```

**Why backward for one-shots?**
One-shot tasks are user-pinned ("remind me at 3pm"). Delaying them breaks the contract. But firing a few seconds _early_ is invisible to the user and still spreads the inference spike. Only `:00` and `:30` get jitter because those are the only minutes humans actually pick.

**Default jitter config values:**
- recurringFrac: 0.1 (10% of interval)
- recurringCapMs: 15 min (900,000 ms)
- oneShotMaxMs: 90 s early
- oneShotMinuteMod: 30 (only :00 and :30)
- recurringMaxAgeMs: 7 days (604,800,000 ms)
- oneShotFloorMs: 0 (no minimum lead)

**Deep dive — Live ops tuning via GrowthBook:**
The jitter config is sourced from a GrowthBook JSON feature flag (`tengu_kairos_cron_config`) rather than being hardcoded. This means ops engineers can push a config change during an incident — for example, widening the one-shot lead window from 90s to 300s and spreading `:00/:15/:30/:45` instead of just `:00/:30` — and already-running REPL sessions will pick it up within 60 seconds without any restart.

The config is validated with a strict Zod schema. If any field is out-of-bounds or the `oneShotFloorMs > oneShotMaxMs` invariant is violated, the whole config falls back to `DEFAULT_CRON_JITTER_CONFIG` rather than partially trusting it.

The SDK daemon does _not_ use GrowthBook for jitter — it gets `DEFAULT_CRON_JITTER_CONFIG` directly. This keeps the scheduler bundle free of the GrowthBook dependency chain.

### 06 The Three Cron Tools

#### CronCreate

The model calls `CronCreate` with a 5-field cron string, a prompt, and optional `recurring` and `durable` flags.

```typescript
// CronCreateTool.ts — input schema
z.strictObject({
  cron:      z.string(),     // "*/5 * * * *" — 5-field local time
  prompt:    z.string(),     // what to run at fire time
  recurring: z.boolean().optional(),  // default: true
  durable:   z.boolean().optional(),  // default: false (session-only)
})
```

Validation is strict: the cron expression is parsed and checked against the next 366 days. A hard cap of **50 jobs** prevents runaway scheduling. Teammate agents cannot create durable crons (their agentId would orphan on restart). After a successful create, `setScheduledTasksEnabled(true)` flips the bootstrap flag to kick the enablePoll loop into action immediately.

**Off-minute heuristic in the system prompt:**
The `CronCreate` system prompt explicitly instructs the model to avoid scheduling at `:00` or `:30` unless the user names that exact time. "Every morning around 9" → `"57 8 * * *"` or `"3 9 * * *"`. This is the _biggest_ lever for fleet load-shedding — jitter adds at most minutes on top of whatever the model picks.

#### CronDelete

Takes a job ID, validates it exists (and belongs to the calling teammate if in teammate context), then calls `removeCronTasks([id])`. Teammate isolation is enforced: a teammate can only delete crons with a matching `agentId`.

#### CronList

Returns all tasks merged from disk and the session store. Teammates see only their own crons; the team lead sees everything. The tool is marked `isReadOnly()` and `isConcurrencySafe()` — it never writes and can run alongside other tool calls.

```typescript
// CronListTool.ts — teammate scoping
const ctx = getTeammateContext()
const tasks = ctx
  ? allTasks.filter(t => t.agentId === ctx.agentId)  // teammate sees own crons only
  : allTasks                                              // lead sees all
```

### 07 REPL Wiring — useScheduledTasks

The `useScheduledTasks` hook is the bridge between the React REPL and the non-React scheduler core. It mounts the scheduler exactly once and tears it down on unmount.

```typescript
// hooks/useScheduledTasks.ts
export function useScheduledTasks({ isLoading, assistantMode, setMessages }: Props) {
  const isLoadingRef = useRef(isLoading)
  isLoadingRef.current = isLoading  // latest-value ref — no stale closure

  useEffect(() => {
    if (!isKairosCronEnabled()) return  // runtime gate

    const scheduler = createCronScheduler({
      onFire: prompt => enqueuePendingNotification({
        value: prompt, mode: 'prompt',
        priority: 'later',   // drains between turns — never interrupts
        isMeta: true,        // hidden from transcript UI
        workload: WORKLOAD_CRON, // lower QoS — no human waiting
      }),
      onFireTask: task => {
        if (task.agentId) {
          // Route to teammate agent instead of main REPL queue
          injectUserMessageToTeammate(teammate.id, task.prompt, setAppState)
          return
        }
        // Show "Running scheduled task (Mar 31 9:03am)" in transcript
        setMessages(prev => [...prev, createScheduledTaskFireMessage(...)])
        enqueueForLead(task.prompt)
      },
      isLoading: () => isLoadingRef.current,
      getJitterConfig: getCronJitterConfig,
      isKilled: () => !isKairosCronEnabled(),  // polled every tick — live killswitch
    })
    scheduler.start()
    return () => scheduler.stop()
  }, [assistantMode])
}
```

Fired prompts are enqueued at **`'later'` priority** via `enqueuePendingNotification`. The REPL's `useCommandQueue` drains this queue between turns — so a cron task never interrupts an active query. The `WORKLOAD_CRON` attribution flows through to the API billing header, allowing lower Quality-of-Service for automated background requests vs interactive ones.

**Orphaned teammate crons:**
When a task fires with an `agentId` but the teammate is gone (terminated or never existed), the hook removes the orphaned cron immediately rather than letting it fire into nowhere every tick. One-shots would self-delete anyway, but recurring crons would loop indefinitely until the 7-day auto-expiry.

### 08 Missed Tasks and Startup Catch-Up

If Claude was not running when a task was scheduled to fire, it detects this on startup. `findMissedTasks()` computes each task's first fire time from `createdAt` and compares to `Date.now()`:

```typescript
// cronTasks.ts
export function findMissedTasks(tasks: CronTask[], nowMs: number): CronTask[] {
  return tasks.filter(t => {
    const next = nextCronRunMs(t.cron, t.createdAt)
    return next !== null && next < nowMs
  })
}
```

Missed **one-shot** tasks are surfaced to the user with a notification prompt built by `buildMissedTaskNotification()`. The notification:
- Includes the task's cron expression in human-readable form and creation timestamp.
- Wraps each prompt in a code fence to prevent accidental prompt injection (uses a fence one backtick longer than any backtick run inside the prompt).
- Explicitly instructs the model _not_ to execute yet — first ask the user via `AskUserQuestion`.
- Deletes the tasks from disk before the model sees the notification.

Missed **recurring** tasks are intentionally NOT surfaced. The scheduler's `check()` handles them correctly by firing on the first tick and rescheduling forward from there. Surfacing them would produce a misleading "missed while Claude was not running" prompt for tasks that were merely overdue, not missed.

**Deep dive — Prompt injection defense in missed-task notification:**
A scheduled task's prompt could contain any string — including backtick sequences that would close a Markdown code fence and allow the outer guidance text to be misread as executable instructions.

`buildMissedTaskNotification()` defends against this with CommonMark's fence-length rule: a fence can only be closed by a fence of equal-or-greater length. It finds the longest run of backticks in the prompt, then opens the fence with one more backtick:

```typescript
const longestRun = (t.prompt.match(/`+/g) ?? []).reduce(
  (max, run) => Math.max(max, run.length), 0
)
const fence = '`'.repeat(Math.max(3, longestRun + 1))
```

This ensures a prompt containing ` ``` ` cannot close the surrounding fence early and expose subsequent text as unguarded instructions.

### 09 End-to-End Flow Diagram

Sequence diagram showing:
- Model calls CronCreate with parameters
- CronCreate validates input
- For durable=false: task added to bootstrap state
- For durable=true: task written to cronTasks file, triggering chokidar event to scheduler
- setScheduledTasksEnabled(true) called
- Hook enablePoll sees flag and calls enable()
- Hook acquires scheduler lock
- Hook starts chokidar file watcher
- Hook starts setInterval(check, 1000ms)
- Every 1 second: check() iterates file tasks (if owner) and session tasks
- If task.nextFireAt <= now:
  - For recurring not aged: enqueue notification and mark as fired
  - For one-shot or aged: enqueue notification and remove task
- Queue drains between REPL turns

### 10 Feature Gates and Kill Switches

The scheduling system has multiple independently-operable gates:

**Build-time:**

#### `feature('AGENT_TRIGGERS')`
Dead-code elimination via Bun. The whole cron module is stripped from builds where triggers are disabled.

**Runtime — env var:**

#### `CLAUDE_CODE_DISABLE_CRON=1`
Local override that wins over GrowthBook. Kills all scheduling including already-running schedulers on next poll.

**Runtime — GrowthBook:**

#### `tengu_kairos_cron`
Fleet-wide kill switch. Polled every 5 min; default true so Bedrock/Vertex/DISABLE_TELEMETRY users get full cron.

#### `tengu_kairos_cron_durable`
Narrower gate — kills disk persistence only. Session-only cron stays alive. Default true.

#### `tengu_kairos_cron_config`
Ops lever for jitter tuning without a deploy. Converges fleet within 60 seconds.

**isKilled is polled every tick:**
The `isKilled` callback is checked at the top of every `check()` call. This means flipping `tengu_kairos_cron` to false in GrowthBook stops all already-running schedulers within 5 minutes (their GrowthBook cache refresh) — not just new sessions. This is the "stop the bleeding" mechanism during an incident.

## Key Takeaways

- Tasks are stored in `.claude/scheduled_tasks.json`; session-only tasks live in `bootstrap/state.ts` memory and are never written to disk.
- The scheduler polls at 1-second intervals but is lazy-enabled — chokidar and timers don't start until tasks actually exist (`setScheduledTasksEnabled(true)`).
- A per-project scheduler lock prevents double-firing when multiple Claude sessions share a working directory. Non-owners probe every 5 seconds to take over if the owner crashes.
- Recurring tasks reschedule from `now` (not from the computed fire time) to avoid rapid catch-up after a blocked session.
- Jitter is deterministic and stable per task ID — same task = same jitter spread across restarts. Recurring tasks spread forward (up to 10% of interval, capped at 15 min); one-shot tasks at :00/:30 spread backward (up to 90s early).
- The jitter config is a live GrowthBook ops lever. Ops can widen jitter during an incident without restarting any clients; the fleet converges within 60 seconds.
- Missed one-shot tasks at startup are surfaced with an injection-resistant prompt (adaptive backtick fence) and require user confirmation before re-execution.
- Recurring tasks auto-expire after 7 days to prevent unbounded session lifetime growth; `permanent: true` marks system assistant tasks exempt from this limit.

## Knowledge Check

**Q1. Where are session-only (`durable: false`) cron tasks stored?**
A) In `.claude/scheduled_tasks.json` with a `durable: false` marker
B) In process memory via `bootstrap/state.ts` — never written to disk
C) In a separate `.claude/session_tasks.json` file
D) In SQLite alongside conversation history

**Q2. Why does `check()` reschedule a recurring task from `now` rather than from the computed fire time?**
A) To avoid rapid catch-up fires if the session was blocked during a long query
B) Because the cron expression is evaluated in UTC, not local time
C) To reset the 7-day auto-expiry timer on every fire
D) To ensure jitter is recomputed with the latest GrowthBook config

**Q3. What is the maximum number of scheduled jobs Claude Code allows at one time?**
A) 10
B) 25
C) 50
D) 100

**Q4. Why does one-shot backward jitter only apply to tasks landing on `:00` or `:30` minutes?**
A) The cron parser only supports minute-granularity at those marks
B) Those are the only minutes humans typically pick, so the thundering herd only happens there
C) API rate limits reset on the half-hour boundary
D) The task ID hash only distributes evenly at those times

**Q5. How does the scheduler prevent two Claude sessions in the same directory from firing the same task twice?**
A) Each session deletes the task immediately before firing so the other sees it gone
B) Tasks include a `claimedBy` field written atomically at check time
C) A per-project scheduler lock — only the lock owner processes file tasks; non-owners probe every 5 seconds
D) The check interval is randomized per session so they rarely collide

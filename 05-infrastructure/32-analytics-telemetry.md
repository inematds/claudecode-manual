# mdENG — Lesson 32 — Analytics and Telemetry — Source Deep Dive

### 01 Architecture Overview

Claude Code's analytics system spans six files in `services/analytics/`. The design separates what gets logged from how events route, keeping the public API dependency-free so any module can call `logEvent()` without importing Datadog or OpenTelemetry transports at import time.

**index.ts — Public API + Queue**
Zero dependencies. Exposes `logEvent` / `logEventAsync`. Queues events until a sink attaches.

**sink.ts — Router**
Initialised at startup. Fans events out to Datadog and first-party systems, applies sampling, checks feature gates, strips PII.

**datadog.ts — Datadog Transport**
Batches logs, flushes every 15 seconds or when 100 entries accumulate. First-party users only.

**metadata.ts — Metadata Enrichment**
Attaches platform, model, session, agent ID, process metrics, and repo hash to every event.

**growthbook.ts — Feature Flags**
GrowthBook remote-eval client. Drives A/B experiments, config values, and kill-switches.

**sinkKillswitch.ts — Emergency Off-Switch**
GrowthBook config `tengu_frond_boric` can disable individual sinks without a release.

---

### 02 The Pre-Sink Queue

Because analytics imports occur before the app fully initialises, events fired during startup would otherwise be lost. `index.ts` solves this with a module-level array:

```typescript
// index.ts — simplified
const eventQueue: QueuedEvent[] = []
let sink: AnalyticsSink | null = null

export function logEvent(eventName: string, metadata: LogEventMetadata): void {
  if (sink === null) {
    eventQueue.push({ eventName, metadata, async: false })
    return
  }
  sink.logEvent(eventName, metadata)
}

export function attachAnalyticsSink(newSink: AnalyticsSink): void {
  if (sink !== null) return  // idempotent
  sink = newSink

  if (eventQueue.length > 0) {
    const queuedEvents = [...eventQueue]
    eventQueue.length = 0
    queueMicrotask(() => {
      for (const event of queuedEvents) { sink!.logEvent(event.eventName, event.metadata) }
    })
  }
}
```

**Design insight**

The queue is drained via `queueMicrotask` rather than synchronously. This avoids adding latency to the startup hot path — startup code calls `attachAnalyticsSink` and returns; the backlog drains in the next microtask checkpoint.

**Idempotency guard**

Both `attachAnalyticsSink` and `initializeAnalyticsSink` (in sink.ts) are explicitly idempotent. They can be called from `preAction` hooks (subcommands) and from `setup()` (the default command) without coordination or double-attaching.

---

### 03 Sink Routing — sink.ts

`sink.ts` is the traffic cop. On every event it applies three sequential checks before dispatch:

```
flowchart TD
A[logEvent called] --> B{shouldSampleEvent?}
B -- sample_rate=0 --> DROP[Drop event]
B -- sampled --> C{isSinkKilled datadog?}
C -- yes --> E[Skip Datadog]
C -- no --> D{shouldTrackDatadog?}
D -- no --> E
D -- yes --> F[stripProtoFields]
F --> G[trackDatadogEvent]
E --> H[logEventTo1P with full payload]
G --> H
```

```typescript
// sink.ts — logEventImpl
function logEventImpl(eventName: string, metadata: LogEventMetadata): void {
  const sampleResult = shouldSampleEvent(eventName)
  if (sampleResult === 0) return  // dropped by sampling

  const metadataWithSampleRate =
    sampleResult !== null
      ? { ...metadata, sample_rate: sampleResult }
      : metadata

  if (shouldTrackDatadog()) {
    // strip _PROTO_* keys — PII tagged, Datadog is general-access
    void trackDatadogEvent(eventName, stripProtoFields(metadataWithSampleRate))
  }
  // 1P exporter receives the full payload including _PROTO_* keys
  logEventTo1P(eventName, metadataWithSampleRate)
}
```

**PII separation**

Keys prefixed `_PROTO_` route to privileged BigQuery columns via the first-party exporter. `stripProtoFields()` is called before the Datadog fanout so those values never reach the general-access backend. The exporter then strips them defensively after hoisting them to proto fields, so an unrecognised `_PROTO_foo` in the future can't silently land in the BigQuery JSON blob.

---

### 04 Datadog Transport

Datadog receives a carefully curated allow-list of events. Events not in `DATADOG_ALLOWED_EVENTS` are silently discarded. This keeps Datadog focused on operational monitoring rather than product analytics.

**Allowed events:**
- tengu_init
- tengu_started
- tengu_api_success
- tengu_api_error
- tengu_tool_use_success
- tengu_tool_use_error
- tengu_exit
- tengu_oauth_success
- tengu_oauth_error
- tengu_cancel
- tengu_uncaught_exception
- tengu_compact_failed
- chrome_bridge_*
- tengu_team_mem_*

```typescript
// datadog.ts — batching + flush
const DEFAULT_FLUSH_INTERVAL_MS = 15000
const MAX_BATCH_SIZE = 100
const NETWORK_TIMEOUT_MS = 5000

function scheduleFlush(): void {
  if (flushTimer) return
  flushTimer = setTimeout(() => {
    flushTimer = null
    void flushLogs()
  }, getFlushIntervalMs()).unref()  // .unref() — never blocks process exit
}

// Flush immediately if batch is full, otherwise schedule
if (logBatch.length >= MAX_BATCH_SIZE) {
  void flushLogs()
} else {
  scheduleFlush()
}
```

**Cardinality Reduction**

Before building Datadog tags, several normalizations reduce metric cardinality:

**Model names**
- For non-Anthropic users, model name is canonicalized to its base name.
- Unknown models bucketed as `"other"` to avoid tag explosion.
- Ants (internal users) keep full model strings for debugging.

**MCP tool names**
- `mcp__slack__post_message` becomes `"mcp"` in Datadog tags.
- User-configured server names are PII-medium per internal taxonomy.
- Official registry servers allowed through in metadata.ts.

**Version strings**
- Dev builds: `2.0.53-dev.20251124.t173302.sha526cc6a` -> `2.0.53-dev.20251124`
- Timestamp and SHA stripped to avoid a new tag per build.

**User buckets**
- User IDs hashed to 1 of 30 buckets (SHA-256, first 8 hex chars mod 30).
- Enables approximate unique-user counting without logging actual IDs.
- Protects privacy while supporting alerting on "users impacted".

---

### 05 Metadata Enrichment — metadata.ts

Every event is enriched with a rich `EventMetadata` object assembled by `getEventMetadata()`. This includes three nested layers:

**EnvContext — platform and runtime snapshot**

Built once per process and memoized. Captures:

```typescript
EnvContext {
  platform: 'darwin' | 'linux' | 'windows'
  platformRaw:          // raw process.platform (freebsd visible here, not bucketed)
  arch: 'arm64' | 'x64'
  nodeVersion:          // runtime version string
  terminal:             // $TERM_PROGRAM
  packageManagers:      // comma-joined: 'npm,bun'
  runtimes:             // node, bun, deno detection
  isCi:                 // $CI env var
  isGithubAction:       // $GITHUB_ACTIONS
  isClaudeAiAuth:       // OAuth vs API key auth
  version:              // full semver build string
  wslVersion:           // WSL 1/2 if applicable
  vcs:                   // 'git' | 'hg' | ... detected VCS
  // + GitHub Actions runner metadata, container IDs, remote session IDs
}
```

**ProcessMetrics — memory and CPU delta per event**

```typescript
ProcessMetrics {
  uptime:           // process.uptime()
  rss:              // resident set size in bytes
  heapTotal:        // V8 heap allocated
  heapUsed:         // V8 heap in use
  cpuPercent:       // delta since last event (user+sys us / wall-clock ms)
  constrainedMemory: // process.constrainedMemory() if available
}
```

CPU percent is a delta: `(userDeltaus + sysDeltaus) / (wallDeltaMs x 1000) x 100`. Module-level vars `prevCpuUsage` and `prevWallTimeMs` persist between calls.

**Agent identification — swarm and subagent attribution**

```typescript
EventMetadata {
  agentId?:         // CLAUDE_CODE_AGENT_ID or subagent UUID
  parentSessionId?:  // lead session for cross-session joining in BQ
  agentType?:        // 'teammate' | 'subagent' | 'standalone'
  teamName?:         // swarm team label
  rh?:               // first 16 chars of SHA-256(repo remote URL)
  subscriptionType?: // 'max' | 'pro' | 'enterprise' | 'team'
  kairosActive?:     // ant-only KAIROS assistant flag
}
```

AsyncLocalStorage is checked first (for subagents running in the same process), then env vars (for swarm teammates running in separate processes). This means attribution is automatic — callers don't pass agent context manually.

---

### 06 PII Sanitization in metadata.ts

The type system itself enforces that strings are reviewed before logging. Any string value in event metadata must be cast to one of two marker types — both typed as `never`, so they can only be used as casts that document developer intent:

```typescript
// You cannot store a value in these types — they're "never".
// They exist solely as cast targets to document a review decision.
export type AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS = never
export type AnalyticsMetadata_I_VERIFIED_THIS_IS_PII_TAGGED = never

// Usage example — sanitizing MCP tool names:
export function sanitizeToolNameForAnalytics(toolName: string) {
  if (toolName.startsWith('mcp__')) {
    return 'mcp_tool' as AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS
  }
  return toolName as AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS
}
```

**Tool Input Truncation**

When `OTEL_LOG_TOOL_DETAILS=1` is set, tool inputs are serialized for OpenTelemetry spans. Strings over 512 chars are truncated to 128 chars with a length annotation. Objects are limited to depth 2 and 20 items. The total JSON is capped at 4 KB.

**File Extension Extraction**

For bash tool calls, Claude Code extracts file extensions from known commands (`cat`, `mv`, `cp`, `rm`, etc.) to track which file types are being operated on — without logging actual file paths. Extensions longer than 10 chars are redacted as `"other"` to prevent hash-based filenames from leaking.

---

### 07 Feature Flags — GrowthBook

GrowthBook drives feature flags, A/B experiments, dynamic configs, and kill-switches. Claude Code uses its remote eval mode: the server evaluates all rules for this user and returns pre-computed values, rather than sending the full rule tree to the client.

**User Attributes for Targeting**

```typescript
GrowthBookUserAttributes {
  id:               // stable device UUID
  sessionId:        // per-session ID
  deviceID:         // same as id
  platform:         // 'win32' | 'darwin' | 'linux'
  organizationUUID?: // enterprise org targeting
  accountUUID?:     // account-level targeting
  subscriptionType?: // 'max' | 'pro' | 'enterprise' | 'team'
  email?:           // ant-only: always included for internal targeting
  appVersion?:      // semver for version-range gates
  github?:          // GitHub Actions runner metadata
}
```

**Three-Level Override Priority**

**Priority 1 — Env Var Overrides**

`CLAUDE_INTERNAL_FC_OVERRIDES` JSON object. Ant-only. Used by eval harnesses for deterministic flag state.

**Priority 2 — Config Overrides**

Set via `/config Gates` tab at runtime. Stored in `~/.claude.json`. Ant-only.

**Priority 3 — Remote Eval**

Fetched from `api.anthropic.com`. Cached to disk on every successful fetch so the next session starts with the last known values.

**SDK Workaround — Remote Eval Format**

The GrowthBook API returns feature values in a `{ "value": ... }` shape, but the SDK expects `{ "defaultValue": ... }`. Claude Code works around this by transforming the payload before loading it and by maintaining its own `remoteEvalFeatureValues` Map that bypasses the SDK's local re-evaluation:

```typescript
// growthbook.ts — processRemoteEvalPayload (simplified)
for (const [key, feature] of Object.entries(payload.features)) {
  if ('value' in feature && !('defaultValue' in feature)) {
    transformedFeatures[key] = { ...feature, defaultValue: feature.value }
  }
  // Cache evaluated value directly to avoid SDK re-evaluation
  remoteEvalFeatureValues.set(key, feature.value ?? feature.defaultValue)
}
```

**Safety guard**

If the server returns an empty `features` object (transient bug or truncated response), `processRemoteEvalPayload` returns `false` without clearing the disk cache. This prevents a total flag blackout for all processes sharing `~/.claude.json`.

**Experiment Exposure Deduplication**

When GrowthBook assigns a user to an experiment variant, Claude Code logs an exposure event to the first-party backend for proper experiment analysis. A module-level `loggedExposures` Set ensures each feature fires its exposure event at most once per session — preventing duplicate entries from hot code paths that call `getFeatureValue` on every render.

---

### 08 The Kill-Switch

If a bug in analytics causes problems (excessive volume, accidental PII, etc.), Anthropic can disable individual sinks remotely without shipping a new release:

```typescript
// sinkKillswitch.ts
// Deliberately obfuscated config key to avoid easy discovery
const SINK_KILLSWITCH_CONFIG_NAME = 'tengu_frond_boric'

export function isSinkKilled(sink: 'datadog' | 'firstParty'): boolean {
  const config = getDynamicConfig_CACHED_MAY_BE_STALE<
    Partial<Record<'datadog' | 'firstParty', boolean>>
  >('tengu_frond_boric', {})
  return config?.[sink] === true
}
```

**Fail-open design**

If the kill-switch config is missing, malformed, or GrowthBook hasn't loaded yet, `isSinkKilled` returns `false` — sinks stay enabled. This prevents a GrowthBook outage from also silencing analytics. The sink is only disabled when the config explicitly sets `{ "datadog": true }`.

---

### 09 When Analytics is Disabled

`config.ts` defines the conditions under which analytics is entirely suppressed:

```typescript
export function isAnalyticsDisabled(): boolean {
  return (
    process.env.NODE_ENV === 'test'               // test environments
    || isEnvTruthy(process.env.CLAUDE_CODE_USE_BEDROCK)  // AWS Bedrock
    || isEnvTruthy(process.env.CLAUDE_CODE_USE_VERTEX)   // GCP Vertex
    || isEnvTruthy(process.env.CLAUDE_CODE_USE_FOUNDRY)  // Azure Foundry
    || isTelemetryDisabled()                          // user privacy setting
  )
}
```

Third-party cloud providers (Bedrock, Vertex, Foundry) disable analytics entirely — event data would be meaningless without Anthropic auth context, and those users have their own logging pipelines. The `isTelemetryDisabled()` path respects the user's privacy level setting.

**Feedback surveys are different**

`isFeedbackSurveyDisabled()` does not check for third-party providers. Survey prompts are local UI with no transcript data, so they remain active on Bedrock/Vertex where analytics would be blocked.

---

### 10 Event Sampling

High-frequency events (like `tengu_api_success` in busy sessions) can generate enormous volume. GrowthBook config key `tengu_event_sampling_config` lets Anthropic set per-event sample rates without a code change:

```typescript
// firstPartyEventLogger.ts — shouldSampleEvent
export function shouldSampleEvent(eventName: string): number | null {
  const config = getEventSamplingConfig()   // from GrowthBook cache
  const eventConfig = config[eventName]

  if (!eventConfig) return null             // no config = 100% logging

  const sampleRate = eventConfig.sample_rate
  if (sampleRate <= 0)  return 0            // 0 = drop everything
  if (sampleRate >= 1)  return null         // 1 = keep everything (no annotation needed)

  // Probabilistic: roll the dice
  return Math.random() < sampleRate
    ? sampleRate    // sampled — return rate so it can be added to event metadata
    : 0            // not sampled — sink drops it
}
```

When an event is sampled, the `sample_rate` is appended to its metadata. This lets analysts apply inverse-probability weighting to reconstruct unsampled totals.

---

### 11 Files Reference

**Core files**
- services/analytics/index.ts
- services/analytics/sink.ts
- services/analytics/config.ts
- services/analytics/sinkKillswitch.ts

**Backends & enrichment**
- services/analytics/datadog.ts
- services/analytics/metadata.ts
- services/analytics/growthbook.ts
- services/analytics/firstPartyEventLogger.ts

---

### 12 Key Takeaways

**Decoupling**

Zero-dep public API — Any module can call `logEvent()` without creating import cycles. The sink wires up later during startup.

**Privacy**

Type-enforced sanitization — Strings require an explicit cast to a marker type. No string can reach a backend without a developer sign-off comment in the code.

**Control**

GrowthBook as control plane — Sampling rates, feature gates, A/B experiments, and emergency kill-switches all flow through a single remote-eval endpoint.

**Resilience**

Fail-open everywhere — Missing configs, failed fetches, and GrowthBook outages all default to keeping analytics running — never silencing data unexpectedly.

**Attribution**

Swarm-aware from day one — Agent type, team name, and parent session ID are automatically attached to every event via AsyncLocalStorage — no manual propagation.

**Operations**

Graceful shutdown — `shutdownDatadog()` is called before `process.exit()` to flush the in-memory batch so the last events of a session are never lost.

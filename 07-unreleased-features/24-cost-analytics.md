# Cost Analytics & Observability

## Lesson 24: Cost Analytics & Observability

### 01 The Two-Pipeline Architecture

Claude Code operates "two entirely separate observability stacks in parallel." One is an internal first-party (1P) pipeline using OpenTelemetry with proto-serialized events sent to `/api/event_logging/batch`. The other is a Datadog HTTP-intake fanout for operational monitoring. A sink layer decides which pipeline receives each event and whether to send it.

**Design Principle:** Events queue before the sink attaches using `queueMicrotask` drain on startup to prevent loss during app initialization. The 1P and Datadog pipelines are completely independent—a killswitch can silence one without affecting the other.

### 02 Cost Tracking: cost-tracker.ts

Every API response carries a `BetaUsage` object. `addToTotalSessionCost()` is the single entry point that distributes cost data to three destinations simultaneously: in-memory accumulators, OpenTelemetry counters, and the analytics event log.

```typescript
// cost-tracker.ts
export function addToTotalSessionCost(
  cost: number,
  usage: Usage,
  model: string,
): number {
  // 1. Accumulate per-model token counters in memory
  const modelUsage = addToTotalModelUsage(cost, usage, model)
  addToTotalCostState(cost, modelUsage, model)

  // 2. Push to OTel counters (for BigQuery / customer OTLP)
  const attrs = isFastModeEnabled() && usage.speed === 'fast'
    ? { model, speed: 'fast' }
    : { model }

  getCostCounter()?.add(cost, attrs)
  getTokenCounter()?.add(usage.input_tokens,  { ...attrs, type: 'input' })
  getTokenCounter()?.add(usage.output_tokens, { ...attrs, type: 'output' })
  getTokenCounter()?.add(usage.cache_read_input_tokens ?? 0,
    { ...attrs, type: 'cacheRead' })
  getTokenCounter()?.add(usage.cache_creation_input_tokens ?? 0,
    { ...attrs, type: 'cacheCreation' })

  // 3. Log advisor sub-usage to 1P analytics
  let totalCost = cost
  for (const advisorUsage of getAdvisorUsage(usage)) {
    const advisorCost = calculateUSDCost(advisorUsage.model, advisorUsage)
    logEvent('tengu_advisor_tool_token_usage', {
      advisor_model: advisorUsage.model as AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS,
      input_tokens: advisorUsage.input_tokens,
      cost_usd_micros: Math.round(advisorCost * 1_000_000),
    })
    totalCost += addToTotalSessionCost(advisorCost, advisorUsage, advisorUsage.model)
  }
  return totalCost
}
```

Cost is stored in **microdollars** when sent as analytics metadata (`cost_usd_micros`) to keep the value as an integer. The raw float is used for OTel counters, preventing floating-point accumulation errors in event payloads.

#### Session Persistence

When the session ends or is suspended, `saveCurrentSessionCosts()` writes the full snapshot to the project config JSON. On resume, `restoreCostStateForSession()` rehydrates the in-memory counters from that snapshot—but only when the `sessionId` in config matches the current session.

```typescript
// Save on process exit via useCostSummary React hook
export function saveCurrentSessionCosts(fpsMetrics?: FpsMetrics): void {
  saveCurrentProjectConfig(current => ({
    ...current,
    lastCost:                          getTotalCostUSD(),
    lastAPIDuration:                   getTotalAPIDuration(),
    lastAPIDurationWithoutRetries:     getTotalAPIDurationWithoutRetries(),
    lastTotalInputTokens:              getTotalInputTokens(),
    lastTotalOutputTokens:             getTotalOutputTokens(),
    lastTotalCacheCreationInputTokens: getTotalCacheCreationInputTokens(),
    lastTotalCacheReadInputTokens:     getTotalCacheReadInputTokens(),
    lastTotalWebSearchRequests:        getTotalWebSearchRequests(),
    lastFpsAverage:                    fpsMetrics?.averageFps,
    lastModelUsage:                    Object.fromEntries(
      Object.entries(getModelUsage()).map(([model, u]) => [
        model,
        { inputTokens: u.inputTokens, outputTokens: u.outputTokens,
          cacheReadInputTokens: u.cacheReadInputTokens,
          cacheCreationInputTokens: u.cacheCreationInputTokens,
          costUSD: u.costUSD },
      ])
    ),
    lastSessionId: getSessionId(),
  }))
}

// Restore only when session IDs match
export function restoreCostStateForSession(sessionId: string): boolean {
  const data = getStoredSessionCosts(sessionId)
  if (!data) return false
  setCostStateForRestore(data)
  return true
}
```

### 03 The costHook.ts React Bridge

`costHook.ts` is a 22-line file that wires cost saving into the React lifecycle so the CLI's exit handler always captures accumulated cost state, regardless of process termination method.

```typescript
// costHook.ts — full file
import { useEffect } from 'react'
import { formatTotalCost, saveCurrentSessionCosts } from './cost-tracker.js'
import { hasConsoleBillingAccess } from './utils/billing.js'
import type { FpsMetrics } from './utils/fpsTracker.js'

export function useCostSummary(
  getFpsMetrics?: () => FpsMetrics | undefined,
): void {
  useEffect(() => {
    const f = () => {
      // Only print the summary if the user has console billing access
      if (hasConsoleBillingAccess()) {
        process.stdout.write('\n' + formatTotalCost() + '\n')
      }
      saveCurrentSessionCosts(getFpsMetrics?.())
    }
    process.on('exit', f)
    return () => { process.off('exit', f) }
  }, [])
}
```

**Why a React hook?** Claude Code's TUI is built with Ink (React for terminals). Registering the `process.on('exit')` handler inside `useEffect` ties the handler lifecycle to the React component tree—if the root component unmounts, the handler is removed, preventing memory leaks in test environments where many sessions run sequentially.

### 04 The Analytics Sink & Queue

The analytics module (`services/analytics/index.ts`) deliberately has **zero dependencies** to prevent import cycles. It exposes only three functions: `logEvent`, `logEventAsync`, and `attachAnalyticsSink`. Everything else—Datadog, 1P, GrowthBook—lives outside this file.

```typescript
// services/analytics/index.ts
const eventQueue: QueuedEvent[] = []
let sink: AnalyticsSink | null = null

export function attachAnalyticsSink(newSink: AnalyticsSink): void {
  if (sink !== null) return  // idempotent
  sink = newSink

  if (eventQueue.length > 0) {
    const queuedEvents = [...eventQueue]
    eventQueue.length = 0

    queueMicrotask(() => {
      for (const event of queuedEvents) {
        event.async
          ? void sink!.logEventAsync(event.eventName, event.metadata)
          :       sink!.logEvent(event.eventName, event.metadata)
      }
    })
  }
}

export function logEvent(eventName: string, metadata: LogEventMetadata): void {
  if (sink === null) {
    eventQueue.push({ eventName, metadata, async: false })
    return
  }
  sink.logEvent(eventName, metadata)
}
```

The sink implementation (`services/analytics/sink.ts`) routes each event through the sampling gate, then fans out to Datadog (after stripping `_PROTO_*` keys) and the 1P pipeline (with the full payload).

```typescript
// services/analytics/sink.ts — core routing
function logEventImpl(eventName: string, metadata: LogEventMetadata): void {
  const sampleResult = shouldSampleEvent(eventName)
  if (sampleResult === 0) return  // dropped

  const metadataWithSampleRate = sampleResult !== null
    ? { ...metadata, sample_rate: sampleResult }
    : metadata

  if (shouldTrackDatadog()) {
    // Strip _PROTO_* keys — Datadog is general-access, PII must stay out
    void trackDatadogEvent(eventName, stripProtoFields(metadataWithSampleRate))
  }
  // 1P receives full payload; exporter handles PII routing internally
  logEventTo1P(eventName, metadataWithSampleRate)
}
```

### 05 Datadog Pipeline

Datadog receives a curated allowlist of approximately 40 events. Events not in `DATADOG_ALLOWED_EVENTS` are silently dropped. This allowlist keeps the Datadog index small and prevents accidental leakage of high-volume internal events.

#### HTTP Batch with 15s Timer

Events accumulate in `logBatch[]`. Flushed every 15 seconds or when the batch hits 100 entries, whichever comes first.

#### Model & Tool Name Normalisation

External user model names are mapped to canonical short names or "other". MCP tool names collapse to "mcp" to prevent per-tool cardinality explosion.

#### SHA-256 → 30 Buckets

User ID is hashed and bucketed into 1 of 30 slots. Dashboards count unique buckets to approximate unique users without tracking individual IDs.

#### First-Party API Only

Datadog events are skipped for Bedrock, Vertex, and Foundry providers—`getAPIProvider() !== 'firstParty'` is a hard exit.

#### ddtags for Queryable Dims

High-cardinality dimensions (model, version, platform) land in `ddtags` so they're queryable from the Datadog aggregation API. The `message` field is reserved and not queryable.

#### Graceful Flush on Exit

`shutdownDatadog()` cancels the flush timer and POSTs any remaining batch immediately before `process.exit()`.

```typescript
// User bucket implementation — privacy-preserving cardinality
const NUM_USER_BUCKETS = 30

const getUserBucket = memoize((): number => {
  const userId = getOrCreateUserID()
  const hash = createHash('sha256').update(userId).digest('hex')
  return parseInt(hash.slice(0, 8), 16) % NUM_USER_BUCKETS
})
```

The hash truncates at 8 hex characters (32 bits) to stay within JavaScript safe-integer range. The modulo maps deterministically to 0–29. This gives alerting dashboards a ~3.3% error margin on unique user counts while making individual re-identification impractical.

### 06 First-Party Pipeline & OpenTelemetry

The 1P pipeline is built on the **OpenTelemetry Logs SDK**. A dedicated `LoggerProvider`—distinct from the customer-facing telemetry provider—drives a `BatchLogRecordProcessor` that buffers events and exports them to Anthropic's internal endpoint in batches.

```
sequenceDiagram
participant App
participant Logger as 1P Logger (OTel Logger)
participant Batch as BatchLogRecordProcessor
participant Exporter as FirstPartyEventLoggingExporter
participant Disk as Failed Events (JSONL on disk)
participant API as /api/event_logging/batch

App->>Logger: emit({ body: eventName, attributes })
Logger->>Batch: onEmit(log record)
Note over Batch: Buffer until scheduledDelayMillis or maxExportBatchSize (200)
Batch->>Exporter: export(logs[])
Exporter->>Exporter: transformLogsToEvents() hoist _PROTO_* keys
Exporter->>API: POST /api/event_logging/batch

alt success
  API-->>Exporter: 200 OK
  Exporter->>Disk: delete failed events file
else failure
  API-->>Exporter: 5xx / timeout
  Exporter->>Disk: appendEventsToFile() JSONL append (atomic)
  Exporter->>Exporter: scheduleBackoffRetry() quadratic: base * attempts²
end
```

#### Resilience: Disk-Backed Retry

When a POST fails, failed events are appended to a per-session JSONL file at `~/.claude/telemetry/1p_failed_events.<sessionId>.<BATCH_UUID>.json`. On the next startup, `retryPreviousBatches()` scans for any leftover files from the same session and retries them. Events survive process crashes and temporary network outages.

```typescript
// Quadratic backoff scheduler
private scheduleBackoffRetry(): void {
  if (this.cancelBackoff || this.isRetrying || this.isShutdown) return

  // Quadratic backoff: base * attempts²  (matches Statsig SDK)
  const delay = Math.min(
    this.baseBackoffDelayMs * this.attempts * this.attempts,
    this.maxBackoffDelayMs,  // caps at 30 000 ms
  )

  this.cancelBackoff = this.schedule(async () => {
    this.cancelBackoff = null
    await this.retryFailedEvents()
  }, delay)
}
```

Attempt 1: 500ms delay. Attempt 2: 2,000ms. Attempt 3: 4,500ms. Attempt 4: 8,000ms. Attempt 5: 12,500ms. Capped at 30,000ms from attempt 8 onward. Max 8 attempts total before the batch is permanently discarded.

#### OTel LoggerProvider Separation

The 1P `LoggerProvider` is **never registered globally** via `logs.setGlobalLoggerProvider()`. Customer-facing OTLP telemetry uses the global provider. Internal event logging uses a module-local provider, so customer OTLP endpoints never see internal Anthropic events and vice versa.

```typescript
// firstPartyEventLogger.ts — two providers, never mixed
// The 1P provider is kept module-local:
let firstPartyEventLoggerProvider: LoggerProvider | null = null
let firstPartyEventLogger: Logger | null = null

// Customer telemetry uses the global provider in instrumentation.ts:
// logs.setGlobalLoggerProvider(loggerProvider)  ← customer endpoint
// setEventLogger(logs.getLogger(...))           ← customer event logger
//
// 1P provider is obtained from the local instance, NOT from logs.getLogger():
firstPartyEventLogger = firstPartyEventLoggerProvider.getLogger(
  'com.anthropic.claude_code.events',
  MACRO.VERSION,
)
```

### 07 PII Segregation & the _PROTO_ Pattern

Claude Code uses a TypeScript phantom-type system to enforce PII hygiene at compile time, and a runtime key-naming convention to route sensitive values only to privileged storage.

#### Compile-Time Guards: never-typed marker types

```typescript
// services/analytics/index.ts

// Type is `never` — cannot hold a value, only used for casting.
// Any string passed to logEvent() must be explicitly cast to this type,
// forcing developers to verify it contains no code/file paths.
export type AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS = never

// Second marker for PII-tagged values destined for privileged BQ columns.
// These values travel with the event but are stripped before Datadog.
export type AnalyticsMetadata_I_VERIFIED_THIS_IS_PII_TAGGED = never
```

**How the never type enforces review:** Because `never` is the bottom type in TypeScript, no value can legitimately be assigned to it. Passing a string as `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` requires an explicit `as` cast. That cast is a signal to code reviewers: "the author consciously confirmed this string is safe to log." The type system can't verify the claim, but it forces a visible declaration of intent.

#### Runtime Guard: _PROTO_* Key Convention

Values that are PII but acceptable in privileged BigQuery columns (e.g. skill names, plugin names) are added to event payloads under keys prefixed with `_PROTO_`. The sink calls `stripProtoFields()` before Datadog fanout. The 1P exporter hoists known `_PROTO_` keys to dedicated proto fields, then calls `stripProtoFields()` again as a defensive catch-all.

```typescript
// services/analytics/index.ts
export function stripProtoFields<V>(
  metadata: Record<string, V>,
): Record<string, V> {
  let result: Record<string, V> | undefined
  for (const key in metadata) {
    if (key.startsWith('_PROTO_')) {
      if (result === undefined) result = { ...metadata }
      delete result[key]
    }
  }
  return result ?? metadata  // same reference when no _PROTO_ keys present
}

// In the 1P exporter — hoist then strip:
const { _PROTO_skill_name, _PROTO_plugin_name, _PROTO_marketplace_name, ...rest } = formatted.additional
const additionalMetadata = stripProtoFields(rest)  // catches future unknown _PROTO_* keys

events.push({
  event_type: 'ClaudeCodeInternalEvent',
  event_data: ClaudeCodeInternalEvent.toJSON({
    skill_name: typeof _PROTO_skill_name === 'string' ? _PROTO_skill_name : undefined,
    additional_metadata: Buffer.from(jsonStringify(additionalMetadata)).toString('base64'),
  }),
})
```

### 08 GrowthBook: Dynamic Config & Sampling

GrowthBook serves three distinct purposes in the analytics stack: feature flags for code behaviour, dynamic configuration for pipeline tuning, and experiment assignment tracking.

#### tengu_log_datadog_events

Enables or disables the entire Datadog pipeline per user/org/rollout. Checked once during `initializeAnalyticsGates()` and cached for the session.

#### tengu_event_sampling_config

JSON config mapping event names to `{ sample_rate: 0–1 }`. Events not in the config log at 100%. Rate 0 = drop all. Uses `getDynamicConfig_CACHED_MAY_BE_STALE`.

#### tengu_1p_event_batch_config

Controls the OTel BatchLogRecordProcessor: `scheduledDelayMillis`, `maxExportBatchSize`, `maxQueueSize`, endpoint, `skipAuth`. Live-reloadable via `reinitialize1PEventLoggingIfConfigChanged()`.

#### tengu_frond_boric

Mangled name for per-sink analytics killswitch. Shape: `{ datadog?: boolean, firstParty?: boolean }`. Value `true` stops all dispatch to that sink immediately.

```typescript
// firstPartyEventLogger.ts
export function shouldSampleEvent(eventName: string): number | null {
  const config = getEventSamplingConfig()   // GrowthBook dynamic config
  const eventConfig = config[eventName]

  if (!eventConfig) return null            // not configured → log 100%

  const sampleRate = eventConfig.sample_rate
  if (typeof sampleRate !== 'number' || sampleRate < 0 || sampleRate > 1)
    return null                              // invalid → fail-open (log 100%)
  if (sampleRate >= 1) return null           // rate=1 → log 100%, no metadata overhead
  if (sampleRate <= 0) return 0             // rate=0 → drop

  return Math.random() < sampleRate ? sampleRate : 0
}

// Caller in sink.ts:
// null   → event passes through, no sample_rate metadata added
// 0      → event dropped
// 0–1   → event passes, sample_rate added to metadata for downstream correction
```

#### Live Pipeline Reinitialisation

Long-running sessions pick up batch config changes automatically. When GrowthBook refreshes, `reinitialize1PEventLoggingIfConfigChanged()` compares the new config against `lastBatchConfig`. If different, it nulls the logger (so concurrent emits see no logger and bail cleanly), force-flushes the old buffer to disk, swaps in a new provider, then shuts down the old one in the background.

### 09 Customer-Facing OpenTelemetry

Enterprises can opt in to receive telemetry at their own OTLP endpoint by setting `CLAUDE_CODE_ENABLE_TELEMETRY=1` and `OTEL_EXPORTER_OTLP_ENDPOINT`. This path uses the **global** OTel providers and is entirely separate from the 1P analytics pipeline.

| Signal | Protocol Options | Notes |
|--------|------------------|-------|
| Metrics | otlp/grpc, otlp/http+json, otlp/http+proto, prometheus, console | Default temporality: delta. 60s interval for BigQuery reader, configurable for customer. |
| Logs | otlp/grpc, otlp/http+json, otlp/http+proto, console | 5s default export interval. User prompts redacted unless OTEL_LOG_USER_PROMPTS=1. |
| Traces | otlp/grpc, otlp/http+json, otlp/http+proto, console | Requires CLAUDE_CODE_ENABLE_TELEMETRY + isEnhancedTelemetryEnabled(). BatchSpanProcessor. |

#### BigQuery Metrics Exporter

API customers, C4E (Claude for Enterprise), and Teams users additionally get a `BigQueryMetricsExporter` that posts to `/api/claude_code/metrics` every 5 minutes. This is separate from customer OTLP—it runs whether or not `CLAUDE_CODE_ENABLE_TELEMETRY` is set.

```typescript
// utils/telemetryAttributes.ts
const METRICS_CARDINALITY_DEFAULTS = {
  OTEL_METRICS_INCLUDE_SESSION_ID:   true,   // included by default
  OTEL_METRICS_INCLUDE_VERSION:      false,  // excluded by default
  OTEL_METRICS_INCLUDE_ACCOUNT_UUID: true,   // included by default
}

export function getTelemetryAttributes(): Attributes {
  const attributes: Attributes = { 'user.id': getOrCreateUserID() }

  if (shouldIncludeAttribute('OTEL_METRICS_INCLUDE_SESSION_ID'))
    attributes['session.id'] = getSessionId()

  if (shouldIncludeAttribute('OTEL_METRICS_INCLUDE_VERSION'))
    attributes['app.version'] = MACRO.VERSION

  // OAuth account data only when actively using OAuth
  const oauthAccount = getOauthAccountInfo()
  if (oauthAccount) {
    if (oauthAccount.organizationUuid) attributes['organization.id'] = oauthAccount.organizationUuid
    if (oauthAccount.email)            attributes['user.email']    = oauthAccount.email
  }
  return attributes
}
```

Version is excluded by default to prevent high-cardinality time-series explosion in customer dashboards. Enterprise users can re-enable it with `OTEL_METRICS_INCLUDE_VERSION=1`.

### 10 Policy Limits & Disable Conditions

Analytics is disabled under four conditions, checked in `isAnalyticsDisabled()`:

```typescript
// services/analytics/config.ts
export function isAnalyticsDisabled(): boolean {
  return (
    process.env.NODE_ENV === 'test'                           ||
    isEnvTruthy(process.env.CLAUDE_CODE_USE_BEDROCK)           ||  // AWS Bedrock
    isEnvTruthy(process.env.CLAUDE_CODE_USE_VERTEX)            ||  // Google Vertex
    isEnvTruthy(process.env.CLAUDE_CODE_USE_FOUNDRY)           ||  // Azure Foundry
    isTelemetryDisabled()                                          // privacy setting
  )
}
```

#### 3P Provider Distinction

The feedback survey uses a separate check, `isFeedbackSurveyDisabled()`, which does _not_ block on 3P providers. The survey is a local UI prompt with no transcript data; enterprise customers capture responses via OTEL rather than Anthropic's internal pipeline.

#### Auth-Aware Event Dispatch

The 1P exporter probes auth state before every POST. If the OAuth token is expired, lacks the `user:profile` scope, or the trust dialog has not been accepted, it falls back to sending events without auth headers. A 401 response also triggers an automatic retry without auth. Events are never dropped solely because auth is unavailable—they degrade to unauthenticated delivery.

| Condition | Behaviour | Recovery |
|-----------|-----------|----------|
| Trust dialog not accepted | Skip auth headers | Automatic on next event after trust accepted |
| OAuth token expired | Skip auth headers | Token refresh; next batch uses fresh headers |
| No user:profile scope | Skip auth headers | Service key sessions — unauthenticated is expected |
| 401 response | Immediate retry without auth | Transparent to caller |
| 5xx / timeout | Queue to disk + quadratic backoff | Max 8 attempts; events dropped after that |
| Killswitch active | Throw; all batches queued to disk | Resumes when GrowthBook cache clears the flag |

### Key Takeaways

- Every token usage event flows through `addToTotalSessionCost()` which fans out to three destinations: in-memory accumulators, OTel counters, and the analytics event log—in a single synchronous call.
- The analytics sink is dependency-free and uses a queue-then-drain pattern so no events are lost during startup, regardless of how long GrowthBook or Datadog initialisation takes.
- Datadog and the 1P pipeline receive different payloads: Datadog gets `_PROTO_*` keys stripped before dispatch; the 1P exporter receives the full payload and routes PII-tagged values to privileged BigQuery columns.
- The `never`-typed marker types (`AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS`) create a compile-time paper trail—no runtime enforcement, but every logging call requires an explicit cast that code reviewers must approve.
- GrowthBook controls three independent dimensions: which events reach Datadog (feature gate), the sampling rate for each event type (dynamic config), and whether any sink is killed entirely (per-sink killswitch).
- The 1P `LoggerProvider` is never registered globally—customer OTLP endpoints and internal Anthropic analytics are completely isolated at the provider level.
- Failed events are durably persisted as append-only JSONL files on disk, retried with quadratic backoff, and survive process crashes. Max 8 attempts before permanent discard.
- Session cost state is serialised to the project config JSON on every exit and rehydrated on resume—costs accumulate correctly across `/resume` sessions without double-counting.

### Knowledge Check

**Q1. When `logEvent()` is called before `attachAnalyticsSink()` has run, what happens to the event?**

A) It is dropped silently.
B) It is pushed to an in-memory queue and drained asynchronously when the sink attaches.
C) It throws a ReferenceError.
D) It is written directly to the Datadog endpoint.

**Answer: B** — Events enqueue before the sink attaches, drained via `queueMicrotask`.

**Q2. What does `shouldSampleEvent()` return when the GrowthBook config has no entry for the event name?**

A) 0 (drop the event)
B) 1 (log at 100% and add sample_rate metadata)
C) null (log at 100%, no sample_rate metadata added)
D) A random number between 0 and 1

**Answer: C** — Missing config defaults to logging at full rate without metadata.

**Q3. Why is the `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` type defined as `never`?**

A) So TypeScript will strip the type from the compiled output, saving bytes.
B) Because `never` is the bottom type—no value can hold it, forcing an explicit `as` cast that acts as a visible code-review signal.
C) To prevent the compiler from widening the type to `string`.
D) It is a placeholder for a future union type that includes all safe string patterns.

**Answer: B** — The `never` type forces explicit casting, signaling code review.

**Q4. Which of the following correctly describes the relationship between the 1P OTel `LoggerProvider` and the customer-facing `LoggerProvider`?**

A) They share the global OTel provider; the exporter is differentiated by instrumentation scope name.
B) The 1P provider is a module-local instance never registered globally; the customer provider is registered via `logs.setGlobalLoggerProvider()`.
C) The 1P provider extends the customer provider and adds an extra processor.
D) They are the same provider but use different `BatchLogRecordProcessor` configurations.

**Answer: B** — The providers are completely isolated: 1P is local, customer is global.

**Q5. What happens to events when the `firstParty` sink killswitch (`tengu_frond_boric`) is set to `true`?**

A) Events are dropped permanently with no recovery path.
B) Events are queued to disk and the backoff timer keeps ticking; delivery resumes when GrowthBook clears the flag.
C) Events are rerouted to the Datadog pipeline instead.
D) The `logEventTo1P()` call blocks until the killswitch is cleared.

**Answer: B** — Batches queue to disk; resumes when GrowthBook clears the flag.

**Q6. Session cost state is saved on exit and restored on resume. What guards against restoring costs from a different session?**

A) A cryptographic signature on the JSON file.
B) The `lastSessionId` field in project config must match the current `getSessionId()`; otherwise `getStoredSessionCosts()` returns undefined.
C) Cost state is keyed by file path hash, not session ID.
D) There is no guard; old costs are always merged into the current session.

**Answer: B** — Session ID matching prevents cross-session cost merging.

**Q7. The `getUserBucket()` function in datadog.ts serves what purpose?**

A) Rate limiting—users in higher buckets receive slower event flushes.
B) Privacy-preserving cardinality estimation—dashboards count unique buckets (0–29) to approximate unique user counts without tracking raw user IDs.
C) Sharding—events from different buckets go to different Datadog indexes.
D) Experiment assignment—bucket determines which GrowthBook variant the user receives.

**Answer: B** — Bucketing allows privacy-preserving user count estimation via SHA-256 hashing.

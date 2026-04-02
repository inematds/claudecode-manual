# Lesson 20: Bridge & Remote Control — Complete Content

## Architecture Overview

Remote Control connects a local Claude Code REPL or headless bridge server to the claude.ai web front-end with bidirectional communication:

- **Outbound (local → cloud):** Claude's messages, tool activities, and result events stream to CCR (Cloud Code Runner) for real-time rendering on claude.ai
- **Inbound (cloud → local):** User prompts, permission decisions, interrupt signals, and control messages flow back to the Claude process

The authentication boundary is critical: bridge API calls use the user's claude.ai OAuth token. CCR worker endpoints additionally validate a short-lived JWT encoding `session_id` and `role=worker` claims—OAuth tokens alone are rejected.

## Bridge v1 vs v2: Env-based vs Env-less

### v1 — Environment-based (`initBridgeCore`)

- Registers environment via `POST /v1/environments`
- Polls for work using `GET /v1/environments/{id}/work`
- Decodes a **WorkSecret** (base64url JSON) to retrieve session JWT
- Supports perpetual mode (crash recovery via `bridge-pointer.json`)
- Supports multi-session spawn (`--spawn`, worktree mode)
- Transport: `HybridTransport` (WebSocket reads + HTTP POSTs)
- Used by: REPL, daemon, `claude remote-control` server mode

### v2 — Environment-less (`initEnvLessBridgeCore`)

- No Environments API—skips register/poll/ack/heartbeat cycle
- `POST /v1/code/sessions` → session ID
- `POST /v1/code/sessions/{id}/bridge` → worker JWT + epoch
- Each `/bridge` call IS the worker registration
- Transport: `SSETransport` (reads) + `CCRClient` (writes)
- Gated by `tengu_bridge_repl_v2` GrowthBook flag
- REPL-only—daemon/print stay on v1

### WorkSecret Decode

When the Environments API delivers work, it attaches an opaque `secret` field that is a base64url-encoded JSON blob:

```typescript
export function decodeWorkSecret(secret: string): WorkSecret {
  const json = Buffer.from(secret, 'base64url').toString('utf-8')
  const parsed: unknown = jsonParse(json)
  return parsed as WorkSecret
}

type WorkSecret = {
  version: 1
  session_ingress_token: string   // JWT for CCR worker endpoints
  api_base_url: string
  sources: Array<{ type: string; git_info?: ... }>
  auth: Array<{ type: string; token: string }>
  use_code_sessions?: boolean    // server-driven CCR v2 selector
}
```

The `session_ingress_token` is the short-lived JWT authorizing worker-tier operations.

### Gate Logic in `initReplBridge`

The v1/v2 branch is resolved after all authentication checks pass:

```typescript
if (isEnvLessBridgeEnabled() && !perpetual) {
  const versionError = await checkEnvLessBridgeMinVersion()
  if (versionError) {
    onStateChange?.('failed', 'run `claude update` to upgrade')
    return null
  }
  const { initEnvLessBridgeCore } = await import('./remoteBridgeCore.js')
  return initEnvLessBridgeCore({ baseUrl, orgUUID, title, ... })
}

// v1 path: env-based register/poll/ack/heartbeat
return initBridgeCore({ dir, machineName, branch, gitRepoUrl, ... })
```

Independent version floors gate each path: `tengu_bridge_min_version` for v1; `tengu_bridge_repl_v2_config.min_version` for v2.

### Session ID Compatibility

Infrastructure endpoints hand out `cse_*` IDs; the claude.ai frontend routes on `session_*`. Both are the same UUID with different prefixes:

```typescript
export function toCompatSessionId(id: string): string {
  if (!id.startsWith('cse_')) return id
  if (_isCseShimEnabled && !_isCseShimEnabled()) return id
  return 'session_' + id.slice('cse_'.length)
}

export function toInfraSessionId(id: string): string {
  if (!id.startsWith('session_')) return id
  return 'cse_' + id.slice('session_'.length)
}

export function sameSessionId(a: string, b: string): boolean {
  const aBody = a.slice(a.lastIndexOf('_') + 1)
  const bBody = b.slice(b.lastIndexOf('_') + 1)
  return aBody.length >= 4 && aBody === bBody
}
```

## Transport Layer: WebSocket, SSE & Hybrid

Transport abstraction lives in `bridge/replBridgeTransport.ts` and defines a single `ReplBridgeTransport` interface with two factory functions—one for v1, one for v2.

### ReplBridgeTransport Interface

```typescript
export type ReplBridgeTransport = {
  write(message: StdoutMessage): Promise<void>
  writeBatch(messages: StdoutMessage[]): Promise<void>
  close(): void
  isConnectedStatus(): boolean
  getStateLabel(): string
  setOnData(cb: (data: string) => void): void
  setOnClose(cb: (closeCode?: number) => void): void
  setOnConnect(cb: () => void): void
  connect(): void
  getLastSequenceNum(): number  // v1 always returns 0
  readonly droppedBatchCount: number
  reportState(state: SessionState): void    // v2 only; v1 no-op
  reportMetadata(m: Record<string, unknown>): void  // v2 only
  reportDelivery(id: string, s: 'processing'|'processed'): void
  flush(): Promise<void>  // v2 drains queue; v1 resolves immediately
}
```

### v1 Transport: `HybridTransport` Adapter

`createV1ReplTransport()` wraps `HybridTransport`, opening a WebSocket to Session-Ingress for inbound messages and using HTTP POST for outbound. v1 never uses SSE sequence numbers—the server-side cursor handles replay:

```typescript
export function createV1ReplTransport(
  hybrid: HybridTransport,
): ReplBridgeTransport {
  return {
    write: msg => hybrid.write(msg),
    writeBatch: msgs => hybrid.writeBatch(msgs),
    close: () => hybrid.close(),
    isConnectedStatus: () => hybrid.isConnectedStatus(),
    getLastSequenceNum: () => 0,     // WS replay != SSE seq nums
    reportState: () => {},              // no-op
    reportMetadata: () => {},
    reportDelivery: () => {},
    flush: () => Promise.resolve(),     // POSTs are awaited per-write
    // ... other pass-throughs
  }
}
```

### v2 Transport: SSETransport + CCRClient

v2 transport is asymmetric: reads come over SSE (Server-Sent Events); writes go through `CCRClient` posting to `/worker/events` via `SerialBatchEventUploader`. This split is intentional—the inbound SSE stream can pause while CCRClient's heartbeat and write path stay alive:

```typescript
export async function createV2ReplTransport(opts: {
  sessionUrl: string
  ingressToken: string
  sessionId: string
  initialSequenceNum?: number  // resume from this SSE seq on reconnect
  epoch?: number               // skip registerWorker if provided by /bridge
  getAuthToken?: () => string | undefined  // multi-session safe
  outboundOnly?: boolean       // skip SSE read (mirror mode)
}): Promise<ReplBridgeTransport> {

  const epoch = opts.epoch ?? (await registerWorker(sessionUrl, ingressToken))

  const sse = new SSETransport(sseUrl, {}, sessionId, ...)
  const ccr = new CCRClient(sse, new URL(sessionUrl), {
    getAuthHeaders,
    onEpochMismatch: () => {
      // 409 from server: our epoch was superseded by another worker
      ccr.close(); sse.close()
      onCloseCb?.(4090)  // poll-loop recovery code
      throw new Error('epoch superseded')
    },
  })
  // ACK 'processed' immediately alongside 'received' to prevent
  // phantom prompt floods on daemon restart (CC-1263)
  sse.setOnEvent(event => {
    ccr.reportDelivery(event.event_id, 'received')
    ccr.reportDelivery(event.event_id, 'processed')
  })
  return { write: msg => ccr.writeEvent(msg), ... }
}
```

#### Epoch Mismatch (409)

When the server detects a second worker has registered for the same session (e.g., bridge restarted), it sends a 409 to the old worker's next heartbeat or write. The old transport closes itself (code 4090) and the poll loop picks up fresh work with a new epoch.

#### FlushGate: Preventing Interleaved History + Live Messages

When a bridge session starts, it POST-flushes historical conversation. Any new messages arriving during that flush would interleave with history on the server. `FlushGate` queues them until flush completes:

```typescript
class FlushGate<T> {
  start(): void            // mark flush in-progress, enqueue() starts queuing
  end(): T[]               // flush done; return queued items for draining
  enqueue(...items: T[]): boolean  // true if active (queued), false if pass-through
  drop(): number           // discard queue (transport closed permanently)
  deactivate(): void       // transport swapped — new one will drain
}
```

`deactivate()` is called when the transport is replaced (e.g., reconnect after env loss)—items are preserved for the new transport's `end()` call.

#### SSE Sequence Numbers Across Reconnects

On v2, `getLastSequenceNum()` returns the SSE high-water mark. When a transport is swapped (epoch mismatch, 401, SSE drop), the new `SSETransport` is created with `initialSequenceNum` so it sends `from_sequence_num` and the server resumes without replaying full history. v1 always returns 0 because WS replay is cursor-based server-side.

## Permission Bridge

Remote Control surfaces tool-use permission prompts through the **control_request / control_response** protocol. When Claude wants to run a potentially dangerous tool, the REPL normally asks the local user. In Remote Control mode, that question travels through the bridge to claude.ai instead.

### Control Message Types

- `control_request` — bridge asks claude.ai "can this tool run?"
- `control_response` — claude.ai answers allow/deny, optionally with updated input
- `control_cancel_request` — server cancels a pending prompt (e.g., session ended)

### Permission Response Type

```typescript
type BridgePermissionResponse = {
  behavior: 'allow' | 'deny'
  updatedInput?: Record<string, unknown>
  updatedPermissions?: PermissionUpdate[]
  message?: string
}

function isBridgePermissionResponse(value: unknown): value is BridgePermissionResponse {
  if (!value || typeof value !== 'object') return false
  return (
    'behavior' in value &&
    (value.behavior === 'allow' || value.behavior === 'deny')
  )
}
```

### RemoteSessionManager: Permission Request/Response Flow

`remote/RemoteSessionManager.ts` coordinates the client side for CCR-hosted sessions. It receives permission requests from CCR via WebSocket and holds them until the user responds:

```typescript
class RemoteSessionManager {
  private pendingPermissionRequests = new Map<string, SDKControlPermissionRequest>()

  private handleControlRequest(req: SDKControlRequest): void {
    const { request_id, request: inner } = req
    if (inner.subtype === 'can_use_tool') {
      this.pendingPermissionRequests.set(request_id, inner)
      this.callbacks.onPermissionRequest(inner, request_id)
    } else {
      // Unsupported subtype: send error response immediately
      this.websocket?.sendControlResponse({ type: 'control_response', response: {
        subtype: 'error', request_id, error: 'Unsupported subtype'
      }})
    }
  }

  respondToPermissionRequest(requestId: string, result: RemotePermissionResponse): void {
    this.pendingPermissionRequests.delete(requestId)
    this.websocket?.sendControlResponse({
      type: 'control_response',
      response: {
        subtype: 'success', request_id: requestId,
        response: {
          behavior: result.behavior,
          ...(result.behavior === 'allow'
            ? { updatedInput: result.updatedInput }
            : { message: result.message }),
        },
      },
    })
  }
}
```

### Synthetic AssistantMessage for Remote Permission Prompts

When a permission request comes from remote CCR, no local message exists. `remote/remotePermissionBridge.ts` fabricates one:

```typescript
export function createSyntheticAssistantMessage(
  request: SDKControlPermissionRequest,
  requestId: string,
): AssistantMessage {
  return {
    type: 'assistant',
    uuid: randomUUID(),
    message: {
      id: `remote-${requestId}`,
      type: 'message',
      role: 'assistant',
      content: [{
        type: 'tool_use',
        id: request.tool_use_id,
        name: request.tool_name,
        input: request.input,
      }],
      // zero-usage stub fields ...
    } as AssistantMessage['message'],
    requestId: undefined,
    timestamp: new Date().toISOString(),
  }
}

// For tools not known locally (e.g. MCP tools on the remote container):
export function createToolStub(toolName: string): Tool {
  return {
    name: toolName,
    isEnabled: () => true,
    needsPermissions: () => true,
    // ... renders first 3 input key:value pairs for display
    call: async () => ({ data: '' }),
  } as unknown as Tool
}
```

### Standalone Bridge Permission Flow

In `claude remote-control` server mode, permission requests from child Claude processes are intercepted via `sessionRunner.ts`, forwarded to the server via `api.sendPermissionResponseEvent()`, and the response is written back to the child's stdin.

## Remote Control Command (`/remote-control`)

The `/remote-control` slash command lives at `commands/bridge/bridge.tsx` as a React component rendered inside the REPL's Ink terminal UI.

### What It Does

- Checks if a bridge is already connected—if yes, shows a disconnect dialog with the session URL and QR code option
- If not connected, runs pre-flight checks (`checkBridgePrerequisites`) then sets `replBridgeEnabled: true` in `AppState`
- `useReplBridge` in `REPL.tsx` watches `replBridgeEnabled` and calls `initReplBridge()`
- The `name` argument (`/remote-control my-session`) sets an explicit session title

### BridgeToggle Component Logic

```typescript
function BridgeToggle({ onDone, name }) {
  const replBridgeConnected = useAppState(s => s.replBridgeConnected)
  const replBridgeEnabled   = useAppState(s => s.replBridgeEnabled)
  const [showDisconnectDialog, setShow] = useState(false)

  useEffect(() => {
    if ((replBridgeConnected || replBridgeEnabled) && !replBridgeOutboundOnly) {
      setShow(true)  // already connected → show dialog
      return
    }
    (async () => {
      const error = await checkBridgePrerequisites()
      if (error) { onDone(error, { display: 'system' }); return }
      setAppState(prev => ({
        ...prev,
        replBridgeEnabled: true,
        replBridgeExplicit: true,
        replBridgeOutboundOnly: false,
        replBridgeInitialName: name,
      }))
      onDone('Remote Control connecting…', { display: 'system' })
    })()
  }, [])  // fires once on mount
}
```

### Entitlement Gates

Five checks must all pass before `initReplBridge` proceeds:

1. **Runtime gate**: `isBridgeEnabledBlocking()`—requires `tengu_ccr_bridge` GrowthBook flag AND a claude.ai OAuth subscription (no Bedrock/Vertex/API-key auth)
2. **OAuth token present**: `getBridgeAccessToken()` must return a value
3. **Organization policy**: `isPolicyAllowed('allow_remote_control')`—enterprise admins can disable RC for all org members
4. **Token freshness**: proactive refresh + skip if expired-and-unrefreshable (avoids guaranteed 401 loops)
5. **Min version**: `checkBridgeMinVersion()` for v1, `checkEnvLessBridgeMinVersion()` for v2—ops can force upgrades fleet-wide

If any gate fails, `onStateChange?.('failed', reason)` is called and the function returns `null`.

### CCR Mirror Mode (`outboundOnly`)

When `isCcrMirrorEnabled()` is true (env var `CLAUDE_CODE_CCR_MIRROR` or GrowthBook flag), every local session starts an outbound-only bridge. The SSE read stream is skipped—the bridge only streams events to claude.ai without accepting inbound prompts. The session shows up in the claude.ai session list as a read-only view.

### Session Title Derivation

Titles are set in two stages: count-1 uses a fast placeholder (first sentence of the first user message, truncated to 50 chars); count-3 fires Haiku (`generateSessionTitle`) over the full conversation text. Explicit titles from `/remote-control <name>` or `/rename` are never auto-overwritten.

## CCR Integration: Cloud Code Runner

CCR is the server-side execution environment processing sessions requested from claude.ai without a local CLI present. The local bridge connects to CCR's session-ingress layer to render output and handle permissions for sessions originally created remotely.

### SessionsWebSocket: Subscribing to a CCR Session

`remote/SessionsWebSocket.ts` connects to `wss://api.anthropic.com/v1/sessions/ws/{sessionId}/subscribe` to receive the real-time event stream of an active CCR session.

#### Connection and Reconnect Logic

```typescript
const RECONNECT_DELAY_MS = 2000
const MAX_RECONNECT_ATTEMPTS = 5
const MAX_SESSION_NOT_FOUND_RETRIES = 3  // 4001 can be transient during compaction

const PERMANENT_CLOSE_CODES = new Set([
  4003,  // unauthorized — stop immediately
])

private handleClose(closeCode: number): void {
  if (PERMANENT_CLOSE_CODES.has(closeCode)) {
    this.callbacks.onClose?.()
    return
  }
  if (closeCode === 4001) {
    // session not found — retry up to 3 times with linear backoff
    this.sessionNotFoundRetries++
    if (this.sessionNotFoundRetries > MAX_SESSION_NOT_FOUND_RETRIES) {
      this.callbacks.onClose?.()
      return
    }
    this.scheduleReconnect(RECONNECT_DELAY_MS * this.sessionNotFoundRetries, ...)
    return
  }
  if (previousState === 'connected' && this.reconnectAttempts < MAX_RECONNECT_ATTEMPTS) {
    this.reconnectAttempts++
    this.scheduleReconnect(RECONNECT_DELAY_MS, ...)
  }
}
```

Bun's native WebSocket passes auth via headers; Node uses the `ws` package with the same auth headers (no post-connect auth message needed for the subscribe endpoint).

### SDKMessage Adapter

CCR sends SDK-format messages (`assistant`, `stream_event`, `result`, `system`, `tool_progress`, etc.). `remote/sdkMessageAdapter.ts` translates them to the REPL's internal `Message` type for local rendering.

#### Message Conversion Table

```typescript
function convertSDKMessage(msg: SDKMessage, opts?: ConvertOptions): ConvertedMessage {
  switch (msg.type) {
    case 'assistant':
      return { type: 'message', message: convertAssistantMessage(msg) }

    case 'stream_event':
      return { type: 'stream_event', event: convertStreamEvent(msg) }

    case 'result':
      // Only show errors — success results are noise in multi-turn
      return msg.subtype !== 'success'
        ? { type: 'message', message: convertResultMessage(msg) }
        : { type: 'ignored' }

    case 'system':
      if (msg.subtype === 'init')            return { type: 'message', message: convertInitMessage(msg) }
      if (msg.subtype === 'status')          return { ... }  // 'compacting' → banner
      if (msg.subtype === 'compact_boundary') return { ... }  // marks compaction point
      return { type: 'ignored' }

    case 'tool_progress':  return { type: 'message', message: convertToolProgressMessage(msg) }
    case 'user':          return { type: 'ignored' }  // already added locally by REPL
    case 'auth_status':   return { type: 'ignored' }
    default:             return { type: 'ignored' }  // forward-compat: unknown types silently dropped
  }
}
```

User messages are intentionally ignored. In live WS mode, the REPL already added the user's typed message locally before sending it to CCR. If the adapter converted inbound user messages, they'd appear twice. `convertUserTextMessages: true` is only set when replaying historical events.

## Standalone Bridge: `claude remote-control` Server Mode

`bridge/bridgeMain.ts` implements the `runBridgeLoop()` function used by `claude remote-control` as a persistent server. Unlike the REPL bridge (one session, inline), the standalone bridge manages a pool of concurrent child Claude processes.

### Key Concepts

- **SpawnMode**: `single-session` (one session, bridge exits), `worktree` (each session gets an isolated git worktree), `same-dir` (sessions share cwd—can stomp each other)
- **maxSessions**: configurable pool size (default 32); bridge pauses polling when at capacity and uses `capacityWake` to resume immediately when a session completes
- **Token refresh**: v1 sessions receive an updated OAuth token via `handle.updateAccessToken()`; v2 sessions call `reconnectSession` to trigger server re-dispatch with a fresh JWT (OAuth tokens can't be used in CCR worker endpoints)

### Poll Loop Backoff and Reconnect Strategy

```typescript
const DEFAULT_BACKOFF: BackoffConfig = {
  connInitialMs:    2_000,
  connCapMs:      120_000,  // 2 min
  connGiveUpMs:   600_000,  // 10 min
  generalInitialMs:   500,
  generalCapMs:    30_000,
  generalGiveUpMs:600_000,
}

// Sleep detection: if a poll tick is delayed by >2× the cap,
// the machine probably slept — reset error budget and reconnect immediately
function pollSleepDetectionThresholdMs(b: BackoffConfig): number {
  return b.connCapMs * 2  // 240_000ms — above max backoff cap
}
```

Connection errors and general poll errors have independent backoff budgets. Connection errors (registration/WebSocket failures) give up at 10 minutes. General errors (HTTP 500 on work poll) also give up at 10 minutes—the server is the authority on session liveness.

### Child Process Spawning and Session Tracking

```typescript
const activeSessions = new Map<string, SessionHandle>()
const sessionStartTimes = new Map<string, number>()
const sessionIngressTokens = new Map<string, string>()
const sessionTimers = new Map<string, ReturnType<typeof setTimeout>>()

// Per-session timeout watchdog (default 24h)
const DEFAULT_SESSION_TIMEOUT_MS = 24 * 60 * 60 * 1000

type SessionHandle = {
  sessionId: string
  done: Promise<SessionDoneStatus>
  kill(): void
  forceKill(): void
  activities: SessionActivity[]      // ring buffer (last 10)
  currentActivity: SessionActivity | null
  accessToken: string
  lastStderr: string[]              // ring buffer (last 10 lines)
  writeStdin(data: string): void
  updateAccessToken(token: string): void
}
```

### Heartbeat and JWT Expiry Recovery

Active work items are heartbeated at a GrowthBook-configured interval. Heartbeats use the session ingress JWT (not OAuth) via `SessionIngressAuth`—a lightweight DB-free JWT validation. On 401/403 (JWT expired), the bridge calls `reconnectSession` to re-queue the work so the next poll delivers fresh credentials:

```typescript
async function heartbeatActiveWorkItems() {
  for (const [sessionId] of activeSessions) {
    const ingressToken = sessionIngressTokens.get(sessionId)
    try {
      await api.heartbeatWork(environmentId, workId, ingressToken)
    } catch (err) {
      if (err.status === 401 || err.status === 403) {
        // JWT expired — re-dispatch so next poll delivers fresh token
        await api.reconnectSession(environmentId, sessionId)
      }
    }
  }
}
```

A proactive token refresh scheduler also fires 5 minutes before expiry. v1 sessions receive the new OAuth token directly; v2 sessions go through `reconnectSession` because CCR worker endpoints reject OAuth tokens.

### BridgeWorkerType

Every environment registration includes a `worker_type` string sent as `metadata.worker_type`. The Web UI uses this to filter sessions in its session picker:

- `"claude_code"` — standard REPL session
- `"claude_code_assistant"` — assistant mode (KAIROS feature flag)
- `"cowork"` — Desktop Cowork (sent by claude.ai desktop app, not this codebase)

## Key Takeaways

1. **Two bridge architectures coexist:** v1 (env-based with poll/ack/heartbeat loop) and v2 ("env-less"—direct OAuth → worker JWT via `POST /bridge`). The GrowthBook flag `tengu_bridge_repl_v2` controls which path the REPL takes.

2. **Transport is asymmetric in v2:** inbound uses SSE (with sequence numbers for gapless reconnect), outbound uses `CCRClient` posting to `/worker/events`. v1 uses WebSocket for both directions via `HybridTransport`.

3. **The `FlushGate` prevents history/live message interleaving:** historical messages are flushed as one HTTP batch on connect; any live messages arriving during that window are queued and drained after the flush completes.

4. **Permission flow is bidirectional over the same transport:** `control_request` travels from Claude → server → claude.ai; `control_response` (allow/deny) travels back. Remote tool stubs are synthesized locally for tools the client doesn't know about.

5. **Session IDs have two costumes:** infrastructure layer uses `cse_*`, the compat/client-facing API uses `session_*`. `sameSessionId()` compares by UUID body so the poll loop doesn't reject its own session. `toCompatSessionId()` is kill-switched via GrowthBook.

6. **Auth for worker endpoints requires a JWT, not OAuth:** CCR validates the JWT's `session_id` claim and `role=worker`. Token refresh for v2 sessions triggers server re-dispatch (`reconnectSession`) rather than pushing a new OAuth token to the running process.

7. **`claude remote-control` server mode** supports multi-session concurrent execution with `single-session`, `worktree`, and `same-dir` spawn modes, a configurable pool size, and a 24-hour per-session timeout watchdog.

## Knowledge Check — Quiz Questions

**Question 1:** In the v2 (env-less) bridge path, what does `POST /v1/code/sessions/{id}/bridge` return that replaces the entire Environments API polling workflow?

A. A WorkSecret encoded in base64url containing the OAuth refresh token

B. A worker JWT (short-lived) and a worker epoch that authorises CCR worker-tier operations

C. A WebSocket URL and session_ingress_token that work like the v1 HybridTransport

D. An environment_id and environment_secret used to poll for queued work

**Question 2:** Why does the v2 transport immediately ACK `processed` (not just `received`) upon receiving an SSE event?

A. The CCRClient spec requires both events to be sent in a single HTTP round-trip

B. The server only uses `processed` for rate-limiting; `received` alone triggers duplicate delivery

C. Without `processed`, events stay in the server's re-queue and flood the session with phantom prompts on every daemon restart

D. SSETransport's `setOnEvent` fires only for `processed`-type delivery acknowledgements

**Question 3:** A `cse_abc123` session ID arrives from the work poll. Which function converts it for use with the _client-facing_ sessions API (`/v1/sessions/{id}/archive`)?

A. `toInfraSessionId()` — re-tags to `cse_*`

B. `toCompatSessionId()` — re-tags to `session_*`

C. `sameSessionId()` — compares UUID bodies and selects the matching ID

D. No conversion is needed — the archive endpoint accepts both prefixes

**Question 4:** What is the purpose of `FlushGate.deactivate()` (as opposed to `drop()`)?

A. It stops all queuing immediately and discards pending items because the transport will never reconnect

B. It marks flush as complete and returns the queued items for the caller to drain

C. It clears the active flag but keeps queued items so the _replacement_ transport can drain them on its next flush

D. It pauses queuing without clearing pending items and schedules a retry after the reconnect backoff

**Question 5:** In `claude remote-control` server mode (standalone bridge), what happens when a v2 session's JWT expires during a heartbeat?

A. `handle.updateAccessToken(newOAuthToken)` is called to push a fresh credential to the running child process

B. The session is killed and re-spawned with a fresh JWT from the next poll

C. `api.reconnectSession()` is called to trigger server re-dispatch so the next poll delivers fresh work with a new JWT

D. The bridge retries the heartbeat up to 5 times with exponential backoff before declaring the session dead

**Question 6:** `createToolStub(toolName)` in `remote/remotePermissionBridge.ts` exists because:

A. MCP tools require a stub to bypass CCRClient's schema validation before being forwarded

B. The remote CCR container may have MCP tools that the local CLI doesn't know about, so a stub routes to FallbackPermissionRequest for display

C. All tools need a stub for the synthetic AssistantMessage because real tool definitions are never serialized over the wire

D. Stubs are only needed when `outboundOnly` is true and no inbound tool messages can arrive

# Upstream Proxy System — Complete Lesson Content

## Lesson 36: Upstream Proxy System

**Subtitle:** How Claude Code routes outbound traffic through a MITM-capable WebSocket tunnel inside CCR session containers — and does so without ever trusting the agent loop with the session token.

---

## 01 What Problem Does This Solve?

Claude Code Remote (CCR) runs inside sandboxed GKE containers. Enterprise customers need outbound HTTP traffic — `curl`, GitHub CLI, `npm install`, Datadog agents — to carry injected credentials (e.g. `DD-API-KEY`) and to be auditable. The solution is an **HTTPS CONNECT proxy** running locally on `127.0.0.1` inside each container, tunnelled back to Anthropic's gateway over a WebSocket. The gateway MITMs TLS, injects org-configured headers, and forwards to the real upstream.

### Architecture constraint

CCR ingress is a GKE L7 load balancer with path-prefix routing. Raw TCP CONNECT tunnels aren't routable there, so the relay uses WebSocket — the same upgrade pattern already used by session tunnels.

### Fails open by design

Every step in `initUpstreamProxy` is wrapped so that _any_ error logs a warning and returns `{ enabled: false }`. A broken proxy must never break a working session.

---

## 02 Initialization Sequence (`upstreamproxy.ts`)

`initUpstreamProxy()` is called exactly once from `init.ts` at startup. It runs six steps, each conditional on the previous succeeding:

### Step 1: Guard env vars

Returns early if `CLAUDE_CODE_REMOTE` or `CCR_UPSTREAM_PROXY_ENABLED` is not truthy. No work on developer machines.

### Step 2: Read session token

Reads a short-lived credential from `/run/ccr/session_token`. File is written by the CCR container orchestrator before the agent starts.

### Step 3: Set non-dumpable

Calls `prctl(PR_SET_DUMPABLE, 0)` via Bun FFI. Blocks same-UID `ptrace` / `gdb`, so a prompt-injected debugger cannot scrape the token from heap memory.

### Step 4: Download CA bundle

Fetches the MITM proxy's CA cert from `/v1/code/upstreamproxy/ca-cert` and concatenates it with the system CA bundle. All runtimes (curl, Python, Node) are patched via env vars.

### Step 5: Start relay

Calls `startUpstreamProxyRelay()`, which opens a local TCP server on an ephemeral port. Returns port number.

### Step 6: Unlink token file

Deletes `/run/ccr/session_token` from disk _only after_ the relay is confirmed listening. Token lives in heap memory only from this point on.

### Security detail

The token is deleted _after_ the relay confirms it is listening, not before. This allows a supervisor process to restart the container and retry — the token file would still be on disk if anything earlier in the sequence failed.

---

## 03 The CONNECT-over-WebSocket Relay (`relay.ts`)

The relay is a local TCP server. Clients (`curl`, `gh`, `kubectl`) connect and send standard HTTP `CONNECT host:port` requests. The relay upgrades a WebSocket to Anthropic's gateway and tunnels bytes bidirectionally.

### Sequence Diagram

```
participant C as curl / gh / npm
participant R as Local Relay 127.0.0.1:<port>
participant G as CCR Gateway (WebSocket)
participant U as Real Upstream

C->>R: HTTP CONNECT api.example.com:443
R->>G: WS upgrade (Bearer token)
G-->>R: 101 Switching Protocols
R->>G: encodeChunk(CONNECT line + Proxy-Auth)
G-->>R: encodeChunk(HTTP/1.1 200 Connection Established)
R-->>C: HTTP/1.1 200 Connection Established
C->>R: TLS ClientHello (raw bytes)
R->>G: encodeChunk(TLS bytes)
G->>U: Forward (MITM TLS, inject headers)
U-->>G: Response
G-->>R: encodeChunk(response)
R-->>C: Response bytes
```

### Two-Phase Connection Handling

The `handleData()` function implements a clean two-phase state machine per TCP connection:

```javascript
// Phase 1: accumulate until CRLF CRLF terminates the CONNECT header
if (!st.ws) {
  st.connectBuf = Buffer.concat([st.connectBuf, data])
  const headerEnd = st.connectBuf.indexOf('\r\n\r\n')
  if (headerEnd === -1) { /* guard: reject if > 8192 bytes */ return }
  // parse CONNECT host:port, stash trailing bytes, open WS tunnel
  openTunnel(sock, st, firstLine, wsUrl, authHeader, wsAuthHeader)
  return
}
// Phase 2: pump bytes to WebSocket (buffer if WS not open yet)
if (!st.wsOpen) { st.pending.push(Buffer.from(data)); return }
forwardToWs(st.ws, data)
```

### Race condition solved

TCP can coalesce the CONNECT header and the TLS ClientHello into a single packet. Any bytes arriving after `\r\n\r\n` go into `st.pending[]` and are flushed in `ws.onopen`. Without this, the ClientHello would be silently dropped.

---

## 04 Hand-Rolled Protobuf Encoding

Bytes crossing the WebSocket are wrapped in an `UpstreamProxyChunk` protobuf message. Rather than pulling in `protobufjs` for a single one-field message, the code encodes and decodes by hand:

```javascript
// message UpstreamProxyChunk { bytes data = 1; }
// tag = (field 1 << 3) | wire_type 2 (length-delimited) = 0x0a
export function encodeChunk(data: Uint8Array): Uint8Array {
  const len = data.length
  const varint: number[] = []
  let n = len
  while (n > 0x7f) { varint.push((n & 0x7f) | 0x80); n >>>= 7 }
  varint.push(n)
  const out = new Uint8Array(1 + varint.length + len)
  out[0] = 0x0a
  out.set(varint, 1)
  out.set(data, 1 + varint.length)
  return out
}
```

The tag byte is always `0x0a`. The length is a standard protobuf varint — 7 bits per byte, MSB set on continuation bytes. `decodeChunk` validates both the tag and that the declared length fits within the buffer, returning `null` for malformed frames and `new Uint8Array(0)` for zero-length keepalive chunks.

### Why protobuf and not plain bytes?

The CCR gateway uses `gateway.NewWebSocketStreamAdapter` server-side, which expects this framing. The `Content-Type: application/proto` upgrade header tells the server to use binary proto deserialization instead of protojson — without it the server silently fails with EOF.

---

## 05 Bun vs Node Runtime Dispatch

`startUpstreamProxyRelay()` dispatches at runtime: if `typeof Bun !== 'undefined'` it calls `startBunRelay`; otherwise `startNodeRelay`. CCR containers run Claude Code under Node, not Bun, so `startNodeRelay` is the production path. `startBunRelay` is used in developer environments and must be tested explicitly.

### Key difference: write backpressure

Node's `net.Socket.write()` always buffers — a `false` return signals backpressure but no bytes are lost. Bun's `sock.write()` does a partial kernel write and **silently drops the remainder**. The Bun relay tracks unwritten bytes in a per-socket `writeBuf: Uint8Array[]` and flushes them in the `drain` handler.

```javascript
// Bun: tail-queue what the kernel didn't accept
const n = sock.write(bytes)
if (n < bytes.length) st.writeBuf.push(bytes.subarray(n))

// drain handler: flush the queue
while (st.writeBuf.length > 0) {
  const chunk = st.writeBuf[0]!
  const n = sock.write(chunk)
  if (n < chunk.length) { st.writeBuf[0] = chunk.subarray(n); return }
  st.writeBuf.shift()
}
```

### WebSocket proxy agent

CCR containers sit behind an egress gateway — direct outbound is blocked. The WS upgrade itself must go through an HTTP CONNECT proxy. Node's `undici`/`globalThis.WebSocket` does not consult the global dispatcher for upgrade requests, so the Node relay imports the `ws` npm package and passes an explicit `agent`. Bun's native WebSocket accepts a `proxy` URL directly.

---

## 06 Environment Variable Propagation

`getUpstreamProxyEnv()` returns a record of env vars that `subprocessEnv()` merges into every child process spawned by the agent — Bash tool, MCP servers, LSP, git hooks. This ensures all subprocess traffic routes through the relay without per-tool configuration.

| Variable | Value | Purpose |
|----------|-------|---------|
| `HTTPS_PROXY` / `https_proxy` | `http://127.0.0.1:<port>` | Route HTTPS traffic through the relay (most tools) |
| `NO_PROXY` / `no_proxy` | loopback, RFC1918, Anthropic API, GitHub, npm, PyPI, crates | Bypass the relay for traffic that must not be intercepted |
| `SSL_CERT_FILE` | `~/.ccr/ca-bundle.crt` | OpenSSL / curl CA trust |
| `NODE_EXTRA_CA_CERTS` | same | Node.js TLS trust |
| `REQUESTS_CA_BUNDLE` | same | Python `requests` / `httpx` trust |
| `CURL_CA_BUNDLE` | same | curl CA trust |

### HTTPS only

Only `HTTPS_PROXY` is set, never `HTTP_PROXY`. The relay only handles `CONNECT` — routing plain HTTP through it would return a `405 Method Not Allowed`.

### NO_PROXY: Anthropic API exclusion

The Anthropic API (`api.anthropic.com`) is listed three ways in `NO_PROXY`: as `anthropic.com`, `.anthropic.com`, and `*.anthropic.com`. This is because NO_PROXY parsing differs across runtimes — Bun and Go use glob match, Python urllib/httpx use suffix match. All three forms are required for reliable bypass.

### Inherited proxy vars

Child CLI processes (e.g. a sub-agent spawned inside the session) cannot re-initialize the relay — the token file was already unlinked. If the parent's `HTTPS_PROXY` and `SSL_CERT_FILE` are both set in the environment, `getUpstreamProxyEnv()` detects this and passes all proxy vars through to grandchild processes, maintaining the chain.

---

## 07 Security Model

### Prompt injection and the token

A malicious prompt could instruct the agent to run `gdb -p $PPID` to read the session token off the heap. The `prctl(PR_SET_DUMPABLE, 0)` call (via Bun FFI into `libc.so.6`) blocks this — same-UID ptrace is rejected by the kernel after this call.

```javascript
const PR_SET_DUMPABLE = 4
const rc = lib.symbols.prctl(PR_SET_DUMPABLE, 0n, 0n, 0n, 0n)
// Linux-only; silently no-ops on macOS/Windows
```

### Dual authentication layers

The relay uses two separate credentials at two different points:

- **WS upgrade**: `Authorization: Bearer <token>` — authenticates the relay to the CCR gateway at the WebSocket protocol level.
- **CONNECT tunnel**: `Proxy-Authorization: Basic base64(sessionId:token)` — carried inside the first protobuf chunk, authorizes the specific upstream tunnel request.

### TLS corruption guard

Once the server's `200 Connection Established` is forwarded, the `st.established` flag is set. If the WebSocket subsequently errors, the relay closes the TCP connection rather than writing a plaintext `502` response — writing ASCII into an active TLS stream would corrupt the client's cipher state.

---

## 08 Keepalive Strategy

The CCR sidecar has a 50-second idle timeout. The relay sends a ping every 30 seconds (`PING_INTERVAL_MS = 30_000`). Not all WebSocket implementations expose a `ping()` frame API, so the relay uses an application-level keepalive: a zero-length `encodeChunk(new Uint8Array(0))` that the server ignores but which resets the idle timer. The pinger is stored in `ConnState.pinger` and cleared in `cleanupConn()`.

---

## 09 Chunk Sizing and Backpressure

`forwardToWs()` splits outbound data into chunks of at most `MAX_CHUNK_BYTES = 512 * 1024` (512 KB). This is Envoy's per-request buffer cap. Git push payloads can exceed this; the chunking ensures they are streamed rather than rejected.

```javascript
for (let off = 0; off < data.length; off += MAX_CHUNK_BYTES) {
  const slice = data.subarray(off, off + MAX_CHUNK_BYTES)
  ws.send(encodeChunk(slice))
}
```

---

## Key Takeaways

- The upstream proxy is **container-only**: two env var guards (`CLAUDE_CODE_REMOTE` + `CCR_UPSTREAM_PROXY_ENABLED`) ensure it never activates on developer machines.
- The session token is deleted from disk _after_ the relay confirms it is listening — allowing supervisor restarts to retry on partial failure.
- `prctl(PR_SET_DUMPABLE, 0)` is the kernel-level defense against prompt-injection-driven `ptrace` attacks on the token in heap memory.
- The WebSocket uses hand-rolled protobuf framing (`encodeChunk` / `decodeChunk`) to satisfy the server's `NewWebSocketStreamAdapter` — one tag byte + varint length + payload.
- Bun and Node relay implementations are identical in protocol but differ in write backpressure: Bun silently truncates partial kernel writes; Node buffers unconditionally.
- All eight proxy-related env vars are injected into every subprocess via `getUpstreamProxyEnv()`, covering curl, Node, Python, and OpenSSL trust stores.
- Zero-length protobuf chunks serve as application-level keepalives when the native WebSocket ping API is unavailable.

---

## Deep Dive: CA Bundle Download and the 5-Second Timeout

`downloadCaBundle()` fetches the MITM proxy's CA certificate from `<baseUrl>/v1/code/upstreamproxy/ca-cert`. Bun has no default fetch timeout, so a hung endpoint would block CLI startup indefinitely. The call uses `AbortSignal.timeout(5000)`.

The downloaded PEM is concatenated with the existing system CA bundle (`/etc/ssl/certs/ca-certificates.crt`), written to `~/.ccr/ca-bundle.crt`, and referenced via all runtime-specific env vars. If the system bundle read fails, the concatenation proceeds with an empty string — the MITM CA alone is sufficient since CCR containers have no other outbound TLS to trust.

The base URL comes from `ANTHROPIC_BASE_URL` (injected by the container orchestrator), not from `getOauthConfig()`. The OAuth config keys off `USER_TYPE` and `USE_{LOCAL,STAGING}_OAUTH`, neither of which is set in CCR containers, so it always returned the prod URL even on staging — which 404'd on the CA endpoint.

---

## Deep Dive: ConnState Fields and the Closed/Established Guards

Each TCP connection carries a `ConnState` object with several boolean guards:

- `wsOpen`: true once `ws.onopen` fires. Until then, incoming client bytes go into `st.pending[]`.
- `established`: set to true when the server's `200 Connection Established` is forwarded. After this point, writing a plaintext error response would corrupt the TLS stream.
- `closed`: prevents double-close. `ws.onerror` is always followed by `ws.onclose`; without this flag, the second handler would call `sock.end()` on an already-ended socket.

The `pending: Buffer[]` array handles two distinct races: bytes that arrived after the CONNECT header but before `ws.onopen` (TCP coalescing), and `data` events that fire while the WS handshake is still in flight.

---

## Deep Dive: Why NO_PROXY Lists Anthropic Domains

The Anthropic API is listed in `NO_PROXY` for two reasons:

1. The CCR gateway never has an upstream route matching Anthropic's own API. Routing API calls through the MITM would cause them to fail at the gateway, not just at TLS verification.
2. Python's `httpx`/`certifi` uses its own bundled CA store and does not pick up `REQUESTS_CA_BUNDLE` in all configurations. Even if routing worked, some runtimes would reject the forged certificate.

GitHub and package registries (`registry.npmjs.org`, `pypi.org`, etc.) are also bypassed because CCR containers already have direct egress to them — routing these through the MITM would be unnecessary overhead and would break if the MITM proxy were unavailable.

---

## Check Your Understanding

**Question 1**

Why is `prctl(PR_SET_DUMPABLE, 0)` called before anything else in the proxy setup?

- To prevent the process from writing core dump files on crash
- **To block same-UID ptrace so a prompt-injected debugger cannot read the session token from heap memory**
- To prevent child processes from inheriting the proxy credentials
- To disable swap so the token cannot be paged to disk

**Question 2**

What is the first byte of every outbound WebSocket message sent by the relay, and why?

- `0x00` — marks the start of a binary frame
- **`0x0a` — the protobuf tag for field 1 (bytes), wire type 2 (length-delimited)**
- `0x01` — a version identifier required by the CCR gateway
- `0xff` — a magic sentinel that distinguishes data from keepalive frames

**Question 3**

The token file is deleted _after_ the relay is confirmed listening. What does this enable?

- It ensures the token is only read once, preventing replay attacks
- **It allows a supervisor to restart the container and retry the entire init sequence if something fails before the relay is up**
- It gives the relay time to copy the token into a secure enclave before it is wiped
- It prevents the token from being inherited by child processes via `/proc/fd`

**Question 4**

Why does the Bun relay need a `writeBuf` drain queue but the Node relay does not?

- **Node buffers writes unconditionally; Bun's `sock.write()` does a partial kernel write and silently drops the remainder**
- Bun processes data faster and generates more backpressure than Node
- Node's `net.Socket` does not support binary data, so no buffering is needed
- Bun uses UDP under the hood, which requires explicit retransmission

**Question 5**

Once the `st.established` flag is set, what does the relay do if the WebSocket errors out?

- Writes `HTTP/1.1 502 Bad Gateway` to the client socket before closing
- Attempts to reconnect the WebSocket with the same session token
- **Closes the TCP connection without writing any response, to avoid corrupting the active TLS stream**
- Buffers all further client data until the WebSocket recovers

# MCP Integration System — Lesson 07 Complete Content

## What MCP Is in Claude Code

MCP (Model Context Protocol) supplies external tools and data sources to AI models. Within Claude Code, every MCP server provides tools registered via the standard `Tool` interface, namespaced as `mcp__<server>__<tool>`.

The architecture spans four layers:
- **services/mcp/** — Connection lifecycle, config loading, OAuth, transport setup
- **tools/MCPTool/** — Proxy tool wrapping remote MCP calls with runtime overrides
- **commands/mcp/** — User-facing `/mcp` slash command interface
- **components/mcp/** — React UI panels for settings and dialogs

## Transport Types

Eight transport types exist, each with distinct setup requirements:

**stdio** (public) — Spawns subprocess via stdin/stdout. Default type. Stderr piped and capped at 64 MB.

**sse** (public) — HTTP Server-Sent Events using `ClaudeAuthProvider` for OAuth. GET requests skip 60s timeout; only POSTs enforce it.

**http** (public) — Streamable HTTP (MCP 2025-03-26 spec). Advertises `Accept: application/json, text/event-stream`. Supports OAuth and session-ingress JWT.

**ws** (public) — WebSocket with `protocols: ['mcp']`. Uses `ws` module on Node, native Bun WS. Supports proxy agent and mTLS.

**sse-ide** (internal) — SSE for IDE extensions (VS Code). No OAuth. Lockfile-based auth token planned but not wired.

**ws-ide** (internal) — WebSocket for IDE extensions. Accepts optional `authToken` via `X-Claude-Code-Ide-Authorization` header.

**sdk** (internal) — Control-message bridge to SDK process. Tool calls route via stdout/stdin.

**in-process** (in-proc) — Used by Chrome MCP and Computer Use. `createLinkedTransportPair()` creates two `InProcessTransport` instances with messages delivered via `queueMicrotask` to avoid stack overflows.

### InProcessTransport Code Example

```typescript
class InProcessTransport implements Transport {
  private peer: InProcessTransport | undefined

  async send(message: JSONRPCMessage): Promise<void> {
    queueMicrotask(() => { this.peer?.onmessage?.(message) })
  }
}

export function createLinkedTransportPair(): [Transport, Transport] {
  const a = new InProcessTransport()
  const b = new InProcessTransport()
  a._setPeer(b); b._setPeer(a)
  return [a, b]
}
```

## Config Scope Cascade

MCP server configs merge from multiple sources with strict precedence:

| Priority | Scope | Source | Notes |
|----------|-------|--------|-------|
| 1 (highest) | `enterprise` | `managed-mcp.json` (MDM path) | Blocks all user-managed add/remove. Exclusive control. |
| 2 | `dynamic` | CLI flag `--mcp-config <path>` | Policy-filtered before use. |
| 3 | `claudeai` | Claude.ai connector API (remote) | Deduplicated by URL signature. |
| 4 | `project` | `.mcp.json` (CWD & parents) | Nearest-to-cwd wins; parents searched, child overrides. |
| 5 | `local` | `~/.claude/projects/<hash>/` | Per-project local state, not git-tracked. |
| 6 | `user` | `~/.claude/settings.json` | User-wide defaults. |
| 7 | `managed` | Plugin-provided servers | Namespaced `plugin:name:server`. Content-deduplicated. |

**Enterprise Lock Warning:** When `managed-mcp.json` exists, `addMcpConfig()` throws: "enterprise MCP configuration is active and has exclusive control." Tools like `claude mcp add` are completely blocked.

### Scope Cascade Assembly (Simplified)

```typescript
const allConfigs = {
  ...enterpriseServers,      // wins if present
  ...dynamicServers,         // --mcp-config flag
  ...claudeAiServers,        // deduplicated by URL sig
  ...projectServers,         // .mcp.json, root-down
  ...localServers,           // ~/.claude/projects/...
  ...userServers,            // ~/.claude/settings.json
  ...pluginServers,          // namespaced, sig-deduplicated
}

// Env vars expanded in all configs before connection
// e.g. command: "npx", args: ["$MY_SERVER_PATH"]
```

### Policy Allow/Deny Lists

Enterprise policy can define `allowedMcpServers` and `deniedMcpServers`. The denylist takes absolute precedence. Matching by:
- **Name** — exact server name string
- **Command** — full command+args array for stdio servers
- **URL pattern** — glob with `*` wildcard for remote servers

## Connection Lifecycle

Every server transitions through a state machine (`MCPServerConnection` union):

**1. Config Assembly** — All scopes merged and policy-filtered. Env vars expanded (`$VAR` / `${VAR}`). Missing vars logged as warnings but don't block.

**2. Batched Connection** — Stdio servers connect in batches of 3 (`MCP_SERVER_CONNECTION_BATCH_SIZE`). Remote servers batch at 20. Each `connectToServer()` call memoized by `name + JSON(config)`.

**3. Transport Construction** — Based on `serverRef.type`, correct SDK transport instantiated. Auth providers, proxy agents, mTLS options attached.

**4. client.connect() with Timeout** — Default 30s (`MCP_TIMEOUT` env var). Races `connectPromise` vs `timeoutPromise`. Timeout closes in-process server if spawned.

**5. Auth Handling** — If `UnauthorizedError` (401): server moves to `needs-auth` state. `McpAuthTool` pseudo-tool injected for model to trigger OAuth. 15-min needs-auth cache expiration.

**6. Capability Negotiation** — Claude Code declares `roots: {}` and `elicitation: {}` capabilities. Server capabilities read via `getServerCapabilities()`. Server instructions truncated to 2048 chars.

**7. Tool/Resource/Prompt Fetch** — Tools, resources, prompts fetched in parallel. Tool names normalized (`mcp__server__tool`). Each tool cloned `MCPTool` with overridden `name`, `description`, `inputSchema`, `call()`.

**8. Live Notifications** — Subscriptions to `ToolListChanged`, `ResourceListChanged`, `PromptListChanged` notifications. Changes trigger re-fetch and AppState update. Exponential backoff reconnect (1s → 30s cap, max 5 attempts).

### Connection with Timeout Race Code

```typescript
const connectPromise = client.connect(transport)
const timeoutPromise = new Promise<never>((_, reject) => {
  const id = setTimeout(() => {
    transport.close().catch(() => {})
    reject(new Error(`MCP server "${name}" timed out after ${timeout}ms`))
  }, getConnectionTimeoutMs())
  connectPromise.then(() => clearTimeout(id), () => clearTimeout(id))
})

await Promise.race([connectPromise, timeoutPromise])
```

## Tool Proxying

`MCPTool` in `tools/MCPTool/MCPTool.ts` is a template. For each tool from a connected server, `fetchToolsForClient()` clones it with real metadata overridden:

MCP Server (tool.name, description, inputSchema) → Normalize (`mcp__server__tool_name`) → Clone MCPTool (override name, desc, schema, call()) → AppState (available to Claude) → Tool call (client.callTool() → JSON-RPC → transport)

### Name Normalization

```typescript
export function normalizeNameForMCP(name: string): string {
  let normalized = name.replace(/[^a-zA-Z0-9_-]/g, '_')
  if (name.startsWith('claude.ai ')) {
    normalized = normalized.replace(/_+/g, '_').replace(/^_|_$/g, '')
  }
  return normalized
}

export function buildMcpToolName(server: string, tool: string): string {
  return `mcp__${normalizeNameForMCP(server)}__${normalizeNameForMCP(tool)}`
}
```

**Tool Description Cap:** OpenAPI-generated servers dump 15-60 KB of docs into `tool.description`. Claude Code hard-caps both tool descriptions and server instructions at **2048 characters** to manage context window.

### Result Handling — Images, Binary Blobs, Truncation

```typescript
// After client.callTool() returns CallToolResult...

// 1. Image content items → resized/downsampled, returned as base64
if (content.type === 'image' && IMAGE_MIME_TYPES.has(mimeType)) {
  const buf = maybeResizeAndDownsampleImageBuffer(rawBuf)
  // → ContentBlockParam with base64 data
}

// 2. Binary blobs → persisted to disk, path returned as text
if (!IMAGE_MIME_TYPES.has(mimeType)) {
  await persistBinaryContent(content)
  // → getBinaryBlobSavedMessage(path)
}

// 3. Total result > 100 KB → truncation with instructions
if (mcpContentNeedsTruncation(result)) {
  result = truncateMcpContentIfNeeded(result)
}
```

## OAuth Authentication

MCP servers using SSE or HTTP transport can require OAuth. The `services/mcp/auth.ts` system implements full PKCE flow with XAA (Cross-App Access) extension support:

**Sequence:**
1. Claude Code calls `client.connect(transport)` → receives 401 UnauthorizedError
2. Server state set to `needs-auth`; `McpAuthTool` pseudo-tool injected
3. Model calls `mcp__server__authenticate`
4. `discoverOAuthServerMetadata()` fetches auth server metadata (PKCE endpoint)
5. `/authorize?code_challenge=...&redirect_uri=...` opened in browser
6. Auth server returns callback with `authorization_code`
7. Claude Code POSTs `/token` with code + verifier
8. Auth server returns `access_token` + `refresh_token`
9. Tokens stored in keychain (macOS) or secure storage
10. `reconnectMcpServerImpl()` reconnects; real tools swap in

### McpAuthTool: Model-Triggered OAuth

When server enters `needs-auth` state, pseudo-tool `mcp__<server>__authenticate` injected. Model calls it to start OAuth flow and receive authorization URL for user. After callback fires, real tools automatically replace pseudo-tool via prefix-based replacement on AppState.

### Token Refresh & Slack Quirk

RFC 6749 `invalid_grant` error triggers token invalidation. Slack returns HTTP 200 with `{"error":"invalid_refresh_token"}` — SDK sees ZodError. Claude Code normalizes these non-standard codes to `invalid_grant` before passing to SDK's error-class mapper.

### Slack 200-Error Normalization Code

```typescript
const NONSTANDARD_INVALID_GRANT_ALIASES = new Set([
  'invalid_refresh_token',
  'expired_refresh_token',
  'token_expired',
])

// Wraps fetch: peeks at 2xx POST responses, rewrites error bodies
// matching OAuthErrorResponseSchema (but NOT OAuthTokensSchema)
// to a synthetic 400 response — so SDK error-class mapping applies.
```

### XAA — Cross-App Access

Enterprise extension for SSO flows. When `xaa: true` set on MCP server config, system exchanges IdP ID-token for MCP server's OAuth token silently instead of launching browser. Configured once in `settings.xaaIdp`, shared across all XAA-enabled servers.

## Elicitation

Elicitation is MCP mechanism for servers to request structured input from user mid-operation. Claude Code supports two modes:

**form mode** — Server sends JSON Schema; user fills form. Response is `accept` with content, or `decline` / `cancel`.

**url mode** — Server sends URL (OAuth step-up, external confirmation). Two-phase: open URL → wait for `ElicitationComplete` notification. User can dismiss or retry.

Requests land in `AppState.elicitation.queue` as `ElicitationRequestEvent` objects with `respond()` callback. `ElicitationDialog` React component polls queue. Hooks (`executeElicitationHooks`) can satisfy requests programmatically without UI.

### Elicitation Handler Registration Code

```typescript
client.setRequestHandler(ElicitRequestSchema, async (request, extra) => {
  // 1. Try hooks first (programmatic response)
  const hookResponse = await runElicitationHooks(serverName, request.params, extra.signal)
  if (hookResponse) return hookResponse

  // 2. Queue for user interaction
  const response = new Promise<ElicitResult>(resolve => {
    setAppState(prev => ({
      ...prev,
      elicitation: {
        queue: [...prev.elicitation.queue, {
          serverName, requestId: extra.requestId,
          params: request.params,
          respond: resolve,
        }],
      },
    }))
  })
  return await response
})
```

## Server Deduplication

Multiple config sources supplying same server deduplicated by content signature, not name. Stdio server signature: `stdio:["cmd","arg1"]`. Remote server signature: `url:https://vendor.example.com/mcp`.

### CCR Proxy Unwrapping

In remote sessions, claude.ai connectors arrive with URLs rewritten through CCR/session-ingress proxy. Original vendor URL preserved in `mcp_url` query param. `unwrapCcrProxyUrl()` extracts it before signature comparison, so plugin pointing at Slack's direct server deduplicates against claude.ai connector routed through CCR.

### Deduplication Rules

- **Manual wins over plugin** — user-configured beats plugin-provided
- **First plugin wins** — if two plugins provide same server, first-loaded wins
- **Enabled manual wins over claude.ai** — disabled manual server doesn't suppress connector twin
- Plugin servers namespaced `plugin:name:server` to avoid key collisions before dedup

## Key Takeaways

- **7 transport types** — stdio, sse, http, ws, sse-ide, ws-ide, sdk plus internal in-process pair for Chrome/Computer Use. Public transports support OAuth; IDE variants auth-free or token-based.

- **7-layer scope cascade** — enterprise > dynamic > claudeai > project > local > user > managed. Enterprise presence locks out manual add/remove.

- **Config files walk directory tree** — `.mcp.json` read from every parent directory to root; child overrides parent.

- **All tool names pass normalization** — `mcp__<server>__<tool>`, non-alphanumeric replaced by underscores. Claude.ai server names get collapse/strip to protect `__` delimiter.

- **OAuth model-triggerable** — `McpAuthTool` pseudo-tool lets Claude start auth autonomously, return URL to user; real tools auto-replace after callback.

- **Tool descriptions hard-capped 2048 chars** — prevents context explosion from OpenAPI servers.

- **Deduplication content-based, not name-based** — same command array or URL = same server regardless of names in different sources.

## Knowledge Check

**1. Config Scope Priority** — You add same MCP server in both `.mcp.json` (project scope) and `~/.claude/settings.json` (user scope). Which wins?

**Answer:** Project-level `.mcp.json` wins because "project scope has higher priority than user scope" in the cascade hierarchy.

**2. Stderr Handling** — A stdio MCP server's stderr prints noise. How prevented?

**Answer:** "StdioClientTransport is created with stderr: 'pipe'; output accumulated in memory and logged via logMCPError, not shown in UI."

**3. Timeout Purpose** — What is purpose of `wrapFetchWithTimeout()` and why skip GET?

**Answer:** Applies 60s timeout to POST requests. "GETs are skipped because SSE GET connections are long-lived streams that must stay open indefinitely."

**4. Slack Error Handling** — Slack returns HTTP 200 with `{"error":"invalid_refresh_token"}`. What happens?

**Answer:** "Non-standard error code normalized to invalid_grant and response rewritten to synthetic 400, triggering SDK's standard InvalidGrantError path and token invalidation."

**5. Enterprise Config Block** — User runs `claude mcp add my-tool` with `managed-mcp.json` present. Result?

**Answer:** "`addMcpConfig()` throws: 'enterprise MCP configuration is active and has exclusive control over MCP servers.'"

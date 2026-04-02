# Entrypoints & Agent SDK — Source Deep Dive

## Lesson 49

### How Claude Code is invoked — CLI, MCP server, headless, bridge, daemon — and how external developers build on top of it via the Agent SDK.

## 01 The Entrypoint Layer

The `src/entrypoints/` directory serves as the boundary between the external world and Claude Code's internals. It contains five surface-level files — `cli.tsx`, `init.ts`, `mcp.ts`, `agentSdkTypes.ts`, and `sandboxTypes.ts` — plus an `sdk/` sub-directory holding serializable contract types. The `main.tsx` file sits one level up and represents the single large Commander-based CLI handler that nearly every interactive and headless path flows through.

**Architecture note**

`cli.tsx` is a thin bootstrap that pattern-matches on `process.argv` and fast-paths special commands _before_ loading the 200+ module import graph in `main.tsx`. This keeps `--version` and daemon-worker startup near-instant.

### Invocation Flow Diagram

The system dispatches through several specialized pathways:

- **claude binary** (Bun single-file exe) routes to `cli.tsx`
- **Fast paths** handle `--version / -v`, `--daemon-worker`, `remote-control / rc / bridge`, daemon subcommands, background session management (`ps`, `logs`, `attach`, `kill`, `--bg`), environment-runner, self-hosted-runner, and MCP modes (`--claude-in-chrome-mcp`, `--computer-use-mcp`)
- **Everything else** falls through to `main.tsx` (Commander CLI)
- **main.tsx** routes to headless execution (`runHeadless()`), interactive REPL (`launchRepl()`), MCP mode (`entrypoints/mcp.ts`), or Agent SDK process transport

## 02 cli.tsx — The Bootstrap Dispatcher

`src/entrypoints/cli.tsx` is the actual binary entrypoint, running before any other module evaluation. Its design philosophy: **load as little as possible for each fast-path**.

### Fast Paths (in order of detection)

**Fast path 1: `--version / -v`**

Zero imports. Prints `MACRO.VERSION` inlined at build time and exits immediately.

**Fast path 2: `--dump-system-prompt`**

Internal eval tool. Loads only config, model, and prompts modules to render and print the system prompt.

**Fast path 3: Chrome / Computer-Use MCP**

`--claude-in-chrome-mcp` and `--computer-use-mcp` launch standalone MCP servers without the full CLI stack.

**Fast path 4: `--daemon-worker=<kind>`**

Spawned by the daemon supervisor. Loads only the worker registry — no configs, no auth, no telemetry at this layer.

**Fast path 5: Bridge / Remote Control**

`remote-control`, `rc`, `sync`, `bridge` — connects the local machine as a remote-controlled environment for claude.ai.

**Fast path 6: `daemon` subcommand**

Long-running supervisor process. Sets up sinks then delegates to `daemon/main.js`.

**Fast path 7: Background sessions**

`ps`, `logs`, `attach`, `kill`, `--bg` — session registry management without loading the interactive UI.

**Fallthrough: Full CLI (`main.tsx`)**

Everything else. Loads the complete Commander-based CLI handler with all 200+ module imports.

### Code Example

```typescript
// From cli.tsx — each fast-path is gated by a build-time feature() flag
if (feature('BRIDGE_MODE') && (args[0] === 'remote-control' || args[0] === 'rc'
    || args[0] === 'remote' || args[0] === 'sync' || args[0] === 'bridge')) {
  // Auth check → GrowthBook gate → policy limits → bridgeMain()
  const { bridgeMain } = await import('../bridge/bridgeMain.js');
  await bridgeMain(args.slice(1));
  return;
}
```

**Design pattern**

Every fast-path also checks a `feature()` flag — a Bun build-time dead-code-elimination gate. This means unsupported features are completely absent from external distribution builds, not just gated at runtime.

## 03 init.ts — Shared Initialization

`src/entrypoints/init.ts` is a memoized `init()` function shared by all non-trivial entrypoints. It is **not called for fast-paths**. It performs all the one-time setup that must happen before the first API call.

### What init() does (in order)

1. **enableConfigs()** — validates and activates the settings system
2. **applySafeConfigEnvironmentVariables()** — applies safe env vars before the trust dialog
3. **applyExtraCACertsFromConfig()** — sets TLS CA certs before the first TLS connection
4. **setupGracefulShutdown()** — registers SIGTERM/SIGINT handlers for flush-on-exit
5. **initialize1PEventLogging()** — lazily loads OpenTelemetry analytics (deferred ~400KB)
6. **populateOAuthAccountInfoIfNeeded()** — fills missing OAuth cache from keychain
7. **initJetBrainsDetection()** — detects IDE host asynchronously
8. **initializeRemoteManagedSettingsLoadingPromise()** — sets up enterprise policy loading
9. **configureGlobalMTLS() / configureGlobalAgents()** — TLS + proxy agents
10. **preconnectAnthropicApi()** — warms TCP+TLS (~150ms) in parallel with CLI parsing
11. **initUpstreamProxy()** — CCR upstream proxy for org-injected credentials (CLAUDE_CODE_REMOTE)
12. **registerCleanup(shutdownLspServerManager)** — LSP teardown on exit
13. **ensureScratchpadDir()** — creates scratch dir if enabled

A separate `initializeTelemetryAfterTrust()` function is called only after the user has accepted the trust dialog. This separates consent-independent setup from consent-gated telemetry, and waits for remote managed settings to load before initializing OpenTelemetry exporters.

## 04 mcp.ts — Claude Code as an MCP Server

When invoked with `claude --mcp`, Claude Code runs as a standard **Model Context Protocol server** over stdio. This exposes all of Claude Code's tools (Bash, Edit, Read, WebFetch, etc.) to _other_ MCP clients — editors, agents, or automation scripts can call them directly without spawning a full REPL.

### Code Example

```typescript
// src/entrypoints/mcp.ts
const server = new Server(
  { name: 'claude/tengu', version: MACRO.VERSION },
  { capabilities: { tools: {} } }
)

// ListTools: enumerate every Claude Code tool with its Zod-derived JSON schema
server.setRequestHandler(ListToolsRequestSchema, async () => ({
  tools: await Promise.all(tools.map(async tool => ({
    ...tool,
    description: await tool.prompt(...),
    inputSchema: zodToJsonSchema(tool.inputSchema),
  })))
}))

// CallTool: validate input, check permissions, execute, return result
server.setRequestHandler(CallToolRequestSchema, async ({ params }) => {
  const tool = findToolByName(tools, params.name)
  return await tool.call(params.arguments, toolUseContext, ...)
})
```

**Key detail**

The MCP server entrypoint forces `isNonInteractiveSession: true` and disables thinking (`thinkingConfig: { type: 'disabled' }`). It also exposes only the `review` slash command since slash commands are not meaningful in this context.

## 05 agentSdkTypes.ts — The Public SDK Contract

`src/entrypoints/agentSdkTypes.ts` is the main export of the Agent SDK package. It re-exports the full public API from three sub-modules and declares the top-level functions that SDK consumers call. All function bodies throw `'not implemented'` in this file — the actual implementations are injected at runtime by the SDK transport layer.

### Module Structure

| Module | Purpose | Examples |
|--------|---------|----------|
| `sdk/coreTypes.ts` | Serializable, transport-safe types generated from Zod schemas | `SDKMessage`, `SDKUserMessage`, `ModelUsage`, `PermissionResult`, `HookInput` |
| `sdk/runtimeTypes.ts` | Non-serializable types with callbacks and method interfaces | `SDKSession`, `Options`, `Query`, `SdkMcpToolDefinition` |
| `sdk/controlTypes.ts` | Control protocol for SDK builders (bridge subpath consumers) | `SDKControlRequest`, `SDKControlResponse` |
| `sdk/settingsTypes.generated.ts` | Full Settings type generated from settings JSON schema | `Settings` |

### Top-Level SDK Functions

```typescript
// V1 API (stable) — headless one-shot query
export function query(params: {
  prompt: string | AsyncIterable<SDKUserMessage>
  options?: Options
}): Query

// V2 API (@alpha) — persistent multi-turn sessions
export function unstable_v2_createSession(options: SDKSessionOptions): SDKSession
export function unstable_v2_resumeSession(sessionId: string, options: SDKSessionOptions): SDKSession
export async function unstable_v2_prompt(message: string, options: SDKSessionOptions): Promise<SDKResultMessage>

// Session management
export async function listSessions(options?: ListSessionsOptions): Promise<SDKSessionInfo[]>
export async function getSessionInfo(sessionId: string): Promise<SDKSessionInfo | undefined>
export async function getSessionMessages(sessionId: string): Promise<SessionMessage[]>
export async function renameSession(sessionId: string, title: string): Promise<void>
export async function tagSession(sessionId: string, tag: string | null): Promise<void>
export async function forkSession(sessionId: string, options?: ForkSessionOptions): Promise<ForkSessionResult>

// In-process MCP server
export function createSdkMcpServer(options: { name: string; tools: SdkMcpToolDefinition[] }): McpSdkServerConfigWithInstance
export function tool<S>(name, description, schema, handler): SdkMcpToolDefinition<S>
```

## 06 The Control Protocol (SDK Builders)

External SDK implementations (like the Python `claude-code-sdk`) communicate with the Claude Code process via a JSON-based **control protocol** layered over stdio. The schemas are defined in `sdk/controlSchemas.ts`.

### Control Request Subtypes

| Subtype | Direction | Purpose |
|---------|-----------|---------|
| `initialize` | SDK → CLI | Start session — pass hooks, MCP servers, agents, system prompt overrides |
| `interrupt` | SDK → CLI | Cancel the currently running turn |
| `can_use_tool` | CLI → SDK | Request permission for a tool use; SDK host responds allow/deny |
| `set_permission_mode` | SDK → CLI | Change permission mode mid-session (default / acceptEdits / bypassPermissions / plan / dontAsk) |
| `set_model` | SDK → CLI | Switch model for subsequent turns |
| `set_max_thinking_tokens` | SDK → CLI | Adjust extended thinking budget |
| `mcp_status` | SDK → CLI | Query current MCP server connection states |
| `get_context_usage` | SDK → CLI | Inspect context window utilization by category |

### Code Example

```typescript
// Initialize request — sent by SDK to start a session
{
  subtype: "initialize",
  hooks: {
    "PreToolUse": [{ hookCallbackIds: ["my-hook"], matcher: "Bash" }]
  },
  sdkMcpServers: ["my-server"],
  systemPrompt: "You are a coding assistant.",
  agents: {
    "reviewer": { description: "Reviews code changes", ... }
  }
}

// Initialize response — returned by CLI
{
  commands: [...],    // available slash commands
  agents: [...],      // available agent types
  models: [...],      // available models
  account: {...},     // account info
  pid: 12345          // @internal CLI PID for tmux socket isolation
}
```

## 07 Hook Events — The SDK Observer Pattern

The SDK exposes a rich hook system that lets external hosts observe and intercept Claude Code's lifecycle. There are 26 named hook events, defined in `sdk/coreTypes.ts`:

```typescript
export const HOOK_EVENTS = [
  // Tool execution
  'PreToolUse', 'PostToolUse', 'PostToolUseFailure',
  // Permission flow
  'PermissionRequest', 'PermissionDenied',
  // Session lifecycle
  'SessionStart', 'SessionEnd', 'Setup',
  // Turn lifecycle
  'Stop', 'StopFailure',
  // Context management
  'PreCompact', 'PostCompact',
  // Agent/swarm lifecycle
  'SubagentStart', 'SubagentStop', 'TeammateIdle',
  'TaskCreated', 'TaskCompleted',
  // Notifications and user input
  'Notification', 'UserPromptSubmit',
  // Config changes
  'ConfigChange', 'InstructionsLoaded', 'CwdChanged', 'FileChanged',
  // Elicitation
  'Elicitation', 'ElicitationResult',
  // Worktree
  'WorktreeCreate', 'WorktreeRemove',
] as const
```

Every hook fires with a `BaseHookInput` that includes `session_id`, `transcript_path`, `cwd`, `agent_id` (if inside a subagent), and `agent_type`. Each event adds its own specific fields on top.

**Subagent detection**

Use `agent_id` (present only inside a subagent) — not `agent_type` — to distinguish subagent hook firings from main-thread firings. The main thread can have an `agent_type` when started with `--agent` but will never have an `agent_id`.

## 08 Daemon & Bridge Mode (@internal)

For advanced host architectures (desktop apps, CI systems, claude.ai integrations), `agentSdkTypes.ts` also exports `@internal` primitives:

### Scheduled Tasks / Cron

```typescript
// Watch .claude/scheduled_tasks.json and yield fire/missed events
export function watchScheduledTasks(opts: {
  dir: string
  signal: AbortSignal
  getJitterConfig?: () => CronJitterConfig
}): ScheduledTasksHandle

// ScheduledTasksHandle — drain with for await
{
  events(): AsyncGenerator<ScheduledTaskEvent>  // fire | missed
  getNextFireTime(): number | null               // epoch ms of next scheduled run
}
```

This lets daemon processes own the scheduler in the parent process. When a task fires, the daemon spawns a `query()` subprocess — if it crashes, the daemon can respawn it while the schedule continues.

### Remote Control / Bridge

```typescript
// Connect the local machine as a claude.ai remote-control environment
export async function connectRemoteControl(opts: ConnectRemoteControlOptions):
  Promise<RemoteControlHandle | null>

// RemoteControlHandle — two-way bridge over WebSocket
{
  sessionUrl: string
  environmentId: string
  bridgeSessionId: string
  write(msg: SDKMessage): void         // pipe query() yields in
  sendResult(): void                     // signal turn complete
  inboundPrompts(): AsyncGenerator<...>  // read user messages from claude.ai
  controlRequests(): AsyncGenerator<...> // interrupt, set_model, etc.
  permissionResponses(): AsyncGenerator<...>
  onStateChange(cb): void               // ready | connected | reconnecting | failed
  teardown(): Promise<void>
}
```

**Architecture difference**

**Daemon mode**: the WebSocket lives in the parent process. If the agent subprocess crashes, the daemon respawns it while claude.ai keeps the same session. **`query.enableRemoteControl`**: the WebSocket lives in the child process and dies with it. Use daemon mode for production-grade reliability.

## 09 sandboxTypes.ts — Process Isolation Config

`src/entrypoints/sandboxTypes.ts` is the single source of truth for sandbox configuration types. Both the SDK and the settings validation system import from here. It is exported through `agentSdkTypes.ts` so SDK consumers can configure sandboxing.

```typescript
// SandboxSettings — full configuration for process-level isolation
{
  enabled: boolean
  failIfUnavailable: boolean    // hard-gate for managed deployments
  autoAllowBashIfSandboxed: boolean
  allowUnsandboxedCommands: boolean
  network: {
    allowedDomains: string[]
    allowManagedDomainsOnly: boolean  // enterprise: only managed domains
    allowUnixSockets: string[]        // macOS only
    allowLocalBinding: boolean
    httpProxyPort: number
    socksProxyPort: number
  }
  filesystem: {
    allowWrite: string[]      // merged with Edit() allow rules
    denyWrite: string[]       // merged with Edit() deny rules
    denyRead: string[]
    allowRead: string[]       // re-allow within denyRead regions
    allowManagedReadPathsOnly: boolean
  }
}
```

## 10 main.tsx — Invocation Modes

Once `cli.tsx` falls through to `main.tsx`, the Commander-based CLI handles the remaining invocation modes. Key ones:

| Flag / Mode | Behavior |
|-------------|----------|
| `-p / --print <prompt>` | Headless mode — runs a single prompt non-interactively, streams results to stdout, exits. Used by scripts and the Agent SDK. |
| `--mcp` | Starts Claude Code as an MCP server via stdio. Delegates to `entrypoints/mcp.ts`. |
| (no flags) | Interactive REPL — renders the full Ink TUI. Calls `launchRepl()`. |
| `--resume [sessionId]` | Resume a previous session by UUID, or show the resume chooser TUI. |
| `--continue` | Continue the most recent session without showing the chooser. |
| `--dangerously-skip-permissions` | Bypass all permission checks (requires explicit opt-in in settings). Used by CI environments. |
| `--allowedTools` | Comma-separated tool allow-list for the session. |
| `--sdk-transport=process` | SDK process transport mode — connects the Agent SDK via stdin/stdout control protocol. |

## 11 Complete Invocation Flow

The system follows this sequence:

1. External Developer calls `query("fix this bug", { cwd: "/project" })` via Agent SDK (Python/TS)
2. SDK spawns Claude Code process with `--sdk-transport=process`
3. CLI executes `cli.tsx` → `main.tsx` initialization
4. CLI sends `initialize` control response to SDK (commands, models, account, PID)
5. SDK sends user message via stdin
6. CLI makes streaming API request to Anthropic API
7. Anthropic API returns assistant tokens
8. CLI streams `SDKAssistantMessage` to SDK
9. CLI requests permission via `can_use_tool?` control request (e.g., Bash)
10. SDK responds with allow/deny
11. CLI executes tool and streams `SDKToolResultMessage` to SDK
12. CLI sends final `SDKResultMessage`
13. SDK yields completed messages to developer

## Key Takeaways

- `cli.tsx` is a pure dispatcher — it fast-paths 8+ special commands before loading the full CLI, keeping startup snappy for version checks, daemon workers, and bridge mode.
- Build-time `feature()` flags perform dead-code elimination — bridge mode, daemon, background sessions, and other features are completely absent from external builds unless enabled.
- `init.ts` is memoized and shared by all entrypoints — it coordinates TLS certs, proxy agents, OAuth, telemetry, and cleanup handlers before the first API call.
- Claude Code can _be_ an MCP server (`--mcp` mode via `mcp.ts`) while also _consuming_ MCP servers — the boundary is symmetric.
- The Agent SDK public API (`agentSdkTypes.ts`) is a stub file — all function bodies throw; the real implementations are injected by the SDK transport at runtime.
- 26 lifecycle hook events let SDK hosts observe and intercept tool execution, permission decisions, session lifecycle, compaction, and worktree operations.
- The control protocol separates SDK consumers (who call `query()`) from SDK builders (who implement the process transport and speak the control protocol directly).
- `sandboxTypes.ts` is the single source of truth for process isolation config — used by both the SDK export and the settings validation system.

## Check Your Understanding

**Question 1**

Which file handles the very first dispatch of `claude --version` and why does it load zero modules?

- `main.tsx` — it checks for `--version` first in Commander's option parsing
- `init.ts` — the init function skips module loading for version checks
- **`cli.tsx` — it checks argv before any dynamic imports, and `MACRO.VERSION` is inlined at build time**
- `agentSdkTypes.ts` — the SDK handles version reporting

**Answer**: `cli.tsx` reads `process.argv.slice(2)` synchronously before any imports. `MACRO.VERSION` is a build-time inline, so no dynamic import is ever issued.

**Question 2**

In the Agent SDK, all top-level functions in `agentSdkTypes.ts` throw `'not implemented'`. What mechanism actually provides the real implementations?

- The functions are monkeypatched by the npm package's index.js at require time
- They are abstract methods overridden by SDK subclasses
- **The SDK transport layer injects the implementations at runtime when the process starts**
- TypeScript declaration merging provides the implementations

**Answer**: The stub bodies are placeholders. When Claude Code runs with `--sdk-transport=process`, the transport layer provides real implementations via the control protocol over stdin/stdout.

**Question 3**

What is the key architectural difference between daemon-mode remote control and `query.enableRemoteControl`?

- Daemon mode uses HTTP; `enableRemoteControl` uses WebSockets
- **In daemon mode, the WebSocket lives in the parent process; in `enableRemoteControl`, it lives in the child process and dies with it**
- Daemon mode requires OAuth; `enableRemoteControl` uses API keys
- They are functionally identical — the difference is only in configuration syntax

**Answer**: With daemon mode, the parent holds the WebSocket so it survives agent subprocess crashes, enabling respawning while claude.ai keeps the same session.

**Question 4**

How does `feature()` in cli.tsx differ from a runtime GrowthBook gate like `isBridgeEnabled()`?

- `feature()` checks the same GrowthBook value but is cached for performance
- **`feature()` is a build-time dead-code-elimination gate — the entire branch is absent from external builds; GrowthBook gates are runtime checks on a live flag service**
- `feature()` reads from an environment variable; GrowthBook reads from a remote config service
- They are the same — `feature()` is just syntactic sugar for GrowthBook

**Answer**: The source comments note that `feature()` is a Bun build-time DCE gate. Code inside disabled feature blocks is stripped from the binary, not just branched-over at runtime.

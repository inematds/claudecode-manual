# Claude Code Lesson 50: Full Architecture Overview

## Overview
This capstone lesson synthesizes 49 previous lessons into a complete mental model of how Claude Code operates from initial keystroke through rendered output. The course covers a TypeScript application built on Bun, React/Ink, and the Anthropic API.

## Six-Layer Architecture

Claude Code organizes into distinct layers:

1. **Boot**: `main.tsx`, `setup.ts`, `entrypoints/init.ts` – startup, settings, migrations, session wiring
2. **UI Shell**: `replLauncher.tsx`, `screens/REPL.tsx` – Ink terminal interface
3. **State**: `state/AppStateStore.ts`, `bootstrap/state.ts` – immutable AppState and global singleton
4. **Query Engine**: `QueryEngine.ts`, `query.ts` – conversation lifecycle and API streaming
5. **Tools**: `tools.ts`, `Tool.ts` – capability registry including Bash, file I/O, agents, MCP
6. **Services**: API client, MCP connections, context compaction

## Data Flow: User Prompt to Response

The sequence from user input to API response follows this pattern:

- User enters prompt in REPL
- `processUserInput()` handles slash commands
- System prompt assembled from multiple sources
- Transcript written to disk **before** API call
- Query loop streams text and tool calls
- Tools execute with permission checks
- Response flows back through UI components
- Session stored as JSONL

**Critical design decision**: "The transcript is written **before** the API call, not after. This is intentional: if the process is killed mid-request, the session is still resumable."

## QueryEngine Internals

Each conversation owns one `QueryEngine` instance managing:
- Full message history across turns
- Token usage tracking
- File state cache
- Shared abort controller for tools
- Permission denials accumulation

The `submitMessage()` method yields `SDKMessage` events through an async generator, enabling streaming without callbacks.

## Boot Sequence Optimization

Three parallel work categories minimize time-to-first-render:

**Parallel at import time:**
- `startMdmRawRead()` – MDM policy subprocesses
- `startKeychainPrefetch()` – macOS keychain reads

**After init():**
- `preconnectAnthropicApi()` – TCP connection warm-up

**After first render:**
- `startDeferredPrefetches()` – user context, tips, model capabilities

## Tool System Architecture

All capabilities are flat-registered `Tool` objects with:
- `name` – stable identifier
- `description` – system prompt injection
- `inputSchema` – Zod validation
- `isEnabled()` – feature-gating
- `call()` – async generator for execution
- `renderToolResult()` – Ink UI rendering

Tools receive `ToolUseContext` bundling messages, model, MCP clients, agent definitions, abort controller, and UI state setter.

## Permission Architecture

The `canUseTool()` function serves as the sole architectural choke point for permission decisions, supporting three modes:

- **default**: Ask user for tools outside allow-list
- **auto**: Automatically allow safe tools
- **bypass**: Allow all without asking

## Two-Layer State Model

**Layer 1 – Global Singleton** (`bootstrap/state.ts`):
Process-lifetime constants like sessionId, cwd, projectRoot, totalCostUSD, token budget counters. Explicitly not React state.

**Layer 2 – React State** (`state/AppStateStore.ts`):
`DeepImmutable<AppState>` containing messages, MCP clients, permission context, tasks, agents, file history. Updated immutably via `setAppState()`.

## Context Compaction

Automatic compaction triggers at ~80% context window fill:
- Calls compaction prompt sending conversation to Claude
- Returns single summary plus preserved recent messages
- Handles 500k token continuation automatically
- Transparent to user

## Session Management

Sessions stored as JSONL under `~/.claude/projects/<cwd-hash>/<session-id>.jsonl`:
- Fire-and-forget transcript writes with 100ms lazy flush
- User messages awaited before API call
- Assistant messages fire-and-forget to preserve streaming
- Sessions resumable via `loadTranscriptFromFile()`

## MCP Integration

MCP servers connect as `MCPServerConnection` objects:
- Initialized before first query
- Passed into every `ToolUseContext`
- Tools, commands, resources dynamically added via `getMcpToolsCommandsAndResources()`
- Separate from base tool registry but integrated at tool list assembly

## Hook System

User-defined lifecycle hooks configured in `settings.json`:
- `PreToolUse`, `PostToolUse`, `PreCompact`, `PostCompact`, `Stop`, `Notification`, `FileChanged`, `SessionStart`
- Snapshotted once at startup via `captureHooksConfigSnapshot()`
- Security invariant prevents mid-session injection of malicious hooks

## Interactive vs Headless Modes

**Interactive** (default):
- Full Ink/React rendering
- Trust dialog on first launch
- Session transcript awaited before API
- Deferred prefetches after first render

**Headless** (`-p`/`--print`):
- Stdout text only
- Implicit trust
- Fire-and-forget transcript
- React never imported
- Controlled via `isBareMode()` flag

## Agent Swarms

The `AgentTool` enables recursive execution:
- Each sub-agent spawns own `QueryEngine` instance
- Restricted tool set from parent
- In swarm mode (`ENABLE_AGENT_SWARMS=true`), agents communicate via UDS messaging
- `TeamCreateTool` for agent registration
- `SendMessageTool` for inbox communication

## Key Design Patterns

1. **Async generator threading**: Data flows as generator yields from API through UI
2. **Dead code elimination**: `feature()` flags completely remove disabled features at bundle time
3. **Cache-warming for latency**: Critical paths pre-warmed in parallel during setup
4. **Immutable AppState + mutable bootstrap/state**: React state immutable; session constants in plain module singleton
5. **isBareMode() fast path**: Single flag skips React, Ink, UDS, plugins for headless execution
6. **Parallel subprocess investment**: Subprocesses fire early, run during JavaScript evaluation

## First Keystroke to First Token Timeline

Complete progression from cold start to streaming:

- t=0ms: CLI entry
- t=1ms: MDM and keychain reads fire (parallel)
- t=136ms: Module eval complete
- t=160ms: TCP warm-up to API
- t=175ms: Setup screens if first run
- t=190ms: First UI render
- t=191ms: Deferred prefetches start
- User types and hits Enter
- t+50ms: First token arrives

## Critical Invariants

- Hooks captured after `setCwd()` but before any query (security)
- Transcript written before API call (crash resilience)
- Tool list stable per build (prompt cache key)
- Feature gates eliminated at bundle time (deterministic tools)
- AppState immutable (React change detection)
- QueryEngine owns full conversation lifecycle

# mdENG Lesson 10 — The Hooks System

## What are hooks?

"Hooks let you inject side effects at precisely defined points in the Claude Code lifecycle — before a tool runs, after sampling completes, when a session starts, when a file changes."

Each hook consists of a matcher and command pair stored in settings files or in-memory for session hooks. When an event fires, Claude Code locates all matching hooks, executes them, then interprets exit codes and stdout to determine whether to continue, block, or return output to the model.

The hooks system serves as the primary extension surface, powering lint-on-save behaviors, policy enforcement, observability pipelines, and structured verification agents without modifying Claude Code itself.

**Entry point:** Hook events are defined in `src/entrypoints/sdk/coreTypes.ts` as the `HOOK_EVENTS` const array. Execution logic resides in `src/utils/hooks.ts` and individual `exec*Hook.ts` files. Configuration is read from `src/utils/hooks/hooksConfigSnapshot.ts`.

---

## All 27 Hook Events

Events are grouped by category:

### Lifecycle (session boundaries)
- SessionStart
- SessionEnd
- Setup
- Stop
- StopFailure

### Tool execution
- PreToolUse
- PostToolUse
- PostToolUseFailure

### Agent & subagent
- SubagentStart
- SubagentStop

### Compaction
- PreCompact
- PostCompact

### Permission & policy
- PermissionRequest
- PermissionDenied
- UserPromptSubmit
- ConfigChange
- InstructionsLoaded

### Collaborative / multi-agent
- TeammateIdle
- TaskCreated
- TaskCompleted
- Notification

### Filesystem & environment
- CwdChanged
- FileChanged
- WorktreeCreate
- WorktreeRemove

### MCP elicitation
- Elicitation
- ElicitationResult

---

## How a Hook Fires

For a **PreToolUse** event, the execution path follows this sequence:

1. Tool call is requested
2. Check for any hooks matching PreToolUse
3. If present, filter by matcher (e.g., tool_name = Write)
4. Run each matched hook command/prompt/agent/http/function
5. Evaluate results based on exit codes

**Exit code semantics determine behavior:**

### PreToolUse exit codes

| Exit code | Effect |
|-----------|--------|
| 0 | stdout/stderr not shown; tool proceeds |
| 2 | stderr shown to model; **tool call blocked** |
| other | stderr shown to user only; tool proceeds |

### PostToolUse exit codes

| Exit code | Effect |
|-----------|--------|
| 0 | stdout shown in transcript mode (Ctrl+O) |
| 2 | stderr shown to model immediately |
| other | stderr shown to user only |

### Stop exit codes

| Exit code | Effect |
|-----------|--------|
| 0 | stdout/stderr not shown; session concludes |
| 2 | stderr shown to model; **conversation continues** (prevents stop) |
| other | stderr shown to user only; session concludes |

**StopFailure** fires instead of Stop when the turn ended with an API error (rate limit, auth failure). It is fire-and-forget — exit codes and output are ignored.

### UserPromptSubmit exit codes

| Exit code | Effect |
|-----------|--------|
| 0 | stdout injected into model context (shown to Claude) |
| 2 | **blocks processing**, erases original prompt, shows stderr to user |
| other | stderr shown to user only |

### SessionStart & Setup exit codes

| Exit code | Effect |
|-----------|--------|
| 0 | stdout shown to Claude (seed context for the session) |
| 2 | blocking errors ignored for both events |
| other | stderr shown to user only |

SessionStart supports a `source` matcher with values: `startup`, `resume`, `clear`, `compact`. Setup supports a `trigger` matcher with values: `init`, `maintenance`.

### PreCompact / PostCompact exit codes

| Exit code | Effect (PreCompact) |
|-----------|--------|
| 0 | stdout appended as custom compact instructions |
| 2 | **block compaction** |
| other | stderr shown to user; compaction continues |

PostCompact: exit 0 shows stdout to user; other exit codes show stderr to user only.

### CwdChanged & FileChanged exit codes

Both events set `CLAUDE_ENV_FILE` — write bash `export` statements to that file path and they will be applied to subsequent BashTool commands. Neither event supports exit code 2 blocking; non-zero exits show stderr to user only.

FileChanged also supports `hookSpecificOutput.watchPaths` in stdout JSON to dynamically register additional paths with the file watcher. The matcher field is used as a pipe-separated filename glob (e.g. `.envrc|.env`).

---

## 5 Hook Command Types

The `type` discriminant field selects the execution engine. All types except `function` can be persisted to `settings.json`; `function` is session-only and defined in TypeScript code.

### command — Shell command

Spawns a subprocess via the configured shell (default: bash / your `$SHELL`; also supports `powershell`). Receives hook input as JSON on stdin. stdout/stderr interpretation follows exit codes above.

**Key options:** `if`, `timeout`, `once`, `async`, `asyncRewake`, `statusMessage`, `shell`

### prompt — LLM prompt

Sends a prompt to an LLM (default: the small fast model). The prompt uses `$ARGUMENTS` as a placeholder for the JSON hook input. The model must respond with `{"ok": true}` or `{"ok": false, "reason": "..."}`. A forced JSON schema output mode ensures parseable responses.

**Key options:** `if`, `timeout` (default 30s), `model`, `once`, `statusMessage`

### agent — Agentic verifier

Launches a full multi-turn sub-agent (up to 50 turns) with access to all tools. The agent reads the conversation transcript at the path injected via system prompt, then calls a `StructuredOutput` tool to return `{"ok": true/false, "reason": "..."}`. Disallowed tools (AgentTool, plan mode) are filtered out to prevent recursion.

**Key options:** `if`, `timeout` (default 60s), `model`, `once`, `statusMessage`

### http — HTTP POST

POSTs the hook input JSON to a configured URL. Responses are interpreted by the caller. Supports env var interpolation in header values (only vars in `allowedEnvVars` are resolved). Protected by an SSRF guard that blocks private/link-local IP ranges; loopback (127.x) is intentionally allowed.

**Key options:** `if`, `timeout` (default 10 min), `headers`, `allowedEnvVars`, `once`, `statusMessage`

### function — TypeScript callback

An in-process TypeScript function registered programmatically via `addFunctionHook()`. Returns a boolean or Promise<boolean>. Session-scoped only — cannot be persisted to settings.json. Used internally by the skill improvement system and structured output enforcement.

**Key options:** `id` (for later removal), `timeout` (default 5s), `errorMessage`

**Important note:** "Prompt hooks do not use tool calls." The `prompt` hook queries the model with `queryModelWithoutStreaming` and uses JSON schema output mode to force parseable responses. It does NOT trigger `UserPromptSubmit` hooks — that would be infinite recursion. The same pattern applies to `agent` hooks.

---

## Config Sources & Priority

Hooks can originate from six sources. When the same event + matcher has hooks from multiple sources, they are merged and all run; the priority ordering determines display order in `/hooks`, but all hooks execute.

| Priority | Source name | File path | Scope |
|----------|------------|-----------|-------|
| 1 | `userSettings` | `~/.claude/settings.json` | All projects for this user |
| 2 | `projectSettings` | `.claude/settings.json` (project root) | Everyone working on this repo |
| 3 | `localSettings` | `.claude/settings.local.json` | Your machine + this project only |
| 4 | `policySettings` | MDM / managed config (read-only) | Enterprise admin enforced |
| 5 | `pluginHook` | `~/.claude/plugins/*/hooks/hooks.json` | Plugin-installed hooks |
| 6 | `sessionHook` | In-memory only | Current session; cleared on exit |

**policySettings can gate everything.** If `allowManagedHooksOnly: true` is set in managed (MDM) settings, _only_ managed hooks run — user, project, local, and plugin hooks are all suppressed. If `disableAllHooks: true` appears in managed settings, zero hooks run (including managed). But if `disableAllHooks` is in a non-managed source, managed hooks still run — non-managed settings cannot disable managed hooks.

### How hooks are matched

Each entry in a hook event's array is a **HookMatcher** — an object with an optional `matcher` string and an array of hook commands. For events that support matchers (e.g. `PreToolUse` matches on `tool_name`; `Notification` matches on `notification_type`; `SessionStart` matches on `source`), hooks with a matching string run; hooks with no matcher run unconditionally.

Example minimal settings.json:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Write",
        "hooks": [
          { "type": "command", "command": "./scripts/pre-write.sh" }
        ]
      },
      {
        "hooks": [
          { "type": "command", "command": "./scripts/audit-log.sh" }
        ]
      }
    ]
  }
}
```

---

## Session Hooks

Session hooks are in-memory only, attached to a specific `sessionId` stored in a `Map<string, SessionStore>`. They are created programmatically via three functions in `sessionHooks.ts`:

```typescript
// Adding a command/prompt/agent/http hook for this session
addSessionHook(setAppState, sessionId, 'Stop', '', {
  type: 'command',
  command: './verify-output.sh'
})

// Adding a TypeScript callback hook (function type)
const hookId = addFunctionHook(
  setAppState, sessionId,
  'Stop',
  '',
  (messages, signal) => checkCondition(messages),
  'Condition not met',
  { timeout: 5000, id: 'my-hook' }
)

// Removing a function hook by ID
removeFunctionHook(setAppState, sessionId, 'Stop', hookId)
```

Session hooks are _not_ reactive — they sit in a `Map` (not a plain object) so that `Object.is(next, prev)` returns true during state updates and does not fire unnecessary re-renders. The map is mutated in place; only the session-scoped data changes.

### Session hooks from frontmatter

When a skill or agent is loaded, its frontmatter `hooks` section is registered as session hooks via `registerSkillHooks()` or `registerFrontmatterHooks()`. These hooks live only while the session (or agent) is active.

One detail: agents trigger `SubagentStop`, not `Stop`. The `registerFrontmatterHooks()` function automatically converts any `Stop` entries in an agent's frontmatter to `SubagentStop` so they fire at the correct point.

**once: true hooks are self-destructing.** When a hook command includes `"once": true`, `registerSkillHooks()` attaches an `onHookSuccess` callback that calls `removeSessionHook()` on first successful execution. This is the mechanism for one-shot initialization patterns.

---

## Async & asyncRewake

Command hooks support two async flags. When `async: true`, the hook process is launched but Claude does not wait for it — the conversation continues immediately. The process is tracked in `AsyncHookRegistry` and polled in the main loop.

When `asyncRewake: true`, the hook runs in background but the main loop wakes the model if the process exits with code 2. This allows a background watcher to interrupt Claude's next response with a blocking error, even though the hook itself ran asynchronously.

Example asyncRewake configuration:

```json
{
  "PostToolUse": [
    {
      "matcher": "Bash",
      "hooks": [
        {
          "type": "command",
          "command": "./run-tests-background.sh",
          "asyncRewake": true
        }
      ]
    }
  ]
}
```

The registry tracks each pending hook by `processId`. When the main loop polls via `checkForAsyncHookResponses()`, completed hooks are finalized and their stdout parsed for a JSON `SyncHookJSONOutput` response. If none is found, the response object defaults to `{}`.

---

## HTTP Hooks: Security Model

HTTP hooks POST JSON to a URL. The security model has three layers:

1. **URL allowlist:** If `allowedHttpHookUrls` is set in merged settings, only URLs matching the listed glob patterns (using `*` as wildcard) are permitted. An empty array blocks all HTTP hooks.

2. **Env var allowlist:** Header values can contain `$VAR_NAME` interpolation, but only variables listed in the hook's `allowedEnvVars` array are resolved. Variables not on the list are replaced with empty strings to prevent secret exfiltration via project hooks.

3. **SSRF guard:** DNS resolution is intercepted by `ssrfGuardedLookup()`, which blocks private IP ranges (10.x, 172.16-31.x, 192.168.x, 169.254.x, 100.64-127.x) and unspecified/link-local IPv6. Loopback (127.0.0.1, ::1) is intentionally allowed for local dev servers. When a sandbox proxy or env-var proxy is active, the guard is bypassed (the proxy performs its own DNS; blocking the proxy's own IP would break corporate proxies on private networks).

Example HTTP hook with auth header:

```json
{
  "Stop": [
    {
      "hooks": [
        {
          "type": "http",
          "url": "https://api.example.com/claude-stop-event",
          "headers": {
            "Authorization": "Bearer $MY_TOKEN"
          },
          "allowedEnvVars": ["MY_TOKEN"],
          "timeout": 10
        }
      ]
    }
  ]
}
```

The `$MY_TOKEN` reference in the header value is only resolved because `MY_TOKEN` appears in `allowedEnvVars`. Any other `$VAR` in the same header template would be silently replaced with an empty string. Header values are also sanitized to strip CR/LF/NUL to prevent HTTP header injection.

---

## Prompt & Agent Hooks: Deep Dive

### Prompt hooks

A prompt hook sends a message to an LLM using `queryModelWithoutStreaming`. The system prompt tells the model to respond as JSON matching `{"ok": boolean, "reason"?: string}`. The output format is enforced via a JSON schema output mode — the model cannot produce anything unparseable.

The hook resolves argument placeholders (`$ARGUMENTS`, `$0`, `$ARGUMENTS[0]`, etc.) before sending. The model used defaults to the "small fast model" (configurable per-hook via `model` field).

If the model returns `{"ok": false}`, the hook result is `blocking` with a `preventContinuation: true` flag. If it returns `{"ok": true}`, the hook is a success. Malformed JSON or schema validation failures produce a non-blocking error.

### Agent hooks

Agent hooks run a full multi-turn agent loop (up to 50 turns) via `query()`. The agent gets a custom system prompt including the path to the conversation transcript, access to all tools filtered by permissions, plus a `StructuredOutput` tool it must call to return results. The structured output enforcement is registered as a session-level `Stop` function hook before the agent loop starts, and cleaned up after.

If the agent hits 50 turns without calling `StructuredOutput`, the outcome is `cancelled` — no error is shown to the user. The same applies if the agent finishes without calling the tool at all.

StructuredOutput enforcement pattern:

```typescript
// hookHelpers.ts — registerStructuredOutputEnforcement
addFunctionHook(
  setAppState,
  sessionId,
  'Stop',
  '',
  messages => hasSuccessfulToolCall(messages, SYNTHETIC_OUTPUT_TOOL_NAME),
  `You MUST call the ${SYNTHETIC_OUTPUT_TOOL_NAME} tool to complete this request. Call this tool now.`,
  { timeout: 5000 }
)
```

This function hook fires before the agent's Stop, checks whether a successful `StructuredOutput` tool call exists in the message history, and if not, injects the error message to force the model to call it before finishing.

---

## Hook Event System (SDK Telemetry)

In addition to executing hooks, Claude Code emits hook execution events to SDK consumers via a separate in-process event bus in `hookEvents.ts`. This is distinct from the hook execution system — it is purely observability/telemetry.

### Three event types

- **started** — emitted when a hook begins executing
- **progress** — emitted on a polling interval (default 1s) while the hook runs, carrying current stdout/stderr
- **response** — emitted when the hook finishes, with full output, exit code, and outcome

### Always-emitted events

Two events are always emitted regardless of the `includeHookEvents` SDK option: `SessionStart` and `Setup`. These are described in the source as "low-noise lifecycle events that were in the original allowlist and are backwards-compatible." All other events require `includeHookEvents: true` in the SDK options or running in `CLAUDE_CODE_REMOTE` mode.

Up to 100 events are buffered in `pendingEvents` if no handler is registered yet (e.g., the SDK consumer attaches after the first few hooks fire). When a handler is registered, buffered events are flushed immediately.

---

## The `if` Filter Field

Every persisted hook command type supports an optional `if` field. It uses permission rule syntax — the same syntax as `allowedTools` patterns. Examples: `"Bash(git *)"`, `"Read(*.ts)"`. The hook only spawns if the tool call matches the pattern.

Example `if` field — only fires for git push:

```json
{
  "PreToolUse": [
    {
      "matcher": "Bash",
      "hooks": [
        {
          "type": "command",
          "command": "./git-safety.sh",
          "if": "Bash(git push*)"
        }
      ]
    }
  ]
}
```

The `if` field is part of hook identity: two hooks with the same command but different `if` values are considered distinct hooks and both run when the session has them registered. The `shell` field is also part of identity — `"command": "foo"` with `"shell": "bash"` and `"command": "foo"` with `"shell": "powershell"` are distinct hooks.

---

## Real-world Patterns

### Pattern 1: Lint-on-write guard (PreToolUse)

```json
{
  "PreToolUse": [
    {
      "matcher": "Write",
      "hooks": [
        {
          "type": "command",
          "command": "jq -e '.tool_input.file_path | test(\"test.*.ts$\")' <<< \"$CLAUDE_HOOK_INPUT\" && echo 'Must write tests alongside implementation' >&2 && exit 2 || exit 0",
          "statusMessage": "Checking test coverage policy..."
        }
      ]
    }
  ]
}
```

If the tool input path does not match the test file pattern, exit 2 blocks the write and the error is shown to Claude, which will then reconsider. Exit 0 allows the write to proceed silently.

### Pattern 2: Session context injection (SessionStart)

```json
{
  "SessionStart": [
    {
      "matcher": "startup",
      "hooks": [
        {
          "type": "command",
          "command": "echo \"Today is $(date). Open PRs: $(gh pr list --json number | jq length)\""
        }
      ]
    }
  ]
}
```

stdout on exit 0 is shown to Claude as session-start context. The model sees this before the first user prompt. Good for injecting dynamic environment info (date, PR state, branch, deployment status) that would otherwise require manual context from the user.

### Pattern 3: Stop verification with an agent hook

```json
{
  "Stop": [
    {
      "hooks": [
        {
          "type": "agent",
          "prompt": "Verify that the implementation includes unit tests and that they all pass. Read the transcript at $ARGUMENTS[transcript_path] to understand what was built, then run the tests.",
          "timeout": 120
        }
      ]
    }
  ]
}
```

The agent hook spawns a full sub-agent with access to all tools. It can read files, run commands, and inspect the transcript. If it returns `{"ok": false, "reason": "Tests failed: 3 assertions"}`, the session continues and Claude must address the failure.

### Pattern 4: .envrc auto-load on cwd change (CwdChanged)

```json
{
  "CwdChanged": [
    {
      "hooks": [
        {
          "type": "command",
          "command": "[ -f .envrc ] && direnv export bash >> \"$CLAUDE_ENV_FILE\" || true"
        }
      ]
    }
  ]
}
```

Claude Code sets `CLAUDE_ENV_FILE` to a temp file path. Writing `export VAR=value` lines there causes those env vars to be applied to subsequent BashTool commands. This pattern mimics direnv integration without running the full direnv daemon.

### Pattern 5: LLM-based policy check (prompt hook)

```json
{
  "PreToolUse": [
    {
      "matcher": "Bash",
      "hooks": [
        {
          "type": "prompt",
          "prompt": "The following bash command is about to run: $ARGUMENTS\nReturn ok: true only if the command does not destructively modify production infrastructure. Common destructive patterns: terraform destroy, kubectl delete on prod namespaces, DROP TABLE, rm -rf on critical paths.",
          "model": "claude-sonnet-4-6"
        }
      ]
    }
  ]
}
```

Prompt hooks are ideal for fuzzy policy decisions that are hard to express as regex. The model evaluates intent, not just pattern matching. Using a stronger model (sonnet rather than haiku) improves accuracy for nuanced cases.

---

## Key Takeaways

1. **Exactly 27 hook events exist.** They are defined in `HOOK_EVENTS` in `src/entrypoints/sdk/coreTypes.ts`. If a new event is added, it appears there first. The metadata (descriptions, matcher fields, valid matcher values) lives in `hooksConfigManager.ts → getHookEventMetadata()`.

2. **Exit code 2 is the universal "block and tell the model" code.** All other non-zero codes are user-visible noise only. This distinction is the most important thing to understand about the hook protocol. StopFailure is the sole exception — it ignores all exit codes.

3. **The 5 hook command types form a hierarchy of power vs. simplicity:** `command` (shell subprocess) → `prompt` (single LLM call) → `agent` (full multi-turn agent) → `http` (POST to URL) → `function` (TypeScript callback, session-only). Simpler types have lower latency and are more predictable.

4. **Six config sources, one merge.** User > Project > Local > Policy > Plugin > Session. Policy settings (MDM) have special powers: `allowManagedHooksOnly` suppresses everything else; `disableAllHooks` in policy kills even managed hooks. Non-managed settings cannot block managed hooks.

5. **Session hooks use a Map, not a Record, intentionally.** The Map is mutated in place so `Object.is(next, prev)` returns true and store listeners don't fire. This matters under parallel agent workflows where dozens of function hooks are registered in a single tick.

6. **HTTP hooks have a three-layer security model:** URL allowlist (glob patterns), env var allowlist (only listed vars interpolated in headers), and SSRF guard (blocks private IP ranges, allows loopback). Project-level hooks can't exfiltrate secrets or reach internal infrastructure by default.

7. **The hook event bus** (`hookEvents.ts`) is separate from hook execution. It's purely observability. Only `SessionStart` and `Setup` events are always emitted; all others require `includeHookEvents: true` or remote mode. Up to 100 events buffer if no SDK consumer is attached yet.

---

## Quiz

**Q1: A PreToolUse hook script exits with code 1 and writes to stderr. What happens?**

A) The tool call is blocked and the model sees the stderr
B) The stderr is shown to the user only; the tool call proceeds
C) The stderr is ignored entirely
D) The tool call is blocked and the user sees the stderr

**Correct answer: B** — "Exit codes other than 0 and 2 show stderr to the USER only. The tool proceeds. Only exit code 2 shows stderr to the MODEL and blocks the tool call."

---

**Q2: Which hook event would you use to inject dynamic context (e.g., current git branch) before Claude's first response?**

A) Setup
B) SessionStart
C) UserPromptSubmit
D) PreToolUse

**Correct answer: B** — "SessionStart fires when a new session is started (source: startup, resume, clear, compact). Its stdout on exit 0 is shown to Claude before the first user prompt — perfect for injecting dynamic context. Setup is for repo init/maintenance tasks."

---

**Q3: An agent hook reaches 50 turns without calling StructuredOutput. What is the outcome?**

A) The hook result is `blocking` and the session cannot continue
B) The hook result is `non_blocking_error` and stderr is shown to the user
C) The hook result is `cancelled` and no message is shown to the user
D) The agent is given 50 more turns automatically

**Correct answer: C** — "The source code (execAgentHook.ts) explicitly sets outcome to "cancelled" on max turns and logs the event without showing any UI message. This is intentional — a silent timeout, not a visible error."

---

**Q4: You want a hook to fire only when Claude runs a `git push` bash command, not other bash commands. Which field controls this?**

A) The `matcher` field set to `"git push"`
B) The `if` field set to `"Bash(git push*)"`
C) The `command` field with a grep filter
D) A separate hook for the GitPush event

**Correct answer: B** — "The matcher field matches against event field values (tool_name for PreToolUse/PostToolUse). The if field uses permission rule syntax to filter within a matched event. "Bash(git push*)" filters to only bash commands starting with "git push"."

---

**Q5: Your company's MDM policy sets `disableAllHooks: true` in `policySettings`. A plugin also registers hooks. What runs?**

A) Only plugin hooks run (they're registered separately)
B) Only managed (policySettings) hooks run
C) Zero hooks run — disableAllHooks in policy disables everything
D) User and plugin hooks run; only project hooks are disabled

**Correct answer: C** — "disableAllHooks: true in policySettings (managed config) disables ALL hooks including plugin and managed hooks. This is the only way to disable managed hooks — non-managed settings cannot disable them."

---

**Q6: You add `"asyncRewake": true` to a PostToolUse command hook. When does the model get interrupted?**

A) Never — asyncRewake is fire-and-forget
B) When the hook process exits with any non-zero code
C) When the hook process exits with code 2 (blocking error)
D) Immediately after the tool use, synchronously

**Correct answer: C** — "asyncRewake runs the hook in the background but wakes the model specifically when the process exits with code 2. Other exit codes and exit 0 do not wake the model. This enables background watchers to inject blocking errors into the next model response."

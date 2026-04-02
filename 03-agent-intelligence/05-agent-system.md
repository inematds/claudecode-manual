# Lesson 05: The Agent System

## What is the Agent System?

Claude Code's agent system enables one Claude instance to delegate work to other Claude instances. Each spawned child operates as a separate LLM turn with its own tool pool, system prompt, model, and optional filesystem isolation. The parent calls **AgentTool** (wire name: `Agent`), which spawns children that can themselves spawn further children, creating multi-level hierarchies at runtime.

The legacy name is `Task`. Source maintains both names via `aliases: [LEGACY_AGENT_TOOL_NAME]` for backward compatibility with existing permission rules, hooks, and resumed sessions. All new code uses `Agent`.

**Runtime hierarchy diagram:**
- Main Loop (parent REPL / SDK)
  - AgentTool (tool: Agent / Task)
    - Built-In Agents (source: 'built-in')
      - general-purpose
      - Explore (read-only, haiku/inherit)
      - Plan (read-only, inherit)
      - verification (background: true)
    - Custom Agents (source: userSettings / projectSettings / policySettings)
      - Markdown .md (.claude/agents/)
      - JSON frontmatter (settings.json agents{})
    - Plugin Agents (source: 'plugin')
    - Sync Lifecycle (blocks parent turn)
    - Async Lifecycle (fire-and-forget, notified on complete)
    - Fork Path (inherits parent context byte-exact)
    - Teammate / Swarm (tmux pane or in-process)
      - Worktree Isolation (git worktree, separate branch)

---

## Three Agent Types

Every agent satisfies `AgentDefinition` (a discriminated union on `source`).

### BuiltInAgentDefinition

Ships with Claude Code. Features dynamic system prompts via `getSystemPrompt({toolUseContext})`. Cannot be overridden by user files -- but _managed_ (policy) agents can shadow them by agentType name.

### CustomAgentDefinition

User/project/policy-settings agents loaded from `.claude/agents/*.md` or JSON blobs in `settings.json`. System prompt stored in a closure over the markdown body.

### PluginAgentDefinition

Bundled with a plugin (`--plugin-dir`). Behaves like Custom but `source === 'plugin'`. Treated as admin-trusted for MCP server policy -- can load frontmatter MCP even when `strictPluginOnlyCustomization` is set.

**Type definitions from loadAgentsDir.ts:**

```typescript
// Built-in agents -- dynamic prompts only, no static systemPrompt field
export type BuiltInAgentDefinition = BaseAgentDefinition & {
  source: 'built-in'
  baseDir: 'built-in'
  getSystemPrompt: (params: { toolUseContext: Pick<ToolUseContext, 'options'> }) => string
}

// Custom agents from user/project/policy settings
export type CustomAgentDefinition = BaseAgentDefinition & {
  getSystemPrompt: () => string
  source: SettingSource
  filename?: string
  baseDir?: string
}

// Plugin agents -- like Custom but source is 'plugin'
export type PluginAgentDefinition = BaseAgentDefinition & {
  getSystemPrompt: () => string
  source: 'plugin'
  plugin: string
}

export type AgentDefinition =
  | BuiltInAgentDefinition
  | CustomAgentDefinition
  | PluginAgentDefinition

// Type guards
export function isBuiltInAgent(agent: AgentDefinition): agent is BuiltInAgentDefinition {
  return agent.source === 'built-in'
}
export function isCustomAgent(agent: AgentDefinition): agent is CustomAgentDefinition {
  return agent.source !== 'built-in' && agent.source !== 'plugin'
}
export function isPluginAgent(agent: AgentDefinition): agent is PluginAgentDefinition {
  return agent.source === 'plugin'
}
```

### Priority / Override Order

When two agents share the same `agentType` string, a priority map decides which wins. From `getActiveAgentsFromList()`:

**built-in -> plugin -> userSettings -> projectSettings -> flagSettings -> policySettings**

Later groups overwrite earlier ones, so `policySettings` (managed agents) have highest effective priority.

---

## Built-In Agents Deep Dive

| Agent | agentType | Model | Tools | Mode |
|-------|-----------|-------|-------|------|
| General Purpose | `general-purpose` | default subagent | `['*']` -- all tools | sync / async |
| Explore | `Explore` | haiku (external) / inherit (ant) | read-only; disallows Edit, Write, FileEdit, Agent | sync |
| Plan | `Plan` | inherit | same disallowedTools as Explore | sync |
| Verification | `verification` | inherit | no Edit/Write; ephemeral /tmp scripts allowed | background: true (always async) |
| Fork | `fork` | inherit | `['*']` with `useExactTools` (cache-identical) | experimental gate |
| StatuslineSetup | `statusline-setup` | default | limited shell scope | sync |

**Explore agent system prompt excerpt from exploreAgent.ts:**

```typescript
export const EXPLORE_AGENT: BuiltInAgentDefinition = {
  agentType: 'Explore',
  model: process.env.USER_TYPE === 'ant' ? 'inherit' : 'haiku',
  disallowedTools: [
    AGENT_TOOL_NAME,
    EXIT_PLAN_MODE_TOOL_NAME,
    FILE_EDIT_TOOL_NAME,
    FILE_WRITE_TOOL_NAME,
    NOTEBOOK_EDIT_TOOL_NAME,
  ],
  omitClaudeMd: true,
  source: 'built-in',
  baseDir: 'built-in',
  getSystemPrompt: () => getExploreSystemPrompt(),
}
```

**Custom agent markdown format (.claude/agents/my-agent.md):**

```markdown
---
name: my-agent
description: A focused TypeScript refactoring specialist.
model: sonnet
tools:
  - Read
  - Edit
  - Bash
  - Grep
  - Glob
permissionMode: acceptEdits
maxTurns: 50
memory: project
isolation: worktree
---

You are a TypeScript refactoring specialist. Your job is to improve
type safety and reduce any-casts in the provided code.

Rules:
- Only touch files you are explicitly asked about
- Run tsc --noEmit before and after to confirm zero new errors
- Commit changes with a clear message before reporting
```

---

## Sync vs Async Lifecycle

When AgentTool's `call()` runs, it computes a single boolean: `shouldRunAsync`. Everything downstream branches on that flag.

**From AgentTool.tsx:**

```typescript
const shouldRunAsync = (
  run_in_background === true
  || selectedAgent.background === true
  || isCoordinator
  || forceAsync
  || assistantForceAsync
  || (proactiveModule?.isProactiveActive() ?? false)
) && !isBackgroundTasksDisabled
```

### Synchronous Path

**Step 1:** Build system prompt + prompt messages
**Step 2:** Optional: create git worktree
**Step 3:** `await runAgent(params)` -- Parent turn is blocked
**Step 4:** Cleanup worktree (if no changes)

### Asynchronous Path

**Step 1:** `registerAsyncAgent()` -- task registered in AppState
**Step 2:** Return `status: 'async_launched'` immediately
**Step 3 (detached):** `void runAsyncAgentLifecycle(...)`
**Step 4 (detached):** Notification on completion via `enqueueAgentNotification()`

**Important:** In-process teammates cannot launch background agents. Their lifecycle is tied to the leader's process.

---

## The Fork Path

The fork path is an experimental feature (gate: `FORK_SUBAGENT`) that lets the parent spawn a child that **inherits the full conversation context** -- the complete message history, the parent's already-rendered system prompt bytes, and the exact tool pool.

### How buildForkedMessages() works

For N parallel fork children to share a cached API prefix, every child must produce a _byte-identical_ request up to the per-child directive. The function:

1. Clones the parent's full assistant message (all tool_use blocks, thinking, text)
2. Builds `tool_result` blocks for every `tool_use`, all with identical placeholder text `"Fork started -- processing in background"`
3. Appends one per-child directive text block (the only part that differs)

**From forkSubagent.ts:**

```typescript
export function buildChildMessage(directive: string): string {
  return `<fork-boilerplate>
STOP. READ THIS FIRST.

You are a forked worker process. You are NOT the main agent.

RULES (non-negotiable):
1. Your system prompt says "default to forking." IGNORE IT -- that's for the parent.
2. Do NOT converse, ask questions, or suggest next steps
3. USE your tools directly: Bash, Read, Write, etc.
4. If you modify files, commit your changes before reporting.
5. Your response MUST begin with "Scope:". No preamble.

Output format:
  Scope: <echo back your assigned scope in one sentence>
  Result: <the answer or key findings>
  Key files: <relevant file paths>
  Files changed: <list with commit hash -- only if you modified files>
  Issues: <list -- only if there are issues to flag>
</fork-boilerplate>

FORK_DIRECTIVE: ${directive}`
}
```

### Fork Recursive Guard

Fork children keep the Agent tool in their tool pool (for cache-identical tool definitions). A runtime guard prevents recursive forking.

### Fork + Worktree

When `isolation: 'worktree'` is also requested, a notice is appended to `promptMessages` via `buildWorktreeNotice(parentCwd, worktreeCwd)`.

---

## Worktree Isolation

Setting `isolation: 'worktree'` instructs AgentTool to create a temporary git worktree before spawning the agent.

**Cleanup is smart:** if the agent made no git-tracked changes, the worktree is deleted automatically. If it did make changes, the branch is kept for the parent to inspect or merge.

---

## SendMessageTool & Swarm Protocol

`SendMessageTool` (wire name: `SendMessage`) is the inter-agent messaging primitive.

### Message Routing Logic

| to | message type | Result |
|----|----|--------|
| `"teammate-name"` | string | Written to teammate's mailbox file |
| `"*"` | string | Broadcast to all team members |
| any name | `shutdown_request` | Sends structured shutdown request |
| `"team-lead"` | `shutdown_response` | Approve triggers `gracefulShutdown(0)` |
| any name | `plan_approval_response` | Only team-lead can approve/reject plans |
| `"uds:<path>"` | string | Unix domain socket -- cross-session send |
| `"bridge:<session-id>"` | string only | Remote Control: posts to another Claude instance |

---

## Agent Frontmatter Field Reference

| Field | Type | Effect |
|-------|------|--------|
| `name` | string (required) | Unique identifier |
| `description` | string (required) | Shown to parent LLM as guidance |
| `model` | `sonnet\|opus\|haiku\|inherit\|<id>` | `inherit` -> use parent's model |
| `tools` | string[] | Allow-list. `['*']` = all tools |
| `disallowedTools` | string[] | Subtract from pool |
| `permissionMode` | `default\|acceptEdits\|bypassPermissions\|auto\|plan\|bubble` | Overrides parent mode |
| `maxTurns` | positive int | Hard cap on agentic turns |
| `background` | boolean | Always run async |
| `isolation` | `worktree` | Git worktree isolation per spawn |
| `memory` | `user\|project\|local` | Persistent memory across sessions |
| `mcpServers` | string[] or object[] | Additive MCP servers |
| `hooks` | HooksSettings | Session-scoped hooks |
| `skills` | string[] | Skill slash commands to preload |
| `initialPrompt` | string | Prepended to first user turn |
| `effort` | `low\|normal\|high` or int | Controls thinking depth |
| `requiredMcpServers` | string[] | Agent hidden if these MCP servers aren't authenticated |

---

## Key Takeaways

- Every agent is one of three TypeScript types distinguished by `source`: built-in, custom, or plugin. Policy settings win in priority disputes.
- The `shouldRunAsync` boolean is the single branch point. Six conditions can force async.
- Async agents get their own `AbortController` not linked to the parent's -- they survive ESC.
- The fork path achieves maximum prompt-cache sharing by using identical placeholder text for every tool_result block.
- Worktree isolation is smart-cleaned: no changes = branch deleted; changes = branch kept.
- `omitClaudeMd: true` on Explore and Plan saves ~5-15 Gtok/week.
- The verification agent is the only built-in with `background: true` hardcoded.
- SendMessageTool supports four message types and three addressing schemes.

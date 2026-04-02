# mdENG — Lesson 39 — System Prompt Construction

## Overview

Claude Code assembles a system prompt spanning thousands of tokens before processing user messages. Rather than a static binary string, it's compiled fresh each session from composable sections—some cached across turns, others recomputed per request—shaped by active tools, user settings, CLAUDE.md files, MCP servers, and feature flags.

**Source files covered:**
- `constants/prompts.ts`
- `utils/systemPrompt.ts`
- `constants/systemPromptSections.ts`
- `utils/claudemd.ts`
- `memdir/memdir.ts`

The architecture separates concerns: `prompts.ts` owns section _content_; `systemPrompt.ts` owns _priority logic_; `systemPromptSections.ts` owns the _caching registry_.

### Six-Layer Architecture

**Layer 1 — Priority resolver:** `systemPrompt.ts` implements override → coordinator → agent → custom → default

**Layer 2 — Content factory:** `prompts.ts` assembles static + dynamic sections per session

**Layer 3 — Section registry:** `systemPromptSections.ts` manages memoized vs. volatile caching

**Layer 4 — CLAUDE.md loader:** `claudemd.ts` handles multi-scope file discovery, @include, frontmatter

**Layer 5 — Memory system:** `memdir/memdir.ts` injects MEMORY.md + auto-memory

**Layer 6 — Cache boundary:** `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` splits global-scope prefix

---

## 02 Priority Resolver — Which Prompt Even Runs?

`buildEffectiveSystemPrompt()` in `utils/systemPrompt.ts` implements a strict priority waterfall. Once a higher-priority source is found, lower layers are skipped.

**Priority order:**

1. **overrideSystemPrompt** (loop mode) — returns override alone; all other sources ignored
2. **COORDINATOR_MODE** (env + feature flag) — returns coordinator prompt + appendSystemPrompt
3. **mainThreadAgentDefinition** set:
   - If PROACTIVE active: returns default prompt + custom agent instructions + appendSystemPrompt
   - Normal mode: returns agent prompt + appendSystemPrompt
4. **customSystemPrompt** (via --system-prompt flag) — returns custom + appendSystemPrompt
5. **Default** — returns default prompt + appendSystemPrompt

**Key insight:** `appendSystemPrompt` always appends to every branch _except_ when `overrideSystemPrompt` is active. This enables `--append-system-prompt` to work regardless of mode.

In proactive/KAIROS mode, agent instructions are _appended_ to the default prompt rather than replacing it, mirroring a "teammate" pattern:

```typescript
// utils/systemPrompt.ts — proactive mode branch
if (agentSystemPrompt && (feature('PROACTIVE') || feature('KAIROS')) && isProactiveActive()) {
  return asSystemPrompt([
    ...defaultSystemPrompt,
    `\n# Custom Agent Instructions\n${agentSystemPrompt}`,
    ...(appendSystemPrompt ? [appendSystemPrompt] : []),
  ])
}
```

---

## 03 Content Factory — getSystemPrompt()

`getSystemPrompt()` in `constants/prompts.ts` is the main content factory building the `defaultSystemPrompt` array.

**Fast escape hatch:** If `CLAUDE_CODE_SIMPLE=1`, returns a three-line stub and exits immediately.

For normal sessions, it gathers four pieces of runtime state in parallel, then assembles two halves separated by a boundary marker controlling prompt-cache scoping:

```typescript
// constants/prompts.ts — abbreviated assembly
export async function getSystemPrompt(tools, model, additionalDirs, mcpClients) {
  const [skillToolCommands, outputStyleConfig, envInfo] = await Promise.all([
    getSkillToolCommands(cwd),
    getOutputStyleConfig(),
    computeSimpleEnvInfo(model, additionalDirs),
  ])

  return [
    // -- Static (globally cacheable) --
    getSimpleIntroSection(outputStyleConfig),
    getSimpleSystemSection(),
    getSimpleDoingTasksSection(),      // code-style, YAGNI, security rules
    getActionsSection(),               // reversibility / blast-radius guidance
    getUsingYourToolsSection(enabledTools),
    getSimpleToneAndStyleSection(),
    getOutputEfficiencySection(),
    // -- Boundary marker --
    SYSTEM_PROMPT_DYNAMIC_BOUNDARY,
    // -- Dynamic (session-specific, registry-managed) --
    ...resolvedDynamicSections,        // session_guidance, memory, env_info_simple, ...
  ].filter(s => s !== null)
}
```

### Static Sections Reference

| Section | Content | Scope |
|---------|---------|-------|
| `getSimpleIntroSection()` | Identity ("interactive agent for software engineering tasks"), URL-generation guard, CYBER_RISK_INSTRUCTION | static |
| `getSimpleSystemSection()` | Markdown rendering, permission-mode tool approval, system-reminder tags, prompt-injection warning, hooks guidance, auto-compression notice | static |
| `getSimpleDoingTasksSection()` | Task scope, YAGNI/no-gold-plating rules, no speculative abstractions, security hygiene, code-style bullets | static |
| `getActionsSection()` | Reversibility and blast-radius guidance—when to pause, risky action taxonomy (destructive, hard-to-reverse, visible-to-others) | static |
| `getUsingYourToolsSection()` | Prefer dedicated tools over Bash, REPL-mode path, parallel tool calls, task-tracking tool | static |
| `getSimpleToneAndStyleSection()` | No emojis, file:line references, GitHub issue format, no colon before tool calls | static |
| `getOutputEfficiencySection()` | Conciseness/prose quality rules — ant build gets rich prose guide, external gets brief instructions | static |

### ANT-Only Branches

Several sections—comment-writing norms, false-claims mitigation, assertiveness counterweights, numeric length anchors—are gated on `process.env.USER_TYPE === 'ant'`. These compile out of the external binary via Bun dead-code elimination. Every `// @[MODEL LAUNCH]` comment marks instructions needing updates at each new model release.

---

## 04 Dynamic Sections — The Registry

Everything after the boundary marker is managed through the **section registry** in `constants/systemPromptSections.ts`. Each section is either memoized (computed once per session, cached until `/clear` or `/compact`) or volatile (recomputed every turn).

```typescript
// systemPromptSections.ts
export function systemPromptSection(name, compute): SystemPromptSection {
  return { name, compute, cacheBreak: false }   // memoized
}

export function DANGEROUS_uncachedSystemPromptSection(name, compute, reason) {
  return { name, compute, cacheBreak: true }    // recomputes every turn
}
```

`resolveSystemPromptSections()` iterates the registry, returns cached values for non-cacheBreak sections, and calls `compute()` for volatile or uncached ones. Cache entries are keyed by section `name` string and stored in `bootstrap/state.ts`.

### Section Registry Reference

| Section name | Content | Cache behaviour |
|--------------|---------|-----------------|
| `session_guidance` | Ask-user-question guidance, interactive shell tip, agent-tool guidance, skill invocation syntax, verification-agent contract | memoized |
| `memory` | CLAUDE.md hierarchy + MEMORY.md auto-memory | memoized |
| `ant_model_override` | Internal ant build suffix injected from remote config | memoized |
| `env_info_simple` | CWD, git status, platform, shell, OS version, model name, knowledge cutoff, model family reference | memoized |
| `language` | Language preference from settings → "Always respond in X" | memoized |
| `output_style` | Named output style loaded from settings (name + prompt string) | memoized |
| `mcp_instructions` | Per-server `instructions` blocks from connected MCP servers | volatile |
| `scratchpad` | Per-session scratchpad directory path and usage rules | memoized |
| `frc` | Function-result-clearing notice (CACHED_MICROCOMPACT feature) | memoized |
| `token_budget` | "Keep working until you approach the target" instruction | memoized |

### Why mcp_instructions Is Volatile

"MCP servers can connect and disconnect between turns. If the instructions" were cached, "a server that connects after the first turn would never" get injected. Making it volatile ensures it's always current—at the cost of potentially busting the prompt cache on server connect/disconnect events.

---

## 05 The Cache Boundary — How Token Cost Is Kept Low

Claude's API supports prompt caching: identical prefixes are billed at a fraction of normal input-token cost. To exploit this, the system prompt splits into a **stable prefix** (same across all users) and a **volatile tail** (per-session content).

```typescript
// constants/prompts.ts
export const SYSTEM_PROMPT_DYNAMIC_BOUNDARY = '__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__'

// WARNING: Do not remove or reorder this marker without updating cache logic in:
// - src/utils/api.ts (splitSysPromptPrefix)
// - src/services/api/claude.ts (buildSystemPromptBlocks)
```

Everything _before_ the marker can use `scope: 'global'` in the API call—cacheable across all organizations. Everything _after_ contains session-specific content (CWD, git status, CLAUDE.md files, model ID) and must not be globally cached.

**Boundary contents:**

- **Static Prefix** (globally cacheable): Intro, System, Doing Tasks, Actions, Tools, Tone, Output Efficiency
- **Dynamic Tail** (session-specific): Session Guidance, Memory/CLAUDE.md, Env Info, Language, Output Style, MCP Instructions, Scratchpad, FRC, Token Budget

### Engineering Constraint

Any runtime-conditional expression placed _before_ the boundary multiplies the number of distinct cache prefix hashes by 2 per condition. Engineers explicitly moved session-specific flags (non-interactive mode, fork-subagent mode) to `getSessionSpecificGuidanceSection`—which lives after the boundary—for exactly this reason. Comments reference "PR #24490, #24171" for the same bug class.

---

## 06 CLAUDE.md Injection — How User Instructions Enter

The `memory` section loads every CLAUDE.md file the user placed in their filesystem. `utils/claudemd.ts` discovers and ranks files across four scopes, loading them in priority order (lower-priority first, higher-priority last—so the model attends more to files seen last).

### Load Order

1. **Managed memory** (`/etc/claude-code/CLAUDE.md`) — Global for all users on machine
2. **User memory** (`~/.claude/CLAUDE.md`) — Private global for all projects
3. **Project memory** (`CLAUDE.md` / `.claude/CLAUDE.md` / `.claude/rules/*.md`) — Checked into codebase
4. **Local memory** (`CLAUDE.local.md`) — Private project-specific

### Discovery Algorithm

Project and Local files are found by walking _upward_ from the current directory to filesystem root, checking each directory for `CLAUDE.md`, `.claude/CLAUDE.md`, and every `.md` file in `.claude/rules/`. Files closer to CWD appear later in the array—giving them higher effective priority.

### The @include Directive

CLAUDE.md files can reference other files with an `@path` directive placed in leaf text (not inside code blocks). Supported forms:

```markdown
# CLAUDE.md
@shared-rules.md                    # relative path (same dir)
@./scripts/lint-conventions.md      # explicit relative
@~/company/global-standards.md      # home-relative
@/absolute/path/to/rules.md        # absolute
```

Included files are inserted as _separate entries before_ the including file. Circular references are prevented via a processed-path set. Non-existent targets are silently skipped. The loader restricts includes to a large allowlist of text file extensions—binary files (images, PDFs) are rejected to prevent accidental context bloat.

### Frontmatter Path Filtering

A CLAUDE.md file can carry YAML frontmatter with a `paths` key. Only files under those glob patterns will receive the instructions. This lets language-specific or directory-specific rules coexist in a single CLAUDE.md:

```markdown
---
paths:
  - src/components/**
  - "*.tsx"
---
Always use named exports in React components.
```

### The Memory Instruction Wrapper

Every loaded memory file is wrapped in a header injected by `claudemd.ts`:

```
Codebase and user instructions are shown below. Be sure to adhere to these instructions. 
IMPORTANT: These instructions OVERRIDE any default behavior and you MUST follow them 
exactly as written.
```

This is the preamble seen in your own `<system-reminder>` context. Individual file contents are separated by clearly labeled blocks identifying each scope (user memory, project memory, local memory) so the model can reason about origin.

### MEMORY.md Auto-Memory

`memdir/memdir.ts` handles a separate auto-memory system layered on top. When enabled, a per-session `MEMORY.md` file is also injected. The file is truncated to **200 lines** or **25,000 bytes**, whichever fires first, with a warning appended if truncated. This prevents an ever-growing memory file from bloating every request.

```typescript
// memdir/memdir.ts
export const MAX_ENTRYPOINT_LINES = 200
export const MAX_ENTRYPOINT_BYTES = 25_000
```

---

## 07 Environment Info — computeSimpleEnvInfo()

The `env_info_simple` section is assembled by `computeSimpleEnvInfo()`. It gives the model a snapshot of the runtime context it's operating in.

### Representative Output

```
# Environment
You have been invoked in the following environment:
 - Primary working directory: /Users/alice/myproject
 - Is a git repository: true
 - Platform: darwin
 - Shell: zsh
 - OS Version: Darwin 25.3.0
 - You are powered by the model named Claude Sonnet 4.6. 
   The exact model ID is claude-sonnet-4-6.
 - Assistant knowledge cutoff is August 2025.
 - The most recent Claude model family is Claude 4.5/4.6 ...
 - Claude Code is available as a CLI in the terminal, desktop app ...
 - Fast mode for Claude Code uses the same Claude Opus 4.6 model 
   with faster output ...
```

Shell detection parses `process.env.SHELL` to extract the name. On Windows, it appends a note to prefer Unix shell syntax (`/dev/null` not `NUL`, forward slashes in paths). Knowledge cutoff dates are hardcoded per model canonical name via `getKnowledgeCutoff()`—every model launch requires a `// @[MODEL LAUNCH]` update.

### Undercover Mode

When Anthropic engineers run internal builds with `isUndercover()` active, all model name references are stripped from the environment section. This prevents internal model IDs from leaking into public commits, PRs, or screenshots.

---

## 08 MCP Server Instructions

When MCP servers are connected, `getMcpInstructions()` iterates the `ConnectedMCPServer` objects and assembles a block from any that expose an `instructions` field.

```typescript
// constants/prompts.ts
function getMcpInstructions(mcpClients) {
  const withInstructions = mcpClients
    .filter(c => c.type === 'connected')
    .filter(c => c.instructions)

  const blocks = withInstructions.map(c => `## ${c.name}\n${c.instructions}`).join('\n\n')

  return `# MCP Server Instructions\n\n...\n\n${blocks}`
}
```

This injects text like "IMPORTANT: Before using any chrome browser tools, you MUST first load them using ToolSearch" into the system prompt—it comes verbatim from the MCP server's `instructions` field, not from Claude Code's own code.

There is also an experimental `mcpInstructionsDelta` path: when that feature is enabled, instructions are delivered as _persisted attachment_ objects rather than being re-injected into the system prompt every turn. This avoids the prompt-cache bust that happens when a late-connecting MCP server forces a new system prompt hash.

---

## 09 Subagent Enhancement — enhanceSystemPromptWithEnvDetails()

Subagents launched by `AgentTool` do not go through `getSystemPrompt()`. They start with whatever system prompt the caller passes—typically `DEFAULT_AGENT_PROMPT`. `enhanceSystemPromptWithEnvDetails()` then appends environment context and agent-specific notes.

```typescript
// The agent default prompt — what every subagent starts with
export const DEFAULT_AGENT_PROMPT = `You are an agent for Claude Code, Anthropic's 
official CLI for Claude. Given the user's message, you should use the tools available 
to complete the task. Complete the task fully—don't gold-plate, but don't leave it 
half-done. When you complete the task, respond with a concise report covering what 
was done and any key findings — the caller will relay this to the user, so it only 
needs the essentials.`

// Notes appended by enhanceSystemPromptWithEnvDetails()
`Notes:
- Agent threads always have their cwd reset between bash calls — use absolute file paths.
- In your final response, share file paths (always absolute, never relative) ...
- For clear communication the assistant MUST avoid using emojis.
- Do not use a colon before tool calls.`
```

`computeEnvInfo()` (the fuller version, not `computeSimpleEnvInfo`) is used here—it adds `uname -sr` output via `getUnameSR()` and wraps the env block in `<env>` XML tags rather than the simpler bullet-list format used in the main session.

---

## 10 Simple Mode & Other Escape Hatches

### CLAUDE_CODE_SIMPLE=1

Setting this environment variable activates a three-line system prompt—no instructions, no CLAUDE.md, no tool guidance, just CWD and date. Useful for benchmarking raw model capability without Claude Code's prompt overhead.

```typescript
if (isEnvTruthy(process.env.CLAUDE_CODE_SIMPLE)) {
  return [`You are Claude Code, Anthropic's official CLI for Claude.\n\nCWD: ${getCwd()}\nDate: ${getSessionStartDate()}`]
}
```

### Proactive / KAIROS Mode

When `feature('PROACTIVE')` or `feature('KAIROS')` is enabled and proactive mode is active, `getSystemPrompt()` returns a short autonomous-agent identity prompt instead of the full interactive-session prompt. It includes memory, env info, MCP instructions, and the proactive section—but none of the interactive coding-task guidance.

```typescript
return [
  `\nYou are an autonomous agent. Use the available tools to do useful work.\n\n${CYBER_RISK_INSTRUCTION}`,
  getSystemRemindersSection(),
  await loadMemoryPrompt(),
  envInfo,
  getLanguageSection(settings.language),
  getMcpInstructionsSection(mcpClients),
  getScratchpadInstructions(),
  getProactiveSection(),
].filter(s => s !== null)
```

---

## Key Takeaways

- `buildEffectiveSystemPrompt()` implements a strict priority waterfall: override > coordinator > agent > custom > default, with `appendSystemPrompt` always appending.
- The prompt splits into a static half (globally cache-scoped, same for all users) and a dynamic tail (session-specific). The boundary marker `__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__` is the seam.
- Dynamic sections are managed by a named registry. Most are memoized for the session; only `mcp_instructions` is volatile because servers connect mid-session.
- CLAUDE.md files load in four scopes (managed → user → project → local), with files closer to CWD taking higher priority by appearing last.
- The `@include` directive lets CLAUDE.md files compose other files. Frontmatter `paths` gates instructions to specific file patterns.
- Subagents bypass `getSystemPrompt()` entirely, starting from `DEFAULT_AGENT_PROMPT` then enhanced with env details.
- Multiple escape hatches exist: `CLAUDE_CODE_SIMPLE=1` for a stub prompt, proactive mode for an autonomous-agent identity, undercover mode to strip all model name references.
- Every `// @[MODEL LAUNCH]` comment marks code needing human attention at each new Claude model release.

---

## Knowledge Check

**Question 1:** What happens when `overrideSystemPrompt` is set in `buildEffectiveSystemPrompt()`?

A) It is appended to the default prompt
B) **It replaces all other prompts including appendSystemPrompt**
C) It replaces the default prompt but appendSystemPrompt is still added
D) It is ignored if a custom system prompt is also set

**Question 2:** Why is `mcp_instructions` marked as `DANGEROUS_uncachedSystemPromptSection` while other dynamic sections are memoized?

A) MCP server instructions are too large to cache
B) **MCP servers can connect and disconnect between turns, so cached instructions would go stale**
C) The Anthropic API does not support caching for MCP content
D) MCP instructions are injected as tool descriptions, not system prompt text

**Question 3:** In CLAUDE.md loading priority, which file has the _highest_ effective priority?

A) `/etc/claude-code/CLAUDE.md` (managed memory)
B) `~/.claude/CLAUDE.md` (user memory)
C) **`CLAUDE.local.md` in the project root closest to CWD**
D) `CLAUDE.md` in the filesystem root

**Question 4:** What is the purpose of `__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__`?

A) It separates user instructions from built-in instructions
B) It marks where `/compact` will truncate the context
C) **It splits the prompt into a globally-cacheable static prefix and a session-specific dynamic tail**
D) It signals the model to switch from planning to execution mode

**Question 5:** How does proactive / KAIROS mode change the system prompt?

A) It adds a proactive section to the end of the standard interactive prompt
B) **It returns a short autonomous-agent identity prompt instead of the full interactive-session prompt**
C) It uses the coordinator system prompt from coordinatorMode.js
D) It disables CLAUDE.md injection entirely

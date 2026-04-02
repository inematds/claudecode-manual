# mdENG -- Lesson 06: The Permission System -- Source Deep Dive

## Complete Lesson Content

### Contents
1. System Overview
2. The Decision Pipeline (Flowchart)
3. The 5 Permission Modes
4. Rule Matching System
5. Rule Sources & Priority
6. Auto Mode & AI Classifier
7. Shadowed Rule Detection
8. Permission Explainer
9. Key Takeaways
10. Quiz

---

## 1. System Overview

Every tool use—bash commands, file edits, MCP endpoints—passes through a permission pipeline producing three outcomes:

- **allow**: Tool executes immediately
- **deny**: Tool is blocked; Claude is told why
- **ask**: User is prompted to approve or reject

Three inputs drive the pipeline: the tool requested, its input parameters, and the current permission context (mode, rules, session state).

---

## 2. The Decision Pipeline

Implemented in `hasPermissionsToUseToolInner()` in `permissions.ts`. Steps are numbered to match inline source comments.

### Step-by-step breakdown

**1a — Deny rule for entire tool**: `getDenyRuleForTool()` checks if the tool appears in any deny list. Immediate deny if found.

**1b — Ask rule for entire tool**: Same check against ask lists. When sandbox auto-allow is on, this may be skipped.

**1c — Tool's own `checkPermissions()`**: Each tool implements its own logic (Bash checks subcommand prefixes; FileWrite checks working directory boundaries).

**1d — Tool denied permission**: If `checkPermissions` returns deny, stop immediately.

**1e — requiresUserInteraction**: Tools like ExitPlanMode, AskUserQuestion, ReviewArtifact always require human approval—even `bypassPermissions` cannot override.

**1f — Content-specific ask rules**: Rules like `Bash(npm publish:*)` are respected even in bypass mode.

**1g — Safety check (bypass-immune)**: Writing to `.git/`, `.claude/`, `.vscode/`, shell config files always requires approval, regardless of mode.

**2a — bypassPermissions mode**: If mode is `bypassPermissions` (or `plan` with bypass), everything passing step 1 is allowed.

**2b — Tool-wide allow rule**: `toolAlwaysAllowedRule()` checks if entire tool is in allow list.

**3 — Mode transformations**: `ask` results transform: `dontAsk` -> `deny`; `auto` -> classifier; headless -> hooks -> auto-deny.

---

## 3. The 5 Permission Modes

Mode controls which pipeline branch executes for `ask` results. Stored in `toolPermissionContext.mode`.

### default
Standard mode. Shows a prompt for every unrecognized tool use.

### plan
Read-only planning phase. Write tools blocked until user exits plan mode.

### acceptEdits
Auto-approves file edits inside the working directory. Shell commands still prompt.

### bypassPermissions
Skips all prompts (except deny rules, content ask rules, safety checks, `requiresUserInteraction`). Requires gated access.

### dontAsk
Converts every `ask` into a `deny`. Claude is told it cannot use the tool.

### auto (ANT-only)
Routes `ask` decisions through an AI classifier instead of human prompt. Feature-flagged via `TRANSCRIPT_CLASSIFIER`.

### What bypassPermissions cannot bypass

- **Step 1a — deny rules**: Always respected, no exceptions
- **Step 1e — requiresUserInteraction**: ExitPlanMode, AskUserQuestion, ReviewArtifact always need human
- **Step 1f — content-specific ask rules**: `Bash(npm publish:*)` in ask list always prompts
- **Step 1g — safety checks**: Writing to `.git/`, `.claude/`, `.vscode/`, shell configs always prompts—even in bypass mode

---

## 4. Rule Matching System

Every rule is a string of form `ToolName` or `ToolName(content)`, stored in allow/deny/ask lists. Parsing handled by `permissionRuleParser.ts`.

### Rule string format

```
// Tool-wide rule — no content
"Bash"                 // match any Bash command
"mcp__myserver"        // match all tools from myserver MCP
"mcp__myserver__*"     // same, wildcard variant

// Content-specific rules (three types)
"Bash(npm install)"    // exact match
"Bash(npm:*)"          // legacy prefix — matches anything starting with "npm"
"Bash(git add *)"      // wildcard — * matches any sequence of chars
"Bash(git ad\* file)"  // escaped * — matches literal asterisk
```

### Three content rule types (shellRuleMatching.ts)

**exact**: Full string equality after trimming. `Bash(npm install)`

**prefix (legacy)**: Ends with `:*`. The part before `:` must be a prefix of the command. `Bash(npm:*)` matches `npm install`, `npm run build`, etc.

**wildcard**: Contains unescaped `*`. Converted to full regex with `dotAll` flag. `Bash(git * --dry-run)`

### Wildcard pattern edge case: trailing `*`

When a pattern ends with space + single wildcard, the match engine makes the trailing part optional:

```
// Rule: "git *"
// Matches both:
"git add"     // command with argument
"git"         // bare command — optional trailing match
```

This aligns wildcard semantics with legacy prefix `git:*`.

Multi-wildcard patterns (`* run *`) are excluded from this optimization to avoid false matches.

### Rule parsing code (permissionRuleParser.ts)

```javascript
function permissionRuleValueFromString(ruleString) {
  const openIdx  = findFirstUnescapedChar(ruleString, '(')
  const closeIdx = findLastUnescapedChar(ruleString, ')')

  if (openIdx === -1) return { toolName: normalizeLegacyToolName(ruleString) }

  const toolName   = ruleString.substring(0, openIdx)
  const rawContent = ruleString.substring(openIdx + 1, closeIdx)

  // Empty or standalone "*" -> treat as tool-wide rule
  if (rawContent === '' || rawContent === '*')
    return { toolName: normalizeLegacyToolName(toolName) }

  return { toolName: normalizeLegacyToolName(toolName), ruleContent: unescapeRuleContent(rawContent) }
}
```

### Legacy tool name aliases

Old rule strings are automatically migrated at parse time:

```javascript
const LEGACY_TOOL_NAME_ALIASES = {
  Task:            AGENT_TOOL_NAME,     // -> "Agent"
  KillShell:       TASK_STOP_TOOL_NAME, // -> "TaskStop"
  AgentOutputTool: TASK_OUTPUT_TOOL_NAME,
  BashOutputTool:  TASK_OUTPUT_TOOL_NAME,
}
```

---

## 5. Rule Sources & Priority

Rules are loaded from multiple sources and merged. When `allowManagedPermissionRulesOnly` is set in `policySettings`, all non-policy sources are ignored.

| Source | File / origin | Shared? | Editable? |
|--------|---------------|---------|-----------|
| `policySettings` | Enterprise managed policy | Yes | No (read-only) |
| `projectSettings` | `.claude/settings.json` (committed) | Yes — git | Yes |
| `userSettings` | `~/.claude/settings.json` | No | Yes |
| `localSettings` | `.claude/settings.local.json` (gitignored) | No | Yes |
| `flagSettings` | `--settings` CLI flag | No | No (read-only) |
| `cliArg` | CLI startup arguments | No | No (runtime) |
| `session` | In-memory (this session only) | No | No (ephemeral) |
| `command` | Slash command frontmatter | Yes | No (read-only) |

### Settings JSON format

```json
{
  "permissions": {
    "allow": ["Bash(npm:*)", "Bash(git status)"],
    "deny":  ["WebFetch"],
    "ask":   ["Bash(npm publish:*)"]
  }
}
```

---

## 6. Auto Mode & the AI Classifier

When mode is `auto`, instead of prompting the user, Claude routes ambiguous tool uses through a secondary Claude model (the "YOLO classifier") that decides allow/deny based on system prompts describing allowed and prohibited action categories.

### Fast paths (skip the classifier API call)

1. **Non-classifiable safety checks**: Writing to `.git/` or shell configs—never auto-approved even by classifier
2. **acceptEdits fast path**: If tool would be allowed in `acceptEdits` mode (file edits inside CWD), allow immediately without calling classifier
3. **Safe tool allowlist**: Read-only tools skip classifier entirely

### Safe tool allowlist (abridged from classifierDecision.ts)

```javascript
const SAFE_YOLO_ALLOWLISTED_TOOLS = new Set([
  // Read-only file operations
  FILE_READ_TOOL_NAME,
  GREP_TOOL_NAME, GLOB_TOOL_NAME, LSP_TOOL_NAME,
  TOOL_SEARCH_TOOL_NAME, LIST_MCP_RESOURCES_TOOL_NAME,
  // Task management (metadata only)
  TODO_WRITE_TOOL_NAME,
  TASK_CREATE_TOOL_NAME, TASK_GET_TOOL_NAME, TASK_UPDATE_TOOL_NAME,
  TASK_LIST_TOOL_NAME, TASK_STOP_TOOL_NAME, TASK_OUTPUT_TOOL_NAME,
  // Plan mode / UI
  ASK_USER_QUESTION_TOOL_NAME,
  ENTER_PLAN_MODE_TOOL_NAME, EXIT_PLAN_MODE_TOOL_NAME,
  // Misc
  SLEEP_TOOL_NAME,
  YOLO_CLASSIFIER_TOOL_NAME, // the classifier itself
])
```

### Denial tracking

The classifier tracks consecutive and total denials to detect runaway loops. When limits are hit, it falls back to prompting:

```javascript
const DENIAL_LIMITS = {
  maxConsecutive: 3,   // 3 denials in a row -> prompt user
  maxTotal:      20,   // 20 total denials this session -> prompt user
}

function shouldFallbackToPrompting(state) {
  return state.consecutiveDenials >= DENIAL_LIMITS.maxConsecutive
      || state.totalDenials       >= DENIAL_LIMITS.maxTotal
}
```

A successful tool use resets the consecutive count (but not the total).

### Dangerous patterns stripped at auto mode entry

When entering auto mode, `permissionSetup.ts` strips allow rules that would let Claude bypass the classifier:

```javascript
const DANGEROUS_BASH_PATTERNS = [
  // Script interpreters (arbitrary code execution)
  'python', 'node', 'deno', 'ruby', 'perl', 'php', 'lua',
  // Package runners
  'npx', 'bunx', 'npm run', 'yarn run', 'bun run',
  // Shells & execution multipliers
  'bash', 'sh', 'eval', 'exec', 'sudo', 'xargs',
  // ... plus ANT-only: gh, curl, wget, git, kubectl, aws, gcloud
]
```

Existing allow rules matching these patterns are removed when auto mode is activated.

### Classifier fail modes

**Fail Closed (default)**: Classifier API error -> deny with retry guidance. Controlled by `tengu_iron_gate_closed` GrowthBook gate.

**Transcript Too Long**: Falls back to normal permission prompting (interactive) or throws AbortError (headless).

**Fail Open (gate off)**: If iron-gate feature is off, classifier error falls back to normal permission handling instead of denying.

---

## 7. Shadowed Rule Detection

A permission rule can be unreachable—written correctly but never evaluated because a more general rule fires first. `shadowedRuleDetection.ts` detects and reports these conflicts.

### Deny-shadowed

A tool-wide deny rule (e.g., `Bash` in deny list) makes any specific allow rule (e.g., `Bash(ls:*)`) unreachable—the deny fires first.

### Ask-shadowed

A tool-wide ask rule (e.g., `Bash` in ask list) always prompts, making a specific allow rule (e.g., `Bash(ls:*)`) unreachable. **Exception**: if ask rule is from personal settings and sandbox auto-allow is on, no warning issued.

```javascript
// Example: allow rule is deny-shadowed
permissions.deny  = ["Bash"]      // <- fires first, blocks everything
permissions.allow = ["Bash(ls:*)"] // <- never reached -> shadowed!

// Fix: remove tool-wide "Bash" from deny,
// or add more specific deny rules instead.
```

---

## 8. Permission Explainer

When a permission prompt appears, `permissionExplainer.ts` makes a side API call to generate human-readable explanation of what the command does, why Claude wants to run it, and risk level.

```javascript
// Structured output schema returned by the explainer model
{
  explanation: "What this command does (1-2 sentences)",
  reasoning:   "I need to check the file contents",  // starts with "I"
  risk:        "Modifies files outside the working directory",
  riskLevel:   "HIGH"  // LOW | MEDIUM | HIGH
}
```

- Uses `sideQuery()`—separate API call NOT counting toward session's token totals
- Model forced to use specific tool (`explain_command`) for guaranteed structured output
- Can be disabled via `permissionExplainerEnabled: false` in global config
- Includes up to 1,000 chars of recent conversation context for grounded "why" answers

---

## 9. Key Takeaways

- **Deny rules checked first**: Before bypass mode, before allow rules, before tool-specific logic. Only unconditional block.
- **bypassPermissions not truly unconditional**: Cannot bypass deny rules, content-specific ask rules, safety checks on protected paths, or tools requiring user interaction.
- **Safety-check paths bypass-immune by design**: `.git/`, `.claude/`, `.vscode/`, and shell config files always prompt, preventing silent corruption.
- **Rule specificity matters for shadowing**: Tool-wide deny or ask rule silently makes specific allow rules unreachable. Shadow detector warns about this.
- **Auto mode has layered fast paths**: Safe allowlist -> acceptEdits check -> classifier. Only last step costs API call.
- **Classifier denial limits prevent loops**: After 3 consecutive or 20 total denials, system falls back to prompting human.
- **Rule sources form layered override system**: Enterprise policy can lock rules with `allowManagedPermissionRulesOnly`, preventing user overrides.
- **Legacy rule names auto-migrated**: Rules using old tool names (`Task`, `KillShell`) transparently rewritten to current canonical names at parse time.

---

## 10. Quiz

### Q1. A user has `Bash` in their deny list AND `Bash(ls:*)` in their allow list. What happens when Claude tries to run `ls -la`?

A. It is allowed because the specific allow rule matches
B. It is denied because the deny rule fires first (step 1a)
C. The user is asked to approve it
D. It depends on which setting source the rules came from

**Correct: B**

### Q2. In `bypassPermissions` mode, which of these WILL still prompt the user?

A. `git commit -m "fix"` — a normal bash command
B. Writing to `src/main.ts` inside the working directory
C. Writing to `.git/hooks/pre-commit`
D. Reading a file with FileRead tool

**Correct: C**

### Q3. The auto mode classifier has made 3 consecutive denials this session. What happens on the 4th ambiguous action?

A. The session is aborted with an error
B. The action is automatically allowed to break the loop
C. The system falls back to showing the user a permission prompt
D. The classifier is disabled for the rest of the session

**Correct: C**

### Q4. A rule string `"Bash(npm:*)"` uses what rule type?

A. Wildcard — the `*` makes it a wildcard pattern
B. Prefix (legacy) — the `:*` suffix signals legacy prefix matching
C. Exact — it matches the literal string "npm:\*"
D. Regex — the `*` is converted to `.*`

**Correct: B**

### Q5. What does the `dontAsk` permission mode do differently from `bypassPermissions`?

A. `dontAsk` allows everything; `bypassPermissions` only allows safe tools
B. `dontAsk` converts every "ask" result to "deny"; `bypassPermissions` converts it to "allow"
C. They are functionally identical — different names for the same behavior
D. `dontAsk` skips safety checks; `bypassPermissions` does not

**Correct: B**

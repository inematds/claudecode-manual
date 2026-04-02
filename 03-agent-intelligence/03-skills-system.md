# Lesson 03: The Skills System

## Overview
This lesson explores how Claude Code discovers, loads, parses, and executes reusable prompt workflows called "skills."

## Key Definitions

A **skill** is "a named, reusable prompt workflow that Claude Code can discover and execute," stored as `SKILL.md` Markdown files with YAML frontmatter or compiled into the CLI. Skills function as slash commands like `/commit` or `/review-pr`.

## Skill Lifecycle (Six Stages)

1. **Discovery**: Walks managed/user/project directories; resolves symlinks to deduplicate
2. **Load**: Reads SKILL.md; parses YAML; estimates token budget from frontmatter only
3. **Parse**: Extracts fields into Command object; validates hooks and shell config
4. **Substitute**: Replaces `$ARGUMENTS`, named args, shell backticks, special variables
5. **Execute**: Runs inline (same context) or forked (isolated sub-agent)
6. **Inject**: Tool result returns to conversation; applies allowed tools and model overrides

## Four Skill Sources & Priority Order

1. **Managed/Policy**: `managed/.claude/skills/` (enterprise-controlled, highest priority)
2. **User**: `~/.claude/skills/` (personal cross-project library)
3. **Project**: `.claude/skills/` (repository-scoped)
4. **Bundled**: Compiled into CLI (lowest priority)

"When two sources define a skill with the same name, the first one loaded wins." Deduplication uses resolved file paths, not names.

## SKILL.md Format

Basic structure:
```yaml
---
name: "Display Name"
description: "One-line summary"
when_to_use: "Auto-invoke hint"
allowed-tools:
  - Bash(gh:*)
  - Read
  - Write
argument-hint: "[branch] [message]"
arguments: branch message
context: fork
model: claude-opus-4-5
effort: high
version: "1.0.0"
user-invocable: true
paths: src/payments/**
hooks:
  PreToolUse: [...]
shell:
  interpreter: bash
---
# Markdown body with $variable substitution
```

### Frontmatter Field Reference

| Field | Type | Purpose |
|-------|------|---------|
| `name` | string | Override directory-derived name |
| `description` | string | One-line summary for listing |
| `when_to_use` | string | Detailed trigger instructions |
| `allowed-tools` | list | Tool permission patterns |
| `argument-hint` | string | Autocomplete placeholder |
| `arguments` | string/list | Named argument identifiers |
| `context` | `fork` | Isolated sub-agent execution |
| `model` | string | Model override for skill |
| `effort` | low/medium/high/int | Thinking budget |
| `version` | string | Informational version tag |
| `user-invocable` | bool | Hide from /skills menu if false |
| `paths` | glob string(s) | Conditional activation by file pattern |
| `hooks` | object | Lifecycle hooks |
| `shell` | object | Shell interpreter config |
| `disable-model-invocation` | bool | Block Skill tool invocation |

## Argument Substitution

Processing order:
1. Named args: `$foo`, `$bar` (from `arguments` frontmatter)
2. Indexed args: `$ARGUMENTS[0]`, `$0`, `$1`
3. Full arg string: `$ARGUMENTS`
4. If no placeholder found, args appended as `ARGUMENTS: ...`
5. Shell injection: `` !`command` `` or ` ```! ` blocks (local skills only)
6. Special variables: `${CLAUDE_SKILL_DIR}`, `${CLAUDE_SESSION_ID}`

Example: `/myskill "hello world" foo` becomes `["hello world", "foo"]` via shell-quote parsing.

## Advanced Skill Patterns

### Conditional Skills (paths-based activation)

"A skill with a `paths` frontmatter field is a conditional skill. It is loaded at startup but not surfaced to Claude until the user opens or edits a file whose path matches one of the glob patterns."

```yaml
---
paths: src/payments/**
description: "Stripe refund workflow"
---
```

Patterns use `.gitignore` syntax. A `**` pattern is treated as unconditional (same as omitting `paths`).

### Bundled Skills

"Bundled skills ship inside the Claude Code binary and are registered at startup via `registerBundledSkill()`." Well-known examples:

- `/simplify` -- spawns three parallel review agents
- `/loop` -- creates cron jobs (feature-flagged)
- `/remember`, `/verify`, `/debug`, `/stuck`
- `/skillify` -- interviews and writes SKILL.md (internal)

Reference files are extracted to nonce directories with secure flags (`O_EXCL | O_NOFOLLOW | 0o600`) to prevent symlink attacks.

Registration example:
```javascript
registerBundledSkill({
  name: 'simplify',
  description: 'Review changed code for reuse, quality, and efficiency.',
  userInvocable: true,
  async getPromptForCommand(args) {
    return [{ type: 'text', text: SIMPLIFY_PROMPT }]
  },
})
```

### MCP Skills (Model Context Protocol)

MCP servers can expose skills fetched at connection time via `fetchMcpSkillsForClient()`. They appear in the skills menu under "MCP skills" with names like `server-name:skill-name`.

**Key differences from local skills:**

- **No shell injection**: `` !backtick `` and ` ```! ` blocks are silently skipped
- **${CLAUDE_SKILL_DIR} unavailable**: MCP skills have no local directory
- **Separate registry**: Live in `AppState.mcp.commands`; merged via `getAllCommands()` at invocation
- **Security rationale**: "MCP skills are remote and untrusted -- never execute inline shell commands from their markdown body"

The `mcpSkillBuilders.ts` module solves circular imports between client and skill loading code.

### Skill Permissions & Auto-Allow Logic

`checkPermissions()` follows this waterfall:

1. **Deny rules**: Block if name or prefix (`:*`) matches deny list
2. **Remote canonical skills**: Auto-allowed (experimental feature)
3. **Allow rules**: Proceed if explicit allow matches
4. **Safe properties check**: Auto-allow if no allowed-tools, model override, hooks, or paths
5. **Ask user**: Otherwise prompt for permission with suggestions

"This means simple informational skills run without any permission dialog, while skills that gain Bash access or override the model must be explicitly approved."

## Live Reloading

The `skillChangeDetector` module watches directories with chokidar. "When any `SKILL.md` changes, it debounces (300 ms), fires `ConfigChange` hooks, then clears all memoization caches." On Bun, stat-polling replaces FSWatcher due to a deadlock bug.

## Key Takeaways

- Skills live at `<skill-name>/SKILL.md`; directory name becomes slash command
- Four sources: **managed -> user -> project -> bundled** (priority order)
- Two execution modes: **inline** (same context) or **forked** (isolated sub-agent)
- MCP skills **never execute shell injection** (hard security boundary)
- `paths` frontmatter makes skills **conditional** (invisible until matching file opened)
- Skill listing capped at **1% of context window**
- Bundled skill descriptions never truncated; custom skills may be shortened
- Files watched live; no restart needed
- Simple skills auto-approved; powerful skills need explicit allow rules

## Legacy Support

The older `/commands/` directory (`.claude/commands/`) still works with both single `.md` files and `skill-name/SKILL.md` format, but new work should use `.claude/skills/`.

## Quiz Section

Six multiple-choice questions test understanding of:
- Q1: Skill source priority (user vs. project)
- Q2: Fork context for isolated execution
- Q3: MCP skill shell injection blocking
- Q4: Conditional activation via paths
- Q5: Implicit argument appending
- Q6: Bundled skill description preservation

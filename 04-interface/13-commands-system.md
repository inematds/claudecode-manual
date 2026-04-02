# Lesson 13: The Commands System

## Overview

How slash commands function within Claude Code, including their types, registration pipeline, input processing, and REPL integration.

## Three Command Types

**Local Commands** -- Pure TypeScript functions, synchronous. Examples: `/clear`, `/compact`, `/cost`.

**Local JSX Commands** -- Render React/Ink components. Examples: `/help`, `/model`, `/config`, `/memory`.

**Prompt Commands** -- Expand to text content blocks sent to the model. Examples: `/commit`, `/review`, `/init`.

## Registration Pipeline

```
Static COMMANDS() + Skills dirs + Plugins + Workflows
-> loadAllCommands(cwd)
-> filter(availability + isEnabled)
-> getCommands(cwd)
```

### Four Skill Sources
1. **skillDirCommands**: `.claude/skills/`
2. **pluginSkills**: Installed plugins
3. **bundledSkills**: Compiled into binary
4. **builtinPluginSkills**: Always-enabled built-in plugins

### Two Independent Gates
- **availability**: Static auth-provider check (`claude-ai` or `console`)
- **isEnabled()**: Runtime feature-flag check

Neither is memoized -- auth changes take effect immediately.

## Shell Command Substitution in Prompts

```typescript
const PROMPT = `
- Current git status: !` + "`" + `git status` + "`" + `
- Current git diff: !` + "`" + `git diff HEAD` + "`" + `
`
```

## Bridge & Remote Mode

- `local-jsx`: Always blocked (renders terminal UI)
- `prompt`: Always safe (expands to text)
- `local`: Must be explicitly listed in `BRIDGE_SAFE_COMMANDS`

## Key Takeaways

- Three execution types: `local`, `local-jsx`, `prompt`
- Lazy loading via `load: () => import('./cmd.js')`
- Two independent gates control visibility
- Shell substitution embeds live output into prompts
- Internal-only commands dead-code-eliminated by Bun

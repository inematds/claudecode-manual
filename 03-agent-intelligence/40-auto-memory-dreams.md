# Lesson 40: Auto-Memory and the Dream System

## Overview

Claude Code operates two background processes: extraction at query-end and consolidation across sessions.

**Source files:** `services/extractMemories/extractMemories.ts`, `services/extractMemories/prompts.ts`, `services/autoDream/autoDream.ts`, `services/autoDream/config.ts`, `services/autoDream/consolidationLock.ts`, `services/autoDream/consolidationPrompt.ts`, `memdir/memdir.ts`, `memdir/memoryTypes.ts`, `memdir/paths.ts`, `memdir/memoryScan.ts`, `memdir/findRelevantMemories.ts`, `memdir/memoryAge.ts`

**Layer 1 -- Per-turn:** Extract Memories reviews new messages after each response and writes topic files.
**Layer 2 -- Across sessions:** Auto Dream merges, prunes, and re-indexes after 24 hours + 5 sessions.
**At query time:** Relevant Recall uses Sonnet to select up to 5 matching topic files.

## The Memory Directory

Resolution order:
1. `CLAUDE_COWORK_MEMORY_PATH_OVERRIDE` (env var)
2. `getSettingsForSource('localSettings').autoMemoryDirectory` (settings.json)
3. `~/.claude/projects/<sanitized-git-root>/memory/` (default)

**Directory structure:**
```
~/.claude/projects/<slug>/memory/
  MEMORY.md            # index -- always injected into system prompt
  user_role.md         # topic files
  feedback_testing.md
  project_auth.md
  .consolidate-lock    # mtime == lastConsolidatedAt
  logs/                # KAIROS/assistant-mode only
    2026/03/2026-03-31.md
```

## The Four Memory Types

**type: user** -- User Profile. Role, goals, knowledge level, communication preferences.
**type: feedback** -- Behavioral Guidance. Corrections and confirmations. Stores rule + Why + How to apply.
**type: project** -- Project Context. Ongoing work, deadlines, incidents. Decays fast.
**type: reference** -- External Pointers. Where to find information.

## Layer 1 -- Extract Memories

At query end, `handleStopHooks` fires `executeExtractMemories()`. Uses forked agent pattern sharing parent's prompt cache.

**Closure-scoped state:**
```javascript
let lastMemoryMessageUuid: string | undefined  // cursor
let inProgress: boolean                          // overlap guard
let pendingContext: ...                           // stash for trailing run
let turnsSinceLastExtraction: number             // throttle counter
```

**Mutual exclusion with main agent:**
```javascript
if (hasMemoryWritesSince(messages, lastMemoryMessageUuid)) {
  logEvent('tengu_extract_memories_skipped_direct_write', { message_count })
  return
}
```

**Tool permissions:**
```javascript
if (tool.name === FILE_READ_TOOL_NAME || GREP || GLOB) return allow()
if (tool.name === BASH_TOOL_NAME && tool.isReadOnly(parsed.data)) return allow()
if ((EDIT || WRITE) && isAutoMemPath(input.file_path)) return allow()
return denyAutoMemTool(tool, reason)
```

**Two-turn extraction strategy:**
"Turn 1 -- issue all Read calls in parallel. Turn 2 -- issue all Write/Edit calls in parallel."

## Layer 2 -- Auto Dream (Consolidation)

Fires at query-loop end via `executeAutoDream()` only when three gates pass:

**Gate 1 -- Time Gate**: Hours since `lastConsolidatedAt` >= `minHours` (default: 24)
**Gate 2 -- Session Gate**: Transcript files >= `minSessions` (default: 5)
**Gate 3 -- Lock Gate**: No other process mid-consolidation

**The lock file:** `.consolidate-lock` body holds holder's PID; **mtime IS `lastConsolidatedAt`**. Reading timestamp costs one `stat()`.

**Consolidation four phases:**
1. **Orient** -- ls memory dir, read MEMORY.md, skim topic files
2. **Gather recent signal** -- Check daily logs, grep transcripts narrowly
3. **Consolidate** -- Write/update topic files, merge duplicates, convert relative dates
4. **Prune and index** -- Update MEMORY.md, keep under 200 lines/25 KB

## Recall -- findRelevantMemories

At query time, Sonnet side-query selects up to five relevant topic files.

**Notable exception:** If model actively using tool (e.g., `mcp__X__spawn` in `recentTools`), that tool's reference documentation memory is suppressed.

**Staleness signals:**
"This memory is 47 days old. Memories are point-in-time observations, not live state."

## Team Memory (TEAMMEM flag)

Scope rules per type:
- **user** -- always private
- **feedback** -- default private; team only for project-wide conventions
- **project** -- strongly bias toward team
- **reference** -- usually team

## KAIROS / Assistant Mode

Long-lived sessions shift to append-only daily logs. Agent writes to `logs/YYYY/MM/YYYY-MM-DD.md`. Separate nightly `/dream` skill distills logs into topic files.

## Cache Efficiency

Forked agent shares parent's prompt cache. Tool list must match main agent's for cache sharing -- tools part of cache key.

## Key Takeaways

- Extraction runs at query-turn end as forked sub-agent sharing prompt cache
- Lock file's mtime IS `lastConsolidatedAt` timestamp -- one `stat()` read
- Four-type taxonomy keeps memory free of derivable content
- Recall selective: Sonnet picks up to 5 relevant files, suppressing reference docs for tools in active use
- "Before recommending from memory" section header outperformed abstract variants in evals
- Team memory scopes enforced via prompt guidance, not filesystem permissions

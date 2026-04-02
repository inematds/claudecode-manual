# Lesson 08: The Memory System

## Overview

Claude Code implements persistent memory across three distinct layers, each solving different persistence scopes with shared file formats but different lifetimes and audiences.

## Three Memory Layers

**Layer 1: Auto Memory**
- Location: `~/.claude/projects/<slug>/memory/`
- Scope: Persistent facts about individual users
- Lifetime: Survives across all future sessions

**Layer 2: Session Memory**
- Location: `~/.claude/session-memory/<uuid>.md`
- Scope: In-session notes updated during conversations
- Purpose: Powers context compaction
- Lifetime: Single session

**Layer 3: Team Memory**
- Location: `.../memory/team/`
- Scope: Shared memories synced server-side per GitHub repository
- Sync: `.../memory/team/ <-> /api/claude_code/team_memory`

## Auto Memory -- The Core Layer

### MEMORY.md Structure
- Maximum: 200 lines / 25,000 bytes
- Functions as pointer list only, not content storage
- Contains no frontmatter

### Topic Files Format
```yaml
---
name: Feedback -- No Mock Database
description: Integration tests must hit real database, never mocks
type: feedback
---
Content with Why and How sections.
```

### Memory Type Taxonomy

**user** -- Role, goals, expertise level. Always private.
**feedback** -- Corrections AND confirmations. Includes Why reasoning.
**project** -- Ongoing work, goals, incidents. Dates must be absolute.
**reference** -- External system pointers (Linear, Grafana, Slack).

### Exclusions -- What NOT to Save
- Code patterns, conventions, architecture, file paths
- Git history and recent changes
- Debugging solutions or fix recipes
- Anything already in CLAUDE.md files
- Ephemeral task details

### Path Resolution and Security
1. `CLAUDE_COWORK_MEMORY_PATH_OVERRIDE` environment variable
2. `autoMemoryDirectory` in `settings.json` (trusted sources only)
3. Default: `<memoryBase>/projects/<sanitized-git-root>/memory/`

**Security:** Project settings intentionally excluded from override to prevent malicious repos from redirecting to `~/.ssh`.

## The Extraction Pipeline

Memories extracted AFTER complete query loops, never during conversations. Mutual exclusion with main agent via `hasMemoryWritesSince()`.

### Relevance Recall -- The Selector Model
Lightweight Sonnet call selects up to 5 relevant files. Reads only first 30 lines (frontmatter range).

### Staleness Detection
"This memory is N days old. Memories are point-in-time observations, not live state."

## Session Memory

### Fixed Section Structure
```
# Session Title
# Current State
# Task specification
# Files and Functions
# Workflow
# Errors & Corrections
# Codebase and System Documentation
# Learnings
# Key results
# Worklog
```

### Extraction Triggers
- `minimumMessageTokensToInit`: 10,000
- `minimumTokensBetweenUpdate`: 5,000
- `toolCallsBetweenUpdates`: 3
- Token budget: 12,000 total, 2,000 per section

## Team Memory Sync

Server-synced subdirectory, gated behind `TEAMMEM` build flag requiring OAuth.

### Sync Contract
- Pull: server wins per-key
- Push: delta upload using SHA-256 hash comparison
- File deletions don't propagate
- PUT body batched at 200KB max

### Secret Scanning -- Client-Side Guard
35+ secret patterns from gitleaks ruleset. Detection blocks push -- "secrets never reach the server."

### Private vs Team Scope Routing
- **user** -- always private
- **feedback** -- private by default; team only for project-wide conventions
- **project** -- strongly bias toward team
- **reference** -- usually team

## Feature Flags and Disable Mechanisms

- `CLAUDE_CODE_DISABLE_AUTO_MEMORY=1` -- Full disable
- `CLAUDE_CODE_SIMPLE=1` -- Bare mode
- `autoMemoryEnabled: false` -- Settings override
- `tengu_passport_quail` -- Extract-memories feature flag

## Key Takeaways

- Memory is three-layer system: Auto, Session, Team
- MEMORY.md index always in context; topic files loaded on-demand by Sonnet selector
- Four-type taxonomy with validation at parse time
- Extraction agent runs after query loops, never during
- Feedback memories should capture both corrections AND confirmations
- Team memory uses delta push with SHA-256 checksums
- 35-rule secret scanner runs client-side before team memory push
- Memory records are point-in-time snapshots; verify before recommending

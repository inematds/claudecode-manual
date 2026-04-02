# Lesson 30: The Notification System

## Overview

Claude Code implements **two completely separate notification pipelines**:

1. **In-REPL Toast Queue**: Status messages in the terminal UI footer
2. **OS/Terminal Notifier**: Native desktop notifications (iTerm2, Kitty, Ghostty, bell)

## Pipeline 1: In-REPL Toast Queue

### Priority Ordering

| Priority | Rank | Behavior |
|----------|------|----------|
| immediate | 0 | Preempts current display |
| high | 1 | Queued, wins over medium/low |
| medium | 2 | Beats low |
| low | 3 | Shown last |

### Key Features
- `fold` field merges same-key notifications like `Array.reduce()`
- Module-level `currentTimeoutId` enables synchronous cancellation
- `immediate` notifications bump displaced items back to queue

### Hook Catalog (14 specialized hooks)

| Hook | Key | Priority |
|------|-----|----------|
| Rate limit warning | `rate-limit-warning` | high |
| Limit reached | `limit-reached` | immediate |
| Model deprecation | `model-deprecation-warning` | high |
| LSP error | `lsp-error-{source}` | medium |
| Fast mode | `fast-mode` | high |
| Model migration | `model-migration` | high |
| MCP connectivity | `mcp-connectivity` | medium |
| IDE status | `ide-status` | medium |
| Settings errors | `settings-error-*` | high |

## Pipeline 2: OS/Terminal Notifier

### Channel Routing

- `iterm2`: OSC 9 sequence
- `kitty`: Three-step OSC 99 sequence
- `ghostty`: Single OSC sequence
- `terminal_bell`: BEL (intentionally unwrapped for tmux)
- `notifications_disabled`: No-op

BEL stays unwrapped so tmux's `bell-action` triggers natively.

### Progress Reporting
Supported by ConEmu, Ghostty 1.2.0+, and iTerm2 3.6.6+ via OSC 9;4 task-progress sequences.

## Background Task Notification Collapsing

Only successful completions collapse. Failed and killed tasks remain individual. Verbose mode bypasses collapsing.

## MCP Channel Notifications (Kairos)

6-layer security gate: Capability -> Runtime flag -> Auth -> Org policy -> Session opt-in -> Allowlist.

## Key Takeaways

- Two pipelines: toast queue for interactive status, OS notifier for backgrounded sessions
- Queue uses priority-based dequeuing, not FIFO
- `immediate` notifications preempt display
- BEL stays unwrapped for tmux compatibility
- Background bash completions collapse at message-list level
- MCP channel notifications pass through 6-layer security

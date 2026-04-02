# Lesson 22: Teams & Swarms

## Overview

A **swarm** represents a named team of Claude agents sharing configuration, tasks, and file-based mailbox infrastructure. One agent functions as the **team lead** creating the team, spawning members, assigning work, and shutting down cleanly. All others are **teammates**. This feature requires `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`.

## TeamCreateTool: Creating a Team

Five synchronous steps:

1. **Uniqueness check**: If name exists, `generateWordSlug()` substitutes a new random name
2. **Write team config**: `TeamFile` JSON writes to `~/.claude/teams/<sanitized-name>/config.json`
3. **Create task list**: Fresh task numbering starts at 1
4. **Update AppState**: Sets `appState.teamContext`
5. **Register for session cleanup**: Shutdown hook cleans up orphaned directories

## File Structure: On-Disk Layout

```
~/.claude/
  teams/
    my-team/
      config.json           <- TeamFile
      permissions/
        pending/
        resolved/
  tasks/
    my-team/
      0001.json
      0002.json
```

### Agent ID Format

Agent IDs follow the pattern `agentName@teamName`, e.g. `researcher@my-team`.

## Spawn Backends: Three Ways to Run a Teammate

| Backend | When Selected | Kill method |
|---------|---------------|-------------|
| tmux | Inside tmux (highest priority) | `kill-pane -t <paneId>` |
| iterm2 | In iTerm2 with `it2` CLI available | `it2 session close -f -s <id>` |
| in-process | Non-interactive or no pane backend | Abort via AbortController |

### Backend Detection

Detection uses environment variables captured at **module load time** -- `TMUX`, `TMUX_PANE`, `TERM_PROGRAM`, and `ITERM_SESSION_ID`.

The `it2` availability check runs `it2 session list` (not `it2 --version`) because `--version` exits 0 even when the iTerm2 Python API is disabled.

### In-Process Backend

In-process teammates run inside the leader's Node.js process with fully isolated identity context via `AsyncLocalStorage`. They share the API client and MCP connections but get independent `AbortController` (not linked to the leader's).

## Mailbox Messaging

Every agent communicates through a **file-based mailbox**. Messages are JSON files written to `~/.claude/teams/<team>/inbox/<agent-name>/`.

### Message Types

| Type | Direction | Purpose |
|------|-----------|---------|
| plain text | any -> any | Task updates, questions, results |
| shutdown_request | lead -> teammate | Graceful shutdown signal |
| idle notification | teammate -> lead | System-generated after every turn end |
| permission_request | worker -> lead | Worker needs UI approval |
| permission_response | lead -> worker | Approval/denial of tool use |
| mode_set_request | lead -> teammate | Change teammate's permission mode |

## Permission Sync

Workers can escalate permission requests to the team lead via the mailbox. The leader sees the request in its UI and approves or denies.

## Full Lifecycle

### Phase 1 -- Setup
* Lead calls `TeamCreate`, creates tasks, spawns teammates

### Phase 2 -- Parallel Work
* Teammates claim tasks, send progress, work in parallel

### Phase 3 -- Shutdown
* Lead sends `shutdown_request`, teammates approve and exit

### Phase 4 -- Cleanup Guard
* If session ends without `TeamDelete`, shutdown hook cleans up

## Key Takeaways

1. **Team = TaskList.** TeamCreate creates a matching task directory
2. **Three backends, one interface.** tmux, iTerm2, and in-process
3. **Detection is captured at module load.** Environment variables read at import time
4. **All messaging is file-based.** Exception: leader permission bridge uses module-level setter
5. **Idle is not dead.** `isActive: false` means turn ended; teammate wakes on new message
6. **TeamDelete blocks on active members.** Always shut teammates down first
7. **Session cleanup is a safety net.** Orphaned panes and directories cleaned on graceful shutdown

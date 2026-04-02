# Lesson 47: ULTRAPLAN

ULTRAPLAN is Claude Code's remote planning mode. When you type `/ultraplan` or embed the word "ultraplan" anywhere in a prompt, the CLI spawns a full Claude Code session in the cloud (CCR), running on the most powerful available model (Opus), with a 30-minute window to iterate a plan with you via the browser. Your local terminal stays free the entire time.

**Source files:** `commands/ultraplan.tsx`, `utils/ultraplan/ccrSession.ts`, `utils/ultraplan/keyword.ts`, `utils/teleport.tsx`, `tasks/RemoteAgentTask/RemoteAgentTask.tsx`

## Four Phases

1. **Trigger Detection** - Keyword scanner finds "ultraplan" in freeform input or slash command routes it
2. **CCR Session Launch** - `teleportToRemote()` creates remote session, uploads git bundle, returns session ID
3. **Long-Poll** - `pollForApprovedExitPlanMode()` polls event stream every 3s for up to 30 min
4. **Plan Delivery** - Plan lands locally via UltraplanChoiceDialog (teleport) or stays in CCR (remote-execute)

## Keyword Trigger: Smart Disambiguation

The word "ultraplan" fires from freeform text unless context makes clear it's not a directive. Filtered OUT:
- Slash-command input (`/ultraplan` routed to command handler)
- Inside paired delimiters (backticks, quotes, tags, braces, brackets, parens)
- Path/identifier context (preceded/followed by `/`, `\`, `-`)
- Followed by `?` (questions about the feature)
- Followed by `.` + word char (file extensions)

## Launch Sequence

The "detached" pattern returns a user-facing message immediately. All async work runs in a `void` closure. The `ultraplanLaunching` flag is set synchronously before any async call to close the double-launch race window.

## Polling Engine

Cursor-based pagination fetches up to 50 pages per tick. ExitPlanModeScanner classifies: approved / teleport / rejected / pending / terminated. Tolerates up to 5 consecutive network failures.

### Phase State Machine

- running -> needs_input (quiet idle)
- needs_input -> running (user replies in browser)
- running -> plan_ready (ExitPlanMode tool_use seen)
- plan_ready -> running (user rejects)
- plan_ready -> approved (remote execute)
- plan_ready -> teleport (back to terminal)
- running -> terminated / timeout

## Plan Delivery: Two Paths

**Path A - Remote execution:** User approves in CCR browser. Local CLI marks task completed, enqueues notification.

**Path B - Teleport:** User clicks "back to terminal." Browser sends `is_error=true` tool_result with sentinel `__ULTRAPLAN_TELEPORT_LOCAL__`. Local CLI shows UltraplanChoiceDialog.

## Key Takeaways

- Offloads planning to remote CCR session (plan-mode only, Opus, 30-min window)
- Keyword scanner filters quoted contexts, file paths, slash commands, and question marks
- Sessions survive `--resume` via sidecar metadata files
- Model selection via GrowthBook feature flag, not hardcoded

# Lesson 21: Coordinator Mode

## Core Concept

Coordinator mode is an operating mode where Claude acts exclusively as a dispatcher, never executing tools directly. All side-effecting operations are delegated to worker agents. This separation is controlled by `CLAUDE_CODE_COORDINATOR_MODE=1`.

## Activation and Session Matching

Two functions manage coordinator mode:

**`isCoordinatorMode()`**: Reads the environment variable at runtime with no caching, allowing mode changes in resumed sessions.

**`matchSessionMode()`**: Ensures resumed sessions restore the correct mode by mutating `process.env.CLAUDE_CODE_COORDINATOR_MODE`.

## Tool Restrictions

The coordinator has access to exactly four tools via `COORDINATOR_MODE_ALLOWED_TOOLS`:

- **Agent**: Spawns new workers
- **SendMessage**: Continues existing workers
- **TaskStop**: Terminates running workers
- **SyntheticOutput**: Internal output signaling

Workers receive the full `ASYNC_AGENT_ALLOWED_TOOLS` set (Bash, Read, Edit, Glob, Grep, WebSearch, Skill, Notebook, Worktree) plus MCP tools.

## Worker Context: `getCoordinatorUserContext()`

Three components:
1. **Tool enumeration**: Computed at runtime
2. **MCP server advertisement**: Connected server names appended
3. **Scratchpad injection**: Shared directory path for cross-worker knowledge

**Simple mode** (`CLAUDE_CODE_SIMPLE=1`) restricts workers to Bash, Read, and Edit only.

## The Coordinator System Prompt

### Four-Phase Task Workflow

| Phase | Who | Purpose |
|-------|-----|---------|
| Research | Workers (parallel) | Investigate codebase |
| Synthesis | **Coordinator** | Craft specific implementation specs with file paths and line numbers |
| Implementation | Workers | Make targeted changes, run tests, commit |
| Verification | Workers | Prove code works independently |

Critical point: "Never write 'based on your findings.'" The coordinator must synthesize findings into specific file paths and line numbers.

### Parallelism Rules

- **Read-only tasks**: Run freely in parallel
- **Write-heavy tasks**: One at a time per overlapping file set
- **Verification**: Can run alongside implementation on different areas

### Continue vs. Spawn Fresh Decision Logic

| Situation | Action | Why |
|-----------|--------|-----|
| Research explored exact files for editing | Continue | Worker has files in context |
| Research broad, implementation narrow | Spawn fresh | Avoid exploration noise |
| Correcting failure or extending work | Continue | Worker has error context |
| Verifying different worker's code | Spawn fresh | Verifier needs fresh eyes |
| First attempt used wrong approach | Spawn fresh | Wrong-approach context pollutes |
| Unrelated task | Spawn fresh | No useful context to reuse |

### Worker Prompt Guidelines

Bad prompts delegate understanding:
- "Fix the bug we discussed"
- "Based on your findings, implement the fix"

Good prompts are fully self-contained with specific details:
- "Fix null pointer in src/auth/validate.ts:42. The user field is undefined when Session.expired is true. Add null check before user.id access -- return 401 with 'Session expired' if null. Run tests, commit, report hash."

## Coordinator vs. Single-Agent Mode

| Dimension | Single-Agent | Coordinator |
|-----------|--------------|-------------|
| Role | Does work itself | Dispatches and synthesizes |
| Tool count | Full set | 4 tools only |
| Filesystem access | Direct read/write | None -- must delegate |
| Parallelism | Sequential tool calls | Multiple concurrent workers |

## The Scratchpad: Cross-Worker Knowledge

Workers can read and write without permission prompts, creating a shared workspace for durable findings.

## Key Takeaways

1. The coordinator is a pure dispatcher with exactly 4 tools
2. Activation is a single env var read live with no caching
3. Four-phase workflow: Research -> Synthesis -> Implementation -> Verification
4. Synthesis is the coordinator's most important job
5. Continue vs. spawn fresh depends on context overlap
6. `matchSessionMode()` ensures resumed sessions restore correct mode
7. Scratchpad is a permission-free shared directory for cross-worker knowledge

# Lesson 17 — The Bash Tool — Source Deep Dive

## Overview

The Bash Tool serves as Claude Code's operating system gateway, processing commands through "seven distinct layers" before subprocess spawning and three more during output return.

## Architecture

The tool processes commands through a documented flow:
1. Input validation (sleep-pattern guard)
2. Permission checking
3. Command execution via shell
4. Output streaming and interpretation
5. Result persistence and mapping

Five key source directories implement this:
- **tools/BashTool/** — tool definition, permissions, UI classification
- **tools/BashTool/bash*.ts** — security validators and permission rules
- **utils/bash/** — AST parsing, command splitting, ShellSnapshot
- **utils/shell/** — BashProvider, output limits, shell detection
- **utils/sandbox/** — filesystem and network policy enforcement

## Input/Output Schema

The tool maintains two schemas: public (exposed to Claude) and full (internal). The `_simulatedSedEdit` field stays hidden from the model to prevent bypassing permission dialogs.

**Public schema omits:**
- `_simulatedSedEdit` — prevents arbitrary file writes paired with harmless commands
- `run_in_background` (conditionally)

**Output schema includes:**
- stdout/stderr (merged at file-descriptor level via `2>&1`)
- interrupted, backgroundTaskId, isImage flags
- returnCodeInterpretation for semantic exit codes
- persistedOutputPath for outputs >30KB

Critical note: "The shell provider uses `2>&1` — stderr is merged into stdout at the file-descriptor level. The `stderr` field in the output schema is therefore always empty for the normal execution path."

## Shell Snapshot System

Each session captures the interactive shell environment once into `~/.claude/shell-snapshots/snapshot-{shell}-{timestamp}-{random}.sh`.

**Snapshot captures:**
1. User aliases and functions (filtered for completion functions)
2. Shell options (setopt/shopt)
3. PATH export
4. Embedded ripgrep shim (if system `rg` absent)
5. find/grep shims (in native builds)

**Graceful degradation:** If snapshot creation fails, the promise resolves to `undefined`. Commands then spawn with `-l` (login shell) flag, losing aliases/functions but remaining functional.

TOCTOU awareness: Before each command, `access(snapshotFilePath)` verifies file existence. If OS cleaned up `/tmp`, the fallback re-engages.

## Command Building Pipeline

`buildExecCommand()` assembles the full shell string through six stages:

1. **Windows null redirect rewrite** — `2>nul` → `2>/dev/null`
2. **Stdin redirect decision** — append `< /dev/null` conditionally
3. **Shell quoting** — wrap in single quotes for eval
4. **Pipe rearrangement** — move `< /dev/null` to first segment if pipes exist
5. **Assembly** — source snapshot, session env, disable extglob, eval command
6. **Prefix formatting** — apply CLAUDE_CODE_SHELL_PREFIX if set

**Why eval quoting matters:** Bash parses the entire line before executing. Aliases from sourced snapshots only become available after `source` runs. Using `eval 'command'` creates a second parse pass where aliases are live.

**Pipe rearrangement fix:** Without special handling, `eval 'rg foo | wc -l' < /dev/null` causes `wc` to read `/dev/null` while `rg` blocks. The fix injects `< /dev/null` between first command and pipe.

**Extglob disabling:** After snapshot sourcing, extended glob patterns are explicitly disabled. Extended globs like `!(pattern)` can be triggered by filenames, creating post-validation expansion attacks.

## The 23 Security Validators

Each validator in `bashCommandIsSafe()` returns one of three results:
- **allow** — early-allow, skip remaining validators
- **passthrough** — no opinion, continue chain
- **ask** — trigger permission dialog

Validators run in order; first non-passthrough result wins.

### Complete Validator List

| # | Name | Purpose |
|---|------|---------|
| 1 | INCOMPLETE_COMMANDS | Detects fragments (tabs, flags, operators) |
| 2 | JQ_SYSTEM_FUNCTION | Blocks jq intrinsics (env, path, builtins, debug) |
| 3 | JQ_FILE_ARGUMENTS | Blocks --args, --jsonargs, --rawfile, --slurpfile, --arg patterns |
| 4 | OBFUSCATED_FLAGS | Detects encoded/spaced flags differing from shell interpretation |
| 5 | SHELL_METACHARACTERS | Unquoted ;, &&, \|\|, \| — compound command detection |
| 6 | DANGEROUS_VARIABLES | $BASH_ENV, $ENV, $CDPATH, $IFS references |
| 7 | NEWLINES | Unquoted newlines (command separators) |
| 8 | DANGEROUS_PATTERNS — substitution | Unquoted $(), ${}, $[], backticks, <(), >(), =() |
| 9 | DANGEROUS_PATTERNS — input redirection | Unquoted < (except < /dev/null) |
| 10 | DANGEROUS_PATTERNS — output redirection | Unquoted > or >> to non-null targets |
| 11 | IFS_INJECTION | IFS= assignment or unquoted $IFS |
| 12 | GIT_COMMIT_SUBSTITUTION | Special early-allow for `git commit -m "msg"` |
| 13 | PROC_ENVIRON_ACCESS | /proc/*/environ or /proc/self/environ references |
| 14 | MALFORMED_TOKEN_INJECTION | Commands with unbalanced quotes causing mis-tokenization |
| 15 | BACKSLASH_ESCAPED_WHITESPACE | tr\ace pattern changing word splitting |
| 16 | BRACE_EXPANSION | Unquoted {a,b,c} or {1..10} |
| 17 | CONTROL_CHARACTERS | Raw ASCII control chars (\x00–\x1f except tab/newline) |
| 18 | UNICODE_WHITESPACE | Non-ASCII whitespace (NBSP, em-space, zero-width space) |
| 19 | MID_WORD_HASH | # adjacent to closing quote ('x'#) |
| 20 | ZSH_DANGEROUS_COMMANDS | zmodload, emulate, sysopen, file builtins |
| 21 | BACKSLASH_ESCAPED_OPERATORS | Backslash before operators (\;, \|, \&) |
| 22 | COMMENT_QUOTE_DESYNC | Quote stripping places # at word boundary |
| 23 | QUOTED_NEWLINE | Literal newline inside quoted string |

### Safe Heredoc Early-Allow

The pattern `cmd $(cat <<'DELIM'\n...\nDELIM\n)` gets early-allow via `isSafeHeredoc()`:

**Safety conditions:**
1. Delimiter must be single-quoted (`<<'EOF'`) or backslash-escaped
2. Closing delimiter on its own line
3. Substitution in argument position (non-whitespace prefix before `$(`)
4. Remainder contains only safe characters (no shell metacharacters)
5. No nested heredocs (would create ambiguity)

### ValidationContext Views

Each validator receives five pre-computed command views:
- **originalCommand** — verbatim from Claude
- **baseCommand** — first token after env vars
- **unquotedContent** — double-quotes stripped
- **fullyUnquotedContent** — both quote types stripped
- **unquotedKeepQuoteChars** — content stripped but delimiters kept (for mid-word-hash detection)

### Subcommand Validation

Compound commands are split and validated individually with a cap of 50 subcommands. Beyond that, the system returns `ask` as safe default. This prevents "CPU-starvation DoS from exponentially growing subcommand arrays" in complex compound commands.

## Permission Plumbing

`checkPermissions()` calls `bashToolHasPermission()` which layers independent checks:

1. **Permission mode** — read-only mode check
2. **Security validators** — 23-validator chain
3. **Read-only constraints** — filesystem write restrictions
4. **Path constraints** — allowlist/denylist checking
5. **Sed constraints** — in-place edit detection
6. **Command operator permissions** — per-subcommand rule matching

### Environment Variable Stripping

Before rule matching, `stripSafeWrappers()` performs two-phase stripping:

**Phase 1:** Strip leading env vars where variable name is in `SAFE_ENV_VARS` (build/locale/display vars). Notably excludes PATH, LD_PRELOAD, PYTHONPATH.

**Phase 2:** Strip wrapper commands (timeout, time, nice, nohup) with strict flag parsing.

Both phases run in fixed-point loop, handling interleaved patterns.

### Rule Matching Modes

| Mode | Format | Example | Matches |
|------|--------|---------|---------|
| Exact | full command | Bash(git status) | Only exact command |
| Prefix | cmd:* | Bash(npm run:*) | npm run + anything |
| Wildcard | pattern with * | Bash(git * --force) | Matching glob |

Auto-generated prefix rules use 2-word extraction. For `git commit -m "fix typo"`, suggestion is `Bash(git commit:*)`. Bare shell names excluded entirely.

## Sandboxing

`shouldUseSandbox()` makes four-way decision:

1. Sandboxing enabled in system
2. Explicit user override (`dangerouslyDisableSandbox`) allowed by policy
3. Non-empty command exists
4. Command not in user-configured excluded list

SandboxManager controls:
- **Filesystem read** — deny-only list (e.g., ~/.ssh)
- **Filesystem write** — allow-only list (e.g., project dir, $TMPDIR)
- **Network** — optional allowedHosts/deniedHosts

**Critical note:** "sandbox.excludedCommands is a convenience feature — it lets users run tools like docker or bazel without sandboxing because those tools need direct process access. But it is NOT a security control."

Excluded commands checked via fixed-point closure of stripped forms, ensuring consistency with permission-check stripping logic.

## Background Execution

Three paths produce background execution:

1. **run_in_background: true** — explicit model-initiated
2. **User Ctrl+B** — manual background (sets backgroundedByUser: true)
3. **Auto-background** — after 15 seconds in assistant mode (assistantAutoBackgrounded: true)

Backgrounded commands return `backgroundTaskId` and stream output to `getTaskOutputPath(taskId)`.

### Sleep Blocker Pattern

Standalone `sleep N` where N >= 2 blocked by `validateInput()` when Monitor tool enabled. Float-duration sleeps (0.5) exempt as legitimate rate-limiting.

## Output Handling

### Size Limits

- **BASH_MAX_OUTPUT_DEFAULT** — 30,000 chars (configurable)
- **BASH_MAX_OUTPUT_UPPER_LIMIT** — 150,000 chars (hard cap)

Outputs exceeding inline limit persist to file in tool-results directory. Full output capped at 64MB via truncation.

### Image Detection

If `stdout` starts with base64-encoded PNG/JPEG header, `isImage: true` set and output wrapped in image content block. Large images resized before sending.

### Semantic Exit Code Interpretation

`interpretCommandResult()` maps non-zero codes to human-readable notes using well-known codes per command. Example: grep returning 1 means "no matches" (not error); diff returning 1 means "files differ" (not error).

### Claude Code Hints Protocol

CLIs setting `CLAUDECODE=1` emit `<claude-code-hint />` tags to stderr (merged into stdout). Tool records hints, strips them before model sees output — "zero-token side channel." Subagent outputs also stripped so hints don't escape agent boundary.

## UI Classification

Commands classified to determine default collapse behavior:

**Search/read commands:**
```
BASH_SEARCH_COMMANDS = {find, grep, rg, ag, ack, ...}
BASH_READ_COMMANDS = {cat, head, tail, jq, awk, ...}
BASH_LIST_COMMANDS = {ls, tree, du}
```

**For pipelines:** All parts must be search/read for whole pipeline to be collapsible. Single non-search command makes pipeline non-collapsible.

**Semantic-neutral commands** (echo, printf, true, false, :) skipped in classification.

**BASH_SILENT_COMMANDS** (mv, cp, rm, mkdir) show "Done" instead of "(No output)".

**Sed special case:** Commands matching `sed -i 's/.../.../' file` render as file-edit operations using FileEdit display name, providing file-diff-style permission dialog.

## Key Takeaways

- Shell snapshot runs once per session capturing aliases, functions, options, PATH without expensive login shell per command
- Command building pipeline has six stages with specific security/correctness reasons
- 23 validators form defense-in-depth against injection at character encoding, shell syntax, and Zsh-specific levels
- Safe heredoc path is positive assertion with stricter conditions than individual validators
- Environment variable stripping uses security-sensitive allowlist omitting PATH, LD_PRELOAD, module-path variables
- stderr always merged into stdout via `2>&1`; stderr field always empty for normal commands
- Sandbox exclusions are convenience features, not security controls; permission dialog is actual security boundary
- Background execution has three distinguishable paths (explicit, user-initiated, auto 15s)
- 50-subcommand cap prevents CPU-starvation DoS from exponential subcommand array growth

## Quiz

Six questions test understanding of:
1. Auto-generated prefix rules (answer: 2-word prefix like `Bash(git commit:*)`)
2. Snapshot fallback behavior (answer: login shell with -l flag)
3. ANSI-C quoting detection (answer: validator #8 catches `$'...'`)
4. stderr merging (answer: 2>&1 redirects at file-descriptor level)
5. Compound command sandbox exclusion (answer: subcommand splitting applies)
6. 50-subcommand cap origin (answer: real CPU-starvation DoS incident)
7. unquotedKeepQuoteChars purpose (answer: detects quote-adjacent # in `'x'#`)

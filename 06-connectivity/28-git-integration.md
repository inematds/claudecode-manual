# Git Integration: Zero-Subprocess, Filesystem-First — Complete Lesson Content

## Lesson 28: Git Integration Overview

Claude Code implements git state reading through direct filesystem access rather than spawning subprocesses. The system comprises five core modules handling different aspects of git integration.

## 01 The Design Philosophy: No Subprocesses

The fundamental principle is: "read git's own filesystem state directly" rather than spawning `git` commands. Git's internal formats (`.git/HEAD`, `.git/config`, `.git/packed-refs`, loose ref files) are plain-text and machine-readable.

**Five source files comprise this system:**

1. **gitFilesystem.ts** (`utils/git/gitFilesystem.ts`)
   - Core filesystem reader
   - Functions: `resolveGitDir`, `readGitHead`, `resolveRef`, `GitFileWatcher`
   - Cached public API functions

2. **gitConfigParser.ts** (`utils/git/gitConfigParser.ts`)
   - Hand-written parser for `.git/config` INI format
   - Handles section headers, subsections, quoted values, escapes, inline comments

3. **gitignore.ts** (`utils/git/gitignore.ts`)
   - Checks `.gitignore` status
   - Manages global gitignore at `~/.config/git/ignore`

4. **gitOperationTracking.ts** (`tools/shared/gitOperationTracking.ts`)
   - Shell-agnostic regex detection of git operations
   - Detects commits, pushes, merges, rebases, PR creation

5. **ghAuthStatus.ts** (`utils/github/ghAuthStatus.ts`)
   - Checks gh CLI installation and authentication
   - Uses local keyring/config only, no network requests

## 02 The Config Parser

Git's config format follows INI-like structure with specific parsing rules. The parser in `gitConfigParser.ts` is verified against git's own `config.c` source.

### Three-Level Lookup

Every config value lookup requires three components:

```typescript
export async function parseGitConfigValue(
  gitDir: string,
  section: string,       // e.g. "remote"    — case-insensitive
  subsection: string | null, // e.g. "origin"    — case-sensitive
  key: string,            // e.g. "url"       — case-insensitive
): Promise<string | null>

// In-memory variant — exported for testing
export function parseConfigString(
  config: string,
  section: string,
  subsection: string | null,
  key: string,
): string | null
```

**Correctness detail:** Section names and key names are case-insensitive. Subsection names (the quoted part, e.g., `"origin"` in `[remote "origin"]`) are case-sensitive. The parser normalizes sections and keys to lowercase before matching, with strict equality comparison for subsections.

### Value Parsing: Quotes, Escapes, and Inline Comments

Values support unquoted, partially quoted, or fully quoted formats with backslash escape sequences inside quotes and inline comments (`#` or `;`) outside quotes.

**Character-by-character processing with `inQuote` boolean:**

```
Inside quotes: recognized escape sequences
"hello\\nworld"  →  "hello\\nworld"   // \\n, \\t, \\b, \\\\, \\" recognized
"foo\\xbar"      →  "foobar"         // unknown escapes: backslash silently dropped

Inline comments outside quotes end the value
url = git@github.com:foo/bar.git \# this is a comment
→ url = "git@github.com:foo/bar.git"
```

### Section Header Parsing Deep Dive

The `matchesSectionHeader` function parses lines like `[remote "origin"]` or `[core]`. The section name is read until `]`, whitespace, or `"`. Subsections must be delimited by quotes with only `\\` and `\"` as valid escapes.

**Examples:**

```
// Simple section — no subsection
[core]              → section="core", subsection=null

// Section with subsection
[remote "origin"]   → section="remote", subsection="origin"
[branch "main"]     → section="branch",  subsection="main"

// Case rules
[Remote "ORIGIN"]   → section matches "remote" (lowercased)
                      subsection is "ORIGIN" (case-sensitive, won't match "origin")
```

The parser returns `false` immediately if the found section name doesn't match. For subsection lookup, it requires the opening `"`, reads the name with escape handling, then checks for closing `"` and `]` in strict order.

## 03 Reading Git Filesystem State

### resolveGitDir: Handling Worktrees and Submodules

A critical preprocessing step resolves the actual `.git` directory. In regular repos, `.git` is a directory. In git worktrees or submodules, `.git` is a plain text file containing a `gitdir: <path>` pointer.

```typescript
async function resolveGitDir(startPath?: string): Promise<string | null> {
  const root = findGitRoot(cwd)           // walk up looking for .git
  const gitPath = join(root, '.git')
  const st = await stat(gitPath)

  if (st.isFile()) {
    // Worktree/submodule: .git is a pointer file
    const content = (await readFile(gitPath, 'utf-8')).trim()
    if (content.startsWith('gitdir:')) {
      const rawDir = content.slice('gitdir:'.length).trim()
      return resolve(root, rawDir)   // may be relative path
    }
  }
  return gitPath  // regular repo: .git is a directory
}
```

Results are **memoized per cwd path** in a `Map<string, string | null>`. Since `.git` pointers don't change during a session, this prevents redundant disk reads.

### readGitHead: Parsing HEAD

The `.git/HEAD` file has exactly two formats. The parser handles both and validates all output before returning:

| HEAD content | Meaning | Return type |
|---|---|---|
| `ref: refs/heads/main\n` | On branch "main" | `{ type: 'branch', name: 'main' }` |
| `ref: refs/remotes/...` | Unusual symref (bisect, etc.) | `{ type: 'detached', sha: '...' }` |
| `a1b2c3d4e5...<40 hex chars>` | Detached HEAD (rebase, tag checkout) | `{ type: 'detached', sha: '...' }` |
| Anything else | Tampered or corrupt | `null` |

### resolveRef: Loose Files and packed-refs

To convert a branch name into a commit SHA, `resolveRef` checks two locations in order:

1. **Loose ref file** — e.g. `.git/refs/heads/main`. Contains a 40-char hex SHA on one line. If the file contains `ref: ...` instead, it's a symref — follow it recursively.

2. **packed-refs** — `.git/packed-refs`. Lines of `<sha> <refname>`. Lines starting with `#` or `^` are skipped (peeled annotated tags).

3. **Worktree fallback** — for `git worktree`, shared refs live in `commonDir` (read from `.git/commondir`), not the per-worktree gitDir. The function retries there if both lookups fail.

### Worktrees and commonDir Deep Dive

When you run `git worktree add`, git creates a new working tree whose `.git` is a pointer file like `gitdir: /main/repo/.git/worktrees/feature`. The per-worktree gitDir has its own `HEAD`, but **shared** objects, refs, and config live in the main repo's `.git`.

The `commondir` file inside the per-worktree gitDir contains the path to the main repo's `.git`:

```typescript
export async function getCommonDir(gitDir: string): Promise<string | null> {
  const content = (await readFile(join(gitDir, 'commondir'), 'utf-8')).trim()
  return resolve(gitDir, content)  // may be relative
}
```

Every function that reads shared state (config, refs, packed-refs) checks commonDir and falls back to it. This ensures correct answers even when used inside a worktree.

## 04 GitFileWatcher: Lazy, Cached, Always Fresh

`GitFileWatcher` is the singleton that keeps branch name, HEAD SHA, remote URL, and default branch in memory — recomputing only when underlying files actually change. It uses Node's `fs.watchFile` (inotify/kqueue-backed) rather than polling.

### What Files Are Watched

| File | Why watched | Action on change |
|---|---|---|
| `.git/HEAD` | Branch switches, rebase start/end, detach | Invalidate cache, update branch ref watcher |
| `.git/config` (or commonDir) | Remote URL changes (`git remote set-url`) | Invalidate cache |
| `.git/refs/heads/<branch>` | New commits on current branch | Invalidate cache |

### Branch Switch Handling

When HEAD changes, the watcher must stop watching the old branch's ref file and start watching the new one. This is done in `onHeadChanged()` which calls `watchCurrentBranchRef()`. The update is deferred via `waitForScrollIdle()` so watchFile callbacks don't compete with the event loop mid-render.

### The Cache: Dirty-Bit Invalidation

Each cached value is a `CacheEntry<T>` with a `dirty` flag. The `get(key, compute)` method is the entire public interface:

```typescript
async get<T>(key: string, compute: () => Promise<T>): Promise<T> {
  await this.ensureStarted()
  const existing = this.cache.get(key)
  if (existing && !existing.dirty) return existing.value as T

  // Clear dirty BEFORE async compute — if the file changes again
  // during compute, invalidate() re-sets dirty so we re-read next call
  if (existing) existing.dirty = false

  const value = await compute()

  // Only write back if no new invalidation arrived during compute
  const entry = this.cache.get(key)
  if (entry && !entry.dirty) entry.value = value
  if (!entry) this.cache.set(key, { value, dirty: false, compute })

  return value
}
```

**Race condition design:** The dirty flag is cleared _before_ the async compute begins. If a file-change event fires during async read, `invalidate()` sets dirty to `true` again. The new value from compute is only written back if dirty is still false — meaning no new invalidation snuck in. If it did, the next caller triggers another compute, ensuring correctness.

### Public API

```typescript
// All four return Promises backed by the watcher cache
getCachedBranch()        // → "main" | "HEAD" (detached)
getCachedHead()          // → "a1b2c3..." | "" (no commits yet)
getCachedRemoteUrl()     // → "git@github.com:org/repo.git" | null
getCachedDefaultBranch() // → "main" | "master" (from remote symref)
```

### Computing the Default Branch Deep Dive

`computeDefaultBranch` follows a three-step preference cascade:

1. Read `refs/remotes/origin/HEAD` as a symref — this is what `git clone` sets to point to the remote's default branch. Parsed via `readRawSymref` with prefix `"refs/remotes/origin/"`.

2. If that file doesn't exist (shallow clone, old git version), check if `refs/remotes/origin/main` resolves to a SHA.

3. Check `refs/remotes/origin/master` as a fallback.

4. Return `"main"` as the default if nothing resolves.

All lookups happen inside `commonDir` for worktrees, since remote-tracking refs are shared state.

## 05 Security: Ref Validation

Because `.git/HEAD` and loose ref files are plain text that _can be written without going through git's own validation_, an attacker who can tamper with those files could inject malicious content into downstream contexts. Claude Code defends with two validators applied to every string read from `.git/`.

### isSafeRefName

```typescript
export function isSafeRefName(name: string): boolean {
  if (!name || name.startsWith('-') || name.startsWith('/')) return false
  if (name.includes('..')) return false      // path traversal
  if (name.split('/').some(c => c === '.' || c === '')) return false
  return /^[a-zA-Z0-9/._+@-]+$/.test(name) // strict allowlist
}
```

The allowlist covers all legitimate git branch names (including `feature/foo`, `release-1.2.3+build`, `dependabot/npm/@types/node-18`) while blocking:

- **Path traversal** — `..`, leading `/`, empty path components (`foo//bar`), single dot components (`foo/./bar`)
- **Argument injection** — leading `-` (would become a CLI flag)
- **Shell metacharacters** — newlines, backticks, `$`, `;`, `|`, `&`, `(`, `)`, `<`, `>`, spaces, tabs, quotes, backslash
- **git's own forbidden sequences** — `@{` is blocked because `{` is not in the allowlist

### isValidGitSha

```typescript
export function isValidGitSha(s: string): boolean {
  return /^[0-9a-f]{40}$/.test(s) || /^[0-9a-f]{64}$/.test(s)
}
```

Only full-length SHAs are accepted — 40 hex chars for SHA-1, 64 for SHA-256. Git never writes abbreviated SHAs to `HEAD` or ref files. An attacker controlling a detached HEAD file could embed shell metacharacters; the allowlist of hex digits prevents any injection.

### When Validation Fails

Both validators are applied in `readGitHead`, `resolveRefInDir`, and `readRawSymref`. A validation failure returns `null`, which propagates up to the public API (e.g., `getCachedBranch` returns `"HEAD"` as the safe fallback). Claude Code silently degrades to a safe value rather than crashing or passing tainted data downstream.

## 06 Operation Tracking: Parsing Command Output

`gitOperationTracking.ts` solves a different problem: after Claude Code runs a bash command, how does it know if a git commit happened, a push succeeded, or a PR was created? It doesn't re-query git state — it _parses the command text and output_.

### Shell-Agnostic Regex Matching

The regexes operate on raw command text and work identically for Bash and PowerShell, because both invoke git/gh/glab/curl as external binaries with the same argv syntax. The key helper handles git's global options:

```typescript
// Builds a regex tolerant of git global flags between "git" and the subcmd
// e.g. "git -c commit.gpgsign=false commit -m 'msg'" still matches "commit"
function gitCmdRe(subcmd: string, suffix = ''): RegExp {
  return new RegExp(
    `\\bgit(?:\\s+-[cC]\\s+\\S+|\\s+--\\S+=\\S+)*\\s+${subcmd}\\b${suffix}`
  )
}

const GIT_COMMIT_RE   = gitCmdRe('commit')
const GIT_PUSH_RE     = gitCmdRe('push')
const GIT_MERGE_RE    = gitCmdRe('merge', '(?!-)')  // excludes "merge-base" etc.
const GIT_REBASE_RE   = gitCmdRe('rebase')
const GIT_CHERRY_PICK = gitCmdRe('cherry-pick')
```

### detectGitOperation: What It Returns

The main export is `detectGitOperation(command, output)` which returns a sparse object with only the fields that actually fired:

```typescript
type DetectedOp = {
  commit?: { sha: string; kind: 'committed' | 'amended' | 'cherry-picked' }
  push?:   { branch: string }
  branch?: { ref: string; action: 'merged' | 'rebased' }
  pr?:     { number: number; url?: string; action: PrAction }
}
```

Commit SHA is extracted from git's output line: `[branch abc1234] message` (or `[branch (root-commit) abc1234]`). Push branch is parsed from the ref update line git writes to stderr: `abc..def branch -> branch`. Merge and rebase are confirmed by checking the output for `Fast-forward` / `Merge made by` or `Successfully rebased`.

### PR Detection Deep Dive — gh, glab, and curl

PR creation is detected across three surfaces:

**gh CLI** — six action patterns cover the full PR lifecycle:

```
// gh pr create  → PrAction 'created'
// gh pr edit    → PrAction 'edited'
// gh pr merge   → PrAction 'merged'
// gh pr comment → PrAction 'commented'
// gh pr close   → PrAction 'closed'
// gh pr ready   → PrAction 'ready'
```

**glab CLI** — GitLab MR creation via `\bglab\s+mr\s+create\b`.

**curl REST API** — two conditions must both match: the command contains `curl` with a POST indicator (`-X POST`, `--request POST`, or a data flag `-d`), AND the URL matches a PR endpoint pattern while excluding sub-resources:

```
// POST indicator (any one of):
/-X\s*POST\b/i | /--request\s*=?\s*POST\b/i | /\s-d\s/

// PR endpoint — matches /pulls, /pull-requests, /merge-requests
// but NOT /pulls/123/comments (sub-resource exclusion)
/https?:\/\/[^\s'"]*\/(pulls|pull-requests|merge[-_]requests)(?!\/\d)/i
```

## 07 PR Auto-Linking: Session → PR

When `gh pr create` succeeds, Claude Code extracts the GitHub PR URL from stdout and links the current session to that PR. This powers the PR-context feature in the session history UI.

```typescript
// Inside trackGitOperations, when prHit.action === 'created':
if (stdout) {
  const prInfo = findPrInStdout(stdout)
  if (prInfo) {
    // Dynamic import avoids circular dependency
    void import('../../utils/sessionStorage.js').then(({ linkSessionToPR }) => {
      void import('../../bootstrap/state.js').then(({ getSessionId }) => {
        const sessionId = getSessionId()
        if (sessionId) {
          void linkSessionToPR(sessionId, prInfo.prNumber, prInfo.prUrl, prInfo.prRepository)
        }
      })
    })
  }
}
```

The PR URL regex (`/https:\/\/github\.com\/([^/]+\/[^/]+)\/pull\/(\d+)/`) extracts both the repository (`owner/repo`) and the PR number from the full URL. These three fields — number, URL, repository — are stored in session storage.

### Dynamic Import Pattern

The double dynamic import is intentional. `sessionStorage` and `bootstrap/state` both transitively import from modules that import `gitOperationTracking`. Doing `import()` at runtime instead of statically at the top of the file breaks the circular dependency graph without restructuring modules.

## 08 GitHub Auth Status: Local-Only Check

`ghAuthStatus.ts` checks whether the `gh` CLI is installed and authenticated. It does this with a careful two-step that avoids making any network request:

```typescript
export async function getGhAuthStatus(): Promise<GhAuthStatus> {
  const ghPath = await which('gh')   // Bun.which — no subprocess
  if (!ghPath) return 'not_installed'

  const { exitCode } = await execa('gh', ['auth', 'token'], {
    stdout: 'ignore',  // token NEVER enters this process
    stderr: 'ignore',
    timeout: 5000,
    reject: false,
  })
  return exitCode === 0 ? 'authenticated' : 'not_authenticated'
}
```

The choice of `gh auth token` over `gh auth status` is deliberate. `auth status` makes a live request to `api.github.com` to verify the token. `auth token` only reads the local keyring or config file and exits zero if a token exists. This keeps auth checking offline and fast.

### Security: stdout: 'ignore'

Setting `stdout: 'ignore'` means the auth token printed by `gh auth token` is discarded at the OS level and never flows through Node's memory. This prevents the token from appearing in logs, core dumps, or accidental `console.log` calls upstream.

## 09 Gitignore Management

The `gitignore.ts` module ensures Claude Code's own files (e.g., conversation logs, local config) don't accidentally get committed to user repos. It writes to the **global gitignore** at `~/.config/git/ignore` rather than any repo's local `.gitignore`.

```typescript
export async function addFileGlobRuleToGitignore(
  filename: string,
  cwd: string = getCwd(),
): Promise<void> {
  if (!(await dirIsInGitRepo(cwd))) return   // no-op outside git repos

  const gitignoreEntry = `**/${filename}`
  const testPath = filename.endsWith('/')
    ? `${filename}sample-file.txt`      // directory pattern check
    : filename

  // Check if already ignored by any .gitignore (local, nested, or global)
  if (await isPathGitignored(testPath, cwd)) return

  // Write to global gitignore, creating it if necessary
  const globalPath = getGlobalGitignorePath()  // ~/.config/git/ignore
  await mkdir(dirname(globalPath), { recursive: true })
  // Append only — checks for existing entry to avoid duplication
}
```

### Why Global?

Writing to `~/.config/git/ignore` means the rule applies across all repos without touching any of them. Claude Code uses this to ignore its own `.claude/` working files and settings. Users get clean diffs without any modification to their own `.gitignore` files.

## Key Takeaways

- Git state (branch, SHA, remote URL) is read directly from `.git/` files using Node's `fs` APIs — never by spawning a git subprocess. This eliminates startup latency on the hot path.

- The `GitFileWatcher` singleton caches all four derived values and recomputes only when the underlying files change, using Node's `watchFile`. The dirty-before-compute pattern prevents stale values from a race between compute and invalidation.

- The config parser faithfully replicates git's INI rules: case-insensitive sections and keys, case-sensitive subsections, backslash escapes inside quoted values, inline comments outside quotes.

- Worktree support is first-class: every function that reads shared git state (config, refs, packed-refs) checks `commonDir` and falls back to it when the per-worktree gitDir doesn't have what's needed.

- All strings read from `.git/` are validated against strict allowlists (`isSafeRefName`, `isValidGitSha`) before use — protecting against path traversal, argument injection, and shell metacharacter injection from tampered git files.

- Operation tracking is regex-based on raw command text and output — shell-agnostic and works for Bash and PowerShell equally. A single `detectGitOperation` call covers commits, pushes, merges, rebases, cherry-picks, and PR lifecycle via gh, glab, and curl.

- PR auto-linking extracts the GitHub PR URL from `gh pr create` stdout and stores it with the session ID via a dynamic import that avoids circular module dependencies.

- The `gh auth token` check (vs `auth status`) is an intentional offline-only design: no network call, and `stdout: 'ignore'` ensures the token never enters Node's memory.

## Check Your Understanding

**Q1. Why does `resolveGitDir` check whether `.git` is a file rather than a directory?**

In git worktrees and submodules, `.git` is a plain text pointer file containing `gitdir: <path>` that points to the actual git directory.

**Q2. In `GitFileWatcher.get()`, why is the `dirty` flag cleared _before_ the async `compute()` call rather than after?**

So that if a file-change event fires during the async read, `invalidate()` can re-set dirty and the next caller will re-read — preventing a stale value from being cached.

**Q3. A branch named `dependabot/npm_and_yarn/@types/node-18` — does `isSafeRefName` accept or reject it?**

Accept — all characters are in the allowlist `[a-zA-Z0-9/._+@-]` and no forbidden patterns appear.

**Q4. Why does `getGhAuthStatus` use `gh auth token` instead of `gh auth status`?**

`auth token` only reads the local keyring/config file, while `auth status` makes a live network request to api.github.com to verify the token.

**Q5. The PR auto-linking code uses `void import('../../utils/sessionStorage.js').then(...)` with a dynamic import. Why not a static `import` at the top of the file?**

Static imports would break because `sessionStorage` and `bootstrap/state` transitively import `gitOperationTracking`, creating a circular dependency that Node cannot resolve statically.

**Q6. `addFileGlobRuleToGitignore` writes to `~/.config/git/ignore` (the global gitignore) rather than the repo's local `.gitignore`. What is the main reason for this choice?**

Writing to the global gitignore prevents Claude Code's own files from appearing in diffs across all repos without modifying any individual repo's tracked files.

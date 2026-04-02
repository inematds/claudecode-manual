# mdENG — Lesson 19 — Search Tools: Glob & Grep

## Complete Lesson Content

### 1. The Two Search Tools

Claude Code provides two distinct search primitives using a shared underlying engine:

| Tool | User-facing name | What it searches | Returns | Hard limit |
|------|------------------|------------------|---------|-----------|
| `Glob` | Search | File _names_ (glob pattern) | Array of file paths sorted by mtime | 100 files (configurable via `globLimits`) |
| `Grep` | Search | File _contents_ (regex) | Paths, lines, or counts depending on mode | 250 lines/files (default `head_limit`) |

Both tools share the user-facing name "Search." In the terminal UI, neither "Glob" nor "Grep" appears—only "Search." The distinct names exist at the API/schema layer where the model decides which to call. `GlobTool` reuses `GrepTool`'s `renderToolResultMessage` directly (see `GlobTool/UI.tsx` line 53).

### 2. Glob Delegates to ripgrep

Despite its name, this tool does not use Node's `fs.glob`, `fast-glob`, or any JavaScript glob library. It delegates entirely to ripgrep via `utils/glob.ts`.

**glob() in utils/glob.ts:**

```typescript
export async function glob(
  filePattern: string,
  cwd: string,
  { limit, offset }: { limit: number; offset: number },
  abortSignal: AbortSignal,
  toolPermissionContext: ToolPermissionContext,
): Promise<{ files: string[]; truncated: boolean }> {
  // ...absolute-path extraction, ignore-pattern assembly...

  const args = [
    '--files',        // list files instead of searching content
    '--glob', searchPattern,
    '--sort=modified',
    ...(noIgnore ? ['--no-ignore'] : []),
    ...(hidden  ? ['--hidden']    : []),
  ]

  const allPaths = await ripGrep(args, searchDir, abortSignal)

  const truncated = allPaths.length > offset + limit
  const files = allPaths.slice(offset, offset + limit)
  return { files, truncated }
}
```

The key insight: ripgrep with `--files --glob <pattern>` performs high-performance glob traversal. "Ripgrep's multi-threaded directory walker is _significantly_ faster than Node's" fs-based implementations on large codebases. The same binary handles both name-search (Glob) and content-search (Grep).

#### Absolute-path patterns are decomposed

Ripgrep's `--glob` flag only accepts relative patterns. When the model passes an absolute path pattern like `/src/utils/*.ts`, `extractGlobBaseDirectory()` splits it into a `baseDir` (`/src/utils`) and a relative `relativePattern` (`*.ts`). The search then runs with `searchDir = /src/utils`.

**extractGlobBaseDirectory() logic:**

```typescript
export function extractGlobBaseDirectory(pattern: string): {
  baseDir: string; relativePattern: string
} {
  // Find first glob special char: * ? [ {
  const match = pattern.match(/[\*?\[{\]/)

  if (!match || match.index === undefined) {
    // Literal path — use dirname / basename split
    return { baseDir: dirname(pattern), relativePattern: basename(pattern) }
  }

  const staticPrefix = pattern.slice(0, match.index)
  const lastSep = Math.max(
    staticPrefix.lastIndexOf('/'),
    staticPrefix.lastIndexOf(sep)
  )

  if (lastSep === -1) return { baseDir: '', relativePattern: pattern }
  return {
    baseDir: staticPrefix.slice(0, lastSep),
    relativePattern: pattern.slice(lastSep + 1)
  }
}
```

#### Environment variables control behavior

- `CLAUDE_CODE_GLOB_NO_IGNORE=false` — respect `.gitignore` (default: ignore it, include everything)
- `CLAUDE_CODE_GLOB_HIDDEN=false` — exclude hidden files (default: include them)

**Why ignore `.gitignore` by default?** Claude needs to see build artifacts, generated files, and `node_modules` structure to answer questions like "is this dependency installed?" or "what files were emitted by the build?" Respecting gitignore by default would hide too much context.

### 3. Grep's Three Output Modes

The `output_mode` parameter controls both the ripgrep flags passed and the shape of the returned data. Each mode is a distinct ripgrep invocation strategy.

#### files_with_matches

Default mode. Returns **only file paths** that contain at least one match. Low token cost — ideal for "which files use this pattern?"

Command: `rg -l`

Output sorted by **mtime** (most recently modified first). Result: `filenames[]`, `numFiles`.

#### content

Returns the **matching lines** with optional context lines before/after. Supports `-n` (line numbers), `-B`/`-A`/`-C` context, multiline mode.

Command: `rg (no -l/-c)`

Result: `content` string, `numLines`.

#### count

Returns per-file **match counts** in `filename:N` format. Useful for "how often does this pattern appear and where?"

Command: `rg -c`

Result: `content` string (raw), `numMatches`, `numFiles`.

**output mode dispatch in GrepTool.ts:**

```typescript
// Add output mode flags
if (output_mode === 'files_with_matches') {
  args.push('-l')
} else if (output_mode === 'count') {
  args.push('-c')
}
// content mode: no flag needed — rg defaults to printing match lines

// Line numbers only apply in content mode
if (show_line_numbers && output_mode === 'content') {
  args.push('-n')
}

// Context flags (-C supersedes -B/-A)
if (output_mode === 'content') {
  if (context !== undefined) {
    args.push('-C', context.toString())
  } else if (context_c !== undefined) {
    args.push('-C', context_c.toString())
  } else {
    if (context_before !== undefined) args.push('-B', context_before.toString())
    if (context_after  !== undefined) args.push('-A', context_after.toString())
  }
}
```

#### The mtime sort in files_with_matches mode

After ripgrep returns file paths, `GrepTool` runs `Promise.allSettled(results.map(_ => fs.stat(_)))` to fetch mtimes, then sorts descending. The most recently modified files appear first. This is intentional: the most recently changed files are the most relevant to the current task.

**Test mode bypasses mtime sort.** When `process.env.NODE_ENV === 'test'`, results are sorted by filename instead, ensuring deterministic ordering in VCR test fixtures. This is a common pattern in the codebase.

**mtime sort logic:**

```typescript
const stats = await Promise.allSettled(
  results.map(_ => getFsImplementation().stat(_))
)
const sortedMatches = results
  .map((_, i) => {
    const r = stats[i]!
    return [_, r.status === 'fulfilled' ? (r.value.mtimeMs ?? 0) : 0] as const
  })
  .sort((a, b) => {
    if (process.env.NODE_ENV === 'test') return a[0].localeCompare(b[0])
    const timeComparison = b[1] - a[1]
    return timeComparison === 0 ? a[0].localeCompare(b[0]) : timeComparison
  })
  .map(_ => _[0])
```

### 4. Pagination: head_limit and offset

Both tools support `head_limit` and `offset` parameters that work like a Unix `tail -n +N | head -N` pipeline. The default head_limit is **250** — generous enough for most searches while preventing context bloat on broad patterns.

**Pagination flow:**
1. ripgrep raw output (e.g. 10,000 lines)
2. applyHeadLimit(items, limit, offset)
3. items.slice(offset, offset+limit) e.g. slice(0, 250)
4. Return to model with appliedLimit hint
5. Model calls again with offset=250

**applyHeadLimit() implementation:**

```typescript
function applyHeadLimit<T>(
  items: T[],
  limit: number | undefined,
  offset: number = 0,
): { items: T[]; appliedLimit: number | undefined } {
  // Explicit 0 = unlimited escape hatch
  if (limit === 0) {
    return { items: items.slice(offset), appliedLimit: undefined }
  }
  const effectiveLimit = limit ?? 250    // DEFAULT_HEAD_LIMIT
  const sliced = items.slice(offset, offset + effectiveLimit)
  // appliedLimit is ONLY set when truncation occurred
  // so the model knows there may be more results to page
  const wasTruncated = items.length - offset > effectiveLimit
  return {
    items: sliced,
    appliedLimit: wasTruncated ? effectiveLimit : undefined,
  }
}
```

Three key design decisions in `applyHeadLimit`:

1. **`limit=0` is the unlimited escape hatch.** The model can pass `head_limit=0` when it explicitly wants all results regardless of size. The schema description warns "use sparingly — large result sets waste context."

2. **`appliedLimit` is only set when truncation occurred.** If the full result fits within the limit, `appliedLimit` is `undefined`. The model only sees the pagination hint when there's actually more to page through.

3. **Head-limiting happens before path relativization.** In content mode, each line needs string manipulation to convert absolute paths to relative. By slicing first, you avoid processing thousands of lines that will be discarded. A broad pattern returning 10k lines only relativizes the 250 that are kept.

**The 20KB persist threshold.** "Unbounded content-mode greps can fill up to the 20KB persist threshold (~6-24K tokens/grep-heavy session). 250 is generous enough for exploratory searches while preventing context bloat." Tool results above 20KB are persisted to disk rather than kept in the prompt, reducing in-context tokens. The 250-line default keeps results well under this threshold.

#### Pagination in the tool result block

When truncation occurs, the tool result block includes a `[Showing results with pagination = limit: 250]` annotation. The model reads this and knows it can call the tool again with `offset=250` to get the next page.

### 5. ripgrep Binary Resolution

Every Grep and Glob invocation ultimately calls ripgrep. But _which_ ripgrep binary runs depends on a three-mode resolution chain, evaluated once per process and memoized.

**Resolution chain:**
1. Check `USE_BUILTIN_RIPGREP`
2. If falsy: try system `rg`
3. If not found on PATH: check bundled mode
4. If bundled (Bun): `embedded`
5. Else: `vendor/ripgrep/<arch>-<platform>/rg`

#### system

User has `USE_BUILTIN_RIPGREP` set to a falsy value AND `rg` is on PATH. Uses the system-installed binary via the command name `rg` (not the resolved path — see security note below).

#### embedded

Running in bundled (native) Bun mode. ripgrep is statically compiled into the Bun executable. Spawned via `process.execPath` with `argv0='rg'` — the process checks `argv[0]` and dispatches as ripgrep.

#### builtin

Default npm install. A platform-specific binary is shipped at `vendor/ripgrep/<arch>-<platform>/rg[.exe]`. On macOS, a code-signing step may be needed (see below).

**getRipgrepConfig() memoized factory:**

```typescript
type RipgrepConfig = {
  mode: 'system' | 'builtin' | 'embedded'
  command: string
  args: string[]
  argv0?: string
}

const getRipgrepConfig = memoize((): RipgrepConfig => {
  const userWantsSystemRipgrep = isEnvDefinedFalsy(process.env.USE_BUILTIN_RIPGREP)

  if (userWantsSystemRipgrep) {
    const { cmd: systemPath } = findExecutable('rg', [])
    if (systemPath !== 'rg') {
      // SECURITY: Use command name 'rg', NOT systemPath
      // Prevents ./rg.exe in cwd from being executed (PATH hijacking)
      return { mode: 'system', command: 'rg', args: [] }
    }
  }

  if (isInBundledMode()) {
    return {
      mode: 'embedded',
      command: process.execPath,
      args: ['--no-config'],
      argv0: 'rg',
    }
  }

  // builtin: platform-specific vendored binary
  const command = process.platform === 'win32'
    ? path.resolve(rgRoot, `${process.arch}-win32`, 'rg.exe')
    : path.resolve(rgRoot, `${process.arch}-${process.platform}`, 'rg')
  return { mode: 'builtin', command, args: [] }
})
```

#### macOS code-signing for the builtin binary

On macOS, the vendored `rg` binary ships with only a linker signature (not an ad-hoc signature). macOS Gatekeeper blocks unsigned or minimally-signed binaries from running. On first use, `codesignRipgrepIfNecessary()` checks for `linker-signed` in `codesign -vv` output and re-signs the binary with `codesign --sign -` (self-sign / ad-hoc). It also strips the quarantine xattr (`com.apple.quarantine`) that macOS adds to downloaded files.

**This only runs for `builtin` mode.** The embedded and system ripgrep binaries are assumed to already be properly signed. The codesign check is also guarded by `alreadyDoneSignCheck` — it runs at most once per process lifetime.

#### Spawn strategy: execFile vs spawn

The embedded mode (with `argv0`) must use `spawn()` — Node's `execFile` does not support the `argv0` override needed to make the Bun executable believe it is `rg`. All other modes use `execFile` with a `maxBuffer: 20MB` cap.

### 6. Performance Architecture

#### Timeout strategy

Timeouts are platform-aware: **20 seconds** on standard platforms, **60 seconds** on WSL (which has a 3-5x file I/O penalty). Overrideable via `CLAUDE_CODE_GLOB_TIMEOUT_SECONDS`. Kill escalation uses SIGTERM first, then SIGKILL after 5 seconds — because ripgrep blocked on deep filesystem traversal may not respond to SIGTERM.

#### EAGAIN retry with single-threaded fallback

In resource-constrained environments (Docker, CI), ripgrep may fail with EAGAIN (`os error 11` / "Resource temporarily unavailable") when spawning worker threads. The retry strategy is precise: one retry with `-j 1` (single thread) for that specific call only. A previous version persisted single-threaded mode globally, but this caused timeouts on large repos where EAGAIN was a transient startup error.

**EAGAIN retry logic:**

```typescript
if (!isRetry && isEagainError(stderr)) {
  logForDebugging(`rg EAGAIN error, retrying with -j 1`)
  logEvent('tengu_ripgrep_eagain_retry', {})
  ripGrepRaw(
    args, target, abortSignal,
    (retryError, retryStdout, retryStderr) =>
      handleResult(retryError, retryStdout, retryStderr, true),
    true, // singleThread = true for this call only
  )
  return
}

function isEagainError(stderr: string): boolean {
  return (
    stderr.includes('os error 11') ||
    stderr.includes('Resource temporarily unavailable')
  )
}
```

#### Streaming for file counting (ripGrepFileCount)

The telemetry call `countFilesRoundedRg()` uses a dedicated streaming counter rather than `ripGrep()`. On a repo with 247k files, buffering all paths into a string then splitting on newlines materializes ~16MB in memory. The streaming version counts newline bytes per chunk — peak memory is one stream chunk (~64KB). Results are rounded to the nearest power of 10 for privacy (e.g., 8 → 10, 42 → 100, 8500 → 10000).

#### Buffer cap: 20MB

The `MAX_BUFFER_SIZE` constant is set to 20MB. Large monorepos with 200k+ files can easily produce stdout larger than Node's default 1MB `execFile` buffer. If the buffer is exceeded, ripgrep returns an `ERR_CHILD_PROCESS_STDIO_MAXBUFFER` error — the code treats this as a partial result and drops the last (potentially incomplete) line.

#### Path relativization saves tokens

All returned paths are converted from absolute to relative (via `toRelativePath`) before being included in the tool result sent to the model. Absolute paths like `/Users/moiz/project/src/utils/ripgrep.ts` cost more tokens than `src/utils/ripgrep.ts`. On a result set of 250 files, this saves hundreds of tokens per call.

#### Concurrent safety

Both Glob and Grep declare `isConcurrencySafe() = true`. They are read-only operations with no shared mutable state — the model can (and does) issue multiple search calls in parallel within the same turn.

#### Line length cap: 500 chars

Grep appends `--max-columns 500` to every ripgrep invocation. This prevents base64-encoded data, minified JavaScript, or other long single-line content from flooding the output. Lines longer than 500 characters are truncated with a `[omitted long line]` annotation from ripgrep.

### 7. VCS Directory Exclusions

Grep automatically excludes these version control system directories from every search:

```typescript
const VCS_DIRECTORIES_TO_EXCLUDE = [
  '.git', '.svn', '.hg', '.bzr', '.jj', '.sl',
] as const
```

This covers Git, Subversion, Mercurial, Bazaar, Jujutsu, and Sapling. These directories contain large binary objects, pack files, and index databases — searching them creates noise and can be extremely slow.

Glob does _not_ apply these exclusions explicitly, but uses `--no-ignore` by default which lets ripgrep's own traversal logic handle them — ripgrep's built-in behavior skips `.git` directories unless overridden.

### 8. Pattern Safety: The Leading-Dash Problem

If a regex pattern starts with a dash (e.g., `-v`), ripgrep would interpret it as a command-line flag rather than a search pattern. GrepTool handles this correctly:

```typescript
// If pattern starts with dash, use -e flag to specify it as a pattern
// This prevents ripgrep from interpreting it as a command-line option
if (pattern.startsWith('-')) {
  args.push('-e', pattern)
} else {
  args.push(pattern)
}
```

The `-e` flag tells ripgrep "what follows is a pattern, not a flag." This is a class of injection vulnerability common to tools that shell out: if user-supplied text is appended to a CLI command naively, a leading dash can hijack the flag parsing.

**Security note: UNC path bypass.** Both Glob and Grep skip filesystem `stat` calls for paths starting with `\\` or `//` (UNC paths on Windows). The comment explains: "SECURITY: Skip filesystem operations for UNC paths to prevent NTLM credential leaks." A `stat` call on a UNC path triggers an SMB authentication handshake that can leak NTLM hashes to a malicious server.

### 9. Glob Pattern Parsing in GrepTool

The `glob` parameter of GrepTool accepts one or more file patterns to filter which files are searched. The parsing logic handles two cases:

**glob pattern splitting logic:**

```typescript
if (glob) {
  const globPatterns: string[] = []
  const rawPatterns = glob.split(/\s+/)

  for (const rawPattern of rawPatterns) {
    // Brace patterns must NOT be split on commas
    // e.g. "*.{ts,tsx}" is one pattern, not ["*.{ts", "tsx}"]
    if (rawPattern.includes('{') && rawPattern.includes('}')) {
      globPatterns.push(rawPattern)
    } else {
      // Split comma-separated patterns without braces
      globPatterns.push(...rawPattern.split(',').filter(Boolean))
    }
  }

  for (const globPattern of globPatterns.filter(Boolean)) {
    args.push('--glob', globPattern)
  }
}
```

The model can pass `glob: "*.js,*.ts"` (comma-separated) or `glob: "*.{ts,tsx}"` (brace-expanded). The parser correctly identifies brace patterns and passes them as single `--glob` arguments to ripgrep, which handles brace expansion natively.

The `type` parameter offers an alternative: `type: "js"` maps to ripgrep's `--type js`, which uses ripgrep's built-in file type definitions (includes both `.js` and `.jsx`). This is _more efficient_ than a glob because ripgrep resolves type definitions at startup rather than matching each path against a pattern.

## Key Takeaways

1. **Glob is ripgrep with --files.** There is no JavaScript glob library involved. Both Glob and Grep delegate to the same ripgrep binary, making them fast on any codebase size and consistent in their traversal behavior.

2. **Three output modes, three ripgrep strategies.** `files_with_matches` (`-l`), `content` (no flag), and `count` (`-c`) are fundamentally different ripgrep invocations — not post-processing variants. Choosing the right mode avoids unnecessary work.

3. **Head-limiting before path relativization is a deliberate optimization.** Broad patterns can return tens of thousands of lines. Slicing to 250 before string-processing each line eliminates redundant work. Explicit `head_limit=0` is the escape hatch.

4. **Binary resolution is three-tier and memoized.** `system → embedded → builtin`. The security detail is non-obvious: when using system ripgrep, the command is spelled `"rg"` (not the resolved path) to prevent PATH-hijacking by a local `rg.exe`.

5. **EAGAIN retry is scoped to a single call.** Persisting single-threaded mode globally caused regressions on large repos. The retry uses `-j 1` only for that one call and restores multi-threaded behavior immediately after. Transient errors should not permanently degrade performance.

6. **mtime sort is the default relevance signal.** In `files_with_matches` mode, results are sorted by modification time (most recent first). The assumption is: files you just edited are more relevant to the current task than files that haven't been touched in months.

## Quiz

**1. When GlobTool receives a pattern like `/src/utils/*.ts`, what transformation happens before the ripgrep call?**

- The pattern is passed directly to ripgrep's `--glob` flag unchanged.
- **The base directory `/src/utils` is extracted as `searchDir` and `*.ts` becomes the `--glob` pattern.**
- GlobTool falls back to Node's `fs.glob` for absolute patterns.
- The pattern is rejected with a validation error because absolute paths are not allowed.

**2. Which ripgrep output mode returns only file paths (not matching lines)?**

- `content` — it returns the content of matching files
- `count` — it returns a count of matches per file
- **`files_with_matches` — maps to `rg -l`, returns only file paths**
- `paths` — there is no `paths` mode

**3. What does `head_limit=0` mean in the Grep tool?**

- Return zero results (empty response)
- Use the default limit of 250
- **Explicitly request unlimited results — the escape hatch for getting all output regardless of size**
- Disable the tool (head_limit must be a positive integer)

**4. Why does the system-mode ripgrep use the command `"rg"` rather than the full resolved path from `findExecutable`?**

- Performance — shorter strings are faster to spawn
- Compatibility — some systems don't support absolute paths in execFile
- **Security — using the resolved path could allow a malicious `./rg.exe` in the current directory to be executed via PATH hijacking**
- The resolved path might be a symlink that breaks on Windows

**5. In `files_with_matches` mode, how are results ordered?**

- Alphabetically by file path
- By file size (largest first)
- **By modification time, most recently modified first (with filename as a tiebreaker)**
- In the order ripgrep traverses the filesystem

**6. What happens when a Grep pattern starts with a dash (e.g., `-v`)?**

- The tool returns a validation error
- The pattern is URL-encoded before being passed to ripgrep
- **The pattern is passed using `-e pattern` syntax to prevent ripgrep from interpreting it as a flag**
- The leading dash is stripped before passing to ripgrep

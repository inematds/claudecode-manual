# mdENG — Lesson 18 — File Tools: Read, Write, Edit — Source Deep Dive

## 01 Three Tools, One Contract

Claude Code provides three file-manipulation primitives sharing one invariant: **every write operation requires a prior read of the target file**. That single rule shapes the entire design.

### Read

Functionality: Read any file — text, images, PDFs, Jupyter notebooks. Read-only, concurrency-safe, with pagination and dedup built in.

### Write

Functionality: Create new files or fully overwrite existing ones. Requires a prior Read of any existing file. Prefer Edit for partial changes.

### Edit

Functionality: Exact string replacement inside a file. Only sends the diff — much cheaper than Write for small changes. Handles quote normalization automatically.

### Capability Comparison Table

| Capability | Read | Write | Edit |
|---|---|---|---|
| Read-only / safe for concurrency | Yes | No | No |
| Requires prior Read of existing file | — | Yes | Yes |
| Handles images natively | Yes | No | No |
| Handles PDFs natively | If supported | No | No |
| Handles Jupyter notebooks | Yes | No | No — use NotebookEdit |
| Quote normalization | — | — | Yes |
| Dedup (skip re-sending unchanged file) | Yes | — | — |
| Token limit enforced | 25 000 tok default | — | — |
| LSP notifications on save | No | Yes | Yes |
| Max file size (Edit) | — | — | 1 GiB |

---

## 02 Read Tool Deep-Dive

### Pagination: offset + limit

By default Read returns up to 2000 lines starting at line 1. For large files, pass `offset` (1-based line number) and `limit`. The default prompt says to read the whole file unless you know the relevant section; the A/B variant (`targetedRangeNudge` flag) flips that: only read what you need.

**Default call — reads from line 1 up to 2000 lines:**
```
Read({ file_path: "/project/src/main.ts" })
```

**Paginated — start at line 500, read 100 lines:**
```
Read({ file_path: "/project/src/main.ts", offset: 500, limit: 100 })
```

### Token limit enforcement

A two-stage gate keeps output bounded. First a fast rough estimate (no API call). If the estimate exceeds 1/4 of the cap, an exact token count via API is fetched. If the result exceeds `maxTokens` (default 25 000), a `MaxFileReadTokenExceededError` is thrown before the content is ever sent to the model.

**Design note:** A 256 KB size gate checks the whole-file size before reading. Throwing rather than truncating is intentional: a truncation path was tested (issue #21841, Mar 2026) and reverted — truncation produced ~25K token responses while the throw yields a ~100-byte error, dramatically reducing wasted context.

### Image support

PNG, JPG, JPEG, GIF, and WebP are detected by extension. The file is loaded into memory, optionally resized via `sharp` (or the native `image-processor-napi` in bundled builds), and returned as a base64 block for Claude's multimodal input. Dimension metadata travels alongside so coordinates stay correct after resizing.

**Read a screenshot — Claude sees the image visually:**
```
Read({ file_path: "~/Documents/screenshots/CleanShot 2026-03-31.png" })
```

**macOS detail:** Screenshot filenames on macOS use either a regular space or a thin space (U+202F) before AM/PM depending on OS version. Read detects an ENOENT and automatically retries with the alternate space character before giving up.

### PDF support

When the runtime supports PDFs (`isPDFSupported()` returns true), the `pages` parameter accepts ranges like `"1-5"`, `"3"`, or `"10-20"`. A hard cap of 20 pages per call is enforced. Large PDFs that exceed an internal size threshold have their pages extracted as images rather than sent as a single document block.

**Read pages 1 through 5 of a PDF:**
```
Read({ file_path: "/docs/spec.pdf", pages: "1-5" })
```

**Large PDF: exceeds 10 pages requires explicit range:**
```
Read({ file_path: "/docs/report.pdf", pages: "6-10" })
```

### Jupyter notebooks

`.ipynb` files get special handling. Every cell — code, markdown, outputs, visualizations — is passed through `mapNotebookCellsToToolResult()` which stitches them into a coherent tool result. The model sees a unified view of the notebook rather than raw JSON.

### Dedup: how Read avoids re-sending unchanged files

Every text or notebook read stores metadata in `readFileState`:

```
readFileState.set(fullFilePath, {
  content,
  timestamp: getFileModificationTime(fullFilePath),
  offset, // undefined means "full read"
  limit,
})
```

On the next Read of the same file and range, the tool checks the on-disk `mtime`. If it matches, it returns a lightweight stub instead of re-sending the content:

> "File unchanged since last read. The content from the earlier Read tool_result in this conversation is still current — refer to that instead of re-reading."

**Why it matters:** A/B data showed ~18% of Read calls are same-file re-reads. Two full copies per turn waste `cache_creation` tokens on every subsequent turn. The dedup path yields a ~100-byte stub vs. up to 25K tokens of content. A GrowthBook killswitch (`tengu_read_dedup_killswitch`) can disable it if the stub message confuses the model.

**Only applies to Read-originated entries.** Edit and Write also write to `readFileState`, but with `offset: undefined`. Dedup skips these entries intentionally — they reflect post-edit mtime, not a prior clean read.

### Blocked device paths

A hardcoded set of device paths is blocked by path string only — no I/O needed. This prevents reads that would hang the process:

```
// Infinite output — never reach EOF
'/dev/zero', '/dev/random', '/dev/urandom', '/dev/full'

// Blocks waiting for input
'/dev/stdin', '/dev/tty', '/dev/console'

// fd aliases for stdio (also Linux /proc/self/fd/0-2)
'/dev/fd/0', '/dev/fd/1', '/dev/fd/2'
```

Safe special files like `/dev/null` are intentionally allowed.

### Limits precedence: env var > GrowthBook > default

Token and byte limits have a three-tier precedence chain, memoized at first call so a GrowthBook flag refresh mid-session cannot change the cap:

1. **Env var** `CLAUDE_CODE_FILE_READ_MAX_OUTPUT_TOKENS` — user-set override, beats everything
2. **GrowthBook flag** `tengu_amber_wren` — per-org experiment infrastructure
3. **Hardcoded default** — 25 000 tokens / 256 KB

Each field (maxTokens, maxSizeBytes, includeMaxSizeInPrompt, targetedRangeNudge) is individually validated. Invalid values fall through to the hardcoded defaults — no path to a cap of zero.

---

## 03 Write Tool Deep-Dive

Write is a full-content replacement. It either creates a new file or overwrites an existing one entirely. For surgical changes to existing files, the prompt explicitly says to prefer Edit — it only sends the diff.

### The read-before-write gate

If a file already exists on disk, Write requires a prior Read of that file. The tool checks `readFileState` for the file path. Three failure modes are distinct error codes:

1. **No read in session** — `errorCode: 2`
   > "File has not been read yet. Read it first before writing to it."

2. **Read was partial (offset/limit)** — same `errorCode: 2`, same message
   Partial reads are flagged `isPartialView: true` — Write refuses them to prevent overwriting content that was never seen.

3. **File modified after read** — `errorCode: 3`
   > "File has been modified since read, either by the user or by a linter. Read it again before attempting to write it."

**Windows special case:** On Windows, mtime can change without content changes (cloud sync, antivirus). For full reads, Write falls back to comparing actual content before refusing. Only partial reads are rejected on timestamp alone.

### Line-ending policy

Write always persists content with LF line endings, regardless of the old file's endings or the repo's conventions. This is intentional. A previous approach that preserved or inferred line endings silently corrupted bash scripts when overwriting a CRLF file on Linux, or when binaries in the cwd poisoned a repo-wide ripgrep sample.

### Atomic write sequence

The critical section between reading current content and writing to disk is kept as synchronous as possible to prevent concurrent edits from interleaving:

```
// 1. mkdir (async, before critical section — safe)
await fs.mkdir(dir)

// 2. Backup for file history (async, keyed on content hash — idempotent)
await fileHistoryTrackEdit(...)

// 3. Sync read + staleness check (critical section starts)
meta = readFileSyncWithMetadata(fullFilePath)
if (lastWriteTime > lastRead.timestamp) throw FILE_UNEXPECTEDLY_MODIFIED_ERROR

// 4. Write to disk (critical section ends)
writeTextContent(fullFilePath, content, enc, 'LF')
```

### What happens after a successful write

- **LSP notifications:** `didChange` + `didSave` sent to all active language servers so diagnostics refresh immediately.
- **VSCode diff:** `notifyVscodeFileUpdated()` fires so the diff panel updates.
- **readFileState updated:** The new mtime and content are recorded so a subsequent Write in the same turn won't fail a staleness check.
- **CLAUDE.md telemetry:** If the written path ends in `/CLAUDE.md`, a `tengu_write_claudemd` event is logged.
- **Skill discovery:** The path is checked for skill directories that should be loaded (fire-and-forget, non-blocking).

---

## 04 Edit Tool Deep-Dive

Edit performs an exact string replacement: find `old_string` in the file and replace it with `new_string`. Only the diff is transmitted — far cheaper than Write for small changes.

### String replacement mechanics

The search is a direct substring match. If the string appears more than once, Edit refuses with error `9` unless `replace_all: true` is passed. This forces Claude to provide enough surrounding context to be unambiguous.

**Unambiguous — narrow match:**
```
Edit({
  file_path: "/src/config.ts",
  old_string: "const MAX_RETRIES = 3",
  new_string: "const MAX_RETRIES = 5",
})
```

**Replace all occurrences of a variable name:**
```
Edit({
  file_path: "/src/utils.ts",
  old_string: "oldFunctionName",
  new_string: "newFunctionName",
  replace_all: true,
})
```

### Quote normalization

Claude cannot output curly (typographic) quotes — the API sanitizes them. The Edit tool solves this with a two-step process:

**File on disk (curly quotes):**
```
const msg = \u201cHello, world\u201d
// \u201cHello\u201d uses U+201C/U+201D
```

**Claude's old_string (straight quotes):**
```
const msg = "Hello, world"
// Claude can only type ASCII "
```

`findActualString()` normalizes both the file and the search string to straight quotes, locates the match, and returns the original curly-quote version from the file. Then `preserveQuoteStyle()` applies the same curly-quote style to `new_string`, so the file's typography is preserved after the edit.

**normalizeQuotes() under the hood:**
```
str
  .replaceAll('\u2018', "'")  // left single curly
  .replaceAll('\u2019', "'")  // right single curly
  .replaceAll('\u201C', '"')  // left double curly
  .replaceAll('\u201D', '"')  // right double curly
```

### Contraction detection

`preserveQuoteStyle()` doesn't blindly convert every `'` to a curly quote. When a single quote is flanked by two Unicode letters (e.g., `don't`, `it's`), it is treated as an apostrophe and gets a right single curly quote rather than an opening one. The heuristic uses `/\p{L}/u` for correct Unicode letter detection.

### New file creation via Edit

Setting `old_string: ""` on a non-existent file creates it. Setting it on an existing empty file replaces it. Setting it on a non-empty file fails with "Cannot create new file — file already exists." This makes the intent explicit.

### Staleness check (same as Write)

Edit shares Write's staleness logic. The critical section reads the file synchronously, checks mtime against `readFileState`, and throws `FILE_UNEXPECTEDLY_MODIFIED_ERROR` if the file changed after the last read. The content-comparison fallback for full reads on Windows applies here too.

### The desanitization table

Claude's API sanitizes certain XML-like tokens to prevent prompt injection (e.g., `<function_results>` becomes `<fnr>`). If an exact match fails, Edit tries a desanitized version of `old_string` before reporting "String to replace not found". The full mapping:

```
'<fnr>'           -> '<function_results>'
'<n>'             -> '<name>'
'<o>'             -> '<output>'
'<e>'             -> '<error>'
'\n\nH:'          -> '\n\nHuman:'
'\n\nA:'          -> '\n\nAssistant:'
// ...and several more special tokens
```

The same replacements are applied to `new_string` so the edited result is consistent with the found match.

### Trailing whitespace stripping

Before applying an edit, `normalizeFileEditInput()` strips trailing whitespace from each line of `new_string`. This prevents a common model error where lines end with invisible spaces. Exception: `.md` and `.mdx` files are skipped — Markdown uses two trailing spaces as a hard line break, and stripping them would silently change semantics.

---

## 05 Write vs Edit: Side-by-Side

The same change — updating a constant from 3 to 5 — done both ways. Edit is almost always the right choice for existing files.

### Write — full file replacement

```
// Must include the ENTIRE file content
Write({
  file_path: "/src/config.ts",
  content: `import { z } from 'zod'

export const config = {
  maxRetries: 5,   // changed
  timeout: 3000,
  endpoint: "https://api.example.com",
}
`
})
// Sends the whole file every time.
// 150+ lines -> 150+ lines of tokens.
```

### Edit — surgical replacement

```
// Only the changed fragment needed
Edit({
  file_path: "/src/config.ts",
  old_string: "maxRetries: 3,",
  new_string: "maxRetries: 5,",
})
// Sends ~20 characters.
// Costs a fraction of Write.
```

### When to use Write instead of Edit

- Creating a **new file from scratch**
- Replacing file content with a **structurally different version** (e.g. complete rewrite of a config)
- The changes are so extensive that building a unique `old_string` would require most of the file anyway

---

## 06 The Read-Before-Write Contract

Both Write and Edit enforce the same three-phase contract. Understanding it prevents the most common tool errors.

### Phase 1: Read the file

This stores `{ content, timestamp, offset, limit }` in `readFileState`. A partial read (`isPartialView: true`) does NOT satisfy the requirement — you must read the whole file.

### Phase 2: validateInput() runs (pre-permission)

Checks readFileState. Rejects if no prior read, partial read, or mtime newer than last read. On Windows, also compares content for full reads before rejecting.

### Phase 3: call() runs the atomic check again

The staleness check runs a second time inside `call()` with a synchronous read to close the race window between `validateInput` and the actual write. If the file changed in between, `FILE_UNEXPECTEDLY_MODIFIED_ERROR` is thrown.

### Phase 4: Write/Edit executes and updates readFileState

New mtime and content are stored so the next write in the same session doesn't fail.

### Common gotcha

A linter or formatter running on file save can modify the file between Claude's Read and Write calls. This triggers the "file has been modified since read" error. The fix: re-read the file after formatting completes, or configure the linter to not auto-save during Claude's session.

---

## Key Takeaways

- Read is the only read-only, concurrency-safe tool. Write and Edit both require a full prior Read of any existing file.
- Read's dedup system (~18% hit rate) avoids re-sending unchanged files by returning a ~100-byte stub when mtime matches the cached timestamp.
- The 25 000-token cap on Read uses a fast rough estimate first; only if suspicious does it make an API call for an exact count.
- Write truncation was tested and reverted: throwing at the 256 KB gate is cheaper than sending 25K tokens of content the model can't use anyway.
- Edit's quote normalization handles the API's sanitization of curly quotes — `findActualString()` searches with straight quotes, but writes back curly ones to preserve file typography.
- Both Write and Edit run a staleness check twice: once in `validateInput` (pre-permission) and once in `call()` (atomic, synchronous) to close the race window.
- PDFs require an explicit `pages` range for files over 10 pages; max is 20 pages per call.
- Jupyter notebooks are read natively by `Read`; they cannot be edited by the Edit tool — use `NotebookEdit` instead.

---

## Check Your Understanding

### Q1. You call `Read` with `offset: 500, limit: 100` on a 2000-line file, then immediately call `Write` to update it. What happens?

- The write succeeds — Read was called, so the requirement is met.
- **The write fails — partial reads set `isPartialView: true` and do not satisfy the read-before-write requirement.**
- The write succeeds but only updates lines 500-600.
- The write fails only if the file has more than 1000 lines.

### Q2. An Edit call fails with "Found 3 matches of the string to replace." What is the correct fix?

- Add more surrounding context to `old_string` to make it uniquely identify one occurrence.
- **Set `replace_all: true` if all occurrences should change, or add more context to `old_string` for a single replacement.**
- Use Write instead, which has no uniqueness constraint.
- Read the file again — this error always clears after a fresh read.

### Q3. Why did the team revert the "truncate instead of throw" approach for files exceeding the 256 KB Read gate?

- Truncation caused data loss in the file on disk.
- **Truncation produced ~25K tokens of content at the cap while throwing produced a ~100-byte error, so truncation wasted context without reducing errors.**
- The token-estimation API does not support partial content.
- Truncation changed the line-ending encoding of the file.

### Q4. You have a file with the content `const title = \u201cHello, World\u201d` where the quotes are curly (U+201C/U+201D). You run Edit with `old_string: 'const title = "Hello, World"'` (straight ASCII quotes). What happens?

- Edit fails — the exact string is not found because the quotes don't match.
- **Edit succeeds. `findActualString()` normalizes both sides to straight quotes for searching, finds the match, then preserves curly quotes in the result.**
- Edit succeeds but replaces the curly quotes with ASCII quotes in the file.
- Edit succeeds only if `replace_all: true` is set.

### Q5. What does the dedup mechanism in Read return when a file has not changed since the last read?

- An empty string — the model should check its memory.
- The full file content again — dedup only applies to binary files.
- **A `file_unchanged` type result with a short stub message pointing the model to the earlier Read result.**
- An error asking the model to use the Grep tool instead.

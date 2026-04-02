# mdENG — Lesson 16 — Settings & Configuration

## Overview & Structure

This lesson covers Claude Code's five-layer settings cascade system, which merges configuration from independent sources to support diverse deployment contexts—from solo developer laptops to enterprise IT-managed fleets.

## The 5-Layer Cascade

Settings merge in priority order from lowest to highest:

1. **userSettings** (`~/.claude/settings.json`) — Global developer preferences, writable
2. **projectSettings** (`.claude/settings.json`) — Repo-committed, team-shared, version-controlled
3. **localSettings** (`.claude/settings.local.json`) — Personal per-project overrides, auto-gitignored
4. **flagSettings** (`--settings <path>`) — CLI flag with optional inline SDK settings, read-only
5. **policySettings** — IT/MDM enforced, uses first-source-wins internally

Key invariant: The cascade is read-only at runtime. Only the first three layers are writable via `updateSettingsForSource()`.

## Policy Settings Sub-Cascade (First-Source-Wins)

Unlike other layers that merge together, policySettings applies exactly one internal source:

1. Remote Anthropic API (Enterprise/Team + Console users)
2. MDM (macOS plist: `/Library/Managed Preferences/com.anthropic.claudecode.plist`; Windows HKLM)
3. File-based (`managed-settings.json` + drop-in directory)
4. HKCU registry (Windows only, user-writable)

## SettingsSchema Validation

The Zod v4 schema validates every settings source before merging. Invalid files surface errors but don't crash—remaining valid configuration loads normally.

### Selected Top-Level Configuration Keys

| Key | Type | Purpose |
|-----|------|---------|
| `permissions` | object | allow/deny/ask arrays, defaultMode, disableBypassPermissionsMode |
| `hooks` | object | PreToolUse, PostToolUse, Notification, SessionStart, Stop handlers |
| `env` | record<string,string> | Environment variables injected into sessions |
| `model` | string | Override default Claude model |
| `availableModels` | string[] | Enterprise allowlist of selectable models |
| `allowedMcpServers` | object[] | Enterprise MCP server allowlist |
| `deniedMcpServers` | object[] | Enterprise MCP server denylist |
| `apiKeyHelper` | string | Script path emitting authentication values |
| `cleanupPeriodDays` | number | Transcript retention in days (0 = disable) |
| `strictPluginOnlyCustomization` | bool \| string[] | Lock skills/agents/hooks/mcp to plugin-only |
| `allowManagedHooksOnly` | boolean | Only managed hooks run when set in policy |
| `allowManagedPermissionRulesOnly` | boolean | Only managed permission rules apply in policy |
| `attribution` | object | Customize commit/PR attribution text |
| `sandbox` | object | Sandbox configuration (enabled, network, filesystem) |
| `worktree` | object | symlinkDirectories, sparsePaths for --worktree flag |

### Backward-Compatibility Guarantees

The codebase enforces strict rules:
- **Allowed:** Adding optional fields, new enum values, more permissive validation
- **Forbidden:** Removing fields, removing enum values, making optional fields required, renaming keys
- `.passthrough()` on permissions objects preserves unknown fields
- `filterInvalidPermissionRules()` strips bad rules before validation so one defective rule doesn't null the entire file

### Example Settings File

```json
{
  "$schema": "https://json.schemastore.org/claude-code-settings.json",
  "model": "claude-opus-4-5",
  "env": {
    "NODE_ENV": "development",
    "LOG_LEVEL": "debug"
  },
  "permissions": {
    "defaultMode": "acceptEdits",
    "allow": ["Bash(git *)", "Read(**)"],
    "deny": ["Bash(rm -rf *)"]
  },
  "hooks": {
    "PreToolUse": [{
      "matcher": "Bash",
      "hooks": [{ "type": "command", "command": "echo pre-bash" }]
    }]
  },
  "cleanupPeriodDays": 7
}
```

## Merge Semantics

Claude Code uses lodash `mergeWith()` with a custom `settingsMergeCustomizer`:

```typescript
export function settingsMergeCustomizer(
  objValue: unknown,
  srcValue: unknown,
): unknown {
  if (Array.isArray(objValue) && Array.isArray(srcValue)) {
    return uniq([...objValue, ...srcValue])  // deduplicated concatenation
  }
  return undefined  // let lodash handle objects / scalars
}
```

### Merge Rules

- **Objects** — Deep-merged; higher-layer keys override lower-layer ones
- **Arrays** — Concatenated and deduplicated (permissions rules from all enabled sources accumulate)
- **Scalars** — Higher layer wins
- **Deletion via `updateSettingsForSource()`** — Pass `undefined` as value; customizer detects and calls `delete object[key]`

**Critical distinction:** When loading sources into the cascade, arrays are merged. When _writing_ via `updateSettingsForSource()`, arrays are _replaced_—callers must compute desired final array state before passing it in.

## The 3-Tier Cache

Claude Code reads settings synchronously on the critical startup path. Three caching layers prevent redundant disk I/O and Zod re-parsing:

| Cache | Keyed by | Holds | Invalidated by |
|-------|----------|-------|----------------|
| `sessionSettingsCache` | (singleton) | Fully merged SettingsWithErrors | `resetSettingsCache()` |
| `perSourceCache` | SettingSource | Per-source SettingsJson \| null | `resetSettingsCache()` |
| `parseFileCache` | File path string | Parsed { settings, errors } | `resetSettingsCache()` |

All three are cleared atomically by `resetSettingsCache()`. This single call is the "fan-out" step in change detection: one cache reset means one disk reload regardless of subscriber count.

**Clone-on-read:** `parseSettingsFile()` clones before returning from cache to prevent callers from inadvertently mutating cache entries via `mergeWith()` mutations.

## Change Detection

The `settingsChangeDetector` watches files with chokidar and polls MDM registries/plists every 30 minutes.

### Key Constants

```typescript
const FILE_STABILITY_THRESHOLD_MS = 1000   // wait for write to stabilize
const FILE_STABILITY_POLL_INTERVAL_MS = 500  // chokidar awaitWriteFinish
const INTERNAL_WRITE_WINDOW_MS = 5000        // suppress own writes
const MDM_POLL_INTERVAL_MS = 30 * 60 * 1000  // 30 min MDM poll
const DELETION_GRACE_MS = 1700              // absorb delete-and-recreate
```

### Internal Writes — Suppressing Self-Triggered Loops

When Claude Code writes settings (e.g., saving a permission rule), it calls `markInternalWrite(filePath)` before writing. When chokidar fires the change event, `consumeInternalWrite(path, 5000)` detects internal origin and silently skips reload, preventing cascade loops.

### Delete-and-Recreate Grace Period

Auto-updaters and editors often delete then recreate files atomically. A 1700ms grace period absorbs this: if a file is deleted but an `add` or `change` event arrives within the window, deletion is cancelled and treated as a normal change.

### fanOut() — Single-Producer Pattern

Before centralization, each subscriber called `resetSettingsCache()` defensively, causing N cache clears per N subscribers per change. The fix centralizes reset in a single `fanOut()` function:

```typescript
function fanOut(source: SettingSource): void {
  resetSettingsCache()        // one clear
  settingsChanged.emit(source)  // N subscribers; first pays disk miss
                                // subsequent ones hit repopulated cache
}
```

## MDM & File-Based Managed Settings

### Platform Paths for managed-settings.json

| Platform | Base Path | Drop-in Dir |
|----------|-----------|-------------|
| macOS | `/Library/Application Support/ClaudeCode/managed-settings.json` | `/Library/Application Support/ClaudeCode/managed-settings.d/` |
| Windows | `C:\Program Files\ClaudeCode\managed-settings.json` | `C:\Program Files\ClaudeCode\managed-settings.d\` |
| Linux | `/etc/claude-code/managed-settings.json` | `/etc/claude-code/managed-settings.d/` |

### Drop-in Directory Merge Order

The `managed-settings.d/` directory enables multiple teams to deliver independent policy fragments without editing shared files. Files are sorted **alphabetically** and merged in order—later files override earlier ones (systemd/sudoers convention).

```typescript
// settings.ts — loadManagedFileSettings()
const { settings } = parseSettingsFile(getManagedSettingsFilePath())

const entries = fs.readdirSync(dropInDir)
  .filter(d => d.isFile() && d.name.endsWith('.json'))
  .map(d => d.name)
  .sort()  // alphabetical: 10-otel.json, 20-security.json ...

for (const name of entries) {
  merged = mergeWith(merged, parseSettingsFile(join(dropInDir, name)).settings, ...)
}
```

### MDM Raw Read — Startup Parallelism

Reading macOS plist (`plutil -convert json`) and Windows registry (`reg query`) spawns subprocesses fired early during startup before module initialization completes, allowing parallel execution with module loading.

```typescript
export function startMdmSettingsLoad(): void {
  mdmLoadPromise = (async () => {
    const rawPromise = getMdmRawReadPromise() ?? fireRawRead()
    const { mdm, hkcu } = consumeRawReadResult(await rawPromise)
    mdmCache = mdm
    hkcuCache = hkcu
  })()
}
```

## Remote Managed Settings

For Enterprise/Team subscribers and all Console (API key) users, Claude Code fetches settings from the Anthropic API at startup—the highest-priority sub-source within policySettings.

### Eligibility

- **Console users (API key):** always eligible
- **OAuth users — Enterprise or Team:** eligible
- **OAuth users — unknown subscription type:** eligible (API returns empty for ineligible orgs)
- **Third-party API provider or custom base URL:** ineligible
- **Cowork (`local-agent` entrypoint):** ineligible

### Fetch Lifecycle

Initial load is synchronous via cached settings. The API call happens asynchronously to unblock waiters immediately. Background polling occurs every 60 minutes.

### ETag-Based Caching (Checksum)

To avoid re-downloading unchanged settings on every startup, SHA-256 checksums of cached settings are sent as `If-None-Match` headers. The server returns `304 Not Modified` if unchanged.

```typescript
export function computeChecksumFromSettings(settings: SettingsJson): string {
  const sorted = sortKeysDeep(settings)
  const normalized = jsonStringify(sorted)    // no spaces after separators
  const hash = createHash('sha256').update(normalized).digest('hex')
  return `sha256:${hash}`
}
```

The checksum algorithm matches the server's Python implementation: `json.dumps(settings, sort_keys=True, separators=(",", ":"))`

### Fail-Open Design

- Failed fetch with cached file -> use stale cache
- Failed fetch without cache -> proceed without remote settings
- Auth errors (401/403) -> do not retry
- Network/timeout errors -> retry up to 5 times with exponential backoff
- 204/404 -> no settings configured -> delete stale cache file

### Security Check for Dangerous Changes

When remote settings differ from cache, `checkManagedSettingsSecurity()` runs before applying. If incoming settings are dangerous (new hooks, changed permission rules), users are prompted for approval. Rejection preserves previous cached settings.

### Background Polling

After initial load, background polling checks the API every 60 minutes (`POLLING_INTERVAL_MS = 60 * 60 * 1000`). Settings changes are detected via JSON string comparison, calling `settingsChangeDetector.notifyChange('policySettings')` only if changed. The interval uses `.unref()` to not prevent process exit.

## Settings Sync (CCR)

Settings sync (orthogonal to remote managed settings) syncs a _user's own_ settings across environments—primarily between interactive CLI and Claude Code in Repositories (CCR) headless mode.

| Direction | Trigger | What is Synced |
|-----------|---------|----------------|
| Upload (CLI -> cloud) | Interactive CLI startup (`preAction`) | user settings.json, user CLAUDE.md, local settings.local.json, project CLAUDE.local.md |
| Download (cloud -> CCR) | CCR headless startup (before plugin install) | Same four files, keyed by file path + git remote hash |

### Sync Keys

```typescript
export const SYNC_KEYS = {
  USER_SETTINGS:  '~/.claude/settings.json',
  USER_MEMORY:    '~/.claude/CLAUDE.md',
  projectSettings: (projectId: string) =>
    `projects/${projectId}/.claude/settings.local.json`,
  projectMemory:  (projectId: string) =>
    `projects/${projectId}/CLAUDE.local.md`,
}
```

Project-specific keys are scoped by SHA of the git remote URL, preventing settings for different repos from overwriting each other.

**Incremental upload:** The upload path fetches the remote copy first, computes a diff, and only sends keys with changed values, keeping PUT requests small.

**Size limit:** Each file is capped at 500 KB (`MAX_FILE_SIZE_BYTES`). Files exceeding this are silently skipped on both upload and download.

## Security Constraints

Several settings keys intentionally exclude `projectSettings` from their trust domain to prevent malicious repos from injecting dangerous settings automatically.

| Setting / Check | Sources Trusted | Why projectSettings is Excluded |
|-----------------|-----------------|--------------------------------|
| `skipDangerousModePermissionPrompt` | user, local, flag, policy | Repo could auto-bypass danger-mode dialog (RCE risk) |
| `skipAutoPermissionPrompt` | user, local, flag, policy | Auto-mode opt-in must be user-driven |
| `useAutoModeDuringPlan` | user, local, flag, policy | Plan-mode semantics are safety-critical |
| `autoMode` classifier config | user, local, flag, policy | Injecting allow/deny rules via repo is RCE vector |

**allowManagedPermissionRulesOnly** — When set to `true` in managed settings, _only_ allow/deny/ask rules from managed settings are respected. All user, project, local, and CLI argument permission rules are silently ignored. This is the enterprise control for preventing employees from granting themselves additional permissions.

**allowManagedHooksOnly** — Similar lockdown for hooks. Only hooks defined in managed settings will execute, preventing users from installing hooks that exfiltrate data or bypass auditing.

## Key Takeaways

- Settings merge from 5 layers: **user -> project -> local -> flag -> policy**. Later layers override earlier ones; arrays are deduplicated-concatenated.
- **policySettings uses first-source-wins internally**: remote beats MDM (plist/HKLM) beats managed-settings.json beats HKCU. Only the first non-empty source applies.
- The **3-tier cache** keeps startup fast. A single `resetSettingsCache()` invalidates all three and is called atomically before notifying subscribers.
- **Change detection** uses chokidar for user/project/local/policy files plus 30-minute MDM polling. Internal writes are suppressed with a 5-second window to prevent reload loops.
- **Remote managed settings** are highest-priority policy sub-source, fetched at startup with ETag caching, retried up to 5 times, polled hourly. The system always fails open.
- **Settings sync** is orthogonal to remote managed settings, syncing _user's own_ settings files (not enterprise policy) between CLI and CCR using cloud key-value store, scoped by git remote hash.
- Security-sensitive flags (`skipDangerousModePermissionPrompt`, auto-mode opt-in, classifier rules) intentionally exclude `projectSettings` to prevent malicious repo privilege escalation.
- The **backward-compatibility contract** forbids removing fields or enum values. New fields must be optional. Invalid individual permission rules are stripped before validation.

## Quiz

**Q1:** A setting is defined in both userSettings and projectSettings. Which value wins?
- **Answer:** projectSettings, because it comes later in SETTING_SOURCES array (higher priority)

**Q2:** Both macOS MDM plist and managed-settings.json file exist. Which supplies policy settings?
- **Answer:** The MDM plist — policySettings uses first-source-wins, and MDM (index 2) beats the file (index 3)

**Q3:** What happens when Claude Code writes a settings file itself?
- **Answer:** It calls `markInternalWrite()` before writing, so chokidar's event is suppressed within 5 seconds

**Q4:** A project's `.claude/settings.json` sets `skipDangerousModePermissionPrompt: true`. Will Claude Code respect this?
- **Answer:** No — projectSettings is intentionally excluded from trusted sources for this flag (RCE risk)

**Q5:** userSettings has `permissions.allow: ["Read(**)", "Bash(git *)"]` and projectSettings has `permissions.allow: ["Bash(git *)", "Write(src/)"]`. What is the merged allow list?
- **Answer:** `["Read(**)", "Bash(git *)", "Write(src/)"]` — arrays are concatenated and deduplicated

**Q6:** Remote managed settings are fetched with an `If-None-Match` header. What value is sent?
- **Answer:** A `sha256:` checksum computed from cached settings JSON with keys sorted alphabetically

**Q7:** What triggers the `managed-settings.d/` drop-in directory merge, and in what order?
- **Answer:** managed-settings.json is the base, then drop-in files are merged alphabetically on top (later filenames win)

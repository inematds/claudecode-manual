# Sandbox & Secure Storage — Source Deep Dive

## Lesson 23: Sandbox & Secure Storage

## Overview

This lesson covers how Claude Code confines AI-generated shell commands using OS-level isolation and protects credentials with platform-native keychain storage. The system prevents sandboxed processes from reading SSH keys, exfiltrating files, or writing to settings before execution.

### Three Core Parts

**Part A: Sandbox Adapter**
Bridges `@anthropic-ai/sandbox-runtime` to Claude Code's settings and tool system.

**Part B: Network & Filesystem Control**
Domain allow/deny lists, filesystem allow/deny paths, and how permission rules map to sandbox configuration.

**Part C: Secure Storage**
macOS Keychain plus plaintext fallback for OAuth tokens and API keys, with stale-while-error caching.

### Source Files Covered

- `utils/sandbox/sandbox-adapter.ts`
- `utils/sandbox/sandbox-ui-utils.ts`
- `commands/sandbox-toggle/index.ts`
- `commands/sandbox-toggle/sandbox-toggle.tsx`
- `components/sandbox/SandboxSettings.tsx`
- `components/sandbox/SandboxDependenciesTab.tsx`
- `components/sandbox/SandboxOverridesTab.tsx`
- `utils/secureStorage/index.ts`
- `utils/secureStorage/macOsKeychainStorage.ts`
- `utils/secureStorage/macOsKeychainHelpers.ts`
- `utils/secureStorage/keychainPrefetch.ts`
- `utils/secureStorage/fallbackStorage.ts`
- `utils/secureStorage/plainTextStorage.ts`

---

## Sandbox Architecture

Claude Code wraps `@anthropic-ai/sandbox-runtime` (the `BaseSandboxManager`) through a thin adapter that adds settings integration, worktree awareness, and Claude-specific security rules. The public surface is a single `SandboxManager` object.

The adapter converts local settings into `SandboxRuntimeConfig`, which `BaseSandboxManager` enforces via platform-specific backends.

### Platform Backends

| Platform | Backend | Dependencies |
|----------|---------|--------------|
| macOS | Apple Seatbelt (`sandbox-exec`) | Built-in — zero install. Only `ripgrep` needed. |
| Linux / WSL2 | bubblewrap (`bwrap`) + seccomp + socat | `apt install bubblewrap socat` + seccomp BPF filter from `@anthropic-ai/sandbox-runtime` |
| WSL1 / Windows | Not supported | `isSupportedPlatform()` returns false; /sandbox command hidden |

### Platform Detection and Dependency Checks

Platform support and dependency health are memoized for the process lifetime using `memoize()` from lodash-es, so subsequent calls hit cache rather than spawning subprocesses.

```typescript
// sandbox-adapter.ts — memoized dependency check
const checkDependencies = memoize((): SandboxDependencyCheck => {
  const { rgPath, rgArgs } = ripgrepCommand()
  return BaseSandboxManager.checkDependencies({
    command: rgPath,
    args: rgArgs,
  })
})

// memoized platform check
const isSupportedPlatform = memoize((): boolean => {
  return BaseSandboxManager.isSupportedPlatform()
})
```

On Linux, three separate dependencies are checked: `bwrap` (containerization), `socat` (network proxy plumbing), and a seccomp BPF filter. Seccomp is a _warning_ (not an error) — sandbox can run without it, but Unix domain socket blocking is unavailable.

```typescript
// SandboxDependenciesTab.tsx — Linux dep check display
// bwrap: bubblewrap (required)
// socat: network proxy (required)
// seccomp filter: optional — blocks unix domain sockets
bwrapMissing   && "apt install bubblewrap"
socatMissing   && "apt install socat"
seccompMissing && "npm install -g @anthropic-ai/sandbox-runtime"
```

---

## The Three Sandbox Modes

Users select sandbox behavior through `/sandbox`, which renders `SandboxSettings` with three choices:

### Mode 1: No Sandbox (disabled)
All Bash commands run unsandboxed. Claude must ask permission for every new command pattern. Default state.

### Mode 2: Regular (sandboxed)
Commands run inside OS isolation. Claude still asks before executing commands outside the pre-approved list.

### Mode 3: Auto-Allow (sandboxed)
Commands auto-approved without a prompt because the sandbox provides safety guarantees. Maximum productivity.

Mode is derived from two boolean settings: `sandbox.enabled` and `sandbox.autoAllowBashIfSandboxed`. There is also a `sandbox.allowUnsandboxedCommands` flag that controls whether commands explicitly excluded from sandboxing (`sandbox.excludedCommands`) are still permitted to run outside the sandbox.

### Mode Selection Logic

```typescript
// SandboxSettings.tsx
type SandboxMode = 'auto-allow' | 'regular' | 'disabled'

const getCurrentMode = (): SandboxMode => {
  if (!currentEnabled) return "disabled"
  if (currentAutoAllow) return "auto-allow"
  return "regular"
}
```

When the user picks a mode, `setSandboxSettings()` is called with a combination of `enabled`, `autoAllowBashIfSandboxed`, and optionally `allowUnsandboxedCommands` — all written to `localSettings` (i.e. `.claude/settings.local.json` in the project directory).

```typescript
// sandbox-adapter.ts — mapping mode → settings
// auto-allow:  { enabled: true,  autoAllowBashIfSandboxed: true  }
// regular:     { enabled: true,  autoAllowBashIfSandboxed: false }
// disabled:    { enabled: false }
async function setSandboxSettings(options: {
  enabled?: boolean
  autoAllowBashIfSandboxed?: boolean
  allowUnsandboxedCommands?: boolean
}): Promise<void> {
  updateSettingsForSource('localSettings', {
    sandbox: {
      ...existingSettings?.sandbox,
      ...options,
    },
  })
}
```

### Excluding Specific Commands

Some tools — like Docker daemon management or hardware-touching commands — cannot run inside the sandbox. The `/sandbox exclude "pattern"` command appends a pattern to `sandbox.excludedCommands` in local settings. The adapter also exposes `addToExcludedCommands()` for programmatic exclusion after a sandbox violation event.

```typescript
// commands/sandbox-toggle/sandbox-toggle.tsx — /sandbox exclude subcommand
if (subcommand === 'exclude') {
  const cleanPattern = commandPattern.replace(/^["']|["']$/g, '')
  addToExcludedCommands(cleanPattern)
  // writes to localSettings: sandbox.excludedCommands
}
```

### Policy Lock: Enterprise-Managed Sandbox Settings

When `flagSettings` or `policySettings` explicitly set any sandbox key, those settings take priority over `localSettings` and the user cannot override them locally. `areSandboxSettingsLockedByPolicy()` detects this and the `/sandbox` command surfaces an error:

```typescript
// sandbox-adapter.ts — policy lock detection
function areSandboxSettingsLockedByPolicy(): boolean {
  const overridingSources = ['flagSettings', 'policySettings'] as const
  for (const source of overridingSources) {
    const settings = getSettingsForSource(source)
    if (
      settings?.sandbox?.enabled !== undefined ||
      settings?.sandbox?.autoAllowBashIfSandboxed !== undefined ||
      settings?.sandbox?.allowUnsandboxedCommands !== undefined
    ) return true
  }
  return false
}
```

There is also an undocumented `sandbox.enabledPlatforms` array — added for an NVIDIA enterprise rollout — that lets admins restrict sandboxing to specific platforms (e.g. `["macos"]`) without disabling it globally.

---

## Network Control

The sandbox enforces network access at the OS level. Claude Code translates its permission rules and settings into a `NetworkRestrictionConfig` that `BaseSandboxManager` enforces via the platform backend.

### Domain Allow/Deny Lists

Allowed domains come from two sources that are merged together:

1. `sandbox.network.allowedDomains` — explicit list in settings JSON
2. `permissions.allow` rules of the form `WebFetch(domain:example.com)`

Denied domains come from `permissions.deny` rules following the same `WebFetch(domain:...)` pattern.

```typescript
// sandbox-adapter.ts — convertToSandboxRuntimeConfig()
for (const ruleString of permissions.allow || []) {
  const rule = permissionRuleValueFromString(ruleString)
  if (rule.toolName === WEB_FETCH_TOOL_NAME &&
      rule.ruleContent?.startsWith('domain:')) {
    allowedDomains.push(rule.ruleContent.substring('domain:'.length))
  }
}
// Same pattern for deniedDomains using permissions.deny
```

### allowManagedDomainsOnly — Enterprise Network Lockdown

When `policySettings.sandbox.network.allowManagedDomainsOnly` is `true`, _only_ domains from `policySettings` are used — user-level allow rules in `localSettings` are ignored entirely. The sandbox ask-callback is also wrapped to silently block unapproved outbound connections without prompting:

```typescript
// sandbox-adapter.ts — initialize()
const wrappedCallback: SandboxAskCallback = async (hostPattern) => {
  if (shouldAllowManagedSandboxDomainsOnly()) {
    logForDebugging(
      `[sandbox] Blocked network request to ${hostPattern.host} (allowManagedDomainsOnly)`
    )
    return false  // silently deny
  }
  return sandboxAskCallback(hostPattern)
}
```

### Additional Network Settings

| Setting Key | Type | Purpose |
|-------------|------|---------|
| `sandbox.network.allowUnixSockets` | `string[]` | Specific Unix domain socket paths that are allowed |
| `sandbox.network.allowAllUnixSockets` | `boolean` | Allow all Unix sockets (disables seccomp socket filter) |
| `sandbox.network.allowLocalBinding` | `boolean` | Allow binding to local ports (e.g. for dev servers) |
| `sandbox.network.httpProxyPort` | `number` | Override the HTTP proxy port the sandbox uses |
| `sandbox.network.socksProxyPort` | `number` | Override the SOCKS proxy port the sandbox uses |

---

## Filesystem Control

The filesystem configuration is built from four lists — `allowWrite`, `denyWrite`, `allowRead`, `denyRead` — that are assembled from multiple sources and passed to `BaseSandboxManager`.

### Always-Allowed Write Paths

Two paths are unconditionally writable regardless of user settings: the current working directory (`'.'`) and the Claude temp directory. This is the minimum required for any useful work.

```typescript
// sandbox-adapter.ts — baseline writable paths
const allowWrite: string[] = ['.', getClaudeTempDir()]
```

### Always-Denied Write Paths (Security Hardcoded)

Several paths are unconditionally denied to prevent sandbox escape via settings manipulation:

- All `settings.json` and `settings.local.json` files for every settings source
- The managed settings drop-in directory
- `.claude/skills` in both original cwd and current cwd — skills run unsandboxed and have full Claude capabilities
- Bare git repo sentinel files (`HEAD`, `objects`, `refs`, `hooks`, `config`) — to block the git `core.fsmonitor` escape vector (CVE pattern, tracked as issue #29316)

```typescript
// sandbox-adapter.ts — settings escape prevention
const settingsPaths = SETTING_SOURCES.map(source =>
  getSettingsFilePathForSource(source)
).filter((p): p is string => p !== undefined)
denyWrite.push(...settingsPaths)

// skills escape prevention
denyWrite.push(resolve(originalCwd, '.claude', 'skills'))
```

### The Bare Git Repo Sandbox Escape — Issue #29316

Git's `is_git_directory()` treats any directory containing `HEAD + objects/ + refs/` as a bare repository. If a sandboxed command plants those files, git's next _unsandboxed_ run could load a `core.fsmonitor` hook from an attacker-controlled `config` file, escaping the sandbox entirely.

The adapter handles this with a two-prong defense:

1. If a sentinel file _already exists_ at cwd, add it to `denyWrite` (bubblewrap ro-binds the real file in place — no stub).
2. If a sentinel file does _not_ exist, add its path to `bareGitRepoScrubPaths` and call `scrubBareGitRepoFiles()` after every command to delete anything planted.

```typescript
// sandbox-adapter.ts — post-command scrub
function scrubBareGitRepoFiles(): void {
  for (const p of bareGitRepoScrubPaths) {
    try {
      rmSync(p, { recursive: true })
    } catch {
      // ENOENT is the expected case — nothing was planted
    }
  }
}
```

### Path Resolution: Permission Rules vs. sandbox.filesystem

Two different path-resolution conventions exist side-by-side, and mixing them up is a documented footgun (issue #30067):

| Source | `/path` means | Resolver |
|--------|---------------|----------|
| Permission rules (`Edit(…)`, `Read(…)`) | Relative to the settings file's directory | `resolvePathPatternForSandbox()` |
| `sandbox.filesystem.allowWrite` etc. | Absolute (as written), `~` expanded | `resolveSandboxFilesystemPath()` |
| Both | `//path` → absolute `/path` (legacy compat) | Both resolvers handle `//` prefix |

```typescript
// resolvePathPatternForSandbox — permission-rule convention
// "/foo/**" → "${settingsRootDir}/foo/**"
if (pattern.startsWith('/') && !pattern.startsWith('//')) {
  const root = getSettingsRootPathForSource(source)
  return resolve(root, pattern.slice(1))
}

// resolveSandboxFilesystemPath — sandbox.filesystem convention
// "/Users/foo/.cargo" → "/Users/foo/.cargo" (absolute, as written)
return expandPath(pattern, getSettingsRootPathForSource(source))
```

### Git Worktree Write Access

In a git worktree, `.git` is a file (not a directory) pointing to the main repo's `.git/worktrees/name`. Bash commands run in the worktree need write access to the _main_ repo's `.git` directory for `index.lock` and similar files.

`initialize()` calls `detectWorktreeMainRepoPath()` once and caches the result for the session. That path is then added to `allowWrite`:

```typescript
// sandbox-adapter.ts
if (worktreeMainRepoPath && worktreeMainRepoPath !== cwd) {
  allowWrite.push(worktreeMainRepoPath)
}
```

### Live Config Refresh

The sandbox config is not static. Every time the user updates a permission or settings file, a `settingsChangeDetector` subscription fires `refreshConfig()`, which calls `BaseSandboxManager.updateConfig(newConfig)` synchronously. This means granting a new file-edit permission takes effect immediately — no restart needed.

```typescript
// sandbox-adapter.ts — live settings subscription
settingsSubscriptionCleanup = settingsChangeDetector.subscribe(() => {
  const settings = getSettings_DEPRECATED()
  const newConfig = convertToSandboxRuntimeConfig(settings)
  BaseSandboxManager.updateConfig(newConfig)
})
```

---

## Secure Storage

Claude Code stores OAuth tokens (and legacy API keys) through a `SecureStorage` interface with two concrete implementations: macOS Keychain and plaintext fallback. The adapter selects the right implementation at runtime and composes them with a `createFallbackStorage()` combinator.

### macOS Keychain Storage

Credentials are stored in the macOS Keychain as a generic password item using Apple's `security(1)` command-line tool. The service name is derived from a stable hash of the config directory path so multiple Claude instances with different `CLAUDE_CONFIG_DIR` values get separate keychain entries:

```typescript
// macOsKeychainHelpers.ts — service name derivation
export function getMacOsKeychainStorageServiceName(
  serviceSuffix: string = '',
): string {
  const isDefaultDir = !process.env.CLAUDE_CONFIG_DIR
  const dirHash = isDefaultDir
    ? ''
    : `-${createHash('sha256').update(configDir).digest('hex').substring(0, 8)}`
  return `Claude Code${OAUTH_FILE_SUFFIX}${serviceSuffix}${dirHash}`
}
```

The credential JSON is serialized to hex before being written. This avoids escaping issues in `security(1)`'s argument parser, and also defeats naive plaintext-grep rules in enterprise endpoint security tools (e.g. CrowdStrike):

```typescript
// macOsKeychainStorage.ts — write path
const hexValue = Buffer.from(jsonString, 'utf-8').toString('hex')

// Prefer stdin so process monitors see only "security -i", not the payload
const command =
  `add-generic-password -U -a "${username}" -s "${serviceName}" -X "${hexValue}"\n`

if (command.length <= SECURITY_STDIN_LINE_LIMIT) {
  result = execaSync('security', ['-i'], { input: command })
} else {
  // Payload too large for stdin buffer (4032B limit): fall back to argv
  result = execaSync('security', ['add-generic-password', '-U', ...])
}
```

### Stdin Buffer Limit

Apple's `security -i` reads stdin with a 4096-byte buffer. Payloads larger than ~4032 bytes (with 64 bytes of headroom) silently corrupt the write — the first 4096 bytes are consumed as a malformed command, the overflow is ignored. Claude Code detects this and falls back to argv, which has no practical size limit on macOS (ARG_MAX is 1 MB).

### Keychain Cache: Stale-While-Error and Generation Tracking

Each `security(1)` spawn costs ~500ms synchronously. With 50+ MCP connectors authenticating at startup, a naive implementation would stall the event loop for 25+ seconds. Claude Code solves this with a 30-second TTL cache and stale-while-error semantics:

```typescript
// macOsKeychainHelpers.ts
export const KEYCHAIN_CACHE_TTL_MS = 30_000

export const keychainCacheState: {
  cache: { data: SecureStorageData | null; cachedAt: number }
  generation: number  // incremented on every invalidation
  readInFlight: Promise<SecureStorageData | null> | null
} = { cache: { data: null, cachedAt: 0 }, generation: 0, readInFlight: null }
```

**Stale-while-error:** if a `security` call fails but the cache already has data, the stale data is served rather than returning null (which would surface as "Not logged in").

**Generation counter:** incremented on every `clearKeychainCache()`. An async read captures the generation before spawning; if a newer generation exists when it completes, the result is discarded — preventing a stale subprocess from overwriting fresh data written by a concurrent `update()`.

**readInFlight dedup:** concurrent `readAsync()` calls during a TTL miss share one subprocess, not N.

### Startup Prefetch

`keychainPrefetch.ts` fires two `security` subprocesses in parallel at the very top of `main.tsx`, before the ~65ms of module evaluation. By the time the app's async initializers need credentials, the results are already cached and the sync `security` spawn is avoided entirely.

```typescript
// keychainPrefetch.ts — parallel startup reads
export function startKeychainPrefetch(): void {
  if (process.platform !== 'darwin' || prefetchPromise || isBareMode()) return

  // Fire both subprocesses immediately — they run in parallel with
  // each other AND with main.tsx import evaluation (~65ms saved)
  const oauthSpawn  = spawnSecurity(getMacOsKeychainStorageServiceName(CREDENTIALS_SERVICE_SUFFIX))
  const legacySpawn = spawnSecurity(getMacOsKeychainStorageServiceName())

  prefetchPromise = Promise.all([oauthSpawn, legacySpawn]).then(([oauth, legacy]) => {
    if (!oauth.timedOut)  primeKeychainCacheFromPrefetch(oauth.stdout)
    if (!legacy.timedOut) legacyApiKeyPrefetch = { stdout: legacy.stdout }
  })
}
```

### Plaintext Fallback Storage

On Linux, or when the macOS Keychain is unavailable (e.g. an SSH session), credentials fall back to `~/.config/claude/.credentials.json` with permissions set to `0o600`. A warning is emitted on every write:

```typescript
// plainTextStorage.ts — write path
writeFileSync_DEPRECATED(storagePath, jsonStringify(data), { encoding: 'utf8' })
chmodSync(storagePath, 0o600)  // owner-only read/write
return {
  success: true,
  warning: 'Warning: Storing credentials in plaintext.',
}
```

### Fallback Storage Combinator

On macOS, `createFallbackStorage(primary, secondary)` wraps both implementations with carefully ordered migration logic:

```typescript
// fallbackStorage.ts — update() with migration
update(data: SecureStorageData) {
  const primaryDataBefore = primary.read()
  const result = primary.update(data)

  if (result.success) {
    // First-time migration to keychain: delete secondary (plaintext)
    // so the stale .credentials.json doesn't shadow the fresh keychain entry
    if (primaryDataBefore === null) secondary.delete()
    return result
  }

  // Keychain write failed — use plaintext fallback
  const fallbackResult = secondary.update(data)
  if (fallbackResult.success) {
    // Best-effort: remove stale keychain entry so it can't shadow
    // the fresh plaintext data (login loop fix, issue #30337)
    if (primaryDataBefore !== null) primary.delete()
    return { success: true, warning: fallbackResult.warning }
  }

  return { success: false }
}
```

The key invariant: `read()` prefers primary whenever it returns non-null. If the keychain holds a stale token and plaintext holds a fresh one, the stale token wins — causing a /login loop. The combinator prevents this by deleting the stale primary entry after a successful fallback write.

---

## Sandbox UI and Violation Reporting

When a sandboxed command is blocked, the violation event is stored in `SandboxViolationStore`. Two UI surfaces consume this:

- **stderr annotation:** `annotateStderrWithSandboxFailures()` appends a human-readable explanation to the command's stderr when a violation matches.
- **SandboxDoctorSection:** rendered in the assistant's response area, showing which path or domain was blocked with suggestions on how to allow it.

Violation tag stripping is also needed when displaying error messages in the UI — the `<sandbox_violations>` XML block that appears in raw stderr is removed for display:

```typescript
// sandbox-ui-utils.ts — strip violation XML from display text
export function removeSandboxViolationTags(text: string): string {
  return text.replace(
    /<sandbox_violations>[\s\S]*?<\/sandbox_violations>/g,
    '',
  )
}
```

### Startup Warning for Misconfigured Sandbox

If a user sets `sandbox.enabled: true` but dependencies are missing, `getSandboxUnavailableReason()` returns a human-readable reason string. Claude Code surfaces this once at startup (not silently ignoring the broken setting) because a user configuring `allowedDomains` expects enforcement — silent failure is a security footgun. (Fix for issue #34044.)

---

## Key Takeaways

- The sandbox adapter is a **thin bridge** — Claude Code translates its settings/permission system into `SandboxRuntimeConfig` and delegates all actual process isolation to `@anthropic-ai/sandbox-runtime`.

- macOS uses Apple Seatbelt (zero install); Linux/WSL2 uses bubblewrap + seccomp + socat. WSL1 and Windows are unsupported.

- Three modes: **disabled**, **regular** (sandboxed, ask permission), **auto-allow** (sandboxed, no prompts). Stored as two booleans in local settings.

- Network access is enforced via `allowedDomains` / `deniedDomains`, assembled from both `WebFetch` permission rules and explicit `sandbox.network` settings.

- Settings files and `.claude/skills` are **always denied write access** to prevent sandbox escape via config injection or skill injection.

- Two path-resolution conventions coexist: permission rules treat `/path` as settings-relative; `sandbox.filesystem` treats `/path` as absolute. Mixing them up causes silent breakage (issue #30067).

- Bare git repo files are scrubbed after each command to block the `core.fsmonitor` sandbox escape vector.

- On macOS, credentials live in the Keychain serialized as hex. A 30-second TTL cache with stale-while-error and generation tracking avoids 500ms-per-spawn event loop stalls.

- The fallback storage combinator deletes stale primary entries after a successful fallback write to prevent /login loops (issue #30337).

- Keychain reads are prefetched at process start in parallel with module evaluation — saving ~65ms on every macOS launch.

---

## Knowledge Check

**1. Which OS-level mechanism does Claude Code use for sandboxing on macOS?**
- A) bubblewrap (bwrap) + seccomp
- B) Apple Seatbelt (sandbox-exec)
- C) Docker container isolation
- D) Node.js vm.runInContext

**Answer: B**

**2. What does "auto-allow" sandbox mode do differently from "regular" mode?**
- A) It removes all network restrictions
- B) It disables the sandbox entirely
- C) It allows commands to execute without a permission prompt, relying on sandbox safety
- D) It grants write access to all files in the home directory

**Answer: C**

**3. Why are `settings.json` files unconditionally added to `denyWrite`?**
- A) To prevent Claude from logging its own configuration
- B) To prevent a sandboxed command from escaping the sandbox by modifying permission settings
- C) Because settings files are read-only in the OS
- D) To improve file I/O performance

**Answer: B**

**4. What is the significant difference between `resolvePathPatternForSandbox()` and `resolveSandboxFilesystemPath()` when given a path like `/Users/foo/.cargo`?**
- A) They produce identical output for absolute paths
- B) `resolvePathPatternForSandbox` treats it as settings-relative; `resolveSandboxFilesystemPath` treats it as absolute
- C) `resolveSandboxFilesystemPath` rejects absolute paths
- D) `resolvePathPatternForSandbox` only works with glob patterns

**Answer: B**

**5. Why does `macOsKeychainStorage.update()` serialize credentials to hex before writing to the Keychain?**
- A) To encrypt the credentials at rest
- B) To reduce storage size via compression
- C) To avoid shell escaping issues and defeat naive plaintext-grep rules in endpoint security software
- D) Because the Keychain API only accepts hex values

**Answer: C**

**6. What happens in `createFallbackStorage` when the keychain write fails but the keychain already holds stale data, and the plaintext fallback write succeeds?**
- A) The stale keychain entry is left in place
- B) The stale keychain entry is deleted so it doesn't shadow the fresh plaintext data
- C) Both entries are deleted and the user must log in again
- D) The plaintext write is rolled back

**Answer: B**

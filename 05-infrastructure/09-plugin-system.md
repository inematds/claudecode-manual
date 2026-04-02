# mdENG Lesson 09: The Claude Code Plugin System — Complete Lesson Content

## 01 Overview

The plugin system is the extensibility backbone of Claude Code. It lets third-party authors (and Anthropic itself) ship slash commands, MCP servers, AI agents, lifecycle hooks, LSP integrations, skills, and output styles — all from a Git repository or npm package — without touching the core binary.

### Source files covered

`utils/plugins/` — schemas.ts · pluginLoader.ts · marketplaceManager.ts · dependencyResolver.ts · pluginVersioning.ts · zipCache.ts · pluginAutoupdate.ts · pluginBlocklist.ts · pluginPolicy.ts · reconciler.ts  
  
`services/plugins/` — PluginInstallationManager.ts · pluginOperations.ts · pluginCliCommands.ts

### Five Conceptual Layers

**Layer 1: Marketplace Sources**
GitHub repos, git URLs, npm packages, local directories, URL-hosted JSON — where plugins come from.

**Layer 2: Manifest Schema**
`plugin.json` declares what a plugin exports: commands, hooks, MCP servers, LSP, skills, agents.

**Layer 3: Versioned Cache**
`~/.claude/plugins/cache/{mkt}/{plugin}/{version}/` — immutable per-version snapshots with seed fallback.

**Layer 4: Dependency Resolution**
DFS closure walk at install-time; fixed-point demote pass at load-time. Cross-marketplace deps are blocked.

**Layer 5: Lifecycle**
Background reconcile → autoupdate → load → hook registration → command registration → MCP connect.

## 02 Plugin Lifecycle Diagram

The diagram traces a full plugin journey from user intent to active capability through five phases:

**STARTUP (non-blocking):**
- getDeclaredMarketplaces() — Merge settings + --add-dir sources
- diffMarketplaces() — Compare declared vs known_marketplaces.json
- Decision point: Missing or source changed?
- reconcileMarketplaces() — git clone / http fetch / npm install
- Auto-refresh plugins or set needsRefresh flag

**AUTOUPDATE (Background):**
- getAutoUpdateEnabledMarketplaces() — Official = true by default
- refreshMarketplace() — git pull / re-fetch
- updatePluginsForMarketplaces() — updatePluginOp() per installation
- Notify REPL via onPluginsAutoUpdated()

**INSTALL (user-triggered):**
- parseMarketplaceInput() — Resolve 'name@marketplace'
- isPluginBlockedByPolicy()
- resolveDependencyClosure() — DFS walk, cycle detect, cross-mkt block
- For each dep in closure: getPluginById() from marketplace
- installPlugin() — clone / fetch / npm
- calculatePluginVersion() — 1. plugin.json 2. provided 3. git SHA 4. 'unknown'
- copyPluginToVersionedCache() — ~/.claude/plugins/cache/{mkt}/{plugin}/{ver}/
- updateSettingsForSource() — enablePlugins[id] = true

**LOAD (cache-only, per session):**
- loadAllPluginsCacheOnly() — Read settings.enabledPlugins
- verifyAndDemote() — Fixed-point dep check, demote broken
- detectAndUninstallDelistedPlugins() — Auto-uninstall removed marketplace entries
- loadPluginManifest() — Parse plugin.json via PluginManifestSchema
- Resolve paths: commands/, agents/, skills/, hooks/, mcpServers, lspServers

**REGISTER:**
- getPluginCommands() — Parse .md files → namespaced /plugin:cmd
- getPluginSkills() — Scan skills/ for SKILL.md subdirs
- loadPluginHooks() — Convert to PluginHookMatcher[]
- loadPluginAgents() — Register agent .md files
- mcpPluginIntegration — Connect MCP servers, inject userConfig
- lspPluginIntegration — Register LSP servers

Flow: STARTUP → AUTOUPDATE → LOAD → REGISTER
Also: INSTALL → LOAD → REGISTER

## 03 Marketplace Sources

A marketplace is a catalog — a `marketplace.json` file that lists plugins. Claude Code supports six source types for fetching that catalog:

### GitHub Repo
Clones `owner/repo` over SSH (or HTTPS in remote mode). Official marketplace uses `anthropics/claude-plugins-official`.

### Arbitrary Git URL
HTTPS, SSH (`git@`), or `file://`. Validated before clone to prevent injection.

### Monorepo Subdirectory
Partial clone (`--filter=tree:0`) + sparse-checkout. Version includes a path hash to prevent collisions.

### HTTP/HTTPS URL
Fetches the marketplace JSON directly. GCS fallback for the official marketplace via `officialMarketplaceGcs.ts`.

### npm Package
Installed to a shared `npm-cache/node_modules/` then copied. Not supported in zip-cache mode.

### Local Path
Points directly to a local directory or JSON file. Excluded from zip-cache (path lives outside cache dir).

### Official Marketplace

The official marketplace (`claude-plugins-official`) is _implicitly declared_ — if any enabled plugin references it, Claude Code auto-clones `anthropics/claude-plugins-official` on first launch. No user configuration needed.

### Reserved name protection

The schema enforces two layers of impersonation defense. The `BLOCKED_OFFICIAL_NAME_PATTERN` regex blocks names like `official-claude-plugins` or `anthropic-marketplace-v2`. For reserved names in the allowlist (like `claude-plugins-official`), the source org must be `anthropics/` on GitHub.

### Deep dive — name validation code

```typescript
// schemas.ts — blocked pattern for impersonation attempts
export const BLOCKED_OFFICIAL_NAME_PATTERN =
  /(?:official[^a-z0-9]*(anthropic|claude)|(?:anthropic|claude)[^a-z0-9]*official|^(?:anthropic|claude)[^a-z0-9]*(marketplace|plugins|official))/i

// Non-ASCII blocks homograph attacks (Cyrillic 'а' ≠ Latin 'a')
const NON_ASCII_PATTERN = /[^\u0020-\u007E]/

export function validateOfficialNameSource(name, source): string | null {
  // Reserved name? Must come from github.com/anthropics/
  if (source.source === 'github') {
    const repo = source.repo || ''
    if (!repo.toLowerCase().startsWith(`anthropics/`)) {
      return `The name '${name}' is reserved for official Anthropic marketplaces.`
    }
  }
  return null
}
```

## 04 The Manifest Schema (`plugin.json`)

Every plugin optionally ships a `plugin.json` at its root. Claude Code parses this with a strict Zod schema. Unknown top-level keys are silently stripped (resilience to future fields); unknown keys inside nested objects like `userConfig` or `channels` still fail validation.

| Field | Type | Notes |
|-------|------|-------|
| name | string | Kebab-case, no spaces. Used for namespacing commands as `/plugin:cmd`. |
| version | string? | Semver. Highest priority version source — overrides git SHA. |
| description | string? | User-facing description shown in `/plugin list`. |
| dependencies | string[]? | Bare names inherit declaring plugin's marketplace. Cross-marketplace blocked by default. |
| commands | path \| path[] \| Record<name, metadata> | Supplement `commands/` directory. Object form supports inline content. |
| hooks | path \| HooksConfig \| array | Supplement `hooks/hooks.json`. All 20+ lifecycle events supported. |
| mcpServers | path \| McpbPath \| Record \| array | Supports `.mcpb`/`.dxt` bundles or inline server config objects. |
| lspServers | path \| Record \| array | Each server: `command`, `extensionToLanguage` map, transport, env, timeouts. |
| agents | path \| path[] | Additional agent `.md` files beyond `agents/` directory. |
| skills | path \| path[] | Additional skill directories. Each must contain `SKILL.md`. |
| outputStyles | path \| path[] | Output style definitions for custom rendering. |
| channels | ChannelDecl[] | Messaging channels (Telegram, Slack, etc.). Binds an MCP server + prompts for userConfig. |
| userConfig | Record<key, Option> | User-configurable values prompted at enable time. Sensitive values go to keychain. |
| settings | Record? | Settings to merge when plugin enabled. Only allowlisted keys kept (currently: `agent`). |
| author / homepage / repository / license / keywords | metadata | Discovery and attribution fields. |

### Full annotated plugin.json example

```json
{
  "name": "my-plugin",
  "version": "1.2.0",
  "description": "Example plugin showing all capability types",
  "author": { "name": "Acme Corp", "url": "https://acme.example" },
  "license": "MIT",

  // Declare a dependency — bare name inherits this plugin's marketplace
  "dependencies": ["shared-utils"],

  // Commands: directory is auto-scanned + extra file + inline content
  "commands": {
    "hello": { "content": "Say hello to ${CLAUDE_PLUGIN_ROOT}" },
    "deploy": { "source": "./docs/deploy.md", "argumentHint": "[env]" }
  },

  // Hooks: inline or path
  "hooks": "./hooks/extra.json",

  // MCP server via .mcpb bundle (pre-packaged binary)
  "mcpServers": "./server.mcpb",

  // LSP server for TypeScript
  "lspServers": {
    "typescript": {
      "command": "typescript-language-server",
      "args": ["--stdio"],
      "extensionToLanguage": { ".ts": "typescript", ".tsx": "typescriptreact" }
    }
  },

  // User-configurable values (prompted at enable time)
  "userConfig": {
    "API_KEY": {
      "type": "string",
      "title": "API Key",
      "description": "Your service API key",
      "sensitive": true,   // → stored in keychain, not settings.json
      "required": true
    }
  }
}
```

### Plugin directory convention

Without any `plugin.json` fields, a plugin is still valid. Claude Code auto-discovers `commands/*.md`, `agents/*.md`, `skills/*/SKILL.md`, `hooks/hooks.json`, and `.mcp.json` by convention. The manifest only needs to exist when you want to opt into non-convention paths or declare metadata.

## 05 Versioned Cache

Plugins are installed into an immutable content-addressed cache under `~/.claude/plugins/cache/`. The path structure is:

```
# Path format
~/.claude/plugins/cache/{marketplace}/{plugin}/{version}/

# Example
~/.claude/plugins/cache/claude-plugins-official/sql-helper/1.3.2/
~/.claude/plugins/cache/acme-mkt/deploy-tool/a3f8c1d94b12/
~/.claude/plugins/cache/acme-mkt/monorepo-plugin/a3f8c1d94b12-e5a7c3f1/
                                                          ^^ path hash for git-subdir
```

### Version priority order

The version string is calculated in `pluginVersioning.ts` with this priority:

1. **plugin.json `version` field** — explicit semver, highest authority
2. **Provided version** from marketplace entry (e.g., pinned in `marketplace.json`)
3. **Pre-resolved git SHA** — captured before the clone is discarded (git-subdir case)
4. **Git commit SHA** read from `.git/HEAD` in the install path (first 12 chars)
5. **`'unknown'`** as last resort — still creates a valid cache path

### git-subdir path hashing

For monorepo plugins (`source: "git-subdir"`), two plugins at different subdirectories of the same commit would collide at the same SHA path. The version becomes `{sha12}-{pathHash8}` where the path hash is a normalized SHA-256 of the subdir. This normalization matches the squashfs cron byte-for-byte: backslash→forward slash, strip leading `./`, strip trailing `/`.

### Seed cache fallback

Enterprise deployments can pre-populate a read-only seed directory (set via `CLAUDE_CODE_PLUGIN_SEED_DIR`). On install, the loader probes seeds before performing any network fetch. This enables zero-network-on-first-run scenarios.

### Zip cache mode

When `CLAUDE_CODE_PLUGIN_USE_ZIP_CACHE=1` is set (headless/container deployments), plugins are stored as `.zip` files instead of directories on a mounted Filestore. At session start they are extracted to a `claude-plugin-session-{hex}/` temp dir and cleaned up on exit. Unix execute bits are preserved via the ZIP central directory's `external_attr` field — critical for hooks and scripts that need `+x`.

### Deep dive — versioned path computation

```typescript
// pluginLoader.ts
export function getVersionedCachePathIn(
  baseDir: string,
  pluginId: string,
  version: string,
): string {
  const { name: pluginName, marketplace } = parsePluginIdentifier(pluginId)
  // Sanitize each segment to prevent path traversal
  const sanitizedMarketplace = (marketplace || 'unknown').replace(/[^a-zA-Z0-9\-_]/g, '-')
  const sanitizedPlugin    = (pluginName    || pluginId).replace(/[^a-zA-Z0-9\-_]/g, '-')
  const sanitizedVersion   = version.replace(/[^a-zA-Z0-9\-_.]/g, '-')
  return join(baseDir, 'cache', sanitizedMarketplace, sanitizedPlugin, sanitizedVersion)
}

// pluginVersioning.ts — git-subdir path hash
const normPath = source.path
  .replace(/\\/g, '/')    // backslash → forward slash
  .replace(/^\.\//, '')    // strip leading ./
  .replace(/\/+$/, '')     // strip trailing /
const pathHash = createHash('sha256').update(normPath).digest('hex').substring(0, 8)
const v = `${shortSha}-${pathHash}`
```

## 06 Dependency Resolution

The plugin system has two dependency resolution passes with distinct semantics, implemented in `dependencyResolver.ts`:

### Install-time: `resolveDependencyClosure()`

A recursive DFS walk that computes the full transitive closure of plugins to install. It enforces three rules:

- **Cycle detection** — if a plugin appears on the current DFS stack, return `{ reason: 'cycle' }`
- **Cross-marketplace block** — plugin A from marketplace X cannot auto-install plugin B from marketplace Y (unless Y is on X's `allowCrossMarketplaceDependenciesOn` allowlist)
- **Already-enabled skip** — deps already in settings are skipped to avoid clobbering version pins, but the root is never skipped (handles "cache cleared but still in settings" case)

### Install-time DFS walk — source excerpt

```typescript
export async function resolveDependencyClosure(
  rootId: PluginId,
  lookup: (id: PluginId) => Promise<DependencyLookupResult | null>,
  alreadyEnabled: ReadonlySet<PluginId>,
  allowedCrossMarketplaces: ReadonlySet<string> = new Set(),
): Promise<ResolutionResult> {
  const closure: PluginId[] = []
  const visited = new Set<PluginId>()
  const stack: PluginId[] = []  // DFS stack for cycle detection

  async function walk(id, requiredBy) {
    // Skip already-enabled DEPS (not root) to avoid stomping version pins
    if (id !== rootId && alreadyEnabled.has(id)) return null

    // Cross-marketplace security gate
    const idMkt = parsePluginIdentifier(id).marketplace
    if (idMkt !== rootMarketplace && !allowedCrossMarketplaces.has(idMkt)) {
      return { ok: false, reason: 'cross-marketplace', dependency: id, requiredBy }
    }

    if (stack.includes(id)) return { ok: false, reason: 'cycle', chain: [...stack, id] }
    if (visited.has(id)) return null
    visited.add(id)

    const entry = await lookup(id)
    if (!entry) return { ok: false, reason: 'not-found', missing: id, requiredBy }

    stack.push(id)
    for (const rawDep of entry.dependencies ?? []) {
      const dep = qualifyDependency(rawDep, id)  // inherit marketplace if bare name
      const err = await walk(dep, id)
      if (err) return err
    }
    stack.pop()
    closure.push(id)   // post-order: deps come before dependents
    return null
  }

  const err = await walk(rootId, rootId)
  if (err) return err
  return { ok: true, closure }
}
```

### Load-time: `verifyAndDemote()`

Run on every session start from the cache-only loaded set. It is a **fixed-point loop**: demoting plugin A (because its dep is missing) may uncover that plugin B depends on A, so B must also be demoted. The loop repeats until no changes occur.

### apt-style semantics

Claude Code's dependency model is inspired by Debian's apt: dependencies are _presence guarantees_, not module imports. Plugin B depending on plugin A means "A's namespaced MCP servers, commands, and agents must be available when B runs" — not that B's code imports A's code.

### Dependency Name Resolution Flow Diagram

```
Bare name 'shared-utils' with declaring plugin acme@acme-mkt 
  → 'shared-utils@acme-mkt' (qualified)

Already qualified 'shared-utils@acme-mkt' 
  → 'shared-utils@acme-mkt' (unchanged)

Bare name 'shared-utils' from --plugin-dir plugin 
  → 'shared-utils' (unchanged with @inline sentinel)
```

## 07 Command, Skill, and Hook Loading

### Commands and skills

Commands live in `commands/*.md`. Each file becomes a slash command named `/plugin-name:command-name`. Subdirectories create namespaces: `commands/ci/build.md` → `/my-plugin:ci:build`.

Skills are directories containing `SKILL.md`. When Claude Code finds `SKILL.md` in a subdirectory of `skills/`, it registers the parent directory name as the skill name and injects `${CLAUDE_SKILL_DIR}` so the skill can reference its own support files.

### Variable substitution in commands and skills

At runtime, these variables are replaced in command/skill content:

| Variable | Resolves to |
|----------|-------------|
| ${CLAUDE_PLUGIN_ROOT} | Absolute path to plugin's installed directory |
| ${CLAUDE_PLUGIN_DATA} | Plugin's writable data directory |
| ${CLAUDE_SKILL_DIR} | This skill's specific subdirectory (skill mode only) |
| ${CLAUDE_SESSION_ID} | Current session identifier |
| ${user_config.KEY} | User-configured option (sensitive keys replaced with placeholder) |

### Hooks

Plugin hooks are converted to `PluginHookMatcher[]` objects and registered alongside the global hook config. Every hook event is supported:

PreToolUse | PostToolUse | PostToolUseFailure | PermissionDenied
Notification | UserPromptSubmit | SessionStart | SessionEnd | Stop | StopFailure
SubagentStart | SubagentStop | PreCompact | PostCompact
PermissionRequest | Setup | TeammateIdle
TaskCreated | TaskCompleted | Elicitation | ElicitationResult
ConfigChange | WorktreeCreate | WorktreeRemove
InstructionsLoaded | CwdChanged | FileChanged

### Hot reload

Hook loading subscribes to `settingsChangeDetector`. When `enabledPlugins` changes in settings (e.g., user enables/disables a plugin without restarting), plugin hooks are automatically reloaded. The snapshot comparison uses JSON serialization to detect changes.

## 08 Security and Policy

### Policy blocking (`managed-settings.json`)

Enterprise admins can force-disable any plugin by setting `enabledPlugins["name@marketplace"] = false` in `managed-settings.json`. The `isPluginBlockedByPolicy()` check runs at the install chokepoint, the enable operation, and in the UI — a single source of truth that cannot be overridden by user or project settings.

### Delisting and auto-uninstall

When a marketplace sets `forceRemoveDeletedPlugins: true`, Claude Code compares `installed_plugins.json` against the current marketplace manifest at startup. Plugins no longer listed are automatically uninstalled from all user-controlled scopes and recorded in a flagged-plugins file to prevent re-install loops.

### Installation scopes

Plugins can be installed at four scopes. Only the first three are user-installable:

**User scope**
`~/.claude/settings.json` — active in all projects for this user.

**Project scope**
`.claude/settings.json` in current project — committed to the repo.

**Local scope**
`.claude/settings.local.json` — project-specific, not committed.

**Managed scope**
Set by org admins via `managed-settings.json`. Read-only to users.

### Uninstall warning: reverse dependent detection

```typescript
// dependencyResolver.ts
export function findReverseDependents(
  pluginId: PluginId,
  plugins: readonly LoadedPlugin[],
): string[] {
  const { name: targetName } = parsePluginIdentifier(pluginId)
  return plugins
    .filter(p =>
      p.enabled &&
      p.source !== pluginId &&
      (p.manifest.dependencies ?? []).some(d => {
        const qualified = qualifyDependency(d, p.source)
        // Bare dep from @inline plugin: match by name only
        return parsePluginIdentifier(qualified).marketplace
          ? qualified === pluginId
          : qualified === targetName
      }),
    )
    .map(p => p.name)
}
// Result shown as: "warning: required by plugin-a, plugin-b"
```

## 09 Background Autoupdate

At startup, `autoUpdateMarketplacesAndPluginsInBackground()` runs silently without blocking the REPL. The flow has three phases:

1. Determine which marketplaces have `autoUpdate: true` — official marketplaces default to `true`, third-party default to `false`
2. Run `refreshMarketplace()` (git pull / re-fetch) for each auto-update marketplace
3. Call `updatePluginOp()` for each installed plugin from those marketplaces

Updates are **non-in-place**: the new version is cached at a new versioned path but the running session continues using the old path. The REPL is notified via `onPluginsAutoUpdated()` and shows a restart prompt.

### Race condition handling

The REPL may not yet be mounted when autoupdate completes. The module stores the update notification in `pendingNotification` and delivers it immediately when `onPluginsAutoUpdated()` is eventually called by the REPL. This prevents "updates finished before anyone was listening" silent drops.

## Key Takeaways

- A plugin is a directory (or ZIP) with a `plugin.json` manifest; the manifest is optional — convention-based auto-discovery handles most cases.
- Marketplaces are catalogs. Six source types are supported: github, git, git-subdir, url, npm, directory/file. The official Anthropic marketplace is implicitly declared from any plugin that references it.
- The versioned cache at `~/.claude/plugins/cache/{mkt}/{plugin}/{ver}/` is immutable per-version. git-subdir plugins encode a path hash in the version to prevent monorepo collisions.
- Dependency resolution uses two passes: a DFS closure walk at install-time (cross-marketplace blocked by default) and a fixed-point demote pass at load-time to catch broken deps at session start.
- Plugins are namespaced: commands become `/plugin-name:command`, hooks are tagged with `pluginId`, MCP server names are prefixed. This prevents collisions across plugins.
- Policy blocking (`managed-settings.json`) is enforced at install time, enable time, and in the UI — it cannot be overridden by user or project settings.
- Autoupdate is background-only and non-blocking. The current session sees old code; the new version is cached for the next restart.
- Sensitive `userConfig` values (marked `sensitive: true`) go to the OS keychain, not `settings.json`. They are available in MCP server env but never substituted into skill/agent content sent to the model.

## Knowledge Check

**Question 1:** When a plugin declares `"dependencies": ["shared-utils"]` and the plugin itself is from marketplace `acme-mkt`, what is the resolved dependency ID?

A) shared-utils  
B) shared-utils@acme-mkt  
C) shared-utils@inline  
D) acme-mkt/shared-utils

**Correct Answer: B**

**Question 2:** Plugin A enables OK. Plugin B depends on A. Later, A is removed from the marketplace and auto-uninstalled. What happens to B at the next session start?

A) B loads normally — dependency checking is install-time only  
B) B is demoted (disabled for this session) by `verifyAndDemote()`  
C) B is permanently uninstalled from all scopes  
D) B reinstalls A automatically

**Correct Answer: B**

**Question 3:** A plugin at `plugins/my-plugin/` in a monorepo is at git SHA `abc123def456` and the subdir path is `./plugins/my-plugin`. What does the version look like?

A) abc123def456  
B) abc123def456-plugins/my-plugin  
C) abc123def4-{8-char path hash}  
D) abc123

**Correct Answer: C**

**Question 4:** Which of the following is the correct path for an enterprise pre-populated plugin cache that prevents network fetches on first run?

A) CLAUDE_CODE_PLUGIN_CACHE_DIR — sets the zip cache directory  
B) CLAUDE_CODE_PLUGIN_SEED_DIR — a read-only fallback probed before network fetches  
C) CLAUDE_CODE_PLUGIN_OFFLINE_DIR — disables network for plugins  
D) Plugins cannot be pre-cached

**Correct Answer: B**

**Question 5:** A plugin sets `"sensitive": true` on a `userConfig` field called `API_KEY`. Where is the value stored, and can it appear in skill content?

A) settings.json; yes it can appear in skill content  
B) OS keychain or .credentials.json; yes it can appear in skill content  
C) OS keychain or .credentials.json; no — a descriptive placeholder is substituted instead  
D) Not stored — it must be set as an environment variable manually

**Correct Answer: C**

**Question 6:** What prevents a marketplace named `anthropic-marketplace-v2` from being registered by a third party?

A) Nothing — any name is allowed as long as it doesn't contain spaces  
B) BLOCKED_OFFICIAL_NAME_PATTERN regex matches names starting with "anthropic" + "marketplace"  
C) It's in the ALLOWED_OFFICIAL_MARKETPLACE_NAMES allowlist so source org is checked  
D) All anthropic-* names require manual review by Anthropic

**Correct Answer: B**

# Claude Code Migration System

## mdENG — Lesson 34 — Migration System — Source Deep Dive

**Subtitle:** How Claude Code upgrades user config and model settings across versions — without breaking anything or surprising users.

---

## 01 Overview

Every time Claude Code launches it may silently fix stale settings, remap deprecated model aliases, move fields between files, or re-surface UI dialogs — all before the first prompt appears. This is the **migration system**: a small but precisely designed set of one-shot functions that run inside `runMigrations()` in `main.tsx`.

**Source files covered:**
- `src/migrations/*.ts`
- `src/main.tsx` (lines 323-352)
- `src/utils/config.ts`
- `src/utils/releaseNotes.ts`

**Key distinction:** Migrations in Claude Code are **not database schema migrations**. There is no migration table, no rollback, and no runner framework. Instead, every migration function is idempotent by design — it detects its own pre/post conditions and exits immediately if the work is already done. The entire set runs at every startup, protected by a single version number that short-circuits the whole block once all migrations have been applied.

### Five Migration Types

**Type 1 — Settings promotions**
- Move fields from `~/.claude.json` (GlobalConfig) into `settings.json`

**Type 2 — Model alias upgrades**
- Remap stale or removed model strings to current aliases in `userSettings`

**Type 3 — Config key renames**
- Rename implementation-detail keys that leaked into public config

**Type 4 — One-shot resets**
- Clear flags to re-surface dialogs when the UX changes and users need a second chance to choose

**Type 5 — Async file migrations**
- Move config data into separate files (changelog cache) without blocking the UI

---

## 02 The Runner: `runMigrations()`

All synchronous migrations are wrapped in a single function defined in `main.tsx`. It is called during the Commander `preAction` hook — after config is loaded, before the REPL starts.

```typescript
// main.tsx — line 323
// @[MODEL LAUNCH]: Consider any migrations you may need for model strings.
// See migrateSonnet1mToSonnet45.ts for an example.

// Bump this when adding a new sync migration so existing users re-run the set.
const CURRENT_MIGRATION_VERSION = 11;

function runMigrations(): void {
  if (getGlobalConfig().migrationVersion !== CURRENT_MIGRATION_VERSION) {
    migrateAutoUpdatesToSettings();
    migrateBypassPermissionsAcceptedToSettings();
    migrateEnableAllProjectMcpServersToSettings();
    resetProToOpusDefault();
    migrateSonnet1mToSonnet45();
    migrateLegacyOpusToCurrent();
    migrateSonnet45ToSonnet46();
    migrateOpusToOpus1m();
    migrateReplBridgeEnabledToRemoteControlAtStartup();
    if (feature('TRANSCRIPT_CLASSIFIER')) {
      resetAutoModeOptInForDefaultOffer();
    }
    if ("external" === 'ant') {   // internal Anthropic builds only
      migrateFennecToOpus();
    }
    // Stamp the version so we skip next time
    saveGlobalConfig(prev => prev.migrationVersion === CURRENT_MIGRATION_VERSION
      ? prev
      : { ...prev, migrationVersion: CURRENT_MIGRATION_VERSION });
  }

  // Async migration — fire-and-forget, non-blocking
  migrateChangelogFromConfig().catch(() => {});
}
```

### Three Design Decisions

1. **Version gate:** `migrationVersion` is stored in `~/.claude.json`. Once it equals `CURRENT_MIGRATION_VERSION`, the entire sync block is skipped — avoiding 11 redundant `saveGlobalConfig` lock+re-read cycles on every startup.

2. **Version bump rule:** The comment explicitly states — bump the constant whenever you add a new sync migration so existing users re-run the full set.

3. **Async separation:** `migrateChangelogFromConfig` is async and involves file I/O, so it runs fire-and-forget after the sync block.

### Startup Placement

`runMigrations()` is called in the Commander `preAction` hook inside `main.tsx`, directly after `init()` and right before `loadRemoteManagedSettings()`. A `profileCheckpoint('preAction_after_migrations')` call immediately follows it for latency tracking.

---

## 03 Migration Catalogue

There are currently 11 sync migration functions plus one async one.

### Migration Details Table

| File | Category | What it does | Idempotency guard |
|------|----------|--------------|-------------------|
| `migrateAutoUpdatesToSettings.ts` | Settings | Moves a user-disabled `autoUpdates` flag from `GlobalConfig` into `userSettings.env.DISABLE_AUTOUPDATER = "1"`. Also sets `process.env` immediately so the change is live without a restart. | Skips if `globalConfig.autoUpdates !== false` or if the flag was set by native protection. Removes the field from GlobalConfig on success. |
| `migrateBypassPermissionsAcceptedToSettings.ts` | Settings | Moves `bypassPermissionsModeAccepted` out of GlobalConfig into `userSettings.skipDangerousModePermissionPrompt`. The old name leaked implementation details into the user-facing config file. | Skips if `bypassPermissionsModeAccepted` is not in GlobalConfig. Checks `hasSkipDangerousModePermissionPrompt()` before writing to avoid overwriting an existing value. |
| `migrateEnableAllProjectMcpServersToSettings.ts` | Settings | Moves three MCP server approval fields (`enableAllProjectMcpServers`, `enabledMcpjsonServers`, `disabledMcpjsonServers`) from the project config into `localSettings`. Merges array fields with deduplication to avoid losing existing data. | Skips if none of the three fields are present in project config. For `enableAllProjectMcpServers`, checks whether the target setting is already set before writing. |
| `resetProToOpusDefault.ts` | Reset | Records a timestamp (`opusProMigrationTimestamp`) for Pro subscribers on first-party who had no custom model set, so the UI can show a one-time notification that Opus 4.5 is now their default. | Completion flag: `globalConfig.opusProMigrationComplete`. Immediately marks complete for non-Pro or non-firstParty users. |
| `migrateSonnet1mToSonnet45.ts` | Model | Pins users who had `sonnet[1m]` saved in `userSettings` to the explicit `sonnet-4-5-20250929[1m]` string. Needed because Sonnet 4.6 1M was offered to a different user group than Sonnet 4.5 1M. Also updates the in-memory main loop model override if it is already set. | Completion flag: `globalConfig.sonnet1m45MigrationComplete`. |
| `migrateLegacyOpusToCurrent.ts` | Model | Rewrites explicit Opus 4.0/4.1 model strings (`claude-opus-4-20250514`, `claude-opus-4-1-20250805`, etc.) to the `opus` alias in `userSettings`. Also records `legacyOpusMigrationTimestamp` so the UI can show a one-time notification. Only runs for first-party users with legacy remap enabled. | Reads and writes the same source (`userSettings`), making it self-idempotent — after migration the model string no longer matches, so it exits early. |
| `migrateSonnet45ToSonnet46.ts` | Model | Upgrades Pro/Max/Team Premium users pinned to Sonnet 4.5 explicit strings back to the `sonnet` (or `sonnet[1m]`) alias, which now resolves to 4.6. Skips brand-new users (`numStartups <= 1`) to avoid showing a notification to people who never used 4.5. | Self-idempotent: only writes if `userSettings.model` matches a Sonnet 4.5 string. Gate: first-party + Pro/Max/TeamPremium only. |
| `migrateOpusToOpus1m.ts` | Model | For Max/Team Premium subscribers on first-party, upgrades `userSettings.model = 'opus'` to `'opus[1m]'` when the Opus 1M merge is enabled. If `opus[1m]` resolves to the same model as the default, it clears the field instead (no unnecessary pin). | Self-idempotent: only writes if model is exactly `'opus'`. Gate: `isOpus1mMergeEnabled()`. |
| `migrateReplBridgeEnabledToRemoteControlAtStartup.ts` | Config | Renames the internal config key `replBridgeEnabled` to `remoteControlAtStartup`. The old name leaked an implementation detail into the public config file. Uses an untyped cast to access a key no longer in the TypeScript type. | Skips if `replBridgeEnabled` is not present, or if `remoteControlAtStartup` is already set. |
| `resetAutoModeOptInForDefaultOffer.ts` | Reset | Clears `skipAutoPermissionPrompt` for users who accepted the old 2-option Auto Mode dialog but never set auto as their default mode. This re-surfaces the dialog so they see the new "make it my default" option. Only fires when the `TRANSCRIPT_CLASSIFIER` feature flag is active. | Completion flag: `globalConfig.hasResetAutoModeOptInForDefaultOffer`. Also guarded by `getAutoModeEnabledState() === 'enabled'`. |
| `migrateFennecToOpus.ts` | Model | Internal Anthropic only (`USER_TYPE === 'ant'`). Migrates removed "fennec" model aliases (`fennec-latest`, `fennec-latest[1m]`, `fennec-fast-latest`, `opus-4-5-fast`) to their Opus equivalents including fast mode. | Self-idempotent: only acts if model starts with a fennec prefix. Only writes `userSettings`. |
| `releaseNotes.ts: migrateChangelogFromConfig()` | Config (async) | Moves the cached changelog string from `globalConfig.cachedChangelog` into a separate file on disk. Runs async so it never blocks the UI. Uses `wx` write flag so it only creates the file if it does not already exist. | Skips if `globalConfig.cachedChangelog` is not set. File write uses `flag: 'wx'` (create-only) to avoid clobbering existing data. |

---

## 04 Two Idempotency Patterns

Every migration must be safe to call multiple times. Two patterns emerge across the codebase:

### Pattern A — Completion flag in GlobalConfig

Used when a migration needs to run exactly once but the "already done" state is not self-evident from the data (e.g. `resetProToOpusDefault`, `migrateSonnet1mToSonnet45`). A boolean or timestamp is written to `~/.claude.json` and checked at the top of the function.

```typescript
// migrateSonnet1mToSonnet45.ts — completion flag pattern
export function migrateSonnet1mToSonnet45(): void {
  const config = getGlobalConfig()
  if (config.sonnet1m45MigrationComplete) {
    return  // already done — exit immediately
  }

  const model = getSettingsForSource('userSettings')?.model
  if (model === 'sonnet[1m]') {
    updateSettingsForSource('userSettings', {
      model: 'sonnet-4-5-20250929[1m]',
    })
  }

  saveGlobalConfig(current => ({
    ...current,
    sonnet1m45MigrationComplete: true,
  }))
}
```

### Pattern B — Self-idempotent (data speaks for itself)

Used when the migration condition is a direct check on the current value — if the data has already been migrated, the check simply returns false and nothing is written (e.g. all model alias migrations). Reading and writing the same settings source (`userSettings`) is key to this pattern — the comment in `migrateLegacyOpusToCurrent.ts` explains: _"Reading and writing the same source keeps this idempotent without a completion flag, and avoids silently promoting 'opus' to the global default for users who only pinned it in one project."_

```typescript
// migrateLegacyOpusToCurrent.ts — self-idempotent pattern
export function migrateLegacyOpusToCurrent(): void {
  if (getAPIProvider() !== 'firstParty') return
  if (!isLegacyModelRemapEnabled()) return

  const model = getSettingsForSource('userSettings')?.model
  if (
    model !== 'claude-opus-4-20250514' &&
    model !== 'claude-opus-4-1-20250805' &&
    model !== 'claude-opus-4-0' &&
    model !== 'claude-opus-4-1'
  ) {
    return  // data already clean — nothing to do
  }

  updateSettingsForSource('userSettings', { model: 'opus' })
  saveGlobalConfig(current => ({
    ...current,
    legacyOpusMigrationTimestamp: Date.now(),
  }))
  logEvent('tengu_legacy_opus_migration', { from_model: model })
}
```

---

## 05 Settings Layer Discipline

Claude Code has multiple settings sources that are merged in priority order: `userSettings` -> `projectSettings` -> `localSettings` -> `policySettings`. Migrations are deliberately scoped to only touch `userSettings` (and occasionally `localSettings`). The comment in nearly every model migration file repeats the same rationale:

### Key Constraint

"Only touches userSettings. Legacy strings in project/local/policy settings are left alone (we can't/shouldn't rewrite those) and are still remapped at runtime by `parseUserSpecifiedModel`. Reading and writing the same source keeps this idempotent without a completion flag, and avoids silently promoting 'opus' to the global default for users who only pinned it in one project."

This constraint prevents a subtle class of bugs: if a migration read from _merged_ settings, it might see a project-scoped setting and "helpfully" write that value into the global `userSettings`, suddenly making a per-project preference the new default everywhere.

### Settings Layer Flow Diagram

```
userSettings (~/.claude/settings.json)
  | (migration reads & writes HERE only)
  |
Migration function -> Merged Settings
  ^
projectSettings (.claude/settings.json)
  | (runtime merge only, never mutated)
  |
Merged Settings
  ^
localSettings (.claude/settings.local.json)
  | (some migrations write here [MCP])
  |
Merged Settings
  ^
policySettings (managed)
  | (runtime merge only, never mutated)
  |
Merged Settings
```

---

## 06 In-Memory Config Migration

Alongside the startup migration functions, `config.ts` contains a lower-level `migrateConfigFields()` that runs every time the config file is _read from disk_. This handles the oldest schema changes — before the current versioned migration system existed.

```typescript
// config.ts — runs on every config read
function migrateConfigFields(config: GlobalConfig): GlobalConfig {
  if (config.installMethod !== undefined) {
    return config  // already migrated
  }

  // autoUpdaterStatus is removed from the type but may exist in old configs
  const legacy = config as GlobalConfig & {
    autoUpdaterStatus?: 'migrated' | 'installed' | 'disabled' | 'enabled' | ...
  }

  switch (legacy.autoUpdaterStatus) {
    case 'migrated':  installMethod = 'local';   break
    case 'installed': installMethod = 'native';  break
    case 'disabled':  autoUpdates  = false;      break
    ...
  }

  return { ...config, installMethod, autoUpdates }
}
```

There is also a `removeProjectHistory()` function that strips the old inline `history` field from project configs on every read — this field was migrated to separate `history.jsonl` files, but old configs still carry the field.

### Pattern

Read-time migration (inline in `migrateConfigFields`) is used for very old schema changes where the old field name no longer exists in the TypeScript type — requiring an untyped cast to access it. New migrations go in the `migrations/` directory and run via `runMigrations()`.

---

## 07 Analytics Instrumentation

Migrations are observable. Most functions call `logEvent()` to record what happened to Anthropic's telemetry pipeline. This lets the team know when a migration is still being applied to users in the wild vs. when it is safe to remove the migration code.

### Analytics Events Table

| Event name | Migration |
|------------|-----------|
| `tengu_migrate_autoupdates_to_settings` | migrateAutoUpdatesToSettings |
| `tengu_migrate_autoupdates_error` | migrateAutoUpdatesToSettings (error path) |
| `tengu_migrate_bypass_permissions_accepted` | migrateBypassPermissionsAcceptedToSettings |
| `tengu_migrate_mcp_approval_fields_success` | migrateEnableAllProjectMcpServersToSettings |
| `tengu_migrate_mcp_approval_fields_error` | migrateEnableAllProjectMcpServersToSettings (error path) |
| `tengu_reset_pro_to_opus_default` | resetProToOpusDefault |
| `tengu_legacy_opus_migration` | migrateLegacyOpusToCurrent |
| `tengu_sonnet45_to_46_migration` | migrateSonnet45ToSonnet46 |
| `tengu_opus_to_opus1m_migration` | migrateOpusToOpus1m |
| `tengu_migrate_reset_auto_opt_in_for_default_offer` | resetAutoModeOptInForDefaultOffer |

**Note:** `migrateSonnet1mToSonnet45` and `migrateReplBridgeEnabledToRemoteControlAtStartup` do not emit analytics events — they are considered lower-risk housekeeping.

---

## 08 Adding a New Migration

The source code even contains a comment pointing future engineers in the right direction:

### Developer Note in main.tsx

`// @[MODEL LAUNCH]: Consider any migrations you may need for model strings. See migrateSonnet1mToSonnet45.ts for an example.`

### Recipe from the Existing Codebase

1. Create `src/migrations/myNewMigration.ts` with a single exported function.
2. Choose an idempotency strategy: completion flag in GlobalConfig, or self-idempotent data check.
3. Only read from and write to `userSettings` (or `localSettings` for project-scoped data). Never read merged settings.
4. Call `logEvent('tengu_my_migration_name', {...})` with relevant metadata.
5. Wrap the body in try/catch and call `logError` in the catch — migrations must never throw and break startup.
6. Import the function in `main.tsx` and add it inside the `if (migrationVersion !== CURRENT_MIGRATION_VERSION)` block.
7. **Bump `CURRENT_MIGRATION_VERSION`** so existing users re-run the updated set.

### Template: src/migrations/myNewMigration.ts

```typescript
import { logEvent } from '../services/analytics/index.js'
import { logError } from '../utils/log.js'
import {
  getSettingsForSource,
  updateSettingsForSource,
} from '../utils/settings/settings.js'

export function myNewMigration(): void {
  // self-idempotent guard: check if work is already done
  const model = getSettingsForSource('userSettings')?.model
  if (model !== 'old-alias') return

  try {
    updateSettingsForSource('userSettings', { model: 'new-alias' })
    logEvent('tengu_my_new_migration', {})
  } catch (error) {
    logError(new Error(`Failed to run myNewMigration: ${error}`))
  }
}
```

---

## Key Takeaways

- `runMigrations()` in `main.tsx` is the single entry point — a flat list of function calls guarded by `CURRENT_MIGRATION_VERSION = 11`.
- The version gate in `globalConfig.migrationVersion` prevents re-running 11 config saves on every startup once all migrations have been applied.
- Two idempotency strategies: **completion flag** in GlobalConfig for non-self-evident one-shots, and **self-idempotent data checks** for cases where the data speaks for itself.
- All model migrations only touch `userSettings` (never merged settings) to avoid accidentally globalizing a project-scoped model preference.
- Migrations must never throw — errors are caught, logged, and silently swallowed to avoid breaking startup.
- Most migrations call `logEvent()` so Anthropic can track when the old data shapes have fully disappeared from the installed base.
- Adding a migration requires bumping `CURRENT_MIGRATION_VERSION` so existing users who already passed the gate will re-run the updated set.
- Async migrations (file I/O like `migrateChangelogFromConfig`) are fire-and-forget after the synchronous block.

---

## Knowledge Check

**Q1. What happens when `getGlobalConfig().migrationVersion === CURRENT_MIGRATION_VERSION`?**

A) All migration functions run but their completion flags stop them from making changes.
B) The entire synchronous migration block is skipped; only the async migration runs.
C) Claude Code exits with an error because the version is unexpected.
D) The migration block runs once more to verify integrity, then exits.

**Correct Answer:** B

**Feedback:** The entire sync block is gated on migrationVersion !== CURRENT_MIGRATION_VERSION. When they match, all 11 sync functions are skipped. Only the async migrateChangelogFromConfig still runs.

---

**Q2. Why do model migration functions read from `userSettings` directly rather than from merged settings?**

A) Merged settings are only available after the REPL starts.
B) Reading merged settings would risk silently promoting a project-scoped model pin into the global user default.
C) `userSettings` is the only settings source that supports model aliases.
D) Merged settings are encrypted and cannot be read by migration code.

**Correct Answer:** B

**Feedback:** The key reason is scope containment — reading merged settings would collapse project-level settings into the global user settings.

---

**Q3. Which migration also updates the in-memory runtime state (not just settings files)?**

A) `migrateLegacyOpusToCurrent`
B) `migrateSonnet45ToSonnet46`
C) `migrateSonnet1mToSonnet45`
D) `migrateReplBridgeEnabledToRemoteControlAtStartup`

**Correct Answer:** C

**Feedback:** `migrateSonnet1mToSonnet45` calls `setMainLoopModelOverride()` to update the in-memory runtime state if the override was already set to "sonnet[1m]", ensuring the current process uses the pinned version without a restart.

---

**Q4. What must you do when adding a new migration to the sync block?**

A) Add a row to the migration database and run `bun migrate:up`.
B) Bump `CURRENT_MIGRATION_VERSION` so existing users who passed the gate will re-run the full set.
C) Update the `DEFAULT_GLOBAL_CONFIG` object to include the new migration's completion flag.
D) Register the migration in `entrypoints/init.ts` before `main.tsx` loads.

**Correct Answer:** B

**Feedback:** The comment above `runMigrations()` says: "Bump this when adding a new sync migration so existing users re-run the set." Without the bump, users whose migrationVersion already equals the old constant will skip the new migration.

---

**Q5. `migrateAutoUpdatesToSettings` calls `process.env.DISABLE_AUTOUPDATER = '1'` after writing to `userSettings`. Why?**

A) Writing to `userSettings` does not take effect until the next launch, so the env var makes the change live immediately in the current process.
B) The env var is required by the operating system's update daemon.
C) It is a backup in case `userSettings` write fails.
D) The code comment says it is set for analytics tracking purposes only.

**Correct Answer:** A

**Feedback:** The code comment explains: "explicitly set, so this takes effect immediately" — the env var applies the change to the current process without needing a restart.

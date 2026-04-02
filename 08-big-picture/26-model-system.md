# The Model System — Source Deep Dive

## Overview

This lesson explains how Claude Code manages AI models across four providers, with five selection layers and two context-window tiers.

## File Map

The model system spans two directories in `src/`:

| File | Responsibility |
|------|-----------------|
| `utils/model/configs.ts` | Provider-specific model IDs; registry for all known models |
| `utils/model/providers.ts` | Detects active API provider (firstParty/bedrock/vertex/foundry) |
| `utils/model/model.ts` | Selection priority chain, alias resolution, defaults |
| `utils/model/aliases.ts` | Canonical aliases: sonnet, opus, haiku, best, opusplan, [1m] variants |
| `utils/model/modelStrings.ts` | Runtime model-string resolution; Bedrock profile discovery |
| `utils/model/modelAllowlist.ts` | availableModels enforcement — three-tier matching |
| `utils/model/modelOptions.ts` | Build model picker options per subscription tier |
| `utils/model/agent.ts` | Subagent model resolution, Bedrock region-prefix inheritance |
| `utils/model/check1mAccess.ts` | Gate 1M context behind extra-usage billing |
| `utils/model/deprecation.ts` | Retirement-date warnings per provider |
| `utils/model/validateModel.ts` | Live model validation via API probe |
| `utils/model/modelSupportOverrides.ts` | 3P env-var overrides for effort/thinking flags |
| `utils/model/modelCapabilities.ts` | Ant-only: fetch/cache model capability metadata |
| `utils/model/antModels.ts` | Ant-only: codename models from Statsig feature flag |
| `utils/model/bedrock.ts` | AWS Bedrock profile listing and region utilities |
| `utils/effort.ts` | Effort levels, API mapping, persistence rules |
| `utils/fastMode.ts` | Fast mode toggle, cooldown, availability checks |
| `commands/model/` | /model slash command + ModelPicker UI |
| `commands/effort/` | /effort slash command |
| `commands/fast/` | /fast slash command |
| `migrations/migrate*.ts` | Automated settings upgrades on startup |

## Model Configs: The Four-Provider Registry

Every model is declared in a single object in `configs.ts`. Each entry is a `ModelConfig`—a record keyed by provider:

```typescript
export type ModelConfig = Record<APIProvider, ModelName>

export const CLAUDE_OPUS_4_6_CONFIG = {
  firstParty: 'claude-opus-4-6',
  bedrock:     'us.anthropic.claude-opus-4-6-v1',
  vertex:      'claude-opus-4-6',
  foundry:     'claude-opus-4-6',
} as const satisfies ModelConfig

export const ALL_MODEL_CONFIGS = {
  haiku35:  CLAUDE_3_5_HAIKU_CONFIG,
  haiku45:  CLAUDE_HAIKU_4_5_CONFIG,
  sonnet35: CLAUDE_3_5_V2_SONNET_CONFIG,
  sonnet37: CLAUDE_3_7_SONNET_CONFIG,
  sonnet40: CLAUDE_SONNET_4_CONFIG,
  sonnet45: CLAUDE_SONNET_4_5_CONFIG,
  sonnet46: CLAUDE_SONNET_4_6_CONFIG,
  opus40:   CLAUDE_OPUS_4_CONFIG,
  opus41:   CLAUDE_OPUS_4_1_CONFIG,
  opus45:   CLAUDE_OPUS_4_5_CONFIG,
  opus46:   CLAUDE_OPUS_4_6_CONFIG,
} as const satisfies Record<string, ModelConfig>
```

**Design note:** The `@[MODEL LAUNCH]` comment marks all seven files needing edits when a new model ships. Grepping for it provides a launch checklist.

### The Four Providers

**firstParty** — `api.anthropic.com`
Default. Clean model IDs like `claude-opus-4-6`.

**bedrock** — `CLAUDE_CODE_USE_BEDROCK=1`
AWS cross-region inference profiles; auto-discovered at boot.

**vertex** — `CLAUDE_CODE_USE_VERTEX=1`
GCP. Format: `claude-opus-4-6` (no date suffix).

**foundry** — `CLAUDE_CODE_USE_FOUNDRY=1`
Azure. Deployment IDs are user-defined; marketing name lookup disabled.

```typescript
export function getAPIProvider(): APIProvider {
  return isEnvTruthy(process.env.CLAUDE_CODE_USE_BEDROCK)
    ? 'bedrock'
    : isEnvTruthy(process.env.CLAUDE_CODE_USE_VERTEX)
      ? 'vertex'
      : isEnvTruthy(process.env.CLAUDE_CODE_USE_FOUNDRY)
        ? 'foundry'
        : 'firstParty'
}
```

### Bedrock Model String Resolution at Runtime

Bedrock users may have custom cross-region inference profiles (e.g., `eu.anthropic.claude-opus-4-6-v1`) differing from hardcoded strings in `configs.ts`. At startup, `getBedrockInferenceProfiles()` lists AWS `SYSTEM_DEFINED` profiles, then matches each config's `firstParty` ID as a substring to find the user's region profile.

This happens asynchronously in the background without blocking the REPL. While the fetch is in flight, requests fall back to hardcoded Bedrock IDs. The fetch is memoized—one call per process lifetime.

```typescript
async function getBedrockModelStrings(): Promise<ModelStrings> {
  const profiles = await getBedrockInferenceProfiles()
  const out = {} as ModelStrings
  for (const key of MODEL_KEYS) {
    const needle = ALL_MODEL_CONFIGS[key].firstParty
    out[key] = findFirstMatch(profiles, needle) || fallback[key]
  }
  return out
}
```

Additionally, users can override individual model strings via `settings.json → modelOverrides`. This is keyed by canonical first-party ID (e.g., `"claude-opus-4-6": "arn:aws:bedrock:..."`) and layered on top of discovered strings every time `getModelStrings()` is called.

## Selection Priority Chain

`getMainLoopModel()` in `model.ts` determines which model runs a conversation. It walks five layers in strict priority order:

1. **/model command during session**
   Stored in bootstrap state as `mainLoopModelOverride`. Highest priority—overrides everything for the rest of the session.

2. **--model CLI flag at startup**
   Also sets `mainLoopModelOverride` in bootstrap state, but only before the session begins.

3. **ANTHROPIC_MODEL env var**
   Checked synchronously. Accepts both full model IDs and aliases.

4. **settings.json → model field**
   Set by `/model` and persisted across sessions. The only layer migrations touch.

5. **Built-in subscription default**
   Max/Team Premium → Opus 4.6. Everyone else → Sonnet 4.6 (3P may default to 4.5).

### Allowlist Gate

Before any of these layers resolves, `getUserSpecifiedModelSetting()` checks the model against `settings.availableModels`. If a user-specified model is not on the allowlist it is silently ignored and the default is used instead.

```typescript
export function getUserSpecifiedModelSetting(): ModelSetting | undefined {
  const modelOverride = getMainLoopModelOverride()          // layers 1+2
  if (modelOverride !== undefined) return modelOverride

  const settings = getSettings_DEPRECATED() || {}
  const specifiedModel =
    process.env.ANTHROPIC_MODEL || settings.model || undefined    // layers 3+4

  if (specifiedModel && !isModelAllowed(specifiedModel)) return undefined
  return specifiedModel
}
```

### opusplan and Runtime Mode

Some model values carry behavior beyond a simple model ID. `opusplan` switches between Sonnet and Opus depending on the current permission mode:

```typescript
export function getRuntimeMainLoopModel(params: {
  permissionMode: PermissionMode
  mainLoopModel: string
  exceeds200kTokens?: boolean
}): ModelName {
  if (getUserSpecifiedModelSetting() === 'opusplan'
      && permissionMode === 'plan'
      && !exceeds200kTokens) {
    return getDefaultOpusModel()    // upgrade to Opus in plan mode
  }
  if (getUserSpecifiedModelSetting() === 'haiku' && permissionMode === 'plan') {
    return getDefaultSonnetModel()   // upgrade haiku to sonnet for plan
  }
  return mainLoopModel
}
```

If a `haiku` alias is active and the user enters plan mode, it is silently upgraded to Sonnet. This prevents plan mode from running on an underpowered model.

## Model Aliases & the [1m] Suffix

Users never need to type a full versioned model ID. The alias system resolves short names to current provider-appropriate model strings:

```typescript
export const MODEL_ALIASES = [
  'sonnet', 'opus', 'haiku', 'best',
  'sonnet[1m]', 'opus[1m]',
  'opusplan',
] as const
```

The `[1m]` suffix is special: it is accepted on any alias or model ID and signals the 1 million token context window variant. `parseUserSpecifiedModel()` strips it before resolving the base model, then re-appends it so the API receives the correct model string:

```typescript
export function parseUserSpecifiedModel(modelInput: ModelName | ModelAlias): ModelName {
  const has1mTag  = has1mContext(normalizedModel)           // checks for [1m] suffix
  const baseModel = has1mTag
    ? normalizedModel.replace(/\[1m]$/i, '').trim()
    : normalizedModel

  if (isModelAlias(baseModel)) {
    switch (baseModel) {
      case 'opus':   return getDefaultOpusModel() + (has1mTag ? '[1m]' : '')
      case 'sonnet': return getDefaultSonnetModel() + (has1mTag ? '[1m]' : '')
      case 'haiku':  return getDefaultHaikuModel() + (has1mTag ? '[1m]' : '')
      // ...
    }
  }
  return modelInputTrimmed  // preserve original case for custom IDs
}
```

### Skill Model Override

When a SKILL.md frontmatter sets `model: opus`, `resolveSkillModelOverride()` automatically carries the `[1m]` suffix from the parent session to the skill invocation—as long as the target family supports 1M context. A skill that sets `model: haiku` will intentionally drop 1M because haiku has no 1M variant.

## 1M Context Window

Extended context is not a separate model; it is a runtime flag on an existing model. The `[1m]` suffix travels through the entire pipeline and is stripped only at the API call boundary by `normalizeModelStringForAPI()`.

### Access Gates

Two functions gate access to 1M, one each for Opus and Sonnet:

```typescript
export function checkOpus1mAccess(): boolean {
  if (is1mContextDisabled()) return false    // CLAUDE_CODE_DISABLE_1M_CONTEXT env
  if (isClaudeAISubscriber()) return isExtraUsageEnabled()  // subscriber: extra-usage billing
  return true                                  // API/PAYG: always available
}
```

### The Opus 1M Merge

For Max and Team Premium subscribers, there is only one Opus option: `opus[1m]`—they always get extended context. This "merge" is controlled by `isOpus1mMergeEnabled()`:

```typescript
export function isOpus1mMergeEnabled(): boolean {
  if (is1mContextDisabled() || isProSubscriber() || getAPIProvider() !== 'firstParty')
    return false
  if (isClaudeAISubscriber() && getSubscriptionType() === null)
    return false  // fail closed: unknown sub type → no 1M (avoids API rate-limit error)
  return true
}
```

**Gotcha:** The `getSubscriptionType() === null` guard exists because OAuth tokens can be refreshed without a `subscriptionType` field. Without this guard, a partially-refreshed token would incorrectly expose `opus[1m]` to Pro users, whose API tier rejects it with a misleading "rate limit reached" error.

### Model Picker Options Per Subscription Tier

The `/model` picker shows a different option list depending on who is logged in:

| Tier | Default | Options shown |
|------|---------|---------------|
| Max / Team Premium | Opus 4.6 (1M) | Opus, Sonnet, Sonnet 1M, Haiku |
| Pro / Team Standard | Sonnet 4.6 | Sonnet, Sonnet 1M, Opus, Opus 1M, Haiku |
| PAYG 1P (API key) | Sonnet 4.6 | Sonnet, Sonnet 1M, Opus 1M, Haiku |
| PAYG 3P (Bedrock/Vertex) | Sonnet 4.5 | Custom or Sonnet 4.6/4.5, Opus 4.1/4.6, Haiku |

The "Ant" tier (Anthropic employees) gets a completely different list built from a Statsig feature flag—see **antModels.ts**.

## Effort Levels

Effort is a thinking-budget hint sent to the API as a parameter. It only applies to models that support it—currently `claude-opus-4-6` and `claude-sonnet-4-6`.

**auto** — Model default. No effort param sent. Equals "high" in practice.

**low** — Minimal thinking budget. Fast, cheaper.

**medium** — Balanced budget.

**high** — Extended thinking. Default API behavior.

**max** — Opus 4.6 only. Non-ants: session-scoped, not persisted.

### Resolution Order

```typescript
// utils/effort.ts — resolveAppliedEffort()
// Priority: env > appState > model default
const envOverride = getEffortEnvOverride()  // CLAUDE_CODE_EFFORT_LEVEL env var
if (envOverride === null) return undefined    // null = explicit 'unset'
const resolved = envOverride ?? appStateEffortValue ?? getDefaultEffortForModel(model)
// API rejects 'max' on non-Opus-4.6 — auto-downgrade
if (resolved === 'max' && !modelSupportsMaxEffort(model)) return 'high'
return resolved
```

### Persistence Rules

Not all effort values can be saved to `settings.json`. The `toPersistableEffort()` function enforces these constraints:

- `low`, `medium`, `high`—always persisted
- `max`—only persisted for Anthropic employees (`USER_TYPE === 'ant'`); session-scoped for everyone else
- Numeric values (e.g., `42`)—never persisted, model-default only

**Env var conflict:** When `CLAUDE_CODE_EFFORT_LEVEL` is set and a user runs `/effort high`, the CLI saves the setting to disk but warns them: "CLAUDE_CODE_EFFORT_LEVEL=max overrides this session—clear it and high takes over." The env var always wins at runtime; the setting takes effect next session after clearing the env.

## Fast Mode

Fast mode is a research-preview feature that enables a lower-latency execution path for Opus 4.6 only on the first-party API. It is toggled via `/fast` or settings, and the command is hidden unless fast mode is available for the session.

```typescript
const fast = {
  type: 'local-jsx',
  name: 'fast',
  get description() { return `Toggle fast mode (${FAST_MODE_MODEL_DISPLAY} only)` },
  availability: ['claude-ai', 'console'],
  isEnabled: () => isFastModeEnabled(),
  get isHidden() { return !isFastModeEnabled() },
}
```

### Availability Checks

Fast mode goes through a layered availability check before being offered to the user:

- `CLAUDE_CODE_DISABLE_FAST_MODE=1` hard-disables it globally
- Only available on `firstParty` provider—not on Bedrock/Vertex/Foundry
- Requires a paid subscription; free accounts see "Fast mode requires a paid subscription"
- Subscribers need extra-usage billing enabled; without it the reason is "Fast mode requires extra usage billing"
- Not available in the Agent SDK (non-interactive sessions) unless `fastMode: true` is explicitly set in flag settings
- Statsig feature flag `tengu_penguins_off` can kill it remotely with a custom reason string

### Model Switching on Toggle

Enabling fast mode only switches the model if the current model does not support fast mode. If you are already on Opus 4.6, the model is left unchanged:

```typescript
if (enable) {
  setAppState(prev => {
    const needsSwitch = !isFastModeSupportedByModel(prev.mainLoopModel)
    return {
      ...prev,
      ...(needsSwitch ? { mainLoopModel: getFastModeModel() } : {}),
      fastMode: true,
    }
  })
}
```

### Cooldown

Fast mode has a cooldown state. When you hit your fast limit or the service is overloaded, the UI shows a "resets in X minutes" timer and fast mode is marked unavailable. The cooldown can be cleared programmatically via `clearFastModeCooldown()`.

## Allowlist (availableModels)

Enterprise and team admins can restrict which models users can select by setting `availableModels` in a policy settings file. The matching logic has three tiers, applied in order:

1. **Family wildcard**—`"opus"` allows any Opus model, unless narrower entries for that family also exist
2. **Version prefix**—`"opus-4-5"` or `"claude-opus-4-5"` matches any build of that version at a segment boundary
3. **Exact ID**—`"claude-opus-4-5-20251101"` matches only that exact string

```typescript
// Example: restrict to Sonnet 4.6 and Haiku 4.5 only
{
  "availableModels": ["sonnet", "haiku"]
}

// Example: allow only Opus 4.5 (not 4.6)
{
  "availableModels": ["opus-4-5"]
}

// Example: empty allowlist blocks ALL user-specified models
{
  "availableModels": []
}
```

### Narrowing Rule

If both `"opus"` and `"opus-4-5"` appear in the allowlist, the family wildcard `"opus"` is ignored. The system detects that a more specific entry exists for the opus family and only applies the version-prefix match. This prevents an admin from accidentally allowing all Opus while intending to restrict to one version.

### Full Allowlist Matching Implementation

```typescript
export function isModelAllowed(model: string): boolean {
  const { availableModels } = settings
  if (!availableModels) return true         // no restriction
  if (availableModels.length === 0) return false // empty blocks all

  // 1. Direct match — but skip family aliases that are narrowed
  if (allowlist.includes(model)) {
    if (!isModelFamilyAlias(model) || !familyHasSpecificEntries(model, allowlist))
      return true
  }

  // 2. Family wildcard (only if no specific entries for that family)
  for (const entry of allowlist) {
    if (isModelFamilyAlias(entry)
        && !familyHasSpecificEntries(entry, allowlist)
        && modelBelongsToFamily(model, entry)) return true
  }

  // 3. Version-prefix match at segment boundary
  for (const entry of allowlist) {
    if (!isModelFamilyAlias(entry) && !isModelAlias(entry)) {
      if (modelMatchesVersionPrefix(model, entry)) return true
    }
  }

  return false
}
```

## Subagent Model Inheritance

When Claude Code spawns a subagent (via Task tool or SDK), the subagent needs to know which model to use. The resolution happens in `utils/model/agent.ts` and follows this precedence:

1. **CLAUDE_CODE_SUBAGENT_MODEL env var**
   Global override for all subagents. Useful in CI pipelines.

2. **Tool-specified model**
   Model param on the Task call. The orchestrating model can request a specific model for a subagent.

3. **Agent config model field**
   Set in the agent definition. Accepts any alias or model ID.

4. **"inherit" (default)**
   Subagent uses the same model as the parent thread, including opusplan resolution.

### Alias-Matches-Parent-Tier Optimization

When a bare family alias (`opus`, `sonnet`, `haiku`) matches the parent model's family, the subagent inherits the parent's exact model string instead of resolving the alias to the provider default. This prevents a Vertex user on a pinned Opus version from having subagents silently downgrade:

```typescript
function aliasMatchesParentTier(alias: string, parentModel: string): boolean {
  const canonical = getCanonicalName(parentModel)
  switch (alias.toLowerCase()) {
    case 'opus':   return canonical.includes('opus')
    case 'sonnet': return canonical.includes('sonnet')
    case 'haiku':  return canonical.includes('haiku')
    default: return false
  }
}
// Note: opus[1m], best, opusplan fall through — they have extra semantics
```

### Bedrock Region Prefix Inheritance

On Bedrock, cross-region inference profiles carry a region prefix (`eu.`, `us.`). If the parent model uses `eu.anthropic.claude-opus-4-6-v1`, subagents using alias models must also use the `eu.` prefix. This is handled by `applyParentRegionPrefix()`—which is skipped if the subagent's model spec already carries its own explicit prefix.

## Model Migrations

When Claude Code upgrades its default models, users who had previously pinned a specific model need their settings migrated automatically. The migration system runs a set of functions on every startup; each migration is idempotent (safe to re-run).

### migrateFennecToOpus

Ant-only. Remaps codename aliases (`fennec-latest`, `fennec-fast-latest`, `opus-4-5-fast`) to their public equivalents (`opus`, `opus[1m]`, and fast mode flag). Only touches `userSettings`—project/policy pins are left alone.

### migrateLegacyOpusToCurrent

First-party only. Rewrites explicit Opus 4.0/4.1 strings (`claude-opus-4-20250514`, `claude-opus-4-1-20250805`) to the `opus` alias. Sets `legacyOpusMigrationTimestamp` in global config so the REPL can show a one-time notification. `CLAUDE_CODE_DISABLE_LEGACY_MODEL_REMAP=1` skips this.

### migrateSonnet1mToSonnet45

Pins `sonnet[1m]` settings to the explicit `sonnet-4-5-20250929[1m]` string. Needed because Sonnet 4.6 1M was offered to a different access group than Sonnet 4.5 1M—the `sonnet` alias now resolves to 4.6. Tracked by `sonnet1m45MigrationComplete` flag.

### migrateSonnet45ToSonnet46

Pro/Max/Team Premium first-party users only. Upgrades explicit Sonnet 4.5 strings back to the `sonnet` alias (which now resolves to 4.6). Sets `sonnet45To46MigrationTimestamp` for new-version notification—but skips the notification for brand-new users (startup count ≤ 1).

### migrateOpusToOpus1m

For Max/Team Premium where the 1M merge is enabled: upgrades a pinned `opus` setting to `opus[1m]`. Skipped if the upgrade would result in the same resolved model as the subscription default (to avoid storing a redundant explicit setting).

### resetProToOpusDefault

Pro subscribers on first-party who had no custom model are given a migration timestamp—this enables the REPL to display a one-time "You've been upgraded to Opus 4.5 as your default" callout. Users who already had a custom model setting are silently marked complete.

**Pattern:** All model migrations share the same constraints: (1) only touch `userSettings`, never merged/project/policy settings; (2) use completion flags or idempotent write logic so reruns are safe; (3) log an analytics event so the team can measure migration success rates.

## Validation & Deprecation

### Live Model Validation

When a user types a custom model ID into `/model`, the system cannot know if it is valid just by string-matching. `validateModel()` sends a real API call with `max_tokens: 1` to probe the model:

```typescript
await sideQuery({
  model: normalizedModel,
  max_tokens: 1,
  maxRetries: 0,
  querySource: 'model_validation',
  messages: [{ role: 'user', content: [{ type: 'text', text: 'Hi' }] }],
})
```

A `NotFoundError (404)` means the model does not exist. For 3P users, a fallback suggestion is returned (e.g., if you ask for `opus-4-6` on Vertex but it is not available yet, it suggests `opus-4-1`).

### Deprecation Warnings

The deprecation table in `deprecation.ts` maps model substrings to retirement dates per provider:

```typescript
const DEPRECATED_MODELS: Record<string, DeprecationEntry> = {
  'claude-3-7-sonnet': {
    modelName: 'Claude 3.7 Sonnet',
    retirementDates: {
      firstParty: 'February 19, 2026',
      bedrock:    'April 28, 2026',
      vertex:     'May 11, 2026',
      foundry:    'February 19, 2026',
    },
  },
  // ...
}
```

If the active model matches any entry and has a retirement date for the current provider, the UI shows a yellow warning banner. Bedrock and Vertex often get longer retirement windows than first-party.

## Key Takeaways

- **One registry, four providers.** `ALL_MODEL_CONFIGS` in `configs.ts` is the single source of truth for every model ID. Adding a new model requires updating that file plus six other tagged locations—all marked with `@[MODEL LAUNCH]`.

- **Selection has exactly five layers.** /model command → --model flag → ANTHROPIC_MODEL env → settings.json → subscription default. The allowlist gate runs before any of them.

- **The [1m] suffix is not a separate model.** It is a runtime flag stripped at the API boundary by `normalizeModelStringForAPI()`. Access is gated by subscription tier and extra-usage billing.

- **Effort and fast mode are orthogonal knobs.** Effort controls thinking budget (low/medium/high/max). Fast mode is a latency optimization, Opus 4.6 first-party only. Both have env-var overrides that win over UI settings.

- **Migrations only touch userSettings.** Project and policy settings are intentionally left alone to avoid silently promoting a project-scoped pin to the global default.

- **Subagents default to "inherit".** The parent's exact model string is passed through, including opusplan → Opus resolution. Bedrock region prefixes are also inherited to avoid IAM permission failures.

- **The allowlist narrowing rule is non-obvious.** `["opus", "opus-4-5"]` does NOT allow all Opus—the specific entry narrows the family wildcard to only opus-4-5.

- **Fast mode is first-party only and requires extra-usage billing for subscribers.** The check is multi-layered: env flag, Statsig kill switch, provider check, subscription check, billing check.

## Quiz — Test Your Understanding

**1. A user sets ANTHROPIC_MODEL=sonnet in their environment and also runs /model opus during their session. Which model is used?**

- A. Sonnet—the environment variable takes priority
- B. Opus—the /model command (session override) has the highest priority
- C. Whatever the settings.json says
- D. The subscription default

**Answer: B**

The priority chain is: session override (/model) → startup flag (--model) → env var → settings.json → default. /model during a session always wins.

**2. An admin sets availableModels: ["opus", "opus-4-5"] in the policy settings. A user tries to select "opus-4-6". What happens?**

- A. Allowed—"opus" wildcard covers all Opus models
- B. Blocked—the specific "opus-4-5" entry narrows the family wildcard to only that version
- C. Allowed—exact matches always win over prefix matches
- D. Error is thrown

**Answer: B**

When the allowlist contains both a family alias and a more specific entry for that family, the family wildcard is suppressed. "opus" alone would allow everything, but alongside "opus-4-5" it restricts to only 4.5 builds.

**3. A Max subscriber has "opus" pinned in settings.json. After upgrading Claude Code, migrateOpusToOpus1m() runs. What does settings.json contain afterward?**

- A. "opus"—no change, migrations don't modify this setting
- B. "claude-opus-4-6"—the full model ID
- C. "opus[1m]"—or undefined if opus[1m] is the same as the default
- D. null—the setting is removed entirely

**Answer: C**

migrateOpusToOpus1m() upgrades "opus" to "opus[1m]" for Max/Team Premium users. But if opus[1m] would resolve to the same model as the subscription default, it writes undefined instead—no point storing a redundant explicit setting.

**4. A skill sets model: haiku in its frontmatter. The parent session is running on opus[1m] at 300K tokens. What model does the skill use?**

- A. Haiku with 1M context—the [1m] suffix is automatically inherited
- B. Haiku without 1M context—haiku has no 1M variant, so the suffix is not carried
- C. Opus 1M—the skill model is ignored for 1M sessions
- D. Sonnet—haiku is upgraded in all long-context sessions

**Answer: B**

resolveSkillModelOverride() only carries the [1m] suffix if modelSupports1M() returns true for the target. Haiku does not support 1M, so the suffix is dropped and the autocompact that follows is correct behavior.

**5. A user on a Bedrock deployment in the eu-west-1 region spawns a subagent with model: sonnet. The parent is running on eu.anthropic.claude-opus-4-6-v1. What model string does the subagent receive?**

- A. us.anthropic.claude-sonnet-4-6—the hardcoded Bedrock default
- B. eu.anthropic.claude-sonnet-4-6—the parent's eu. prefix is inherited
- C. claude-sonnet-4-6—aliases are resolved without any prefix
- D. The same as the parent: eu.anthropic.claude-opus-4-6-v1

**Answer: B**

applyParentRegionPrefix() extracts the eu. prefix from the parent model and applies it to the subagent's resolved model string. This ensures subagents use the same cross-region inference profile as the parent—required when IAM permissions are scoped to a specific region.

**6. A user sets CLAUDE_CODE_EFFORT_LEVEL=max in their shell and runs /effort low. What happens?**

- A. low is applied immediately—/effort always wins
- B. low is saved to settings, but the env var overrides this session; low takes effect next session after clearing the env
- C. An error is shown—conflicting effort settings are rejected
- D. max is cleared from the env var automatically

**Answer: B**

The env var wins at resolveAppliedEffort() time. /effort saves to settings.json and warns the user that CLAUDE_CODE_EFFORT_LEVEL is overriding this session. The setting takes effect once the env var is cleared.

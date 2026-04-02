# OAuth 2.0 Authentication — Lesson 25 Complete Content

## Overview

Claude Code implements **OAuth 2.0 Authorization Code flow with PKCE** (Proof Key for Code Exchange) for user authentication against either the Anthropic Console or Claude.ai. The architecture spans three layers:

- **Crypto primitives** (`services/oauth/crypto.ts`): generates PKCE verifier, S256 challenge, and CSRF state
- **Network client** (`services/oauth/client.ts`): builds auth URLs, exchanges codes for tokens, refreshes tokens, fetches profiles
- **Orchestrator** (`services/oauth/index.ts`): wires up the listener, races auto vs. manual flows, formats tokens

Two authorization targets exist: Console (`platform.claude.com/oauth/authorize`) for API-key workflows, and Claude.ai (`claude.com/cai/oauth/authorize`) for Pro/Max/Team/Enterprise subscribers.

---

## PKCE Primitives

PKCE prevents authorization code interception by binding each flow to a one-time secret known only to the initiating process.

### crypto.ts Implementation

```typescript
import { createHash, randomBytes } from 'crypto'

function base64URLEncode(buffer: Buffer): string {
  return buffer
    .toString('base64')
    .replace(/\+/g, '-')
    .replace(/\//g, '_')
    .replace(/=/g, '')   // RFC 4648 §5 — URL-safe, no padding
}

export function generateCodeVerifier(): string {
  return base64URLEncode(randomBytes(32))  // 256-bit entropy
}

export function generateCodeChallenge(verifier: string): string {
  const hash = createHash('sha256')
  hash.update(verifier)
  return base64URLEncode(hash.digest())  // S256 method
}

export function generateState(): string {
  return base64URLEncode(randomBytes(32))  // CSRF protection
}
```

### PKCE Values Reference

| Value | How Generated | Sent to Server | Purpose |
|-------|---------------|----------------|---------|
| `codeVerifier` | 32 random bytes, base64url | Token exchange only | Proves this process started the flow |
| `codeChallenge` | SHA-256(verifier), base64url | Authorization URL param | Server stores and later verifies verifier |
| `state` | 32 random bytes, base64url | Auth URL + callback URL | CSRF — callback must echo exact state |

The `codeVerifier` is generated in the `OAuthService` constructor before the server starts, existing in memory only for the login attempt lifetime.

---

## Full PKCE Sequence Diagram

```
sequenceDiagram
  participant U as User
  participant CC as Claude Code (CLI)
  participant LS as LocalServer :PORT/callback
  participant B as Browser
  participant AS as Auth Server claude.com / platform
  participant TS as Token Server platform.claude.com/v1/oauth/token
  participant PS as Profile API api.anthropic.com

  CC->>CC: generateCodeVerifier()<br/>generateCodeChallenge()<br/>generateState()
  CC->>LS: AuthCodeListener.start()<br/>(OS assigns port)
  CC->>B: openBrowser(automaticFlowUrl)<br/>?code_challenge=S256&state=...&redirect_uri=localhost:PORT
  CC-->>U: Show manual URL fallback in terminal
  B->>AS: GET /oauth/authorize?client_id=...&code_challenge=...
  AS->>U: Login page (email / SSO / magic link)
  U->>AS: Authenticate
  AS->>B: 302 → http://localhost:PORT/callback?code=AUTH_CODE&state=STATE
  B->>LS: GET /callback?code=AUTH_CODE&state=STATE
  LS->>LS: validate state === expectedState
  LS-->>CC: resolve(authorizationCode)
  CC->>TS: POST /v1/oauth/token<br/>{ grant_type: authorization_code, code, code_verifier, redirect_uri }
  TS-->>CC: { access_token, refresh_token, expires_in, scope }
  CC->>PS: GET /api/oauth/profile<br/>Authorization: Bearer access_token
  PS-->>CC: { account, organization }
  CC->>LS: handleSuccessRedirect(scopes)<br/>→ 302 success page
  CC->>CC: installOAuthTokens()<br/>save to keychain / secure storage
```

---

## Automatic vs. Manual Auth Flow

Both flows are started simultaneously. Whichever delivers an authorization code first wins. The key insight: "the `OAuthService` races two resolvers against a single Promise."

### Automatic (browser redirect)

- `AuthCodeListener` starts an HTTP server on an OS-assigned port
- Browser opens `automaticFlowUrl` with `redirect_uri=localhost:PORT/callback`
- After login, auth server redirects browser to the local server
- Callback handler validates `state`, resolves the auth code promise
- Browser receives a success redirect to `platform.claude.com/oauth/code/success`

### Manual (copy-paste fallback)

- Terminal displays the `manualFlowUrl` with `redirect_uri=platform.claude.com/oauth/code/callback`
- User opens the URL, authenticates, and copies the resulting code from the browser
- User pastes the code into the Claude Code terminal prompt
- `handleManualAuthCodeInput()` calls the stored resolver directly
- Used in SSH sessions or environments where `localhost` is unreachable

### OAuthService.startOAuthFlow() — orchestration logic

```typescript
async startOAuthFlow(
  authURLHandler: (url: string, automaticUrl?: string) => Promise<void>,
  options?: { skipBrowserOpen?: boolean; inferenceOnly?: boolean; ... }
): Promise<OAuthTokens> {
  // 1. Start the localhost callback server
  this.authCodeListener = new AuthCodeListener()
  this.port = await this.authCodeListener.start()

  // 2. Build both URLs from same PKCE values
  const manualFlowUrl    = client.buildAuthUrl({ ...opts, isManual: true })
  const automaticFlowUrl = client.buildAuthUrl({ ...opts, isManual: false })

  // 3. Race: automatic (localhost) vs manual (paste)
  const authorizationCode = await this.waitForAuthorizationCode(state, async () => {
    if (options?.skipBrowserOpen) {
      await authURLHandler(manualFlowUrl, automaticFlowUrl)  // SDK mode
    } else {
      await authURLHandler(manualFlowUrl)   // Show manual to user
      await openBrowser(automaticFlowUrl)  // Try automatic
    }
  })

  // 4. Which flow won?
  const isAutomatic = this.authCodeListener?.hasPendingResponse() ?? false

  // 5. Exchange code for tokens
  const tokenResponse = await client.exchangeCodeForTokens(
    authorizationCode, state, this.codeVerifier, this.port!,
    !isAutomatic  // isManual = true if auto did NOT win
  )

  // 6. Fetch subscription/rate-limit tier from profile API
  const profileInfo = await client.fetchProfileInfo(tokenResponse.access_token)

  // 7. Redirect browser to success page, then cleanup
  if (isAutomatic) this.authCodeListener?.handleSuccessRedirect(scopes)
  return this.formatTokens(tokenResponse, profileInfo.subscriptionType, ...)
}
```

**skipBrowserOpen mode:** When the SDK control protocol (`claude_authenticate`) drives login, it sets `skipBrowserOpen: true`. Both URLs are handed to the caller via `authURLHandler`. The SDK client—not Claude Code—decides where to open them.

---

## AuthCodeListener — The Localhost Capture Server

`AuthCodeListener` is a minimal Node.js HTTP server whose only job is to receive the OAuth provider's redirect and hand the authorization code to a waiting Promise.

### Key Implementation Details

#### Port Assignment
Listening on port `0` lets the OS pick a free port. This avoids the "port already in use" class of errors entirely. The chosen port is embedded in the `redirect_uri` that the auth server uses for its callback.

#### State Validation
When the callback arrives, `validateAndRespond()` checks that `state === expectedState` before resolving. A mismatch returns HTTP 400 and rejects the promise—protecting against CSRF.

#### Pending Response Pattern
The server stores the browser's `ServerResponse` object in `pendingResponse` before resolving the auth code promise. After the token exchange succeeds, `handleSuccessRedirect()` completes the browser request with a 302 to the success page. This keeps the browser tab from hanging.

#### Cleanup on Close
If `close()` is called while a response is still pending (e.g., token exchange failed), the server automatically calls `handleErrorRedirect()` first, ensuring the browser always gets a response.

### AuthCodeListener.validateAndRespond() — CSRF check

```typescript
private validateAndRespond(
  authCode: string | undefined,
  state:    string | undefined,
  res:      ServerResponse,
): void {
  if (!authCode) {
    res.writeHead(400)
    res.end('Authorization code not found')
    this.reject(new Error('No authorization code received'))
    return
  }
  if (state !== this.expectedState) {
    res.writeHead(400)
    res.end('Invalid state parameter')
    this.reject(new Error('Invalid state parameter'))
    return
  }
  // Store response for later redirect — keeps browser from hanging
  this.pendingResponse = res
  this.resolve(authCode)
}
```

---

## Token Exchange

Once the authorization code is in hand, it is exchanged for access + refresh tokens via a POST to the token endpoint.

### exchangeCodeForTokens() — request body

```typescript
const requestBody = {
  grant_type:    'authorization_code',
  code:           authorizationCode,
  redirect_uri:   useManualRedirect
    ? getOauthConfig().MANUAL_REDIRECT_URL    // https://platform.claude.com/oauth/code/callback
    : `http://localhost:${port}/callback`,    // must match what was in auth URL
  client_id:      getOauthConfig().CLIENT_ID, // '9d1c250a-e61b-44d9-88ed-5944d1962f5e'
  code_verifier:  codeVerifier,               // proves we started this flow
  state,
}

// POST https://platform.claude.com/v1/oauth/token
const response = await axios.post(TOKEN_URL, requestBody, {
  headers: { 'Content-Type': 'application/json' },
  timeout: 15000,
})
```

The redirect_uri in the token request must exactly match the one used in the authorization request. This is enforced server-side and is another anti-replay measure.

**inferenceOnly tokens:** When `inferenceOnly: true`, only `user:inference` scope is requested. These are long-lived tokens designed for SDK programmatic access where the full scope set would be excessive.

---

## OAuth Scopes

Scopes determine what the issued access token can do. Claude Code requests the union of Console and Claude.ai scopes at login time so one token can serve both paths.

### All Requested Scopes (`ALL_OAUTH_SCOPES`)

- `org:create_api_key`
- `user:profile`
- `user:inference`
- `user:sessions:claude_code`
- `user:mcp_servers`
- `user:file_upload`

### Scope Reference

| Scope | Used For |
|-------|----------|
| `org:create_api_key` | Console path — create a permanent API key for the organization |
| `user:profile` | Fetch subscription type, rate-limit tier, account/org info from `/api/oauth/profile` |
| `user:inference` | Claude.ai path — route inference requests directly via Claude.ai subscription |
| `user:sessions:claude_code` | Session management for the Claude Code client specifically |
| `user:mcp_servers` | Access and configure MCP servers associated with the account |
| `user:file_upload` | Upload files to Anthropic infrastructure for processing |

The function `shouldUseClaudeAIAuth(scopes)` checks whether `user:inference` is present. If it is, inference calls route through Claude.ai's infrastructure; otherwise the Console API key path is used.

---

## Profile Fetching

Immediately after the token exchange, Claude Code fetches the user's profile to determine subscription type and rate-limit tier. The profile drives UI choices, model availability, and feature flags.

### fetchProfileInfo() — subscription type mapping

```typescript
export async function fetchProfileInfo(accessToken: string) {
  const profile = await getOauthProfileFromOauthToken(accessToken)
  const orgType = profile?.organization?.organization_type

  let subscriptionType: SubscriptionType | null = null
  switch (orgType) {
    case 'claude_max':        subscriptionType = 'max';        break
    case 'claude_pro':        subscriptionType = 'pro';        break
    case 'claude_enterprise': subscriptionType = 'enterprise'; break
    case 'claude_team':       subscriptionType = 'team';       break
    default: subscriptionType = null
  }
  return {
    subscriptionType,
    rateLimitTier:       profile?.organization?.rate_limit_tier ?? null,
    hasExtraUsageEnabled: profile?.organization?.has_extra_usage_enabled ?? null,
    billingType:         profile?.organization?.billing_type ?? null,
    displayName:         profile?.account?.display_name,
    accountCreatedAt:    profile?.account?.created_at,
    subscriptionCreatedAt: profile?.organization?.subscription_created_at,
    rawProfile:          profile,
  }
}
```

### Subscription Types and Their org_type Keys

| Subscription | org_type Key |
|--------------|--------------|
| max | `claude_max` |
| pro | `claude_pro` |
| team | `claude_team` |
| enterprise | `claude_enterprise` |

**Profile skip optimization:** During routine token refresh, if the global config already has `billingType`, `accountCreatedAt`, and `subscriptionCreatedAt`, AND secure storage already has a non-null `subscriptionType` and `rateLimitTier`, the profile endpoint call is skipped entirely. This optimization cuts roughly 7 million requests per day fleet-wide.

---

## Token Storage — The Keychain Architecture

Tokens are stored in platform-specific secure storage, not in a plain config file. The storage layer is selected at runtime by `getSecureStorage()`.

### Platform Storage Options

| Platform | Primary Storage | Fallback |
|----------|-----------------|----------|
| macOS | `macOsKeychainStorage` — uses macOS `security` CLI (`add-generic-password` / `find-generic-password`) | `plainTextStorage` — encrypted JSON in `~/.claude/` |
| Linux | `plainTextStorage` (libsecret support planned) | — |
| Windows | `plainTextStorage` | — |

### macOS Keychain: Why Hex and stdin?

The macOS `security` command is invoked to store credentials. Two notable security engineering decisions:

- **Hex encoding:** The JSON token payload is converted to hexadecimal before storage (`-X` flag). This avoids shell quoting issues and—more importantly—prevents process monitors like CrowdStrike from seeing the raw token value in `ps` output or system call logs.
- **stdin preference (`security -i`):** When the payload fits within 4032 bytes (4096 - 64 headroom), it is passed via stdin instead of as a command-line argument. This prevents the token from appearing in process argument lists. If the payload exceeds the limit, argv fallback is used with a debug warning.
- **Stale-while-error:** If a `security` subprocess fails transiently, the last known good value is served from cache rather than surfacing a "Not logged in" error to the user.

### installOAuthTokens() — what happens after the flow

```typescript
export async function installOAuthTokens(tokens: OAuthTokens): Promise<void> {
  // 1. Wipe old state first (clear keychain, reset caches)
  await performLogout({ clearOnboarding: false })

  // 2. Store account info in global config (non-sensitive, JSON)
  const profile = tokens.profile ?? await getOauthProfileFromOauthToken(tokens.accessToken)
  if (profile) {
    storeOAuthAccountInfo({ accountUuid, emailAddress, organizationUuid, ... })
  }

  // 3. Save tokens to secure storage (keychain on macOS)
  const storageResult = saveOAuthTokensIfNeeded(tokens)
  clearOAuthTokenCache()

  // 4. Fetch roles (org/workspace role) — non-critical, failure tolerated
  await fetchAndStoreUserRoles(tokens.accessToken).catch(logForDebugging)

  // 5. Console path: create a permanent API key via the token
  if (!shouldUseClaudeAIAuth(tokens.scopes)) {
    await createAndStoreApiKey(tokens.accessToken)
  }
}
```

---

## Token Refresh

Access tokens expire. Claude Code proactively refreshes them before the expiry using a 5-minute buffer window. The refresh flow is designed to be invisible to users.

### isOAuthTokenExpired() — the buffer check

```typescript
export function isOAuthTokenExpired(expiresAt: number | null): boolean {
  if (expiresAt === null) return false

  const bufferTime = 5 * 60 * 1000  // 5 minutes early
  const expiresWithBuffer = Date.now() + bufferTime
  return expiresWithBuffer >= expiresAt
}
```

### refreshOAuthToken() — scope expansion and dedup

```typescript
export async function refreshOAuthToken(
  refreshToken: string,
  { scopes: requestedScopes }: { scopes?: string[] } = {},
): Promise<OAuthTokens> {
  const requestBody = {
    grant_type:    'refresh_token',
    refresh_token: refreshToken,
    client_id:     getOauthConfig().CLIENT_ID,
    // Backend allows scope expansion on refresh (ALLOWED_SCOPE_EXPANSIONS)
    scope: (requestedScopes?.length ? requestedScopes : CLAUDE_AI_OAUTH_SCOPES).join(' '),
  }

  // Skip profile fetch if we already have all fields cached
  const haveProfileAlready =
    config.oauthAccount?.billingType !== undefined &&
    config.oauthAccount?.accountCreatedAt !== undefined &&
    existing?.subscriptionType != null  // must check secure storage too

  const profileInfo = haveProfileAlready ? null : await fetchProfileInfo(accessToken)

  return {
    accessToken,
    refreshToken: newRefreshToken,   // server may rotate refresh token
    expiresAt:    Date.now() + expiresIn * 1000,
    scopes,
    subscriptionType: profileInfo?.subscriptionType ?? existing?.subscriptionType ?? null,
    rateLimitTier:    profileInfo?.rateLimitTier    ?? existing?.rateLimitTier    ?? null,
  }
}
```

**Re-login via env var:** When `CLAUDE_CODE_OAUTH_REFRESH_TOKEN` is set, Claude Code uses it to perform a fresh token exchange. The critical subtlety: `installOAuthTokens` calls `performLogout()` _after_ the refresh returns. If `refreshOAuthToken` returned `subscriptionType: null` because it saw missing profile fields (already wiped by the logout), subsequent refreshes would permanently lose the subscription type. The fix: pass through the cached value from secure storage before the logout wipes it.

---

## Logout Flow

Logout is more than deleting a token. It needs to clear every layer of state that depends on the current identity.

### performLogout() — full teardown sequence

```typescript
export async function performLogout({ clearOnboarding = false }): Promise<void> {
  // 1. Flush telemetry BEFORE clearing credentials
  //    Prevents sending org-attributed events after account is wiped
  const { flushTelemetry } = await import('../../utils/telemetry/instrumentation.js')
  await flushTelemetry()

  await removeApiKey()

  // 2. Wipe secure storage (tokens from keychain)
  const secureStorage = getSecureStorage()
  secureStorage.delete()

  // 3. Clear all auth-dependent in-memory caches
  await clearAuthRelatedCaches()

  // 4. Update global config
  saveGlobalConfig(current => ({
    ...current,
    oauthAccount: undefined,        // clear account info
    ...(clearOnboarding && {
      hasCompletedOnboarding: false,
      subscriptionNoticeCount: 0,
      hasAvailableSubscription: false,
    }),
  }))
}
```

### clearAuthRelatedCaches() invalidates:

- OAuth token memoize cache (`getClaudeAIOAuthTokens.cache?.clear()`)
- Trusted device token cache
- Betas and tool schema caches
- User data cache (must clear before GrowthBook refresh)
- GrowthBook feature flag cache
- Grove config cache (notice + settings)
- Remotely managed settings cache
- Policy limits cache

**Telemetry-first ordering:** Flushing telemetry before credential wipe ensures that in-flight events carrying org attribution are delivered under the correct account context. Flushing after wipe would send them as "anonymous", potentially corrupting usage analytics.

---

## Authorization URL Construction

`buildAuthUrl()` assembles the authorization URL with all required OAuth + PKCE parameters plus optional hints.

### buildAuthUrl() — full parameter set

```typescript
export function buildAuthUrl({ codeChallenge, state, port, isManual,
  loginWithClaudeAi, inferenceOnly, orgUUID, loginHint, loginMethod }) {

  // Choose authorization server based on account type
  const authUrlBase = loginWithClaudeAi
    ? 'https://claude.com/cai/oauth/authorize'   // 307s to claude.ai
    : 'https://platform.claude.com/oauth/authorize'

  const authUrl = new URL(authUrlBase)
  authUrl.searchParams.append('code',              'true')  // show Claude Max upsell
  authUrl.searchParams.append('client_id',         CLIENT_ID)
  authUrl.searchParams.append('response_type',     'code')
  authUrl.searchParams.append('redirect_uri',      isManual
    ? 'https://platform.claude.com/oauth/code/callback'
    : `http://localhost:${port}/callback`)
  authUrl.searchParams.append('scope',             scopesToUse.join(' '))
  authUrl.searchParams.append('code_challenge',    codeChallenge)
  authUrl.searchParams.append('code_challenge_method', 'S256')
  authUrl.searchParams.append('state',             state)

  // Optional: pre-fill login form (standard OIDC)
  if (loginHint)   authUrl.searchParams.append('login_hint',   loginHint)
  // Optional: request specific login method
  if (loginMethod) authUrl.searchParams.append('login_method', loginMethod)
  // Optional: target specific org
  if (orgUUID)     authUrl.searchParams.append('orgUUID',      orgUUID)

  return authUrl.toString()
}
```

The `?code=true` parameter is a Claude-specific flag telling the login page to display the Claude Max subscription upsell. It is not a standard OAuth parameter.

---

## Enterprise and FedStart Configuration

For US federal / FedRAMP deployments (_FedStart_), all OAuth endpoints can be redirected to an approved base URL via the `CLAUDE_CODE_CUSTOM_OAUTH_URL` environment variable.

### Allowlist Enforcement in getOauthConfig()

```typescript
const ALLOWED_OAUTH_BASE_URLS = [
  'https://beacon.claude-ai.staging.ant.dev',
  'https://claude.fedstart.com',
  'https://claude-staging.fedstart.com',
]

const oauthBaseUrl = process.env.CLAUDE_CODE_CUSTOM_OAUTH_URL
if (oauthBaseUrl) {
  const base = oauthBaseUrl.replace(/\/$/, '')
  if (!ALLOWED_OAUTH_BASE_URLS.includes(base)) {
    throw new Error('CLAUDE_CODE_CUSTOM_OAUTH_URL is not an approved endpoint.')
  }
  // Override all OAuth URLs to point to FedStart deployment
  config = { ...config, BASE_API_URL: base, CONSOLE_AUTHORIZE_URL: `${base}/oauth/authorize`, ... }
}
```

Strict allowlisting prevents the override from being used to route OAuth tokens to arbitrary servers (credential exfiltration attack).

---

## Key Takeaways

1. **No client secret in the binary.** PKCE replaces it. The code verifier is generated fresh each login attempt, lives only in memory, and is never stored. The S256 challenge is what the server sees.

2. **Auto and manual flows race each other.** Both are started simultaneously. The automatic flow opens a browser and captures the redirect via a temporary localhost server. The manual flow displays a URL for users to visit in any browser and paste the resulting code.

3. **Tokens live in the OS keychain on macOS,** stored as hex via `security -i` (stdin) to keep them out of process argument lists and process monitors. A stale-while-error cache prevents transient subprocess failures from logging users out.

4. **Profile fetching is optimized away during refresh.** If all profile fields are already cached in global config and secure storage, the `/api/oauth/profile` call is skipped. This optimization eliminates millions of API calls per day fleet-wide.

5. **Logout flushes telemetry first.** In-flight analytics events carry org attribution. Clearing credentials before flushing would send those events as anonymous, corrupting usage data. The ordering is intentional and documented in code.

6. **Two separate authentication paths based on scopes.** `user:inference` scope means Claude.ai subscriber—inference goes directly through Claude.ai infrastructure. Absence of that scope means Console path—an API key is created post-login and used for all requests.

7. **Token refresh can expand scopes.** The backend's `ALLOWED_SCOPE_EXPANSIONS` allows refresh grants to include scopes beyond what the initial authorization granted. This lets new scopes (added after the user first logged in) be picked up on next token refresh without requiring re-login.

---

## Knowledge Check

**Q1: Why does the token request include `code_verifier` but the authorization request only includes `code_challenge`?**

A) The verifier is too long to fit in a redirect URL
B) The server stores the challenge, then verifies SHA-256(verifier) matches it — if the auth code was intercepted, the attacker doesn't know the verifier
C) The verifier is the client secret under a different name
D) PKCE is only needed on the token endpoint, not the auth endpoint

**Answer:** B

---

**Q2: What happens if the user's browser can't reach `localhost:PORT/callback`?**

A) Login fails with an error
B) Claude Code retries with a different port
C) The user can fall back to the manual URL shown in the terminal and paste the code
D) The flow times out and re-opens the browser

**Answer:** C

---

**Q3: What triggers `shouldUseClaudeAIAuth()` to return `true`?**

A) The user has a Pro subscription
B) The granted scopes include `user:inference`
C) The user authenticated via `claude.ai` as the auth server base URL
D) The token has `user:profile` scope

**Answer:** B

---

**Q4: Why is telemetry flushed before clearing credentials in `performLogout()`?**

A) Telemetry needs the API key to authenticate its export
B) The telemetry endpoint is the same as the OAuth endpoint
C) In-flight events carry org attribution; flushing after wipe would send them as anonymous, corrupting usage data
D) It's just convention with no functional consequence

**Answer:** C

---

**Q5: On macOS, why is the keychain payload converted to hex before storage?**

A) The keychain only accepts hexadecimal values
B) To avoid shell quoting issues and prevent process monitors (e.g., CrowdStrike) from seeing the raw token in process argument lists
C) Hex is more compact than base64 for JSON payloads
D) The `security` CLI requires hex for the `-X` flag to work on all macOS versions

**Answer:** B

---

**Q6: When is the `/api/oauth/profile` call skipped during token refresh?**

A) When the refresh token is less than 1 hour old
B) When global config has billingType/accountCreatedAt/subscriptionCreatedAt AND secure storage already has non-null subscriptionType and rateLimitTier
C) Always—profile is only fetched on initial login
D) When the user has an enterprise subscription

**Answer:** B

# Lesson 48: Desktop App Integration

## Overview

Claude Code operates as a terminal application with extensive desktop integration capabilities. It can transfer active sessions to Claude Desktop, automatically recognize IDEs (VS Code, Cursor, JetBrains family), establish Chrome extension connections via native messaging, and provide computer-use MCP server functionality. These four integration layers operate independently without shared global state.

**Source files covered:**
- `commands/desktop/index.ts`
- `commands/desktop/desktop.tsx`
- `components/DesktopHandoff.tsx`
- `components/DesktopUpsell/DesktopUpsellStartup.tsx`
- `utils/desktopDeepLink.ts`
- `utils/claudeDesktop.ts`
- `utils/ide.ts`
- `utils/claudeInChrome/`
- `utils/computerUse/`
- `entrypoints/cli.tsx`

**Four integration layers:**

1. **Claude Desktop Handoff** — `/desktop` command flushes sessions and opens via deep link
2. **IDE Integration** — Auto-detects VS Code, Cursor, Windsurf, 14+ JetBrains IDEs via lockfiles
3. **Claude in Chrome** — Native messaging host manifest for browser extension tool invocation
4. **Computer Use MCP** — Subscription-gated OS-level automation (screenshots, input, app control)

---

## Section 02: The `/desktop` Command and Deep Links

### Command Registration

The `/desktop` command (alias `/app`) resides in `commands/desktop/index.ts` and is restricted to `claude-ai` product builds. Platform support check:

```javascript
function isSupportedPlatform(): boolean {
  if (process.platform === 'darwin') return true
  if (process.platform === 'win32' && process.arch === 'x64') return true
  return false
}

const desktop = {
  type: 'local-jsx',
  name: 'desktop',
  aliases: ['app'],
  description: 'Continue the current session in Claude Desktop',
  availability: ['claude-ai'],
  isEnabled: isSupportedPlatform,
  get isHidden() { return !isSupportedPlatform() },
  load: () => import('./desktop.js'),
}
```

Windows support is limited to x64 architecture; ARM Windows and Linux are excluded.

### The Handoff State Machine

`DesktopHandoff` component (`components/DesktopHandoff.tsx`) implements a six-state machine:

- `checking` → validates desktop app installation and version
- `prompt_download` → user prompted if version too old or not installed
- `flushing` → writes buffered conversation to disk
- `opening` → initiates deep link URL opening
- `success` → confirms successful handoff
- `error` → handles OS URL-opening failures

Minimum required desktop version: `MIN_DESKTOP_VERSION = '1.1.2396'` (hard-coded in `utils/desktopDeepLink.ts`).

### Deep Link Construction

The handoff URL is built by `buildDesktopDeepLink`:

```javascript
function buildDesktopDeepLink(sessionId: string): string {
  const protocol = isDevMode() ? 'claude-dev' : 'claude'
  const url = new URL(`${protocol}://resume`)
  url.searchParams.set('session', sessionId)
  url.searchParams.set('cwd', getCwd())
  return url.toString()
}
```

Final URL format: `claude://resume?session=<uuid>&cwd=<path>`

Claude Desktop registers the `claude://` protocol handler via Electron's `setAsDefaultProtocolClient`. Development mode uses AppleScript instead of the `open` command because the dev Electron binary registers itself rather than the app bundle:

```javascript
const { code } = await execFileNoThrow('osascript', [
  '-e',
  `tell application "Electron" to open location "${deepLinkUrl}"`,
])
```

### Platform-Specific Install Detection

**macOS:**
- Checks `/Applications/Claude.app` existence
- Reads version from `defaults read …/Info.plist CFBundleShortVersionString`

**Linux:**
- Runs `xdg-mime query default x-scheme-handler/claude`
- Verifies non-empty stdout (exit code alone unreliable)

**Windows:**
- Queries `HKEY_CLASSES_ROOT\claude` via `reg query`
- Discovers version in `%LOCALAPPDATA%\AnthropicClaude\app-*\`

### Session Flush: The Critical Step

Before opening the deep link, the `flushing` state calls `flushSessionStorage()`. This writes all buffered conversation turns to disk in `~/.claude/projects/`. Without the flush, Claude Desktop would open an incomplete session transcript.

---

## Section 03: Startup Upsell Dialog

`components/DesktopUpsell/DesktopUpsellStartup.tsx` proactively suggests desktop app migration at startup.

### Dialog Visibility Conditions

```javascript
export function shouldShowDesktopUpsellStartup(): boolean {
  if (!isSupportedPlatform()) return false
  if (!getDesktopUpsellConfig().enable_startup_dialog) return false
  const config = getGlobalConfig()
  if (config.desktopUpsellDismissed) return false
  if ((config.desktopUpsellSeenCount ?? 0) >= 3) return false
  return true
}
```

The dialog presents three options:
- **Open in Claude Code Desktop** — triggers full handoff
- **Not now** — increments display counter
- **Never ask again** — sets `desktopUpsellDismissed: true`

Controlled by `tengu_desktop_upsell` GrowthBook experiment (ships disabled by default). Config persists in `~/.claude/config.json` via `saveGlobalConfig`.

---

## Section 04: Importing MCP Servers from Claude Desktop

`utils/claudeDesktop.ts` reads MCP server configuration from Claude Desktop. Platform-specific config paths:

```javascript
export async function getClaudeDesktopConfigPath(): Promise<string> {
  if (platform === 'macos') {
    return join(homedir(), 'Library', 'Application Support', 'Claude',
                'claude_desktop_config.json')
  }
  // WSL: tries USERPROFILE, then scans /mnt/c/Users/* for AppData/Roaming/Claude/
}
```

The file extracts `mcpServers` map. Each entry validates against `McpStdioServerConfigSchema()` (Zod). `MCPServerDesktopImportDialog` component provides multi-select for cherry-picking servers.

**WSL path conversion:** Windows paths like `C:\Users\moiz\AppData\Roaming\Claude` convert to `/mnt/c/Users/moiz/AppData/Roaming/Claude` by stripping drive letters and prepending `/mnt/c`.

---

## Section 05: IDE Integration

IDE detection occurs in `utils/ide.ts` and adjusts behavior when Claude Code runs in embedded IDE terminals (IDE-specific file APIs, faster query channels).

### Supported IDEs

`supportedIdeConfigs` map covers two families:

**VS Code family:** cursor, windsurf, vscode
- Detection: process names like `Cursor Helper`, `Code Helper`
- Transport: HTTP/WebSocket to local port

**JetBrains family:** intellij, pycharm, webstorm, phpstorm, rubymine, clion, goland, rider, datagrip, appcode, dataspell, aqua, gateway, fleet, androidstudio (15 total)
- Transport: SSE or WebSocket via lockfile-advertised port

### Lockfile-Based Discovery

IDE plugins write to `~/.claude/ide/<port>.lock` at startup. Claude Code reads these to locate running IDEs and open workspace folders:

```javascript
type LockfileJsonContent = {
  workspaceFolders?: string[]
  pid?: number
  ideName?: string
  transport?: 'ws' | 'sse'
  runningInWindows?: boolean
  authToken?: string
}
```

Lockfiles sort by modification time (newest first) for IDE preference. Stale locks skip via `isProcessRunning(pid)` using `process.kill(pid, 0)` for PID validation.

### Terminal Detection Shortcut

Before scanning lockfiles, the code checks `TERM_PROGRAM` environment variable. If it matches a known IDE process name, `isSupportedTerminal()` returns `true` immediately without filesystem I/O.

### Windows-in-WSL Path Conversion

When IDE runs in native Windows but Claude Code runs in WSL, `utils/idePathConversion.ts` provides `WindowsToWSLConverter` transforming `C:\Users\moiz\project` to `/mnt/c/Users/moiz/project`. `checkWSLDistroMatch` validates both processes target the same WSL distribution before accepting cross-boundary connections.

---

## Section 06: Claude in Chrome — Native Messaging

Claude Code acts as a native messaging host for Chrome/Chromium extensions, allowing browser extensions to invoke Claude Code tools from webpages. Implementation in `utils/claudeInChrome/`.

### Entrypoint Flags

Two special CLI flags in `entrypoints/cli.tsx` (before normal startup):

```javascript
if (process.argv[2] === '--claude-in-chrome-mcp') {
  await runClaudeInChromeMcpServer()
  return
} else if (process.argv[2] === '--chrome-native-host') {
  await runChromeNativeHost()
  return
}
```

### Native Messaging Host Manifest

Chrome requires a JSON manifest mapping host identifier to binary path. Claude Code auto-installs at startup when `shouldEnableClaudeInChrome()` returns true:

```javascript
const NATIVE_HOST_IDENTIFIER = 'com.anthropic.claude_code_browser_extension'

const execCommand = `"${process.execPath}" --chrome-native-host`
void createWrapperScript(execCommand)
  .then(manifestBinaryPath => installChromeNativeHostManifest(manifestBinaryPath))
```

(Native host manifests cannot contain CLI arguments directly, requiring a wrapper script.)

### Supported Browsers

`CHROMIUM_BROWSERS` map in `utils/claudeInChrome/common.ts` covers Chromium-based browsers with platform-specific data paths and native messaging directories:

```javascript
export const CHROMIUM_BROWSERS: Record<ChromiumBrowser, BrowserConfig> = {
  chrome:  { macos: { nativeMessagingPath: ['Library', 'Application Support',
                      'Google', 'Chrome', 'NativeMessagingHosts'] }, ... },
  // Also: chromium, brave, edge, opera, vivaldi, arc ...
}
```

### Activation Conditions

`shouldEnableClaudeInChrome()` checks in priority order:

1. Disabled by default in non-interactive (SDK/CI) sessions
2. `--chrome` / `--no-chrome` CLI flags override all
3. `CLAUDE_CODE_ENABLE_CFC` environment variable
4. `claudeInChromeDefaultEnabled` field in `~/.claude/config.json`
5. Auto-enable: interactive session + extension installed + `tengu_chrome_auto_enable` GrowthBook flag

---

## Section 07: Computer Use MCP — OS-Level Automation

The most powerful (most heavily gated) integration enables screenshots, mouse/keyboard control, and arbitrary desktop app interaction. Exposed as separate MCP server under codename "Chicago" / "Malort".

### Entrypoint Flag

```javascript
if (feature('CHICAGO_MCP') && process.argv[2] === '--computer-use-mcp') {
  await runComputerUseMcpServer()
  return
}
```

The `feature('CHICAGO_MCP')` call is a build-time dead-code-elimination gate — entire branch removed from external builds lacking this flag.

### Access Gates

`utils/computerUse/gates.ts` implements layered gating:

```javascript
export function getChicagoEnabled(): boolean {
  if (process.env.USER_TYPE === 'ant' && process.env.MONOREPO_ROOT_DIR
      && !isEnvTruthy(process.env.ALLOW_ANT_COMPUTER_USE_MCP)) {
    return false
  }
  return hasRequiredSubscription() && readConfig().enabled
}
```

Ant employees with monorepo access are excluded unless `ALLOW_ANT_COMPUTER_USE_MCP=1`. External users need Max or Pro subscription plus GrowthBook gate.

GrowthBook experiment key: `tengu_malort_pedway`. Ships `ChicagoConfig` object with fine-grained sub-gates:

```javascript
type ChicagoConfig = CuSubGates & {
  enabled: boolean
  coordinateMode: 'pixels' | 'normalized'
  pixelValidation: boolean
  clipboardPasteMultiline: boolean
  mouseAnimation: boolean
  hideBeforeAction: boolean
  autoTargetDisplay: boolean
  clipboardGuard: boolean
}
```

### Terminal Bundle ID Awareness

Since Claude Code has no window, computer-use needs to identify the hosting terminal to exclude it from screenshots and avoid blocking its own input. `utils/computerUse/common.ts` resolves via `__CFBundleIdentifier` with static fallback table:

```javascript
const TERMINAL_BUNDLE_ID_FALLBACK = {
  'iTerm.app':    'com.googlecode.iterm2',
  'Apple_Terminal': 'com.apple.Terminal',
  'ghostty':      'com.mitchellh.ghostty',
  'WarpTerminal': 'dev.warp.Warp-Stable',
  'vscode':       'com.microsoft.VSCode',
  // ...
}
```

Coordinate mode (`pixels` vs `normalized`) freezes at first read — a live GrowthBook flip mid-session would otherwise cause the model to think in one space while the executor transforms in another.

---

## Section 08: CLI Entrypoint — Desktop Fast Paths

All desktop subsystems hook into CLI via special argument checks at top of `entrypoints/cli.tsx`, before module loading or config initialization. This keeps host/server code paths lean.

Fast-path flow:
- `--claude-in-chrome-mcp` → `runClaudeInChromeMcpServer` → exit
- `--chrome-native-host` → `runChromeNativeHost` → exit
- `--computer-use-mcp` → `runComputerUseMcpServer` (requires `CHICAGO_MCP` feature) → exit
- `--daemon-worker` → `runDaemonWorker` → exit
- `remote-control` / `rc` → `bridgeMain` → exit
- Normal → full CLI boot

Each fast path returns immediately after completion, eliminating overhead from normal boot pipeline. This matters for Chrome native messaging — Chrome spawns host on demand expecting response within milliseconds.

---

## Section 09: The Native Installer

Claude Code ships in two modes: npm-installed JavaScript bundle and pre-compiled native binary. Native installer in `utils/nativeInstaller/` manages binary lifecycle alongside npm installation.

### Directory Layout

```
~/.local/share/claude/versions/     # permanent — one dir per version
~/.cache/claude/staging/            # temporary — download target before rename
~/.local/state/claude/locks/        # PID-based lock files for coordination
~/.local/bin/claude                 # symlink → active version
```

Installer keeps `VERSION_RETENTION_COUNT = 2` versions on disk, deleting older ones after successful update. Updates are multi-process safe via PID-based lockfiles with 7-day stale timeout (survives laptop sleep).

### Platform Targeting

`getPlatform()` produces strings like `darwin-arm64`, `linux-x64`, `linux-x64-musl`, `win32-x64`. Musl variant auto-detected via `envDynamic.isMuslEnvironment()` for Alpine Linux / Docker.

### Portable Session Storage

`utils/sessionStoragePortable.ts` contains pure-Node.js utilities deliberately free of internal dependencies. This file is shared verbatim with VS Code extension package at `packages/claude-vscode/src/common-host/sessionStorage.ts` — same code reads session transcripts in both CLI and IDE plugin.

---

## Section 10: The Full Integration Map

Claude Code CLI bridges to:

**Claude Desktop (Electron):**
- `/desktop` command flushes session + opens `claude://resume` deep link
- Session reader accesses `~/.claude/projects/`

**IDE Plugin (VS Code / JetBrains):**
- Reads `~/.claude/ide/*.lock` lockfiles
- Communicates via local HTTP/WS RPC server

**Chrome Extension:**
- Native messaging manifest at `com.anthropic.claude_code_browser_extension`
- MCP server spawned with `--claude-in-chrome-mcp`

**OS (macOS):**
- Computer Use MCP with `--computer-use-mcp`
- Screenshots, mouse control, keyboard input

---

## Key Takeaways

- The `/desktop` command is a 6-state React machine that "flushes the session transcript, builds a `claude://resume?session=…&cwd=…` URL, then hands off to the desktop app via OS-native URL opening"
- Desktop install detection: path check (macOS), `xdg-mime` (Linux), registry query (Windows)
- IDE detection uses `~/.claude/ide/<port>.lock` files supplemented by `TERM_PROGRAM` environment variable
- Claude in Chrome uses Chrome's native messaging protocol with manifest mapping `com.anthropic.claude_code_browser_extension` to wrapper script invoking CLI with `--chrome-native-host`
- Computer use is triple-gated: build-time feature flag (`CHICAGO_MCP`), Max/Pro subscription, GrowthBook experiment key `tengu_malort_pedway`
- All desktop-mode fast paths handled before config/analytics initialization to minimize startup overhead
- `utils/sessionStoragePortable.ts` is dependency-free for sharing with VS Code extension package

---

## Knowledge Check

**Q1:** What is the minimum Claude Desktop version for `/desktop` handoff?
- A: 1.0.0
- B: 1.1.2396 (correct)
- C: 2.0.0
- D: No minimum version check

**Q2:** Why does `DesktopHandoff` call `flushSessionStorage()` before opening the deep link?
- A: Free disk space
- B: Encrypt session for transfer
- C: "Write all buffered conversation turns to disk so Claude Desktop can read the complete session transcript" (correct)
- D: Close open file handles

**Q3:** Why does macOS dev mode use AppleScript instead of `open` command?
- A: AppleScript is faster
- B: "The dev Electron binary (not a .app bundle) registers the protocol handler, so `open` would launch a bare Electron executable without app code" (correct)
- C: `open` unavailable on dev machines
- D: AppleScript works on older macOS

**Q4:** How does Claude Code detect IDE before reading lockfiles?
- A: Scan all running processes
- B: "Check the `TERM_PROGRAM` environment variable against a table of known IDE names" (correct)
- C: Read IDE install-time config file
- D: Check parent process via `/proc/ppid/exe`

**Q5:** What is the native messaging host identifier for Claude in Chrome?
- A: `com.google.chrome.claude`
- B: `com.anthropic.claude_code_browser_extension` (correct)
- C: `claude-in-chrome`
- D: `anthropic.claude.native`

**Q6:** Why is computer use coordinate mode "frozen at first read"?
- A: Performance optimization
- B: "A live GrowthBook flip mid-session could make the model think in pixels while the executor transforms in normalized coordinates, causing misclicks" (correct)
- C: OS sets coordinate mode at runtime
- D: Prevent concurrent session conflicts

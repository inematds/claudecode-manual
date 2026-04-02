# mdENG -- Lesson 27: The Voice System — Complete Content

## Overview
This lesson covers Claude Code's push-to-talk voice input pipeline, including WebSocket speech-to-text, multi-backend audio capture, and resilience patterns.

## 1. What the Voice System Does

The system converts spoken speech into prompt text via key hold. Two independent gates protect access: a GrowthBook feature flag (`VOICE_MODE`) and Anthropic OAuth token—neither alone is sufficient.

**High-level flow diagram:**
- `useVoiceIntegration.tsx` receives keypress
- `useVoice.ts` handles hold detection
- `voice.ts` manages audio backend (NAPI/arecord/SoX)
- `voiceStreamSTT.ts` sends PCM chunks via WebSocket
- Anthropic `voice_stream` endpoint returns TranscriptText

**Four key files:**
- `voice/voiceModeEnabled.ts` — auth + kill-switch checks
- `services/voice.ts` — audio recording backends
- `services/voiceStreamSTT.ts` — WebSocket STT client
- `hooks/useVoice.ts` — React hook wiring audio → WS → transcript
- `hooks/useVoiceIntegration.tsx` — prompt-input integration, hold-threshold, interim rendering

## 2. Feature Gating and Auth

**isVoiceGrowthBookEnabled()** — the kill-switch

```
export function isVoiceGrowthBookEnabled(): boolean {
  return feature('VOICE_MODE')
    ? !getFeatureValue_CACHED_MAY_BE_STALE('tengu_amber_quartz_disabled', false)
    : false
}
```

GrowthBook flag `tengu_amber_quartz_disabled` defaults to `false` (not disabled). Missing or stale disk cache reads as "not killed"—fresh installs work immediately. Flipping to `true` emergency-disables voice for all users within GrowthBook's next cache refresh cycle.

**hasVoiceAuth()** — OAuth token check

```
export function hasVoiceAuth(): boolean {
  if (!isAnthropicAuthEnabled()) return false
  const tokens = getClaudeAIOAuthTokens()
  return Boolean(tokens?.accessToken)
}
```

`getClaudeAIOAuthTokens()` spawns `security` on macOS (~20-50ms cold, cached afterward). Memoize clears on token refresh (~once per hour). API keys, Bedrock, Vertex, and Foundry return false—`voice_stream` only available via Claude.ai OAuth.

**/voice command preflight checks**

Five sequential checks before writing `voiceEnabled: true` to settings:

1. **isVoiceModeEnabled()** — auth + GB gate
2. **checkRecordingAvailability()** — rejects remote environments; probes audio backend chain
3. **isVoiceStreamAvailable()** — re-checks OAuth token freshness
4. **checkVoiceDependencies()** — verifies ≥1 recording tool usable
5. **requestMicrophonePermission()** — fires OS TCC dialog (macOS) early rather than on first hold-to-talk

```
if (!(await requestMicrophonePermission())) {
  const guidance = process.platform === 'darwin'
    ? 'System Settings → Privacy & Security → Microphone'
    : 'your system\'s audio settings'
  return { type: 'text', value: `Microphone access denied. Go to ${guidance}.` }
}
```

## 3. Audio Recording Backends

**services/voice.ts**

Single `startRecording(onData, onEnd, options?)` interface over three possible backends, tried in priority order:

| Backend | Platforms | Notes | Status |
|---------|-----------|-------|--------|
| audio-capture-napi | macOS, Linux (ALSA), Windows | In-process via cpal + CoreAudio/AudioUnit. `dlopen` blocks ~1s warm, ~8s cold on macOS—loaded lazily on first voice keypress only | Primary |
| arecord | Linux only | ALSA userspace utility. Probed via 150ms race: if alive = device opened OK. Handles WSL2+WSLg (Win11) via PulseAudio RDP pipes; fails on WSL1/Win10-WSL2 | Fallback 1 |
| SoX (rec) | macOS, Linux | External process piping raw PCM. Requires `--buffer 1024` to prevent internal buffering delay. Built-in silence detection | Fallback 2 |
| Windows (no native) | Windows only | No subprocess fallback on Windows. Native module required | No fallback |

**Why arecord needs runtime probe, not just PATH check**

On WSL1 and headless Linux, `arecord` is installed but `open()` fails immediately—no ALSA card and no PulseAudio server. `hasCommand('arecord')` returns `true` in all cases. Probe works by spawning `arecord` with the same arguments as the real session and racing a 150ms timer:

```
const timer = setTimeout((child, resolve) => {
  child.kill('SIGTERM')
  resolve({ ok: true, stderr: '' })   // still alive = opened OK
}, 150, child, resolve)

child.once('close', code => {
  clearTimeout(timer)
  resolve({ ok: code === 0, stderr: stderr.trim() }) // exited early = failed
})
```

Result is memoized—audio device availability doesn't change mid-session. Runs on every voice keypress via `checkRecordingAvailability()`.

**SoX argument details and silence detection**

```
const args = [
  '-q',               // quiet: no progress output
  '--buffer', '1024', // flush in small chunks
  '-t', 'raw',        // raw PCM, no WAV header
  '-r', '16000',      // 16kHz sample rate (STT requirement)
  '-e', 'signed',     // signed PCM
  '-b', '16',         // 16-bit depth
  '-c', '1',          // mono
  '-',                // write to stdout
]
// Silence detection (only when NOT in push-to-talk mode)
if (useSilenceDetection) {
  args.push('silence', '1', '0.1', '3%', '1', '2.0', '3%')
  // ↑ stop after 2 seconds of audio below 3% threshold
}
```

Push-to-talk passes `{ silenceDetection: false }`—user controls start/stop. Native NAPI module ignores its built-in `onEnd` in push-to-talk mode.

## 4. WebSocket STT Protocol

**services/voiceStreamSTT.ts**

Connects to `wss://api.anthropic.com/api/ws/speech_to_text/voice_stream`. Uses same OAuth Bearer token as rest of Claude Code. URL includes query parameters configuring STT session:

```
const params = new URLSearchParams({
  encoding:        'linear16',   // 16-bit signed PCM
  sample_rate:     '16000',
  channels:        '1',          // mono
  endpointing_ms:  '300',        // endpoint detection window
  utterance_end_ms:'1000',
  language:        options?.language ?? 'en',
})
```

**Wire protocol messages**

→ **KeepAlive**: JSON control. Sent immediately on open (prevents server close before audio starts), then every 8s.

→ **binary frame**: Raw PCM audio chunks from recording backend. Copied via `Buffer.from()` to prevent NAPI shared-ArrayBuffer races.

→ **CloseStream**: JSON control. Signals end of audio. Sent in `setTimeout(0)` to flush any queued NAPI callbacks first.

← **TranscriptText**: Interim transcript chunk. May be revised by subsequent messages. Emitted as `isFinal=false`.

← **TranscriptEndpoint**: Signals utterance boundary. Promotes last `TranscriptText` to `isFinal=true`. After `CloseStream`, resolves `finalize()` fast (~300ms).

← **TranscriptError**: STT error (e.g., unsupported language closes with code 1008). Forwarded to caller's `onError`.

**Why api.anthropic.com instead of claude.ai**

The claude.ai Cloudflare zone uses TLS fingerprinting and blocks non-browser clients (JA3 fingerprint mismatch). The `api.anthropic.com` listener exposes the same private-api pod with same OAuth auth but is on a CF zone that doesn't enforce browser-class TLS fingerprinting. Desktop dictation still uses `claude.ai` because Swift's URLSession has browser-class JA3 fingerprint and passes challenge.

```
// Override via env var for testing/staging
const wsBaseUrl =
  process.env.VOICE_STREAM_BASE_URL ||
  getOauthConfig().BASE_API_URL
    .replace('https://', 'wss://')
    .replace('http://', 'ws://')
```

**finalize() — four resolution triggers**

`finalize()` returns `Promise<FinalizeSource>` resolving via whichever of four paths fires first:

| Source | Condition | Typical latency |
|--------|-----------|-----------------|
| post_closestream_endpoint | TranscriptEndpoint arrives after CloseStream sent | ~300ms |
| no_data_timeout | No TranscriptText arrived after CloseStream (1.5s) | 1.5s |
| ws_close | WebSocket close event fires | 3–5s |
| safety_timeout | Last-resort cap | 5s |

`no_data_timeout` is the _silent-drop signature_—if it fires with `hadAudioSignal=true`, session hit known server bug (sticky CE pod returning zero transcripts, ~1% of sessions).

## 5. Hold-to-Talk Mechanics and Hold Threshold

**hooks/useVoiceIntegration.tsx**

Terminal key events arrive as stream: one event on initial press, then auto-repeat every 30–80ms while held. No "keyup" event in terminal. System reconstructs hold by timing gaps between events.

**The hold threshold problem:** bare-character binding like Space could be normal typed space or start of hold. Requiring N rapid consecutive presses before activating voice prevents accidental triggers during normal typing.

```
const RAPID_KEY_GAP_MS = 120   // auto-repeat fires every 30-80ms
const HOLD_THRESHOLD = 5       // 5 rapid presses required
const WARMUP_THRESHOLD = 2     // show "keep holding…" at press #2
```

Modifier-combo bindings (e.g., Ctrl+Space) activate on first press—no warmup required, because modifier combo is unambiguously intentional.

**stripTrailing() — cleaning up leaked spaces**

While user holds Space for warmup, some space characters leak into text input before `stopImmediatePropagation()` takes effect (listener registration order not guaranteed). `stripTrailing()` removes exactly the leakage count without touching pre-existing spaces at boundary:

```
const stripTrailing = (maxStrip, { char = ' ', anchor = false, floor = 0 } = {}) => {
  // Also counts full-width spaces (U+3000) for CJK IME compatibility
  const scan = char === ' '
    ? normalizeFullWidthSpace(beforeCursor)
    : beforeCursor
  // ...
  if (anchor) {
    voicePrefixRef.current = stripped         // save text before cursor
    voiceSuffixRef.current = afterCursor      // save text after cursor
  }
}
```

When `anchor=true`, call also captures cursor position for interim transcript injection. Gap space inserted between prefix and suffix ensures waveform cursor sits on gap rather than first suffix letter.

**Release detection: RELEASE_TIMEOUT_MS and REPEAT_FALLBACK_MS**

```
const RELEASE_TIMEOUT_MS = 200    // gap signals key release (auto-repeat is 30-80ms)
const REPEAT_FALLBACK_MS = 600    // arm release timer if no auto-repeat seen yet
const FIRST_PRESS_FALLBACK_MS = 2000 // modifier combos: OS initial repeat up to ~2s
```

When no second keypress arrives within 600ms, fallback timer arms release detection. For modifier combos, callers pass 2000ms to cover long OS initial repeat delay (macOS slider at "Long" = ~2s before auto-repeat starts).

## 6. Session State Machine

Each recording session moves through three states managed by `useVoice`:

**State: idle**
No recording active. WS closed. Prompt input normal.

**State: recording**
Audio capturing. PCM streaming to WS. Interim transcript shown in prompt.

**State: processing**
Key released. Audio stopped. Waiting for finalize() and final transcript.

**Critical implementation detail:** `updateState('recording')` called _synchronously before any await_ in `startRecordingSession()`. `useVoiceIntegration` reads `voiceState` from store immediately after `void startRecordingSession()` to gate whether leaked space keypresses should be swallowed. If await ran first, guard would see stale `'idle'` and let spaces leak.

**Full session lifecycle timeline**

**Hold threshold reached (press #5)**
stripTrailing with anchor=true captures prefix/suffix. State → recording synchronously. sessionGenRef++.

**checkRecordingAvailability() await**
Probes backend chain (memoized after first call).

**startRecording() + connectVoiceStream() — parallel**
Audio capture starts immediately. PCM chunks buffer in audioBuffer[] until WS opens. Keyterms gathered (git branch, recent files) before WS connect.

**onReady fires (WS open)**
audioBuffer flushed to WS. Subsequent chunks sent directly. KeepAlive starts (8s interval).

**TranscriptText messages arrive**
Interim shown in prompt at cursor position. Non-cumulative new segments auto-finalized (legacy Deepgram only).

**Key released (gap > 200ms)**
stopRecording(). State → processing. finalize() called. recordingDurationMs captured before await.

**finalize() resolves**
Typically via TranscriptEndpoint (~300ms) or no_data_timeout (1.5s).

**onTranscript(text) called**
Final transcript injected at cursor anchor. State → idle.

## 7. Silent-Drop Replay

**hooks/useVoice.ts**

Approximately 1% of sessions hit server-side bug (session-sticky CE pod accepting audio but returning zero transcripts). Symptom: `finalize()` resolves via `no_data_timeout` despite real speech. Client detects pattern and replays full audio buffer on fresh WebSocket once.

**Silent-drop detection conditions and replay code**

All six conditions must be true to trigger replay:

1. `finalizeSource === 'no_data_timeout'`
2. `hadAudioSignal === true` (non-trivial mic signal detected)
3. `wsConnected === true` (WS did open—backend received audio)
4. `!focusTriggered` (not a focus-mode session)
5. `accumulatedRef.current.trim() === ''` (no partial transcript accumulated)
6. `!silentDropRetriedRef.current` (replay only once per session)

```
if (finalizeSource === 'no_data_timeout' && hadAudioSignal && wsConnected
    && !focusTriggered && focusFlushedChars === 0
    && accumulatedRef.current.trim() === ''
    && !silentDropRetriedRef.current
    && fullAudioRef.current.length > 0) {
  silentDropRetriedRef.current = true
  await sleep(250)          // backoff to clear rapid-reconnect same-pod race
  if (isStale()) return

  // Replay full buffer in 32 KB slices on fresh connection
  const SLICE = 32_000
  for (const chunk of replayBuffer) {
    // ... batch into SLICE-sized sends ...
    conn.send(Buffer.concat(slice))
  }
  await conn.finalize()
}
```

Audio buffer is bounded: `fullAudioRef.current` skips buffering in focus mode (where sessions can last minutes and buffer could reach ~20MB). 32KB slice size batches small NAPI chunks into reasonable WS frame size without exceeding WS message limits.

## 8. Focus Mode

Focus mode is "multi-clauding army" workflow: recording starts when terminal window gains focus and stops when it loses focus. Transcript chunks flushed immediately (rather than accumulated) so continuous dictation across long sessions stays responsive.

**Focus mode differences from hold-to-talk**

| Behavior | Hold-to-talk | Focus mode |
|----------|--------------|-----------|
| Trigger | Key hold | Terminal focus gain |
| Stop trigger | Key release (gap > 200ms) | Terminal focus lost |
| Transcript delivery | Accumulated, injected on stop | Each final flushed immediately; anchor advanced |
| Silence timeout | None | 5s (FOCUS_SILENCE_TIMEOUT_MS)—tears down session to free WS |
| Silent-drop replay | Yes | No (gated on !focusTriggered) |
| Audio buffer | Full buffer kept for replay | Skipped (dead weight in long sessions) |

```
// Arms / resets silence timer after each flushed transcript
function armFocusSilenceTimer(): void {
  if (focusSilenceTimerRef.current) clearTimeout(focusSilenceTimerRef.current)
  focusSilenceTimerRef.current = setTimeout(() => {
    if (stateRef.current === 'recording' && focusTriggeredRef.current) {
      silenceTimedOutRef.current = true
      finishRecording()           // tears down WS gracefully
    }
  }, FOCUS_SILENCE_TIMEOUT_MS)   // 5000 ms
}
```

## 9. Language Normalization and Keyterms

**hooks/useVoice.ts   services/voiceKeyterms.ts**

### Language normalization

`normalizeLanguageForSTT()` maps user's `settings.language` string (could be "Japanese", "日本語", "ja-JP", etc.) to BCP-47 code from hardcoded allowlist—subset of server's `speech_to_text_voice_stream_config` GrowthBook allowlist. Sending unsupported code closes WebSocket with code 1008 "Unsupported language".

```
// Falls back to 'en' with warning if language unsupported
export function normalizeLanguageForSTT(language): { code: string, fellBackFrom?: string } {
  if (!language) return { code: 'en' }
  const lower = language.toLowerCase().trim()
  if (SUPPORTED_LANGUAGE_CODES.has(lower)) return { code: lower }
  const fromName = LANGUAGE_NAME_TO_CODE[lower]   // "japanese" → "ja"
  if (fromName) return { code: fromName }
  const base = lower.split('-')[0]              // "ja-JP" → "ja"
  if (SUPPORTED_LANGUAGE_CODES.has(base)) return { code: base }
  return { code: 'en', fellBackFrom: language }
}
```

### Keyterms (STT boosting)

`getVoiceKeyterms()` builds list of up to 50 domain-specific terms sent as `keyterms` query parameters. STT backend applies boosting so "MCP", "OAuth", "TypeScript", and project-specific vocabulary are correctly recognized.

**Keyterm sources and identifier splitting**

Keyterms come from three sources, merged into deduplicated Set:

1. **Global hardcoded terms**: MCP, symlink, grep, regex, localhost, TypeScript, OAuth, webhook, gRPC, dotfiles, subagent, worktree (Claude and Anthropic are already base keyterms on server)
2. **Project root basename**: e.g., `claude-code` added as whole term
3. **Git branch words**: `feat/voice-keyterms` → ["feat", "voice", "keyterms"]

```
export function splitIdentifier(name: string): string[] {
  return name
    .replace(/([a-z])([A-Z])/g, '$1 $2')  // camelCase → camel Case
    .split(/[-_./\s]+/)                    // split on separators
    .filter(w => w.length > 2 && w.length <= 20) // discard noise
}
```

## 10. Audio Level Visualization and RMS Computation

While recording, prompt input shows 16-bar waveform. Each new PCM chunk updates rightmost bar by computing RMS amplitude from raw 16-bit signed PCM buffer.

```
const AUDIO_LEVEL_BARS = 16

export function computeLevel(chunk: Buffer): number {
  const samples = chunk.length >> 1   // 16-bit = 2 bytes per sample
  if (samples === 0) return 0
  let sumSq = 0
  for (let i = 0; i < chunk.length - 1; i += 2) {
    // Read 16-bit signed little-endian sample
    const sample = ((chunk[i]! | (chunk[i+1]! << 8)) << 16) >> 16
    sumSq += sample * sample
  }
  const rms = Math.sqrt(sumSq / samples)
  const normalized = Math.min(rms / 2000, 1)
  return Math.sqrt(normalized)   // sqrt curve spreads quieter levels visually
}
```

**The sqrt curve is intentional.** Linear scale compresses most speech energy into top 20% of visual range—waveform looks flat except for loud peaks. Taking `sqrt(normalized)` spreads quieter levels (0.0–0.5) across more bars, making visualization responsive to normal conversational speech.

## 11. React Context Layer

**context/voice.tsx   hooks/useVoiceEnabled.ts**

Voice state stored in custom `Store<VoiceState>` (not React state) held in context. Enables `useSyncExternalStore`-based subscriptions that only re-render when selected slice changes.

```
export type VoiceState = {
  voiceState:             'idle' | 'recording' | 'processing'
  voiceError:             string | null
  voiceInterimTranscript: string   // live preview text shown in prompt
  voiceAudioLevels:       number[] // 16 bars, 0–1 normalized
  voiceWarmingUp:         boolean  // show "keep holding…" hint
}
```

**useVoiceEnabled — why auth is memoized on authVersion**

```
export function useVoiceEnabled(): boolean {
  const userIntent   = useAppState(s => s.settings.voiceEnabled === true)
  const authVersion  = useAppState(s => s.authVersion)
  // authVersion bumps on /login only.
  // getClaudeAIOAuthTokens() spawns `security` (~60ms cold)—can't call every render.
  const authed = useMemo(hasVoiceAuth, [authVersion])
  return userIntent && authed && isVoiceGrowthBookEnabled()
}
```

`isVoiceGrowthBookEnabled()` call stays outside memo so mid-session kill-switch flip takes effect on next render without waiting for login event.

## 12. Early-Error Retry

CE proxy can reject rapid reconnects (~1/N_pods same-pod collision), and Deepgram's upstream can fail during its own teardown window. These manifest as errors before any transcript arrives. System retries once with 250ms backoff.

```
// Only retry if: not fatal (4xx), no transcript seen yet, still recording
if (!opts?.fatal && !sawTranscript && stateRef.current === 'recording') {
  if (!retryUsedRef.current) {
    retryUsedRef.current = true
    connectionRef.current = null       // null → audio re-buffers until new onReady
    attemptGenRef.current++             // stale conn's trailing close is ignored
    setTimeout(() => {
      if (stateRef.current === 'recording') attemptConnect(keyterms)
    }, 250)
    return
  }
}
// Fatal errors (4xx) surface message to user
```

**Fatal vs. transient errors:** 4xx HTTP upgrade rejections (Cloudflare bot challenge, auth rejection) marked `fatal: true` by `unexpected-response` handler. Fatal errors never retried—same request gets same rejection.

## Key Takeaways

1. **Voice is double-gated.** Both GrowthBook `VOICE_MODE` feature flag (compile-time dead-code elimination) and Anthropic OAuth token required. Kill-switch defaults to "not killed" so fresh installs work immediately. API keys, Bedrock, Vertex, Foundry excluded by design.

2. **Backend fallback chain matches availability check chain.** `startRecording()` and `checkRecordingAvailability()` walk same NAPI → arecord → SoX priority order. Memoized `probeArecord()` result ensures if availability check falls through to SoX (broken arecord), recording call does too.

3. **Audio starts before WebSocket opens.** PCM chunks buffer in `audioBuffer[]` until `onReady` fires. Eliminates 1–2s OAuth+WS connect latency from user's perceived recording start.

4. **State transitions to 'recording' synchronously before any await.** Any async work before this transition would let hold-detection code see stale 'idle' and allow auto-repeat key characters to leak into text input.

5. **Hold threshold (5 rapid presses) prevents accidental activation for bare-character bindings.** Modifier combos bypass threshold entirely. `stripTrailing()` cleans up leaked chars without disturbing pre-existing content at cursor boundary.

6. **Silent-drop replay is client-side workaround for server-side bug.** Full audio buffer kept specifically for one-shot replay. Focus mode skips buffering to avoid multi-MB accumulation in long sessions.

7. **`finalize()` has four resolution triggers with different latencies.** Fast path (`post_closestream_endpoint`, ~300ms) is normal case. `no_data_timeout` (1.5s) is silent-drop detector. Always capturing `recordingDurationMs` before `finalize()` await prevents WebSocket teardown time from inflating metric.

8. **Focus mode is architecturally different from hold-to-talk.** Disables audio buffering, replaces accumulation with immediate flush, uses 5-second silence timer instead of key release as stop trigger.

## Quiz

**1. Why does `updateState('recording')` run synchronously before any `await` in `startRecordingSession()`?**

*Correct Answer: B*

Because `useVoiceIntegration` reads `voiceState` synchronously after `void startRecordingSession()` to decide whether to swallow auto-repeat spaces. If any await ran first, code would see stale `'idle'` and fail to swallow auto-repeat spaces, causing spaces to leak into prompt input.

**2. On headless Linux with both `arecord` and `sox` installed, what determines which backend is actually used?**

*Correct Answer: C*

Runtime `probeArecord()` result—if arecord can't open device (exits before 150ms), it falls through to SoX. `hasCommand('arecord')` only checks PATH. On headless Linux, `arecord` exists but `open()` immediately fails with no ALSA card. 150ms race detects this: if arecord exits before timer fires, `probe.ok = false` and SoX used instead. Decision is memoized for session.

**3. What is the `no_data_timeout` resolution source of `finalize()` designed to detect?**

*Correct Answer: C*

Session-sticky CE pod that accepted audio but returned zero transcripts (silent-drop bug). When `no_data_timeout` fires with `hadAudioSignal=true` and `wsConnected=true`, audio reached backend but no transcript came back—signature of ~1% silent-drop bug. Client then replays full audio buffer on fresh WebSocket connection.

**4. Why does `connectVoiceStream()` target `api.anthropic.com` rather than `claude.ai`?**

*Correct Answer: C*

Claude.ai Cloudflare zone blocks non-browser TLS fingerprints; api.anthropic.com exposes same pod without that restriction. Cloudflare TLS fingerprinting (JA3) on claude.ai zone challenges Node.js/Bun WebSocket connections. api.anthropic.com listener exposes same private-api pod with same OAuth Bearer auth but on CF zone that doesn't enforce browser-class fingerprints.

**5. Why is full audio buffer (`fullAudioRef`) skipped in focus mode?**

*Correct Answer: B*

Focus mode sessions can last minutes, making buffer potentially 20+MB; replay gated on `!focusTriggered` anyway. At 32KB/s PCM, 10-minute focus session would accumulate ~20MB. Since replay explicitly gated on `!focusTriggered`, buffer serves no purpose in focus mode and is waste of memory.

**6. What happens when `normalizeLanguageForSTT()` receives unsupported language like "Swahili"?**

*Correct Answer: C*

Falls back to `'en'` and sets `fellBackFrom: 'Swahili'`, which surfaces warning in `/voice` toggle response. Function returns `{ code: 'en', fellBackFrom: 'Swahili' }`. `/voice` command handler checks `stt.fellBackFrom` and appends note like "Swahili is not supported; using English" to enable confirmation message.

# Voice Bot — Documentation

**Repository:** https://github.com/shakthiswaran-M/Voice-bot

A single-file, client-side voice assistant that runs entirely in the browser. It listens through the microphone, transcribes speech with Sarvam's Saaras STT model, sends the transcript to a Sarvam LLM (streamed), and speaks the reply back using the browser's built-in text-to-speech.

No backend, no build step — everything lives in one `.html` file.

---

## 1. Overview

| Layer | Technology |
|---|---|
| UI | Vanilla HTML/CSS (dark "terminal" theme), no framework |
| Speech-to-text | Sarvam `saaras:v3` via `POST https://api.sarvam.ai/speech-to-text` |
| Chat/LLM | Sarvam `sarvam-105b` / `sarvam-105b-conversations` via `POST https://api.sarvam.ai/v1/chat/completions` (streamed SSE) |
| Text-to-speech | Browser-native `speechSynthesis` (Web Speech API) — not a Sarvam service |
| Audio capture | Web Audio API (`getUserMedia` + `ScriptProcessorNode`) with a hand-rolled voice-activity detector (VAD) |

Everything (API key included) lives only in browser memory for the current tab — nothing is persisted to disk or a server.

---

## 2. Feature summary

- **Hands-free turn-taking.** Tap the mic once; the app auto-detects when you start and stop talking (VAD) and sends each utterance without you pressing anything else.
- **Auto language detection.** Sarvam STT can auto-detect the spoken language per utterance (English/Hindi/Tamil/etc., including code-mixed speech) and updates the TTS voice to match.
- **Streaming replies.** The LLM response streams in via SSE and is rendered token-by-token.
- **Fast mode.** Splits the streaming text into complete sentences and starts speaking each sentence as soon as it's ready, instead of waiting for the full reply — lowers perceived latency.
- **Barge-in / interruption.** If the user starts speaking while the bot is talking, playback stops immediately and the in-flight request is aborted.
- **Manual interrupt button.** "■ stop reply" cancels TTS playback and any pending fetch.
- **Voice test button.** Speaks a sample sentence in the currently selected language so users can confirm a system TTS voice is installed before relying on it.
- **Live oscilloscope.** A canvas waveform visualizer (teal while recording, amber while the bot speaks, grey when idle) driven by an `AnalyserNode`.

---

## 3. UI components

| Element | Purpose |
|---|---|
| ⚙ Settings button | Toggles a panel with API key, language, model, and mode controls |
| API key field | Sarvam API key (password-masked, kept in memory only) |
| Auto-detect toggle | On: STT detects language per utterance. Off: STT is locked to the selected language |
| Language select | Fallback STT language when auto-detect is off; always used to choose the TTS voice |
| Model select | `sarvam-105b-conversations` (low-latency) or `sarvam-105b` (128K context, heavier reasoning) |
| Fast mode toggle | Stream + skip deep reasoning + speak sentence-by-sentence |
| Oscilloscope | Live mic/TTS waveform (visual only) |
| Status row | `idle` / `listening` / `thinking` / `speaking` / `error`, plus a first-token latency readout |
| Transcript | Scrolling chat log (user right-aligned/teal, bot left-aligned/grey, system messages centered) |
| Interim line | Shows "listening…" / "transcribing…" while an utterance is in flight |
| Mic button (●) | Starts/stops the always-on listening loop |
| Stop reply button | Interrupts TTS playback and aborts the current LLM request |
| Test voice button | Speaks a canned sentence in the current language |

---

## 4. Pipeline: what happens on one turn

1. **Capture.** While listening, a `ScriptProcessorNode` continuously buffers raw `Float32Array` PCM frames from the mic.
2. **VAD.** Every 120 ms, `vadTick()` computes the RMS level of the input. Above `VAD_THRESHOLD` (0.02) the app considers the user "speaking" and starts/continues recording. Once level drops below threshold, a `VAD_SILENCE_MS` (900 ms) timer starts; if it fires, the utterance is considered finished.
3. **Barge-in.** If the bot is speaking (or a request is in flight) when the user starts talking, TTS is cancelled and the fetch is aborted immediately — the user's new utterance takes over.
4. **Discard short noise.** Utterances shorter than `MIN_UTTERANCE_MS` (350 ms) or whose resulting WAV is under 3 KB are discarded as noise rather than sent.
5. **WAV encoding.** Buffered PCM is downsampled to 16 kHz, converted to 16-bit signed PCM, and wrapped in a minimal WAV header — done manually because Sarvam's STT endpoint accepts wav/mp3/pcm, not the webm/opus that `MediaRecorder` produces in Chrome/Edge/Firefox.
6. **STT request.** The WAV blob is POSTed as `multipart/form-data` to `/speech-to-text` with `model=saaras:v3`, `mode=transcribe`, and `language_code` set to `unknown` (auto-detect) or the selected language.
7. **Language sync.** If auto-detect returned a `language_code`, the language dropdown updates to match (used for the reply's TTS voice).
8. **LLM call.** The transcript is appended to `history` and POSTed to `/v1/chat/completions` with `stream: true`. A system prompt instructs the model to reply in the same language the user spoke and to keep responses conversational/voice-appropriate. `reasoning_effort` is `null` in fast mode, `'low'` otherwise.
9. **Streaming render + sentence-splitting.** Each SSE `data:` chunk's `delta.content` is appended to the transcript bubble live. In fast mode, a regex (`/[^.!?।॥]+[.!?।॥]+[\s]*|[^.!?।॥]+$/g`) extracts complete sentences (including Devanagari/other Indic terminators `।` `॥`) from the growing buffer and queues each for speech as soon as it's complete, keeping the trailing partial sentence buffered.
10. **TTS playback.** Queued sentences are spoken serially via `speechSynthesis`, using a voice matched to the current language code (exact match, then language-prefix match, then system default). A periodic "resume nudge" every 4 s works around a known Chrome bug where `speechSynthesis` can silently pause on long utterances.
11. **Idle.** Status returns to `listening` (if the mic is still on) or `idle`.

---

## 5. Key configuration constants

| Constant | Value | Meaning |
|---|---|---|
| `VAD_THRESHOLD` | `0.02` | RMS level above which audio counts as speech. Raise if it false-triggers on background noise. |
| `VAD_SILENCE_MS` | `900` | Pause length (ms) that ends an utterance and triggers sending it. |
| `MIN_UTTERANCE_MS` | `350` | Utterances shorter than this (after silence) are discarded as noise. |
| WAV min size | `3000` bytes | Encoded clips smaller than this are dropped as non-speech. |
| STT target sample rate | `16000` Hz | Audio is downsampled to this before encoding, per Sarvam's recommendation. |
| TTS resume-nudge interval | `4000` ms | Periodic `speechSynthesis.resume()` call to counter browser auto-pause bugs. |

---

## 6. API usage details

### Speech-to-text — `POST https://api.sarvam.ai/speech-to-text`
- Auth header: `api-subscription-key: <key>`
- Body: `multipart/form-data` with `file` (WAV blob), `model=saaras:v3`, `mode=transcribe`, `language_code` (`unknown` or a BCP-47-style code like `hi-IN`)
- Expected response fields used: `transcript`, `language_code`

### Chat completions — `POST https://api.sarvam.ai/v1/chat/completions`
- Auth headers: both `Authorization: Bearer <key>` and `api-subscription-key: <key>` are sent
- Body: OpenAI-style chat payload — `model`, `messages` (system + rolling `history`), `temperature: 0.3`, `stream: true`, `reasoning_effort` (`null` or `'low'`), `max_tokens: 1024`
- Response: Server-Sent Events; each `data:` line is JSON with `choices[0].delta.content` (or `.text` / `.message.content` fallback), terminated by `data: [DONE]`

### Text-to-speech
- Uses the browser's native `speechSynthesis`/`SpeechSynthesisUtterance` — **not** a Sarvam API. Voice availability, quality, and language coverage depend entirely on the operating system/browser's installed voices.

---

## 7. Compatibility matrix

The app is built entirely on standard browser APIs — no native app, no OS-level install.

| Requirement | Chrome / Edge (desktop) | Firefox (desktop) | Safari 16+ (desktop) | Chrome (Android) | Safari (iOS) |
|---|---|---|---|---|---|
| `getUserMedia` (mic) | ✅ | ✅ | ✅ | ✅ | ✅ (requires HTTPS or `localhost`) |
| `AudioContext` / `ScriptProcessorNode` | ✅ | ✅ | ✅ | ✅ | ✅ |
| `speechSynthesis` (TTS) | ✅, wide voice selection | ✅, fewer installed voices by default | ✅, uses macOS/iOS system voices | ✅, uses Android TTS engine | ✅, uses iOS system voices |
| Indic-language TTS voices | Depends on OS language packs installed | Same | Generally good coverage on macOS/iOS | Depends on device TTS engine/language packs | Generally good coverage |

Practical notes:
- **HTTPS is required** for microphone access in every browser except when testing on `localhost`. A GitHub Pages deployment (or any HTTPS static host) satisfies this automatically.
- **Voice availability is the biggest real-world compatibility gap** — the code itself runs everywhere the table above is ✅, but a user on Windows without an Indic language pack installed will get silence or a fallback English voice. The in-app "test voice" button exists specifically to catch this per-user, per-machine.
- Not tested against IE or any browser without ES2017+/Fetch/`AbortController` support — none are targeted.
- No native mobile app; on mobile it works as a regular web page (add-to-homescreen works but gives no extra capability).

---

## 8. Scalability

This is a **client-side-only** app: every user's browser talks directly to Sarvam's API. There is no server component in the current design, which has specific scaling implications:

- **No app-side bottleneck.** Because there's no backend, the app itself doesn't need to scale — each browser tab is an independent client. Serving the static HTML file (e.g. from GitHub Pages or any static host/CDN) scales trivially to any number of users.
- **The real scaling limit is the Sarvam API tier** the deployer's key is on — request-per-minute and concurrency limits are enforced by Sarvam, not by this app. Check current limits on the [Sarvam dashboard](https://dashboard.sarvam.ai) before deploying to many concurrent users.
- **Shared-key risk.** As shipped, each user pastes in their *own* key via the settings panel — this is actually what keeps the app scalable without a backend, since load is naturally spread across whoever owns each key. If you instead hard-code a single shared key for all users, you push all traffic (and cost) through one account's rate limit and it becomes the bottleneck.
- **No queuing or retry/backoff logic.** STT and LLM calls are fire-and-forget with a single retry only for TTS `speak()` failures. Under rate-limiting or transient errors, requests fail with a visible "error" status rather than retrying — fine for a personal/small-scale tool, not resilient enough for high-concurrency production use without adding a retry/backoff layer.
- **If you outgrow the direct-to-Sarvam model:** the natural next step is a thin backend proxy (also solves the CORS caveat below) that can add request queuing, connection pooling, per-user rate limiting, and centralized key management.

---

## 9. Cost considerations

All costs come from Sarvam API usage — there is no hosting cost beyond serving one static HTML file (free on GitHub Pages or nearly any static host).

- **Two billable calls per conversational turn:** one STT call (`/speech-to-text`) and one LLM call (`/v1/chat/completions`), per user utterance.
- **STT cost** scales with audio duration sent — the app already discards non-speech noise and short blips (see `MIN_UTTERANCE_MS` / WAV min-size checks in §5), which keeps STT usage roughly proportional to actual speech time.
- **LLM cost** scales with tokens in + out. Levers already in the app:
  - `max_tokens: 1024` caps a single reply's output size.
  - The full rolling `history` array is resent on every request (standard for chat APIs, but means cost per turn grows as a conversation gets longer — there's no truncation/summarization of older turns).
  - `sarvam-105b-conversations` (default) is positioned as the low-latency/cheaper conversational model; `sarvam-105b` (128K context) is the heavier option — model choice directly affects cost.
  - `fast` mode sets `reasoning_effort: null` (skips extended reasoning) which is both faster and typically cheaper than `reasoning_effort: 'low'`.
- **TTS is free** — it's the browser/OS's built-in voice engine, not a paid API call.
- **No cost caps or usage tracking in-app.** There's no token/spend counter or per-session limit; cost control is entirely whatever limits are set on the Sarvam account/key itself. For a public deployment, pair this with server-side rate limiting or per-user quotas if cost exposure is a concern.
- Current Sarvam pricing isn't reproduced here since it can change — check the [Sarvam dashboard](https://dashboard.sarvam.ai) for up-to-date rates before estimating cost at scale.

---

## 10. Security & privacy

- **API key handling:** stored only in a JS variable in the current browser tab (`apiKeyInput.value`); never written to disk, `localStorage`, or `sessionStorage`, and never sent anywhere except Sarvam's API. It is lost on page reload — there is intentionally no "remember my key" feature.
- **No server, so no server-side data collection** by this app itself. Audio and transcripts are sent only to Sarvam's API for processing; refer to Sarvam's own privacy/data-retention policy for how they handle that data.
- **Client-side key exposure.** Because the key lives in browser JS and is sent directly from the browser, anyone with access to that browser tab/devtools can read it. This is an acceptable trade-off for a personal tool but is **not suitable for a multi-user public deployment** with a shared key — use a backend proxy to keep the key server-side in that case (see §11 below).
- **No conversation persistence** — the `history` array is in-memory only and clears on refresh; nothing is logged locally.

---

## 11. Deployment / hosting

- **Static hosting is sufficient** — GitHub Pages, Netlify, Vercel, S3+CloudFront, or literally any static file host will work, since the app is a single self-contained HTML file with no server-side code.
- **HTTPS is mandatory** for microphone access outside of `localhost` (see §7). All the hosts above serve HTTPS by default.
- **Recommended for anything beyond personal/single-user use:** add a lightweight backend that:
  1. Proxies requests to `api.sarvam.ai`, resolving the CORS caveat noted in §6/§8.
  2. Holds the Sarvam API key server-side instead of asking each visitor to paste one in.
  3. Adds per-user rate limiting / cost caps (see §9).

---

## 12. Requirements & known limitations

- **Browser support:** needs `getUserMedia`, `AudioContext`/`webkitAudioContext`, and `speechSynthesis`. The app checks for these on load and shows an error message if missing. Recommended: recent Chrome, Edge, or Safari 16+.
- **CORS:** calls go directly from the browser to `api.sarvam.ai`. If a request is blocked by CORS in a given deployment, it needs to be proxied through a backend — the UI hint already surfaces this.
- **`ScriptProcessorNode` is deprecated** in the Web Audio spec (superseded by `AudioWorklet`) but is still broadly supported; used here for simplicity of synchronous PCM access.
- **TTS voice availability** varies a lot by OS — e.g. Windows may report 0 voices for some languages until a language pack is installed (`Settings > Time & language > Speech`). The "test voice" button and `ttsDebug` line surface this.
- **No conversation persistence** — `history` is an in-memory array; refreshing the page clears it.
- **VAD is amplitude-based only** — it doesn't distinguish speech from other loud noises, so it may misfire in noisy environments (tunable via `VAD_THRESHOLD`).
- **No retry/backoff or request queuing** — see §8 for scaling implications.
- **No usage/cost tracking in-app** — see §9.

---

## 13. File structure

Single file: `sarvam-voice-bot.html`
- Inline `<style>` — all CSS, theme driven by CSS custom properties under `:root`
- Inline `<script>` — one IIFE containing all state, UI wiring, audio capture/VAD, WAV encoding, STT/LLM/TTS calls
- No external JS dependencies; Google Fonts (`Inter`, `IBM Plex Mono`) loaded via `<link>`

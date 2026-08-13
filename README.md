# Voice Bot

A single-file, browser-based voice assistant. Talk to it in English, Hindi, Tamil, or any mix of Indian languages — it transcribes your speech with Sarvam's Saaras STT model, gets a streamed reply from a Sarvam LLM, and speaks it back using your browser's built-in text-to-speech.

No backend, no build step, no dependencies to install — it's one `.html` file you can open directly or host anywhere static files are served.

## Features

- 🎙️ **Hands-free turn-taking** — tap the mic once, then just talk; voice-activity detection (VAD) figures out when your turn starts and ends
- 🌐 **Auto language detection** — speak English, Hindi, Tamil, Telugu, Kannada, Malayalam, Marathi, Bengali, Gujarati, or Punjabi (even mixed mid-sentence); the reply's voice follows automatically
- ⚡ **Fast mode** — streams the reply and starts speaking each sentence as soon as it's ready, instead of waiting for the full response
- 🛑 **Barge-in** — start talking while the bot is mid-reply and it stops and listens immediately
- 📊 **Live waveform** — oscilloscope-style visual feedback for mic input and playback
- 🔊 **Voice test button** — check that your browser/OS has a TTS voice installed for your language before relying on it

## Demo / Screenshot

_Add a screenshot or GIF of the app here._

## Requirements

- A modern browser with:
  - `getUserMedia` (microphone access)
  - Web Audio API (`AudioContext`)
  - `speechSynthesis` (Web Speech API)
  - Recommended: recent **Chrome**, **Edge**, or **Safari 16+**
- A [Sarvam AI](https://dashboard.sarvam.ai) API key (used for both speech-to-text and the chat model)
- A working microphone

## Getting started

1. Clone the repo:
   ```bash
   git clone https://github.com/<your-username>/<your-repo>.git
   cd <your-repo>
   ```
2. Open `sarvam-voice-bot.html` directly in a browser, **or** serve it locally:
   ```bash
   python3 -m http.server 8000
   # then visit http://localhost:8000/sarvam-voice-bot.html
   ```
3. Click the ⚙ settings icon and paste in your Sarvam API key ([get one here](https://dashboard.sarvam.ai)).
4. Click the mic button and start talking.

> **Note on CORS:** by default the app calls `api.sarvam.ai` directly from the browser. If your deployment hits a CORS error, put a small backend/reverse proxy in front of the Sarvam API and point the fetch calls at it.

## Configuration options (in the Settings panel)

| Setting | What it does |
|---|---|
| API key | Sarvam key, used for both STT and the chat model. Kept in memory for the tab only — never saved to disk. |
| Auto-detect language | On: detects spoken language per utterance. Off: locks speech recognition to the language you pick. |
| Speech language | Fallback STT language, and always the language used to pick the reply's TTS voice. |
| Model | `sarvam-105b-conversations` (low-latency, default) or `sarvam-105b` (128K context, heavier reasoning). |
| Fast mode | Streams the reply and speaks sentence-by-sentence for lower perceived latency. |

## How it works (short version)

```
mic → voice-activity detection → WAV encode (16kHz PCM) → Sarvam STT (Saaras v3)
    → Sarvam chat completions (streamed) → sentence splitter → browser TTS
```

Full architecture and pipeline details are in [`docs/sarvam-voice-bot-docs.md`](docs/sarvam-voice-bot-docs.md).

## Privacy

Your API key and conversation history live only in browser memory for the current tab. Nothing is written to disk, localStorage, or any server other than Sarvam's API. Reloading the page clears everything.

## Known limitations

- TTS quality/voice availability depends on your OS — some systems need a language pack installed before certain voices are available.
- Voice-activity detection is amplitude-based, so it can misfire in noisy environments (tune `VAD_THRESHOLD` in the script if needed).
- Conversation history is not persisted between sessions.

## License

_Add your license here (e.g. MIT)._

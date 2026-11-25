# Gemini API Audio Decisions

Live decisions on any audio, in a single HTML page, powered by the Gemini API.

Define your questions (multiple choice, yes/no or score), then play a video, an audio file or your microphone. Every couple of seconds the page sends the latest few seconds of audio to Google's [Gemini Decisions API](https://aistudio.google.com/docs/decisions-api) (`gemini-decision-1`). The answers appear on live cards with a probability for every option.

It works with speech in any language the model understands, and with non-speech audio such as music or background noise. Questions and options can be written in any language too.

There is **no server, no install and no build step**: the page calls the Gemini API directly from the browser.

**▶ Try it live: <https://ayman3000.github.io/gemini-api-audio-decisions/>** (bring your own Gemini API key; it stays in your browser).

## Quick start

1. **Get a Gemini API key** at <https://aistudio.google.com/apikey>. The free tier is enough for testing.
2. **Open the [live page](https://ayman3000.github.io/gemini-api-audio-decisions/)**, or download `index.html` and double-click it, or serve the folder with any static web server (see [Hosting](#hosting)).
3. **Paste the key** in the **API key** field in the header. The badge turns green ("Ready · Gemini") once a tiny test request succeeds.
4. **Step 1, Decisions:** start from an example or write your own questions.
5. **Step 2, Live test:** load a video or audio file, or click **Microphone**, then press play.

## How it works

```
 audio file / video / microphone
          │  decoded in the browser, resampled to 16 kHz mono
          ▼
 sliding window: every N s, take the last M s   (default: every 2 s, last 8 s)
          │  encoded as WAV, base64
          ▼
 POST https://generativelanguage.googleapis.com/v1beta/decisions
   { model: "gemini-decision-1",
     input: [{ type: "audio", data: <base64>, mime_type: "audio/wav" }],
     questions: { q1: {...}, q2: {...} } }       ← all questions in ONE request
          │
          ▼
 probabilities per option → cards, confidence, timeline
```

- **One request per update.** All questions go in the same request, so the audio is sent and billed once.
- **No backlog.** Only one request runs at a time, and the next one always takes the newest audio, so decisions never fall behind in a queue.
- **First decision after 3 s of audio,** because a 1 s clip is mostly a guess.
- **Stopping the microphone** gives a final decision at once. By default it hears the whole recording (up to 30 s); untick the option to judge only the last window. Final decisions are marked "Final" and outlined in the timeline.
- Updates are driven by the audio itself, not the page's animation loop, so they keep running in a background tab.

## Writing decisions

Up to 8 questions. Each one is one of:

| Type | You write | You get back |
|---|---|---|
| **Multiple choice** | 2–8 options, one per line. Add a hint for the model after `::`, e.g. `Complaint :: the caller is unhappy with the service` | The top option and a probability for every option |
| **Yes / No** | Just the question | Probability of yes |
| **Score** | 2–8 ordered levels, lowest first | The most likely level and a probability per level |

Tips (from Google's guidance):
- Ask positive questions ("Is the caller satisfied?"), not negated ones.
- A choice question always spreads 100% over its options, even when none fits. Add a catch-all option with a hint (e.g. `Other :: none of the above`), or watch the confidence.
- Probabilities are calibrated, so 0.66 really means "unsure". Pick your thresholds on your own recordings.

Questions are saved in the browser. **Export JSON** / **Import JSON** moves them between machines.

Built-in examples:
- **Any audio (English):** language spoken, topic, music, speaker tone. Works on any recording.
- **Arabic examples:** talk-show commentary, customer service call, meeting and scheduling, sports commentary, news topic. These show writing questions and options in another language.

## Your API key

- The key is **not in this repository**. You type it into the page, and it's stored only in **your browser's localStorage** for that page.
- The browser sends it straight to `generativelanguage.googleapis.com` and nowhere else.
- Anyone using that browser profile can read it, and so can any script on the same web origin. If you host the page, don't share a browser where your key is saved, and consider [restricting the key](https://console.cloud.google.com/apis/credentials) to the Generative Language API and to your site's address.
- To remove it, clear the field. This saves an empty key.

## Hosting

Any static host works, since it's one HTML file.

```bash
python3 -m http.server 8000        # then open http://localhost:8000
```

The **microphone** needs a secure context: a file opened from disk, `localhost`, or an HTTPS site. Over plain `http://` on another machine the browser blocks it.

## Cost and limits

- **Price:** about $0.10 per 1M input tokens on the paid tier; output is always 0 tokens. One update with 8 s of audio and 2 questions was ~400 tokens, so about $0.00004.
- **Free tier:** 200 requests/minute and 50,000 tokens/minute. One update every 2 s is 30 requests/minute.
- **Speed:** ~1 s per update, whatever the number of questions.
- **No streaming:** each update is a normal request on a short audio window.

## Measured

A sanity check on 12 recordings from [FLEURS](https://huggingface.co/datasets/google/fleurs) `ar_eg` (Egyptian Arabic, CC-BY 4.0): the topic out of 8 shuffled options, plus 6 detail questions. For comparison, the same questions with `gemma4:e4b` running locally on Ollama:

| | `gemini-decision-1` | `gemma4:e4b` (local) |
|---|---|---|
| Topic | **12/12** | 11/12 |
| Detail questions | 6/6 | 6/6 |
| Hardest clip | Right, 66% sure | Wrong, 98% sure |
| Silence: "is anyone speaking?" | No (73%) | Yes (65%) |
| Time per update | ~1.1 s for all questions | ~0.8 s for the first question, ~0.15 s for each extra one |

This is a small test in one language, not a benchmark.

## Troubleshooting

| You see | Fix |
|---|---|
| Badge: *Add your Gemini API key* | Paste the key in the header. |
| Badge: *Key not working* (hover for details) | Check the key in AI Studio. Google's message is in the tooltip and the log. |
| `... failed:` with a quota or rate-limit message | You hit the per-minute limit: use a longer *Every* interval or fewer questions. |
| Microphone button does nothing | Open the page from disk, `localhost` or HTTPS, and allow microphone access. |
| A file won't load | The browser must be able to decode it. Most mp4, webm, mp3, wav and m4a files work. |

## Credits

The interface is inspired by the live demo of [AADI – Arabic Audio Decisions](https://github.com/Hazem-azim/AADI-Arabic-Audio-Decisions) by Hazem Abdelazim, which uses the same idea of scoring audio against predefined decisions. This page builds on the live test page I contributed to that project. It adds presets, custom question types, file and microphone sources, any-language use, and calibrated probabilities from Google's Gemini Decisions API.

This repository is independent and not affiliated with AADI or Google.

## License

[MIT](LICENSE) © 2026 Ayman H. Moustafa

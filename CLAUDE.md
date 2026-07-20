# shut-up-brain

## Project Overview

Green noise for sleep. Built out of spite after other apps kept shutting down.

A dependency-free PWA that synthesises noise in the browser via the Web Audio API.
Nothing is streamed or downloaded — the audio is generated locally, which is the
whole point: it cannot stop working because someone else's service shut down.

Live at https://tbarthen.github.io/shut-up-brain/ (GitHub Pages, served from `main`).

## Tech Stack

- Vanilla HTML/CSS/JS — no framework, no bundler, no `package.json`, no dependencies
- **Web Audio API** — noise buffer generation + `BiquadFilterNode` tone shaping
- **Wake Lock API** — keeps the device from suspending audio overnight
- **Service Worker** — offline caching
- PWA install metadata via `manifest.json`

## Architecture

`index.html` is the entire application — markup, styles, and the audio engine all
inline. There is no build step; the file that is edited is the file that ships.

Audio path: `AudioBufferSourceNode` (looping noise buffer) → optional
`BiquadFilterNode` (tone) → `GainNode` (volume) → destination. White, pink, and
brown each have a dedicated buffer generator; **green reuses the white-noise buffer
and is shaped entirely by a bandpass filter**.

Tone and noise are a matrix, not independent settings: `toneFilters[tone][noise]`
resolves to a specific filter config. Adding a tone or a noise colour means adding
every combination, or `startNoise()` falls through to the unfiltered branch.

## Coding Conventions

- **SRP** — Each function does exactly one thing. If you have to use "and" to describe it, split it.
- **DRY** — No duplicated logic. Extract repeated patterns to shared utilities. Check existing helpers before writing new ones.
- **Self-documenting code** — Use clear, descriptive names. No inline comments that explain *what* — only *why* when truly necessary.
- **Defensive programming** — Validate inputs at function boundaries. Handle null/undefined/edge cases explicitly. Prefer explicit errors over silent failures.
- **Small functions** — Keep functions short and focused. If a function is getting long, it probably has more than one responsibility.

## Key Files & Directories

| Path | Purpose |
|------|---------|
| `index.html` | The whole app — markup, CSS, and audio engine |
| `sw.js` | Service worker; cache-first offline support |
| `manifest.json` | PWA metadata; icon is an inline SVG data URI |

## Environment & Setup

No install step. For quick edits, open `index.html` directly in a browser.

For anything touching the service worker or install behaviour, serve over HTTP —
service workers do not register from `file://`:

```
python -m http.server 8000
```

## Deployment

Push to `main`; GitHub Pages publishes automatically.

**Bump `CACHE_NAME` in `sw.js` on every user-facing change.** The fetch handler is
cache-first, so if `sw.js` is byte-identical the browser detects no update and
already-installed clients keep serving the stale `index.html` forever. Changing the
version string is what triggers `install`/`activate` and evicts the old cache.

## Decisions & Notes

- **Defaults live in the markup, not in JS.** The `active` class on a button (and
  the slider's `value`) is the single source of truth; `readSelection()` derives the
  JS state from the DOM at startup. Previously these were two hardcoded copies that
  could silently disagree. To change a default, move the `active` class — do not add
  a second copy in the script.

- **`stopNoise()` does double duty** — it serves both "the user pressed Stop" and
  "tear down before restarting after a noise/tone change", because `startNoise()`
  calls it internally. This is why `clearTimer()` must *not* null `timerEnd`: the
  restart path reads `timerEnd` afterwards to resume the remaining countdown.
  `resetTimer()` is the destructive variant, called only from genuine stop points
  (the Stop branch and the "Off" timer button). Conflating the two silently drops
  the sleep timer whenever the noise or tone is changed mid-playback.

- **Timer expiry fades out rather than cutting off** — `setTargetAtTime` over ~5s,
  then `stopNoise()`. Abrupt silence wakes people up.

- **Tone pills are a fixed 60×34px.** The active pill swaps its name for an icon,
  so the box must not be allowed to resize or the whole row reflows on every tone
  change. 60px is the ceiling: `.controls` is capped at 340px, which leaves 335px
  on a 375px phone, and five pills plus four 8px gaps must fit inside it. Anything
  wider wraps Abyss onto a second line. The name of the selected tone moves into
  the section label (`TONE — DRIFT`) since the pill no longer shows it.

- **Icons set `stroke` explicitly, not `currentColor`.** Inside a `<button>`,
  `currentColor` resolves against the button's own colour rather than an inherited
  one, and gets a UA default if that rule is ever moved or removed — which renders
  the icons black on a near-black background. Silent and invisible when it fails.

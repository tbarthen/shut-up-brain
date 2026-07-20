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

## Noise Colour Accuracy

Measured by FFT on the generated buffers, power averaged into octave bands:

| Colour | Measured | Textbook | Notes |
|--------|----------|----------|-------|
| White  | +0.03 dB/oct | 0  | Exact |
| Pink   | −3.03 dB/oct | −3 | Exact (Kellett IIR filter bank) |
| Brown  | −5.34 dB/oct | −6 | −6 above ~150Hz; flattens below |
| Green  | pink + ~5dB mid lift | — | Correct (see below) |

Brown's low-end flattening is correct, not a defect: `lastOut = (lastOut +
0.02*white) / 1.02` is a leaky integrator, i.e. a one-pole lowpass with a corner
near 150Hz. A true integrator is an unbounded random walk that drifts and clips.

**Green is pink noise with a peaking boost at 500Hz**, baked into the buffer by
`createGreenNoiseBuffer`, so it is a generated colour like the other three.
Measured octave bands (relative to 500Hz): +3.3, +0.7, −0.3, 0, −6.2, −11.0,
−14.4, −17.6.

It was previously white noise through a **bandpass**, which is the opposite
operation — subtractive band-limiting rather than an additive lift on a
broadband base. That left it ~12dB *down* at 62.5Hz where the accepted shape is
~3dB *up*, a 15dB low-end deficit that made it sound thin and hissy compared to
green noise elsewhere. Do not reintroduce a bandpass for green: the low end is
the point.

`createGreenNoiseBuffer` renormalises RMS back to pink's level after the boost,
so switching colours does not jump in volume (measured delta: −0.06dB). Green's
`toneFilters` entries are lowpasses like every other colour; the tone filter must
not double as the colour definition.

Note also that the tone filter always applies — there is no unfiltered path — so
what is heard is generator × filter. Brown is the least altered, because its
energy sits where the tone lowpass barely reaches (already −23dB by 8kHz).

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

- **Saved settings are restored into the markup, not into the JS state.**
  `restoreSettings()` moves the `active` class and sets the slider value before
  the `readSelection()` calls run, so the rest of the app is unaware persistence
  exists and the markup stays the single source of truth.

  Three constraints, each load-bearing:
  - **Every `localStorage` call is wrapped.** It throws — not returns null — when
    storage is disabled or full. An unguarded read breaks the app for those users
    entirely.
  - **Restored values are matched against the buttons actually in the DOM**, not
    a hardcoded list. Renaming or removing a tone therefore discards stale
    storage automatically. This matters because `startNoise()` does
    `toneFilters[selectedTone][selectedNoise]` — an unrecognised tone makes the
    first index `undefined` and the second throw, killing playback on every load
    until storage is cleared by hand.
  - **The sleep timer is not saved.** Picking 15m for an afternoon nap and having
    it carry silently into the night is the exact failure the timer exists to
    prevent. It always returns to the 8h markup default.

  Volume saves on `change`, not `input` — `input` fires continuously while
  dragging and would write on every pixel of travel.

- **Tone pills are a fixed 60×34px.** The active pill swaps its name for an icon,
  so the box must not be allowed to resize or the whole row reflows on every tone
  change. 60px is the ceiling: `.controls` is capped at 340px, which leaves 335px
  on a 375px phone, and five pills plus four 8px gaps must fit inside it. Anything
  wider wraps Abyss onto a second line. The name of the selected tone moves into
  the section label (`TONE — DRIFT`) since the pill no longer shows it.

- **Depth of the selected tone is carried by the fill, not the border.** Tone
  pills mirror the noise buttons' active treatment (bright accent + glow), but
  since the active pill hides its label and shows an icon, the noise row's
  "text goes white" becomes "icon goes white" — the icon is `#ffffff` so it
  never dims as the colour deepens.

  Ramping the *border* darker was tried first and does not work. There is very
  little room between a deep blue and the `#1a1a2e` background, so the steps
  compress badly (luminance gaps of 28.8 / 16.7 / 7.6 / 3.0) and Abyss lands at
  1.17:1 against the background — the selected state becoming the least visible
  thing on screen. The fill has room to ramp because it is bounded by a border
  that stays bright, and white-on-fill contrast *improves* as it darkens
  (5.7:1 at Clear up to 16.2:1 at Abyss). Keep the border bright; deepen the
  fill.

  **Space the ramp by CIE L\*, not by luminance.** Luminance is linear light, so
  evenly spaced luminance values look increasingly cramped toward the dark end —
  which is what collapsed the earlier attempts at Deep and Abyss. The fills sit
  at L\* 43.5 / 35.6 / 28.0 / 20.2 / 12.5, giving steps of 7.9, 7.6, 7.8, 7.7 —
  a spread of 0.3. If a fill is ever retuned, convert to L\* first
  (`L* = 116·Y^⅓ − 16`) and keep the steps even there.

- **Icons set `stroke` explicitly, not `currentColor`.** Inside a `<button>`,
  `currentColor` resolves against the button's own colour rather than an inherited
  one, and gets a UA default if that rule is ever moved or removed — which renders
  the icons black on a near-black background. Silent and invisible when it fails.

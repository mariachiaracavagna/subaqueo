# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

subaqueo is a live-subtitle web app: it listens to a Spanish-speaking lecturer through the phone mic and shows the translation as bold, lyric-style lines (the current line is bright, earlier lines fade and scroll up). The primary target is **iPhone Safari** during hour-long classes. Sessions are saved on the device and can be exported as Markdown for note-taking.

Everything lives in a single file, `index.html` (inline CSS and JS, no build step or dependencies). It is deployed with GitHub Pages from the `main` branch root. HTTPS is required, because the microphone doesn't work on plain HTTP.

## Commands

- Run locally: `python3 -m http.server 8765` and open http://localhost:8765 (`localhost` counts as a secure context for the mic).
- Syntax-check the inline script: extract the `<script>` body to a file and run `node --check` on it.
- Deploy: push to `main`. GitHub Pages serves https://mariachiaracavagna.github.io/subaqueo/

There is no test suite. To check the logic headlessly, drive the page with puppeteer-core and the system Chrome, and replace `webkitSpeechRecognition` with a mock through `Object.defineProperty` (plain assignment doesn't override Chrome's built-in). The mock should behave like iOS: one growing interim result that never becomes final.

## Architecture (all inside `index.html`)

**Speech pipeline**: this is the fragile part. iOS Safari's `webkitSpeechRecognition` often never sets `isFinal`, can replay the whole transcript after speech stops, and silently hangs. For that reason the code doesn't rely on final results:
- On each result it rebuilds the current recognizer instance's full transcript as a list of words and tracks a `committed` word offset. The uncommitted tail is the live line.
- A tail becomes a committed line when there's a `PAUSE_MS` silence, a final result arrives, or it passes `MAX_WORDS`. In the last case it splits at punctuation where possible.
- It creates a fresh recognizer instance (never reuses one) on `onend` and on errors. Handlers ignore events from stale instances (`r === rec`).
- Planned restarts (after about 120 words, or from the `STALL_MS` watchdog) go through `recycle()`, which calls `stop()` so the recognizer still delivers pending results. `killRecognizer()` (`abort()`) is only for user stop and replay duplicates. Recognizers can take many seconds to return words, so an aggressive watchdog or `abort()` silently loses speech. That was the "detects almost nothing" bug.
- If continuous mode ends twice with sound but no words, `mode` switches to `'phrase'` (`continuous = false`, a new recognizer per utterance).
- Settings → Show diagnostics displays a live event log (`log()`), which is the main way to debug on a real phone.

**Translation**: `translate()` works through a chain of free, keyless, CORS-enabled endpoints, in order: Google translate-pa (Chrome's endpoint), Google clients5, Google translate.googleapis `dict-chrome-ex`, and MyMemory. It uses a 5s timeout and gives each failing provider an exponential cooldown. `client=gtx` is intentionally left out because it rate-limits quickly. Committed lines retry with backoff. The live line is re-translated at most about every 700ms, and a sequence number drops stale responses.

**Lyric view**: when a line is committed, the live element is promoted in place so the text doesn't jump. Classes: `.active` for the bright line, `.near`, `.pending` for a provisional translation, and `.failed`. Auto-follow scrolls the active line to about 58% of the viewport height. A user touch or wheel turns follow off and shows "Back to live". Scrolling back down to the live position turns it on again.

**Storage** (`localStorage`, prefix `subaqueo.`): `settings`, `index` (session metadata list), and `s.<id>` (the `{t, es, en}` lines for each session). Writes are debounced and flushed immediately on `pagehide` or when the page is hidden. `toMarkdown()` defines the export format: a full translation, the full original, then line by line with timestamps.

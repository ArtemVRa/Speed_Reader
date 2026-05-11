# Speed Reader — Development Notes

## What the app does

A web-based speed reading tool using **RSVP** (Rapid Serial Visual Presentation): displays a text one word at a time, allowing the eye to stay in one spot while consuming text at high speed. Designed for iPhone/iPad in Safari but works in any modern desktop browser. Can be saved to the iOS home screen as a PWA.

Core features:
- **ORP highlighting** — the "Optimal Recognition Point" letter is shown in accent color, helping the eye anchor on a fixed focal point
- **Horizontal and vertical** display modes
- **Adjustable WPM** (50–1200) and **font size** (16–120pt)
- **Sources**: `.txt`, `.md`, `.epub` files; clipboard paste; manual textarea paste fallback
- **Markdown**: parses headings (`#…######`) and strips inline formatting; heading words are rendered underlined
- **Sentence-end pause**: words ending in `. ! ? …` (optionally followed by quotes/brackets) are held for 2× the normal duration
- **Touch gestures**: tap on word = pause/play; swipe left/right = ±1 word
- **Keyboard** (Smart/Magic Keyboard on iPad, or desktop): Space, ←/→, ↑/↓, `[`/`]`, `V`

## Main files

The entire app is a **single HTML file**: `index.html`. No build step, no bundler, no framework. All CSS and JS are inline.

External dependency: **JSZip** loaded from CDN for EPUB support. The script tag is in `<head>`. If JSZip fails to load (offline), `.txt` and `.md` still work; only EPUB is unavailable.

## Architecture

### State (global vars in `<script>`)

```
words    : string[]              // tokenized words
wordMeta : {isHeading, level}[]  // parallel array, same length as words
index    : int                   // currently displayed word index
wpm, fontSize, vertical, running, paused, timer
```

### Display

- **Horizontal mode**: three `<span>`s (`#wLeft`, `#wOrp`, `#wRight`) inside `#hDisplay`. ORP span is bold + accent color.
- **Vertical mode**: dynamically generated columns of fixed-height `<span>`s inside `#vDisplay`. Each letter has `height = line-height = round(fontSize * 0.7) + 4` — see decision notes below.
- `setMode(vert)` swaps which display is visible.

### Playback (two-phase tick)

```
tick()    → showWord(); schedule advance() after delay
advance() → index++; tick()
```

This split is intentional: `index` is **always** the index of the currently displayed word. If we did `show → index++ → schedule(tick)` (single-phase), then on pause `index` would point to the *next* word, breaking `toggle_vertical` and any "refresh current display" operation. Do not collapse the two functions.

Delay calculation: `base = 60000 / wpm`; doubled if `isSentenceEnd(words[index])`.

### Loading

- `loadContent(ws, meta, source)` — canonical loader; resets state, sets `words` and `wordMeta`, displays first word.
- `loadWords(ws, source)` — convenience wrapper that calls `loadContent(ws, null, source)` (auto-fills `wordMeta` with empty metadata).
- `tokenize(text)` — splits on whitespace, drops empties.
- `parseMarkdown(text)` — returns `{ws, meta}`. Line-by-line: detects `#…######` headings, strips inline formatting via `stripMarkdown()`, marks heading words with `{isHeading: true, level}`.
- `loadEpub(buf, filename)` — uses JSZip + DOMParser to extract text from spine items in order.
- `askMarkdownAndLoad(text, source)` — prompts the user "Это Markdown?" before loading raw text (used by `.txt` files and clipboard paste; `.md` skips this and parses directly).

## Important decisions

### 1. Two-phase tick

Pause is implemented by setting `paused = true` and cancelling `timer`. Because `index` always equals the displayed word, refreshes during pause (toggling vertical, changing font size) just call `showWord()` and get the right word. Earlier single-phase implementation had `index` pointing one ahead after a tick, causing "next word flashes when toggling on pause" bugs.

### 2. Vertical line-height formula

```
lineH = round(fontSize * 0.7) + 4
```

This was the result of multiple iterations. Naive approaches that failed:
- Using `f.metrics().linespace` — too spaced; on macOS Georgia the linespace already equals the bounding box height with no extra leading.
- Using `ascent + descent` — same as linespace on this font, no change.
- Measuring canvas `bbox()` of a probe character — also returns full bounding box ≈ linespace.

The fix uses **font geometry directly**: Georgia's visible cap height is approximately `0.7 × emSize`, and `font-size` in CSS/Canvas is the em size. So `0.7 × fontSize + small_gap` gives consistent "almost touching" packing regardless of DPI scaling or platform-specific font metrics. The `+4` is a small visual gap; reducing further causes descenders ('р', 'у') to touch the next character.

This same formula is used in the Python desktop version's canvas drawing — keep them in sync if you change one.

### 3. Markdown handling: ask, don't auto-detect

`.md` files are always parsed as Markdown. For `.txt` and clipboard paste, the user is shown a modal with a 120-char preview and two buttons. Auto-detection (looking for `#`, `**`, etc.) was rejected because `#` appears in plain text (hashtags, IDs) and would mis-classify.

### 4. ORP table

```js
n <= 1  → 0
n <= 5  → 1
n <= 9  → 2
n <= 13 → 3
else    → 4
```

These are standard RSVP/Spritz-style ORP positions. The same table is in the Python desktop app.

### 5. Single-file deployment

Everything inline. The user can drag `index.html` to Netlify Drop, push it to a GitHub Pages repo, or open it from any static host. No build pipeline needed.

## Known issues / limitations

- **Clipboard API on iOS**: `navigator.clipboard.readText()` requires HTTPS or `localhost` and may require a permission prompt. The fallback is the manual paste textarea modal — this triggers automatically on failure.
- **EPUB needs network the first time** to fetch JSZip from cdnjs. Browser cache keeps it offline-available after that, but a fresh PWA install with no network will fail on EPUB.
- **PDF**: not implemented despite being mentioned in early plans. Removed from file picker `accept` attribute.
- **Vertical mode column count** uses `card.clientHeight - 60` as the estimate of available height (60 = approx space for the bottom `.pos-lbl` separator). If the layout changes significantly, recompute this constant.
- **No service worker**, so true offline behaviour for the first visit depends on browser cache. Saving to home screen on iOS gives PWA-like behavior but not full offline-first.
- **No persistence**: WPM, font size, vertical mode, and last position reset on reload.

## How to run / edit

### Run locally

Just open `index.html` in a browser. No server needed for `.txt`, `.md`, paste, or vertical mode.

For testing on a phone over Wi-Fi from the same Mac:

```bash
cd <project>
python3 -m http.server 8080
# Find Mac IP: ipconfig getifaddr en0
# Open http://<MAC_IP>:8080 on phone
```

### Deploy

- **GitHub Pages**: push `index.html` to a repo, Settings → Pages → Branch: main / (root) → Save.
- **Netlify Drop**: drag the file to `netlify.com/drop`.

### Edit

Open `index.html` in any text editor. The structure is:

1. `<head>` — meta tags (PWA-friendly), JSZip CDN script, all CSS
2. `<body>` — markup for header, card, progress, controls, file input
3. `<script>` — state, ORP, display functions, playback, settings, loaders, gestures, keyboard

Sections in the JS are separated by `═══` comment dividers and labelled in Russian.

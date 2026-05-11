# Guidance for Claude

This is a **single-file vanilla HTML/CSS/JS** speed-reader for mobile Safari. No frameworks, no build step. The whole app is `index.html`.

User is a programmer with Pascal/C++ background, learning Python/JS conventions through this project. He prefers concise explanations and terse code. UI strings are in Russian; comments and identifiers in code are mostly Russian too (мatching that). Keep it that way unless asked otherwise.

## Things that look weird but are intentional — do not "fix"

### Two-phase tick (`tick` + `advance`)

```js
function tick()    { ... showWord(); timer = setTimeout(advance, delay); }
function advance() { ... index++; tick(); }
```

Do **not** collapse this into `show → index++ → schedule(tick)`. The split exists so that `index` always points to the currently displayed word, which makes pause + refresh (toggle vertical, change font) work correctly. We had a bug where toggling vertical on pause showed the *next* word; this split was the fix.

### Vertical line-height formula

```js
const lineH = Math.round(fontSize * 0.7) + 4;
```

Looks arbitrary. It isn't. Font metrics (linespace, ascent+descent, canvas bbox) all return ≈ full bounding box on macOS Georgia, giving identical (too loose) spacing. Using `0.7 × fontSize` exploits the fact that visible cap height ≈ 0.7 em, giving "almost touching" letters regardless of platform/DPI. See `DEVELOPMENT_NOTES.md` for the full reasoning.

If the user complains about vertical spacing, the lever is the `0.7` multiplier and the `+4` constant — not the font metrics.

### Parallel `wordMeta` array

`words[]` is `string[]` and `wordMeta[]` is `{isHeading, level}[]`, same length. Don't merge them into `{text, meta}[]` — keeps `words[index]` as a plain string throughout the codebase, simpler call sites.

### `askMarkdownAndLoad` modal

The user explicitly chose "ask, don't auto-detect" for Markdown. Don't add auto-detection without asking.

### JSZip from CDN

Loaded via `<script src="cdnjs…">` in `<head>`. Inlining it would bloat the file by ~90KB for a feature most loads won't use. EPUB gracefully degrades (`alert`) if JSZip didn't load.

## Conventions

- **Russian UI text**, English-only inside `<head>` meta and dev docs.
- **Color tokens** in `:root` CSS custom properties: `--bg --card --accent --btn --btn-hv --fg --dim --side`. Use these, don't hardcode hex.
- **Georgia serif** for the word display. **System sans-serif** (`-apple-system, Helvetica, Arial`) for UI chrome.
- **Buttons**: class `.btn` (with `.accent`, `.on`, `.btn-sm` modifiers). Both `onclick` and `ontouchend` are bound on `+/−` rapid-tap buttons to defeat iOS 300ms click delay.
- **No emojis** in code or docs unless the user asks.
- **No comments** that just restate what the code does. Russian-language `// объяснение` comments are fine for non-obvious *why*.

## How to run / test

```bash
cd <project>
python3 -m http.server 8080
# then on phone: http://<mac-ip>:8080
```

Or just open `index.html` directly.

## Related project

There is a Python/tkinter desktop version of the same app (separate codebase). The ORP table and the `0.7 × fontSize + 4` vertical formula were ported from there. If you change algorithm-level behavior in one, mention the other to the user.

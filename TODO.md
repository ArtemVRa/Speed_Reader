# Speed Reader — TODO

## Likely next tasks

### Persistence
- [ ] Save WPM, font size, vertical-mode toggle to `localStorage` and restore on load
- [ ] Save last-read position per file/source (key on filename + length, or a hash) so the user can resume

### Better pause logic
- [ ] Shorter extra pause on `,` `;` `:` (e.g. 1.3× instead of 2×) — distinguish soft and hard sentence breaks
- [ ] Longer extra pause on paragraph breaks (detect `\n\n` during tokenization and inject a pause marker)
- [ ] Slightly longer dwell on long words (more letters = more to absorb)

### Markdown
- [ ] Render the *previous* line above the current word when the current word belongs to a heading, so the user has more context
- [ ] Optional: also visually mark **bold** words (e.g. brighter color) instead of stripping
- [ ] Currently `stripMarkdown` may eat asterisks inside non-Markdown text loaded as Markdown — review false positives

### PWA / offline
- [ ] Add a `manifest.json` for proper "Add to Home Screen" experience (icon, splash, theme color)
- [ ] Add a service worker so the app works fully offline after first load (incl. cached JSZip)

### File handling
- [ ] PDF support (was promised in early versions, not implemented). Use `pdf.js` from CDN
- [ ] Drag-and-drop a file onto the window on desktop
- [ ] Remember the last opened file in some way (within session at minimum)

## Smaller polish

- [ ] iPad landscape: control rows wrap awkwardly at narrow widths — review breakpoints
- [ ] On very small phones (iPhone SE), the controls take >50% of the screen; consider collapsing rows or moving Skip into a popup
- [ ] Long filenames in the header get ellipsised — fine, but maybe show full name on tap
- [ ] When loading a large `.epub`, no progress indicator — consider a simple spinner
- [ ] Modal close on tap-outside or Esc

## Refactors worth considering (not urgent)

- [ ] Extract repeated `document.getElementById('xxx')` calls into cached refs at script top
- [ ] Vertical mode currently rebuilds the entire column DOM every word — fine for performance but could be a Canvas like the desktop version if it ever gets slow
- [ ] Tokenization currently splits only on whitespace; long unbroken strings (URLs, code paths) become one giant "word". Consider a max-length split.

## Out of scope (decided no)

- ~~Auto-detect Markdown in pasted text~~ — user wants explicit prompt
- ~~Bundle JSZip inline~~ — adds 90KB for a rarely-used feature; CDN cache is fine
- ~~Switch to a JS framework~~ — single-file vanilla is a feature, not a limitation

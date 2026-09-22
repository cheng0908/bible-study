# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A static website for 頌主堂 (a church): a homepage, a growing collection of members' daily Bible-reading
notes and Sunday sermon transcripts, a PDF-export tool for those notes, and standalone sermon slide decks.
Deployed as-is via GitHub Pages (see `CNAME` — custom domain `sz-home.my`). There is no build step, no
package manager, and no test suite — every page is a single hand-authored HTML file using the Tailwind CDN
script and vanilla JS.

## Commands

There is no build/lint/test tooling. The only thing you need locally is a static file server, because
`blog/reading-notes.html` and `blog/print.html` load data via `fetch()`, which browsers block under the
`file://` protocol:

```bash
python3 -m http.server 4173
```

(This matches the `static` configuration already in `.claude/launch.json`.) Then open
`http://localhost:4173/blog/reading-notes.html` etc. Opening these two files directly by double-click will
show an empty/broken page.

Pages with no `fetch()` calls (`index.html`, `blog/2026-06-reading-notes.html`, `slides/*/index.html`) can
be opened directly from disk.

## Architecture

### Reading notes hub (`blog/reading-notes.html` + `blog/reading-notes.jsonl`)

This is the main, continuously-growing artifact. It was deliberately split into a thin HTML template/renderer
plus a JSONL data file so that adding a new month's notes is a data-only append, not an HTML edit:

- `reading-notes.jsonl` — one JSON object per line, two shapes:
  - reading note: `{type:"note", book, date, color, ref, text}`
  - sermon: `{type:"sermon", book:"主日講道", date, color, title, top:[...], sections:[{heading, verse, paragraphs:[...]}], prayer:[...]}`
  - `date` is `"YYYY-MM"` (month granularity only — the source data has no day-level dates).
  - `color` is a key into the shared "morandi" Tailwind palette (see below), stored per-record rather than
    derived from a book→color config file, so a record is self-contained.
- `reading-notes.html` — on `DOMContentLoaded`, fetches the JSONL, splits/`JSON.parse`s each line, and
  renders: `<option>`s for the book `<select>`, book "quick nav" pill buttons, month pill buttons, and one
  `<article>` per record (`noteCardHTML()` / `sermonCardHTML()`). Only *after* rendering does it run
  `initPage()` (the original interaction logic: font-size stepper, per-card collapse/expand, copy/share/
  speak-aloud buttons, filtering). Book and month filters combine (AND); picking a month greys out and
  disables any book pill/option with zero matching records for that month, and auto-resets the book filter
  to "all" if the current selection would end up empty.
- Book/date lists and their pill order come from first-appearance order while iterating the JSONL, not a
  hardcoded list — new books/months just need new records.

### PDF export (`blog/print.html`)

Reads the same `reading-notes.jsonl` (not by scraping `reading-notes.html`'s DOM — that page renders
client-side so there's nothing to scrape). Lets the user pick a book range and a month range (both optional,
overridable via `?book=`/`?date=`/`?src=` query params), multi-select which entries to include, adjust the
print font size, and optionally add a cover page, then calls `window.print()`. Sermon records are flattened
back into plain paragraphs (`flattenSermon()`) since the print layout is plain-paragraph only. Entries are
sorted by book (in first-appearance order) then by the leading chapter number parsed out of `ref`/`title`
(`chapterOf()`), not by insertion order, so a book's entries read in chapter order even when they came from
different months.

### Superseded/frozen pages

`blog/2026-06-reading-notes.html` is an older, fully static (non-JSONL) single-month page that predates the
JSONL split. **Do not delete or restructure it** — its URL may already be shared. New months go into
`reading-notes.jsonl` instead.

### Raw source data (`blog/2026-08/`, `blog/2026-10/`, ...)

Per-month folders of OCR'd CSV exports of handwritten/typed notes (`經文,經文出處` columns: content, then
book+chapter reference). These folders are gitignored (raw personal drafts, not published content) — see
`.gitignore`. There is no committed conversion script; turning a new month's CSVs into JSONL records has so
far been done as an ad-hoc one-off pass each time, with two recurring gotchas worth checking for again:
- OCR'd CSV cells sometimes carry a stray literal `"` character at the very start/end of the field (an
  artifact of the CSV quoting, not part of the content) that must be stripped.
- A month's sermon CSV (when present) has a different shape than reading notes: a title line, then
  `經文：`/`核心經文：` header lines, then a body organized as `引言`, numbered sections (`一、`, `二、`, ...)
  each optionally preceded by its own `經文：` line, then `結論`, then `結束禱告` — this needs to be parsed
  into the `sermon` JSON shape above, not the flat `note` shape.

### Checking `reading-notes.jsonl` for duplicate/near-duplicate `text`

When re-importing a month (e.g. after the author fixes typos in a raw CSV) or just auditing the file, check
for duplicate `text` across `note` records both exactly and fuzzily — a chapter re-entered under a different
month, or OCR'd twice, won't always match byte-for-byte. Stdlib `difflib.SequenceMatcher` ratio works well
here; nothing above ~0.3 has turned up as a real duplicate so far, so ratio > 0.5 is a reasonable flag
threshold, with anything above ~0.8 worth treating as a near-certain duplicate:

```python
import json
from difflib import SequenceMatcher

records = []
with open('blog/reading-notes.jsonl', encoding='utf-8') as f:
    for line in f:
        line = line.strip()
        if not line:
            continue
        obj = json.loads(line)
        if obj.get('type') == 'note':
            records.append((obj['date'], obj['ref'], obj['text']))

for i in range(len(records)):
    for j in range(i + 1, len(records)):
        ratio = SequenceMatcher(None, records[i][2], records[j][2]).ratio()
        if ratio > 0.5:
            print(f'{ratio:.2f}  {records[i][:2]}  vs  {records[j][:2]}')
```

When keeping one of a duplicate pair, keep the earlier month/line (the first occurrence) unless told
otherwise.

### Shared design system

Every page independently declares the same Tailwind config (inline `<script>` in `<head>`) defining a
"morandi" color palette: `bg`, `card`, `text`, `muted`, `blue`, `sage`, `rose`, `ochre`, `slate`, `plum`
(the last reserved for 主日講道/sermon accents), plus Noto Serif TC / Noto Sans TC from Google Fonts. There's
no shared stylesheet or component file — keep new pages visually consistent by copying this block rather than
inventing new colors.

### Slide decks (`slides/YYYYMMDD/`)

Standalone, one folder per sermon date: `sermon.md` (source text) and `index.html` (the rendered deck, same
Tailwind/morandi setup as above). Self-contained; not linked into the JSONL/reading-notes system.

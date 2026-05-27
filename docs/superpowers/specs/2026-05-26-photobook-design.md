# Photobook — Design Spec

Date: 2026-05-26

## Goal
A single, dependency-free `index.html` web app to build print-ready photo books.
Everything runs client-side and offline. Print-to-PDF reproduces the page exactly.

## Features
1. Create photo book projects, choosing a page size. Multiple books (a Library).
2. Upload photos (JPEG/PNG).
3. Datestamp on photos (EXIF date), as a non-destructive overlay.
4. Per-photo captions.
5. Separators (forced page breaks) and Sections (titled, contiguous groups of photos
   that may only share pages with other photos in the same section).
6. Photos auto-arranged as efficiently as possible across pages within the constraints.
7. Live re-layout on any edit (upload more, change captions, etc.).
8. Dates come from EXIF and can prefix the caption.
9. Print-optimized output.

## Key insight for arrangement (5/6/7)
"Within a page order is free, but page-to-page is chronological" ⇒ every page is a
**contiguous slice of the time-ordered sequence**. So arrangement is not 2D bin-packing;
it is a 1D partition problem with an optimal dynamic-programming solution.

### Two-level engine
- **Level 1 — Pagination (DP, Knuth–Plass style).** For each segment of photos,
  `best[j] = min over i<j ( best[i] + cost(i,j) )`, where `cost(i,j)` is the badness of
  putting photos `i..j-1` on one page. Cap photos/page (≤8) ⇒ O(N·maxPerPage), globally
  optimal. Forced cuts at separators and section boundaries.
- **Level 2 — Per-page layout (justified rows).** Photos flow into rows; each row is
  justified to fill the content width (heights equal within a row, widths by aspect).
  `cost(i,j) = min over candidate layouts of badness`, so each page independently keeps
  the best layout — no global "which engine" choice. badness = whitespace fraction +
  λ·row-height variation + penalties for photos that are too small / too large / too
  uneven. Candidate layouts = all contiguous row partitions (≤8 photos ⇒ brute force),
  optionally aspect-sorted within the page (within-page order is free).

### Segmentation rules
Canonical order = units sorted by date (a section = one unit keyed by its earliest
photo date; a free photo = a singleton unit; a separator keeps a stored sort position).
This guarantees sections stay contiguous and the book is chronological at unit
granularity. A new page (DP segment break) is forced when: a separator is hit, or the
sectionId changes. The section title is shown only on the first page of its first segment.

## Captions & dates
- Per-photo placement with a book-level default.
- Caption: **below** (reserves layout space) or **in-photo** (overlay + scrim).
- Date: **below** (prefixes caption) or **in-photo** (corner datestamp).
- Constraint: caption in-photo ⇒ date in-photo (UI disables "date below" then).
- Item 3 (stamp on all photos) = book default date placement = in-photo.
- All overlays are non-destructive DOM, so they stay editable and re-layout cleanly.

## Data model
`Library` of `Book`s. A Book:
```
{ id, title, pageSize:{name,w,h}, margin, gap,
  defaults:{ capPlace:'below'|'in', datePlace:'below'|'in', stampStyle, ... },
  sections:{ [sectionId]:{ title } },
  timeline:[
    { type:'photo', id, blobId, w, h, date(ISO), dateOverride?, caption,
      capPlace?, datePlace?, sectionId? }
    { type:'sep', id, sortDate }
  ] }
```

## EXIF
Hand-parse JPEG APP1/Exif `DateTimeOriginal` (0x9003), fall back to `DateTime` (0x0132),
then file `lastModified`. Per-photo date override re-sorts the timeline.

## Storage
**IndexedDB** (built-in, no dependency, offline, holds Blobs): store `books` (metadata
JSON) and `blobs` (image Blobs, written once). localStorage is too small (~5 MB) for
photos. Autosave debounced on every edit. Object URLs cached per photo for display.

## Print
Dynamic `@page { size: W in H in; margin:0 }` matching page size; each book page is one
print page (`break-after:page; break-inside:avoid`); geometry in inches so output is
exact; `@media print` hides editor chrome and removes preview zoom;
`print-color-adjust:exact` so stamps/scrims print.

## Page sizes
Presets at creation (changeable later, triggers relayout): Square 8×8, Landscape 11×8.5,
Portrait 8.5×11, Large Square 12×12.

## UI
- **Library view:** grid of book cards (title, size, count, cover thumb); New / Open / Delete.
- **Editor view:** toolbar (back, title, page-size, Add photos, defaults, Print);
  sidebar timeline list (per-item: caption edit, placement toggles, date override,
  group-into-section, insert separator, delete; multi-select for sectioning);
  main area = live paginated book preview.

## Verification
- EXIF: crafted JPEG fixtures with known `DateTimeOriginal` parse to the right date.
- Layout: injected photo sets produce pages with no overlaps, full-width rows, whitespace
  within tolerance, sections contiguous & titled, separators forcing breaks, chronological
  page order.
- Print: `@page` size tracks the book; chrome hidden; one book page per sheet.
- Persistence: reload restores books, photos, captions, sections.

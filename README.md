# Photobook

A single-file, dependency-free web app for building print-ready photo books — entirely in your browser, fully offline. Open `index.html` (or the hosted page), make a book, and Print → Save as PDF.

**Live:** https://spacerat.github.io/photobook/

## Features
- Multiple photo-book projects, each with a chosen page size (square / landscape / portrait).
- Upload photos by button or by dragging image files onto the timeline. Identical re-uploads are de-duplicated.
- Dates read from EXIF; photos arrange **chronologically** across pages.
- Captions and dates, either below each photo or overlaid on it.
- Full-width **titles**, body **text blocks**, **line breaks** (new row) and **page breaks**, inserted between any rows.
- "Keep together" to stop a set of photos splitting across a page.
- Automatic, efficient page layout (see below). Live re-layout on every edit.
- Print-optimized output: each book page is one PDF page at its exact physical size.

## How the layout works
Because within a page order is free but page-to-page must stay chronological, every page is a
contiguous slice of the time-ordered photos. That turns arrangement into a 1-D problem solved
optimally with **Knuth–Plass-style dynamic-programming pagination** plus a **justified-rows**
per-page layout — no heavyweight optimizer needed. A "density" control tunes how tightly pages pack.

## Tech
- One `index.html`. No build step, no dependencies, no backend.
- Photos and project data stored locally in **IndexedDB**; the open book is reflected in the `?book=` URL.
- Images are downscaled/recompressed on import to keep printed PDFs small.

## Deploy
It's a static file — commit `index.html` and enable GitHub Pages (root of the default branch).

## Develop
Just edit `index.html` and open it via any static server, e.g. `python3 -m http.server`.

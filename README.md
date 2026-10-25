# Merge PDF — Browser Edition

> A complete PDF merger that fits in **one HTML file**. No server, no upload, no build step. Drag, drop, reorder, merge — your files never leave your device.

[![No Backend](https://img.shields.io/badge/backend-none-success)](.)
[![Single File](https://img.shields.io/badge/files-1-blue)](.)
[![Privacy](https://img.shields.io/badge/privacy-100%25-brightgreen)](.)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

---

## What It Is

A self-contained `merge_pdf.html` file that reimplements the core functionality of [Merge PDF tool]() — but with a fundamental difference: **the merge happens in your browser, not on a server**.

Open the file → drag in PDFs → reorder → click merge → download. That's the entire workflow. No sign-ups, no uploads, no waiting for a server, no privacy trade-offs.

```
┌─────────────────────────────────────────────────────────┐
│                    BROWSER (Your Device)                │
│                                                         │
│  ┌──────────────┐   ┌──────────────┐   ┌────────────┐   │
│  │  File Input  │──▶│  In-Memory   │──▶│  pdf-lib   │   │
│  │  / Drag-Drop │   │  File Array  │   │  (merge)   │   │
│  └──────────────┘   └──────────────┘   └────────────┘   │
│         ▲                  ▲                  │         │
│         │                  │                  ▼         │
│  ┌──────────────┐   ┌──────────────┐   ┌────────────┐   │
│  │  SortableJS  │──▶│  DOM <ul>    │   │  Blob URL  │   │
│  │  (reorder)   │   │  (visual)    │   │ (download) │   │
│  └──────────────┘   └──────────────┘   └────────────┘   │
│                                                         │
│              ⚡ Nothing leaves this box ⚡               │
└─────────────────────────────────────────────────────────┘
```

---

## Try It

**Option 1 — Just open it**

1. Download [`merge_pdf.html`](merge_pdf.html)
2. Double-click it (opens in your browser)
3. Drop some PDFs, click merge

**Option 2 — Host it**

Drop the single file on GitHub Pages, Netlify, Vercel, or any static host. It requires no server-side logic — just HTML.

**Option 3 — Serve locally**

```bash
python3 -m http.server 8000
# then open http://localhost:8000/merge_pdf.html
```

That's it. There is no step 4.

---

## Features

- 🔒 **100% private** — files are processed in-memory, never uploaded
- 📦 **Single-file deployment** — literally one `.html` file, zero dependencies to install
- 🎯 **Drag-and-drop reordering** via [SortableJS](https://sortablejs.github.io/Sortable/)
- ✨ **Instant feedback** — no waiting on server round-trips
- 🧹 **Memory-safe** — Blob URLs revoked after download
- 📱 **Responsive** — works on mobile, tablet, desktop
- 🌐 **Works offline** — once loaded, no network needed
- 🚫 **No tracking, no analytics, no cookies**

---

## How It Works

Two CDN-loaded libraries do all the heavy lifting:

| Library | Size | Purpose |
| :--- | :---: | :--- |
| [`pdf-lib`](https://pdf-lib.js.org) | ~1.2 MB | Parse, merge, and re-serialize PDFs in pure JS |
| [`SortableJS`](https://sortablejs.github.io/Sortable/) | ~45 KB | Drag-and-drop reordering of the file list |

The pipeline, in four phases:

### Phase 1 — Collect

The File API hands over `File` objects (references to files on disk — not the bytes). Nothing is read yet. Each file gets a temporary `id` for DOM tracking.

```javascript
if (file.type !== 'application/pdf') {
  setStatus(`Skipped non-PDF file: ${file.name}`, 'error');
  continue;
}
file.id = 'file-' + Date.now() + '-' + Math.random().toString(36).substr(2, 9);
selectedFiles.push(file);
```

### Phase 2 — Arrange

`SortableJS` lets the user drag items in the `<ul>`. The DOM order **is** the merge order — no separate state to sync. Each `<li>` carries `data-file-id`, so when we merge we walk the DOM (not the array) to get the sequence.

```javascript
new Sortable(fileListEl, {
  animation: 150,
  ghostClass: 'sortable-ghost',
});
```

### Phase 3 — Merge

This is where `pdf-lib` earns its megabytes. For each file in the current DOM order:

1. **`file.arrayBuffer()`** — reads the raw bytes (this is when disk I/O happens)
2. **`PDFDocument.load(buffer)`** — parses the PDF's object graph
3. **`mergedPdf.copyPages(pdf, pdf.getPageIndices())`** — deep-copies each page's content streams, fonts, images
4. **`mergedPdf.addPage(page)`** — appends to the merged document

```javascript
const mergedPdf = await PDFLib.PDFDocument.create();
for (let li of listItems) {
  const file = selectedFiles.find(f => f.id === li.dataset.fileId);
  const arrayBuffer = await file.arrayBuffer();
  const pdf = await PDFLib.PDFDocument.load(arrayBuffer);
  const pages = await mergedPdf.copyPages(pdf, pdf.getPageIndices());
  pages.forEach(p => mergedPdf.addPage(p));
}
const mergedBytes = await mergedPdf.save();
```

### Phase 4 — Download

The serialized bytes become a `Blob`, which gets a temporary in-memory URL. A hidden `<a download>` element is clicked programmatically. Then the URL is revoked — critical for freeing memory.

```javascript
const blob = new Blob([mergedBytes], { type: 'application/pdf' });
const url = URL.createObjectURL(blob);
const a = document.createElement('a');
a.href = url;
a.download = 'merged_document.pdf';
a.click();
URL.revokeObjectURL(url);   // ← without this, every merge leaks memory
```

That's the entire app — ~250 lines of JavaScript, ~150 lines of CSS, no backend.

---

## Browser Support

| Browser | Supported | Notes |
| :--- | :---: | :--- |
| Chrome / Edge (desktop) | ✅ | Full support, best performance |
| Firefox (desktop) | ✅ | Full support |
| Safari (macOS) | ✅ | Full support |
| Chrome (Android) | ✅ | Works; see size limits below |
| Safari (iOS) | ✅ | Works; **aggressive memory limits** apply |

Requires a browser with support for `File API`, `Blob`, `URL.createObjectURL`, `async/await`, and `File.arrayBuffer()` — i.e. any browser from 2019 onward.

---

## ⚠️ File Size Limits

This is the **fundamental trade-off** of a client-side approach. Because `pdf-lib` runs in JavaScript, it's constrained by the browser's per-tab heap.

| Device | Safe Total Input | Risky Zone | Likely Crash |
| :--- | :---: | :---: | :---: |
| Desktop (16 GB RAM) | ≤ 200 MB | 200–500 MB | > 500 MB |
| Desktop (8 GB RAM) | ≤ 100 MB | 100–300 MB | > 300 MB |
| Mobile (Android/iOS) | ≤ 30 MB | 30–80 MB | > 80 MB |
| iOS Safari | ≤ 50 MB | 50–150 MB | > 150 MB (tab reloads) |

**Why so low?** `pdf-lib` needs roughly **5–8× the total input size** in peak RAM. A 100 MB merge can momentarily consume 500–800 MB — enough to kill a mobile tab.

**The math:**

| Phase | Approximate Memory |
| :--- | :--- |
| `file.arrayBuffer()` | 1× input |
| `PDFDocument.load()` parse | 2–4× input |
| `copyPages()` duplication | 1–2× input |
| `mergedPdf.save()` output | 1× input |
| **Peak** | **5–8× input** |

**Mitigations the code already uses:**

- `URL.revokeObjectURL()` after each download — prevents Blob leaks
- Files read one at a time, not all at once
- File list capped at 100 items

**If you need larger files:** see the [Server Edition](https://github.com/YOUR_USERNAME/merge-pdf-server) companion project, which uses `qpdf` on a Node.js backend to handle GB-sized merges.

---

## Known Limitations

| Limitation | Reason |
| :--- | :--- |
| **Encrypted PDFs fail** | `pdf-lib` can't parse password-protected files |
| **Form fields not merged** | `copyPages()` copies content but not AcroForm state |
| **Bookmarks/TOC dropped** | Outline trees aren't preserved by `copyPages` |
| **Digital signatures invalidated** | Any modification breaks signatures (inherent) |
| **Very large files crash** | Browser heap limit (see above) |

These are inherent to the client-side approach — not bugs to fix, but boundaries to accept. The server edition addresses most of them.

---

## Security & Privacy

**What leaves your device:** Nothing.

**What stays on your device:**

- The PDF files you select
- The merged output
- All processing in between

**Network activity:**

- Page load fetches two libraries from CDN (unpkg.com, cdn.jsdelivr.net)
- That's it. No analytics, no telemetry, no API calls.

**To self-host everything:** download the two library files, place them next to `merge_pdf.html`, and change the `<script src>` tags to relative paths. Now the tool works 100% offline with zero third-party requests.

---

## Customization

Everything is in one file, so customizing is just editing the HTML:

### Change the accent color

```css
:root {
  --primary: #e74c3c; /* → your brand color */
  --primary-dark: #c0392b;
}
```

### Change the output filename

```javascript
a.download = 'merged_document.pdf';  // ← change this string
```

### Add a per-file size warning

```javascript
if (file.size > 50 * 1024 * 1024) {
  if (!confirm(`${file.name} is ${(file.size/1024/1024).toFixed(0)} MB and may cause issues. Continue?`)) return;
}
```

### Allow merging a single file

```javascript
mergeBtn.disabled = selectedFiles.length < 1;   // change < 2 to < 1
```

---

## Comparison

| Feature | This project | iLovePDF | pdf-lib tutorials |
| :--- | :---: | :---: | :---: |
| Files uploaded to server | ❌ Never | ✅ Yes | ❌ Never |
| Works offline | ✅ | ❌ | ✅ |
| Handles >100 MB | ❌ | ✅ | ❌ |
| Setup required | ❌ None | N/A | ⚙️ Some |
| Single HTML file | ✅ | ❌ | ❌ |
| Drag reorder | ✅ | ✅ | ❌ |
| Free forever | ✅ | ⚠️ Freemium | ✅ |

Each approach has its place. This one wins on **privacy, simplicity, and setup cost** — and loses on **maximum file size**.

---

## Roadmap

- [ ] Password-protected PDF support (via `pdf-lib`'s `ignoreEncryption`)
- [ ] Page-level reordering (currently file-level only)
- [ ] Merge-from-clipboard (paste PDFs directly)
- [ ] PDF preview thumbnails before merge
- [ ] Optional compression after merge
- [ ] Web Worker offload for larger merges
- [ ] WebAssembly engine for 5× memory efficiency

Contributions welcome — open an issue with the tag `enhancement`.

---

## Contributing

Pull requests are welcome. This project has an intentional constraint: **it must stay a single file**.

If you want to add features that require splitting into multiple files or adding a build step, that belongs in the [Server Edition](https://github.com/YOUR_USERNAME/merge-pdf-server) instead.

```bash
git clone https://github.com/YOUR_USERNAME/merge-pdf-browser.git
cd merge-pdf-browser
# Edit merge_pdf.html, open it, test it, commit
```

No `npm install`, no build, no test framework. Just open the file and check that it works.

---

## License

[MIT](LICENSE) — copy it, host it, embed it in your product, no attribution required (though appreciated).

---

## Acknowledgments

- [`pdf-lib`](https://github.com/Hopding/pdf-lib) — the library that makes browser PDF manipulation possible
- [`SortableJS`](https://github.com/SortableJS/Sortable) — the smoothest drag-drop library in existence
- []() — the gold standard for PDF tool UX

---

<div align="center">

**Made with ☕ and zero backend**

*If this saved you from uploading sensitive documents to a stranger's server, star the repo ⭐*

</div>

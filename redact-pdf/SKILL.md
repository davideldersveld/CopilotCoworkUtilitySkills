---
name: redact-pdf
description: |
  Redact specific words or phrases from a PDF so they are both visually
  covered with black bars AND removed from the text layer (uncopyable,
  unsearchable, not recoverable via pdftotext). Preserves all other text
  on the affected pages as selectable, extractable text. Use when the user
  asks to "redact", "black out", "hide names", "censor", or "remove from
  PDF" specific words while keeping the rest of the document intact.
---

# PDF Redaction Guide

## Overview

True redaction requires two operations together:
1. **Remove the target text from the PDF content stream** so it cannot be
   selected, copied, or extracted with `pdftotext`.
2. **Overlay solid black rectangles** at the target text's bounding boxes
   so the page remains visually consistent (otherwise the layout shows
   gaps where text used to be).

Doing only step 1 leaves visible gaps. Doing only step 2 leaves the text
recoverable in the text layer — selection, copy/paste, and `pdftotext`
all still reveal the "redacted" content.

Rasterizing the entire page is NOT a valid redaction strategy: it
destroys the text layer for the rest of the page, breaking accessibility,
search, and copy/paste for content the user wanted to keep.

## Execution Pattern

Wrap the redaction and verification in a single Task tool call. Do not
narrate steps to the user. Present only the final result.

**Bash tool descriptions (BLOCKING):** Every Bash call inside the Task
emits a visible event. Use user-facing intent language:
- Main step → `"Redacting your PDF"`
- Verification → `"Verifying the redactions"`
- Retry → `"Retrying the redaction"`
- Save → `"Saving your PDF"`

**What stays OUTSIDE the Task:**
- Reading the source PDF to identify the target strings (if the user
  hasn't explicitly listed them).
- Confirming with the user which strings to redact when ambiguous.
- Writing the redaction script.

**What goes INSIDE the Task:**
- Running the redaction script.
- Verifying the output (text-layer extraction + visual inspection).
- Retrying with a more granular strategy if verification fails.

## Task Template

```
Task(
  description="Redacting your PDF",
  prompt="Redact a PDF by removing specific strings from the text layer and covering them with black bars. Do not output script references or technical details. Present only the final result.

<instructions>
Bash descriptions MUST be user-facing:
- Main step → 'Redacting your PDF'
- Verification → 'Verifying the redactions'
- Retry → 'Retrying the redaction'
- Save → 'Saving your PDF'
</instructions>

<environment>
Sandboxed container. Pre-installed: python3 + pypdf, pdfplumber, reportlab, Pillow; pdftoppm, pdftotext, pdfinfo, qpdf. NEVER install packages.
</environment>

Source: [path to input PDF]
Output: [path to output PDF — must NOT overwrite the source unless the user explicitly asked to]
Target strings: [list of exact strings to redact]
Target pages: [page numbers (1-indexed), or 'all']

REQUIREMENTS — both must hold:
A. Target strings must NOT appear in `pdftotext` output and must not be selectable.
B. All OTHER text on the affected pages MUST remain in the text layer — selectable and extractable.

Use the two-stage approach in this skill (strip content stream + overlay black rects). Do NOT rasterize the page.

Verification (all must pass):
1. `pdftotext <output> - | grep -F` each target string → zero matches.
2. `pdftotext <output> -` still contains representative non-target strings from the affected pages (sample a few).
3. `pdfinfo` shows the expected page count.
4. The source PDF is byte-identical to before (sha256 unchanged).
5. Visual check via `pdftoppm -jpeg -r 100` confirms black bars cover the target text.

If step 1 fails, retry with the granular TJ-array strategy described in the skill reference. Do not report success until all checks pass.
[Include the redaction script here]"
)
```

## Core Algorithm

```python
from pypdf import PdfReader, PdfWriter
from pypdf.generic import ContentStream, NameObject, TextStringObject, ByteStringObject, ArrayObject
import pdfplumber, io
from reportlab.pdfgen import canvas
from reportlab.lib.colors import black

def redact_pdf(src, dst, targets, pages=None):
    """
    src: input PDF path
    dst: output PDF path (must differ from src unless overwrite is explicit)
    targets: set of exact strings to redact
    pages: 0-indexed list of pages to redact, or None for all pages
    """
    targets = set(targets)
    reader = PdfReader(src)
    writer = PdfWriter()
    page_indices = pages if pages is not None else range(len(reader.pages))

    # Find bounding boxes per page using pdfplumber
    boxes_by_page = {}
    with pdfplumber.open(src) as pdf:
        for i in page_indices:
            pg = pdf.pages[i]
            pw, ph = pg.width, pg.height
            boxes = []
            for w in pg.extract_words():
                if w["text"] in targets:
                    # reportlab coords: origin bottom-left
                    boxes.append((w["x0"], ph - w["bottom"], w["x1"], ph - w["top"]))
            boxes_by_page[i] = (pw, ph, boxes)

    for i, page in enumerate(reader.pages):
        if i not in boxes_by_page:
            writer.add_page(page)
            continue

        # 1) Strip target strings from the content stream
        cs = ContentStream(page["/Contents"].get_object(), reader)
        new_ops = []
        for operands, operator in cs.operations:
            if operator in (b"Tj", b"'", b'"'):
                if operands and isinstance(operands[-1], (TextStringObject, ByteStringObject)):
                    if str(operands[-1]).strip() in targets:
                        operands[-1] = TextStringObject("")
            elif operator == b"TJ":
                if operands and isinstance(operands[0], ArrayObject):
                    arr = operands[0]
                    combined = "".join(
                        str(it) for it in arr
                        if isinstance(it, (TextStringObject, ByteStringObject))
                    )
                    if combined.strip() in targets:
                        for j, it in enumerate(arr):
                            if isinstance(it, (TextStringObject, ByteStringObject)):
                                arr[j] = TextStringObject("")
            new_ops.append((operands, operator))
        cs.operations = new_ops
        page[NameObject("/Contents")] = cs

        # 2) Overlay black rectangles at the bounding boxes
        pw, ph, boxes = boxes_by_page[i]
        buf = io.BytesIO()
        c = canvas.Canvas(buf, pagesize=(pw, ph))
        c.setFillColor(black); c.setStrokeColor(black)
        pad = 1
        for (x0, y0, x1, y1) in boxes:
            c.rect(x0 - pad, y0 - pad, (x1 - x0) + 2*pad, (y1 - y0) + 2*pad,
                   fill=1, stroke=0)
        c.showPage(); c.save()
        buf.seek(0)
        page.merge_page(PdfReader(buf).pages[0])

        writer.add_page(page)

    with open(dst, "wb") as f:
        writer.write(f)
```

## Handling Edge Cases

### Strings split across TJ array elements (kerning)

The simple "combined array equals target" check works for most PDFs, but
kerned text may split a name across multiple array entries with
positioning numbers between glyphs:

```
[(Al) -2 (ex)] TJ    # "Alex" rendered with kerning
```

If verification step 1 still finds a target string after the simple pass,
fall back to the granular strategy:

1. For each TJ operator, walk the array entries left-to-right and
   accumulate the visible glyphs (skipping numeric positioning entries).
2. Track which array index each glyph came from.
3. If the accumulated text contains a target substring, blank out every
   array entry whose glyphs fall inside that substring range.

### Targets with overlapping or substring matches

If one target is a substring of another (e.g. "Sam" and "Samuel"),
process the longer string first and use word-boundary checks on bounding
box extraction. `pdfplumber.extract_words()` already tokenizes by
whitespace, so exact word matches are safe by default; substring redaction
requires character-level extraction via `extract_text(use_text_flow=True)`
or `chars` and merging adjacent matching glyphs into bounding boxes.

### Scanned PDFs (no text layer)

If `pdftotext` on the source returns empty or garbled output, the PDF is
image-based and has no text layer to strip. In that case, OCR the page
to locate the target text (`pytesseract` + `pdf2image`), then rasterize
the page with black bars drawn over the OCR'd coordinates. Disclose to
the user that the entire page becomes an image (no text layer for
anything) — this is unavoidable for scanned input.

### Mixed-case and whitespace variants

The simple comparison uses `.strip()` and exact match. If the user wants
case-insensitive redaction or whitespace-tolerant matching, normalize
both sides before comparing. Be careful: aggressive normalization can
match unintended text. Confirm scope with the user when in doubt.

## Verification Protocol (BLOCKING)

Before reporting success, all five checks must pass:

| # | Check | Command |
|---|-------|---------|
| 1 | Targets gone from text layer | `pdftotext <out> - \| grep -F <each target>` returns nothing |
| 2 | Non-target text preserved | `pdftotext <out> -` contains representative samples |
| 3 | Page count unchanged | `pdfinfo <out>` shows expected count |
| 4 | Source untouched (if not overwriting) | `sha256sum` matches pre-run hash |
| 5 | Visual coverage | `pdftoppm` render shows black bars over targets |

If any check fails, retry with a more granular strategy or escalate the
limitation to the user. Do not claim success on partial redaction —
incomplete redaction is worse than none, because users assume it worked.

## What This Skill Does NOT Do

- Does not remove text from headers, footers, annotations, form fields,
  XFA layers, attached files, or document metadata (Title, Author,
  Subject, Keywords). Those require separate handling — see the `pdf`
  skill's reference for metadata removal.
- Does not redact text inside embedded images or scanned content unless
  the OCR fallback is used (which rasterizes the page).
- Does not provide cryptographic guarantees against forensic recovery.
  PDF redaction at the content-stream level is reliable against normal
  copy/paste, search, and `pdftotext`, but determined forensic analysis
  of compressed object streams can sometimes recover removed content.
  For legal-grade redaction, the user should re-export the result
  through a flattening tool (e.g. `qpdf --linearize` followed by
  `gs -dPDFSETTINGS=/prepress`) and confirm with their legal team.

---
name: signature-pdf
description: |
  Create signature-ready PDFs that work in three ways: (1) clickable AcroForm
  fields for typing directly in any PDF reader, (2) invisible Adobe Sign text
  tags for cloud-upload auto-detection, and (3) invisible DocuSign anchor
  strings for cloud-upload auto-detection. Use when the user asks for a
  "signature-ready PDF", "contract template", "NDA for signing", "e-sign
  template", "DocuSign-ready document", "Adobe Sign template", or any
  PDF where someone will sign — whether locally with Acrobat Reader, or
  via Adobe Sign / DocuSign cloud services.
---

# Signature-Ready PDF Guide

## Overview

A truly signature-ready PDF satisfies three independent signing workflows
at once:

| Workflow | What the user does | What the PDF needs |
|----------|--------------------|--------------------|
| Type in any PDF reader | Click a line, type a name/date | Real AcroForm text fields with underlined borders |
| Adobe Sign upload | Upload to Adobe Sign cloud | Text tags like `{{Sig_es_:signer1:signature}}` somewhere in the text layer |
| DocuSign upload | Upload to DocuSign cloud | Anchor strings like `\s1\` somewhere in the text layer |

A common mistake is rendering the platform tags as visible placeholder text.
The tags are *upload-time markers* — both services scan for them and
*replace* them with real fields during ingestion. In the source PDF the
tags should be invisible: 1-point white-on-white. They remain in the
text layer (so the services find them), but no human reader ever sees
them.

The second common mistake is omitting AcroForm fields entirely and
assuming the signature lines will be clickable. They won't. A plain PDF
has no interactive elements. If you want clickable signature lines that
work without uploading to a cloud service, you must embed AcroForm
widgets.

This skill produces a PDF that works in all three workflows with no
conflicts. Adobe Sign and DocuSign both gracefully handle pre-existing
AcroForm fields — they either use them directly or map them to their
own fields based on the anchors. Either path is fine.

## Execution Pattern

Wrap generation and verification in a single Task tool call. Do not
narrate steps to the user. Present only the final result.

**Bash tool descriptions (BLOCKING):**
- Generation → `"Building your contract"` (or `"Building your NDA"`, etc.)
- Retry → `"Rebuilding your document"`
- Verification → `"Verifying the document"`
- Visual inspection → `"Checking the document for errors"`
- Save → `"Saving your PDF"`

**What stays OUTSIDE the Task:**
- Confirming the document type, party labels, and signer count via
  `AskUserQuestion` if not provided.
- Writing the generation script.

**What goes INSIDE the Task:**
- Running the generation script.
- Verifying tag invisibility, field presence, and text-layer integrity.
- Retrying if any verification step fails.

## Up-Front Questions (When to Ask)

If the user said only "make a signature-ready PDF", ask three short
questions via `AskUserQuestion`:

1. **Document type** — NDA (mutual), simple agreement/letter, services
   contract, or blank template?
2. **Signature platform conventions** — both Adobe + DocuSign anchors,
   Adobe only, DocuSign only, or visible signature lines only (manual
   field drop after upload)?
3. **Number of signers** — one, two, or three?

If the user already specified all three, skip the questions.

## Task Template

```
Task(
  description="Building your contract",
  prompt="Create a signature-ready PDF. Do not output script references or technical details. Present only the final result.

<instructions>
Bash descriptions MUST be user-facing:
- Generation → 'Building your [document type]'
- Retry → 'Rebuilding your [document type]'
- Verification → 'Verifying the document'
- Visual inspection → 'Checking the document for errors'
- Save → 'Saving your PDF'
</instructions>

<environment>
Sandboxed container. Pre-installed: python3 + pypdf, pdfplumber, reportlab, Pillow; pdftoppm, pdftotext, pdfinfo, qpdf. NEVER install packages.
</environment>

OUTPUT: output/[descriptive_filename].pdf

REQUIREMENTS — all three must hold:
A. All signature tags (Adobe Sign and DocuSign) must be INVISIBLE in the visual document but PRESENT in the text layer. Render every tag as 1pt white-on-white text.
B. Real clickable AcroForm text fields must be embedded at every signature/name/title/date line, with underlined borders.
C. The document must be a clean flattened PDF — body content with no conflicting form widgets, signature region with the clickable fields.

DESIGN: [document body specification — title, intro, sections, signature block layout]

ANCHOR CONVENTIONS — use these EXACT strings per signer (N = 1, 2, 3, ...):
Adobe Sign text tags:
- {{Sig_es_:signerN:signature}}, {{N_es_:signerN:fullname}}, {{*Ttl_es_:signerN:title}}, {{Dte_es_:signerN:date}}
DocuSign anchors (escape backslashes in Python source — write \"\\\\sN\\\\\" for the literal \\sN\\):
- \\sN\\, \\nN\\, \\tN\\, \\dN\\

ACROFORM FIELD NAMES (one per visible line, per signer):
- partyA_signature, partyA_name, partyA_title, partyA_date
- partyB_signature, partyB_name, partyB_title, partyB_date
(Use semantic prefixes — 'client_'/'provider_', 'disclosing_'/'receiving_', etc. — matching the document's role names.)

Use reportlab's canvas API with canvas.acroForm.textfield(name=..., borderStyle='underlined', forceBorder=True, ...). Position each field exactly on its signature line — the field's underlined border IS the signature line; do not also draw underscores.

VERIFICATION (all must pass):
1. pdfinfo shows valid PDF and 'Form: AcroForm'.
2. pdftotext output contains the document body landmarks and every tag/anchor (grep each — DocuSign anchors must contain literal backslashes).
3. Visual render via pdftoppm shows NO visible tag text anywhere — all anchors and Adobe tags must be invisible.
4. pypdf.PdfReader.get_fields() returns all expected field names.
5. Glob output/**/*.pdf returns the new file.

If any verification fails, fix and retry. Do not claim success on partial completion."
)
```

## Reference Anchor Strings

**Adobe Sign text tags** (place near corresponding field, render as 1pt white):

| Field | Tag pattern |
|-------|-------------|
| Signature | `{{Sig_es_:signerN:signature}}` |
| Full name | `{{N_es_:signerN:fullname}}` |
| Title | `{{*Ttl_es_:signerN:title}}` (the `*` marks the field optional) |
| Date | `{{Dte_es_:signerN:date}}` |
| Initials | `{{Int_es_:signerN:initials}}` |
| Email | `{{Em_es_:signerN:email}}` |
| Checkbox | `{{c_es_:signerN}}` |

Required vs. optional: prefix with `*` to make optional (`{{*Ttl_es_...}}`).
Default is required.

**DocuSign anchors** (place near corresponding field, render as 1pt white,
preserve literal backslashes):

| Field | Anchor pattern |
|-------|---------------|
| Signature | `\sN\` |
| Full name | `\nN\` |
| Title | `\tN\` |
| Date | `\dN\` |
| Initials | `\iN\` |
| Email | `\eN\` |

The `N` is the signer number. Pick any tokens you like — DocuSign matches
on the literal string. The convention `\sN\` is just a widely-used
shorthand; `[[sig1]]` or `<<sig_alice>>` work equally well as long as
the user configures the same string in DocuSign's anchor settings.

**Critical escaping in Python source:** to emit the literal four-character
string `\s1\` into the PDF text layer, write `"\\s1\\"` in Python. The
`pdftotext` extraction MUST show the literal backslashes — verify with
`pdftotext file.pdf - | grep -F '\s1\'`.

## Core Implementation Pattern

```python
from reportlab.lib.pagesizes import LETTER
from reportlab.pdfgen import canvas
from reportlab.lib.colors import black, white
from reportlab.platypus import Paragraph
from reportlab.lib.styles import getSampleStyleSheet

W, H = LETTER
MARGIN = 72  # 1 inch
c = canvas.Canvas("output/contract.pdf", pagesize=LETTER)
styles = getSampleStyleSheet()

# --- BODY: title, intro, numbered sections ---
# Use Paragraph.wrapOn / drawOn to flow prose at fixed Y positions.
# (Use SimpleDocTemplate for multi-page prose if length is dynamic, then
#  overlay the signature region on the final page via a second pass.)

# --- SIGNATURE REGION (last page, bottom ~3 inches) ---

def hidden(c, text, x, y):
    """Render a tag invisibly at (x, y) — 1pt white text."""
    c.saveState()
    c.setFillColor(white)
    c.setFont("Helvetica", 1)
    c.drawString(x, y, text)
    c.restoreState()

def sig_block(c, label, name_prefix, signer_n, x, y_top):
    """Draw one signer's block at (x, y_top) — label, four fields, anchors."""
    # Heading
    c.setFillColor(black)
    c.setFont("Helvetica-Bold", 10)
    c.drawString(x, y_top, label + ":")

    # Field rows
    fields = [
        ("signature", "Signature", 200, 18),
        ("name",      "Name",      200, 14),
        ("title",     "Title",     200, 14),
        ("date",      "Date",      120, 14),
    ]
    y = y_top - 32
    for short, lbl, w, h in fields:
        c.acroForm.textfield(
            name=f"{name_prefix}_{short}",
            tooltip=f"{label} {lbl}",
            x=x, y=y, width=w, height=h,
            borderStyle="underlined",
            borderColor=black,
            fillColor=None,
            textColor=black,
            forceBorder=True,
            fontSize=10,
        )
        # Label below the field
        c.setFillColor(black)
        c.setFont("Helvetica", 8)
        c.drawString(x, y - 10, lbl)
        y -= 40

    # Invisible anchors and tags near the signature line
    hidden(c, f"{{{{Sig_es_:signer{signer_n}:signature}}}}", x, y_top - 32)
    hidden(c, f"\\s{signer_n}\\",                            x + 1, y_top - 32)
    hidden(c, f"{{{{N_es_:signer{signer_n}:fullname}}}}",    x, y_top - 72)
    hidden(c, f"\\n{signer_n}\\",                            x + 1, y_top - 72)
    hidden(c, f"{{{{*Ttl_es_:signer{signer_n}:title}}}}",    x, y_top - 112)
    hidden(c, f"\\t{signer_n}\\",                            x + 1, y_top - 112)
    hidden(c, f"{{{{Dte_es_:signer{signer_n}:date}}}}",      x, y_top - 152)
    hidden(c, f"\\d{signer_n}\\",                            x + 1, y_top - 152)

# Define signer blocks explicitly so numbering and field prefixes stay aligned.
# Two signers fit side-by-side on Letter; a third signer usually needs a
# stacked layout or another page.
signers = [
    ("CLIENT", "partyA", 1, MARGIN, MARGIN + 200),
    ("SERVICE PROVIDER", "partyB", 2, W/2 + MARGIN/2, MARGIN + 200),
    # Example third signer:
    # ("WITNESS", "partyC", 3, MARGIN, MARGIN + 20),
]

for label, name_prefix, signer_n, x, y_top in signers:
    sig_block(c, label, name_prefix, signer_n, x, y_top)

c.showPage()
c.save()
```

Note: `canvas.acroForm.textfield(...)` automatically creates the
document-level `/AcroForm` dictionary in the PDF catalog the first time
it's called. If you're combining pages from multiple sources via pypdf,
verify the `/AcroForm` entry survives the merge — if not, copy it from
the source explicitly.

## Mixing Platypus Prose With AcroForm Signature Region

If the document body is long enough to need multi-page flow, use
SimpleDocTemplate for the prose, then merge with a second canvas for the
signature region:

1. Build the prose with `SimpleDocTemplate` into a buffer. Reserve empty
   space on the last page for signatures (e.g., end the story with a
   `Spacer` sized to leave ~3 inches at the bottom).
2. Build a second canvas with only the signature region drawn on the
   page-N coordinates.
3. Merge with pypdf: `page.merge_page(signature_overlay_page)` for the
   last page; copy `/AcroForm` from the overlay's reader to the writer's
   root catalog.

Alternative: build the entire document with manual `canvas.Canvas`
positioning. Simpler for short documents (≤2 pages).

## Visible vs. Invisible Field Borders

| Choice | Behavior | When to use |
|--------|----------|-------------|
| `borderStyle='underlined', forceBorder=True` | Solid underline on field | DEFAULT — the underline IS the signature line, professional look |
| `borderStyle='solid'` | Box around field | Form-style documents (intake forms) |
| `borderStyle='inset', fillColor=lightyellow` | Boxed yellow fill | Hint to user "fill this in" — less formal |
| No border | Invisible field | Background data capture — not appropriate for signatures |

Do not draw underscores under a field that has its own underlined
border. The underscores double up with the field border and look broken.

## Signer Naming Conventions

Use semantic names that match the document's role labels, not generic
indexes:

| Document type | Suggested prefixes |
|---------------|--------------------|
| Mutual NDA | `disclosing_`, `receiving_` (or `partyA_`, `partyB_`) |
| Services agreement | `client_`, `provider_` |
| Employment offer | `employer_`, `employee_` |
| Lease | `landlord_`, `tenant_` |
| Three-party agreement | `partyA_`, `partyB_`, `partyC_` (or role-specific) |

Adobe Sign and DocuSign both display these field names to the sender
when configuring the envelope. Clear names speed up envelope setup.

## Verification Protocol (BLOCKING)

Before reporting success, all five checks must pass:

| # | Check | Command |
|---|-------|---------|
| 1 | Valid PDF, has AcroForm | `pdfinfo <out>` shows "Form: AcroForm" |
| 2 | Body landmarks present | `pdftotext <out> -` contains title, key section headings |
| 3 | All tags in text layer | `pdftotext <out> - \| grep -F` each tag and anchor (DocuSign anchors must contain literal `\`) |
| 4 | No visible tag text | `pdftoppm -jpeg -r 100 <out> working/check` — visual inspection shows no `{{...}}` or `\sN\` strings |
| 5 | AcroForm fields detected | `pypdf.PdfReader(<out>).get_fields()` returns all expected field names |

If check 4 finds visible tag text, the `setFillColor(white)` or
`setFont(..., 1)` call was wrong — diagnose which tag rendered black or
at a readable size and fix.

If check 5 returns `None` or missing fields, the `/AcroForm` catalog
entry is missing. Add it explicitly via pypdf:
```python
from pypdf.generic import NameObject, ArrayObject
reader = PdfReader(src)
# Collect all widget annotations across pages
widgets = []
for page in reader.pages:
    for annot in page.get("/Annots", []):
        obj = annot.get_object()
        if obj.get("/Subtype") == "/Widget":
            widgets.append(annot)
reader.trailer["/Root"][NameObject("/AcroForm")] = DictionaryObject({
    NameObject("/Fields"): ArrayObject(widgets),
    NameObject("/NeedAppearances"): BooleanObject(True),
})
```

## Common Failure Modes

| Symptom | Cause | Fix |
|---------|-------|-----|
| Tags visible as text in viewer | Drew in black or default font size | Wrap in `saveState`/`restoreState` with `setFillColor(white)` and `setFont(..., 1)` |
| `\s1\` becomes `s1` in text layer | Backslashes unescaped in Python source | Write `"\\s1\\"` in source |
| Signature lines not clickable | No AcroForm fields, only visual lines | Add `canvas.acroForm.textfield(...)` calls |
| Fields clickable but tab order is wrong | Fields added in wrong order | Call `acroForm.textfield` in desired tab order (top-to-bottom, left-to-right) |
| Adobe Sign creates duplicate fields on upload | Anchors AND AcroForm fields both present, and Adobe couldn't reconcile them | Use ONE strategy per upload destination: if uploading to Adobe Sign, the existing AcroForm fields are enough — drop the Adobe tags. Keep DocuSign anchors only when targeting DocuSign. Use the three-way approach only when you don't know which platform the user will pick. |
| DocuSign doesn't detect anchors | Anchor rendered without backslashes, or font color not actually white | Verify with `pdftotext` — output must contain literal `\sN\` strings |

## What This Skill Does NOT Do

- Does not apply cryptographic signatures (PKCS#7, X.509). Those require
  a signing certificate and a signature handler — typically applied by
  the signer's software, not the document author.
- Does not configure routing, signing order, reminders, or
  authentication settings. Those are envelope settings in Adobe Sign /
  DocuSign, configured per-send, not embedded in the PDF.
- Does not generate legally binding contract content. The template
  language is a starting point — users should have legal counsel review
  before use in production.
- Does not handle attachments, supporting exhibits, or schedule pages
  beyond the main signature block. For multi-document envelopes, send
  each document separately and let the e-sign platform bundle them.

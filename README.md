# Label Verification 42 — TTB Prototype

A browser-based prototype that checks an alcohol beverage label image against the
application's declared values and reports, per mandatory field, whether they **match**,
need **human review**, or are a **mismatch**. Everything runs in the browser — the
application and label image never leave the user's machine, and the app makes no
external network calls at runtime.

The BETA's goal is to take the legwork off agents so they can spend their time on the
cases that actually need judgment. Agents shouldn't have to eyeball every routine label
that comes in — the BETA screens the straightforward ones and surfaces only what needs
a human. Over time it's meant to streamline that intake pipeline end to end.

---

## Project structure

All of the application code is in a single file:

- **`index.html`** — the entire app: markup, CSS, and all logic (OCR pipeline, per-field
  detection and comparison rules, file pairing, UI, the mock notice) in the `<script>` blocks.
  **This is the file to read.** The core field checks are `checkAlcoholField`, `detectVolume` /
  `checkNetField`, `checkBrandField`, `checkClassField`, `checkImport`, and `checkWarning` /
  `checkWarningSize`, orchestrated by `runChecks`.
- `vendor/` — bundled third-party libraries (Tesseract.js, PDF.js, traineddata, WASM); not app code.
- `test-labels/` — sample data (the 88 products: label image + matching application).
- `BLANK_application_template.pdf` — the empty fillable COLA form.
- `Label_Verification_42_Results.pdf` — a sample results export.
- `README.md` — this file.

---

## What it does

Two interfaces share one checking engine:

**BETA Submitter interface — single submission.** You provide one COLA application (PDF) and
its label image; the app parses the application's declared values, reads the label with
on-device OCR (Tesseract.js / WebAssembly), and returns a per-field verdict for that one
product. Intended as the applicant-facing front door, so obvious problems get caught at
submission time before anything reaches an agent.

**Agent Interface tool — batch triage.** You drop anywhere from 1 to 100 label + application
pairs at once (paired by folder or by matching filename). The app runs every pair and routes the
results into **Failed**, **Passed — Imports**, and **Passed**, so an agent works the problem
cases first instead of opening each submission by hand. When a label comes in with **no
application paired to it**, the tool doesn't skip it — it reads the label on its own and
**infers** each mandatory field straight from the image, running the intrinsic checks
(present? warning exact and properly sized? alcohol/proof internally consistent?) without a
cross-comparison. Each result lets you view the label and the application, print the page,
and draft a mock applicant notice grouped into Must-Fix and Needs-Review items.

Under the hood, for every field both interfaces:

- read the application value (when present) and the detected label value, and return a
  three-state result — **match / needs review / mismatch** — with the two values side by side;
- check brand, class/type, alcohol content, net contents, health warning, warning size, and
  (for imports) a U.S. importer statement;
- always test the health warning against the fixed statutory text (27 CFR 16.21).

---

## Run it locally

This is a static site, but it **must be served over HTTP** — opening `index.html` directly
(`file://`) will not work, because browsers block the OCR worker/WASM on `file://`. You need
[Node.js](https://nodejs.org) **or** Python installed.

1. **Get the code.** On the repo page, green **`<> Code`** button → **Download ZIP** → unzip
   it. (Or `git clone https://github.com/williamherndon07/label-verifier.git`.)
2. **Open a terminal in the project folder** — the folder that contains `index.html`.
3. **Start a static server:**

   ```bash
   # Node — serves at http://localhost:3000
   npx serve

   # …or Python — serves at http://localhost:8000
   python -m http.server 8000
   ```

4. **Open the URL it prints** in your browser (`http://localhost:3000` for `npx serve`,
   `http://localhost:8000` for Python).

First load takes a few seconds to initialise the OCR engine; after that, each scan is fast.

### Test data

A full sample set lives in `test-labels/` (88 products):

- **`test-labels/single entry/`** — one folder per product, each holding a label image and its
  matching application PDF, grouped into `beer/` (16), `wine/` (23), `mead/` (19), `spirit/`
  (17), `imports/` (12), and `defects/` (1). Use these to feed the BETA Submitter interface or
  to point the Agent tool at a whole category.
- **`test-labels/bulk/`** — the same 88 products as a flat set (`<category>_<product>.png` +
  matching `_application.pdf`), for dropping straight into the Agent Interface tool.

`BLANK_application_template.pdf` is the empty fillable COLA form the applications are built
from — fill it in a PDF reader and **Save** (don't flatten or scan a printout; the parser reads
the saved form fields).

### Testing it

The tool reads files from **your machine**, not from the server, so get the test set local
first. The whole repo — app plus all 88 products — downloads in one click:

> On the repository page, click the green **`<> Code`** button (top-right, next to **Add file**),
> then **Download ZIP** at the bottom of the panel. Unzip it; the labels and applications are in
> `test-labels/`. (Or `git clone` the repo if you prefer.)

On the live GitHub Pages site you must do this — the app can't pull the files from the repo on
its own.

**Single submission — BETA Submitter interface.** Expand **BETA Submitter interface** and, from
any product in `test-labels/single entry/<category>/<product>/`, add the label image
(`label_*.png`) and its application (`*_application.pdf`), then verify.

**Batch — Agent Interface tool.** Expand **Agent Interface tool** and drag in a whole folder:

- **`test-labels/single entry/`** — each product is its own subfolder, paired by folder; or
- **`test-labels/bulk/`** — a flat set, paired by matching filename.

Don't rename or flatten the files — pairing depends on the subfolder structure or the matching
filenames.

---

## Deploy to GitHub Pages

1. Create a new repository and upload every file in this folder (keep the `vendor/`
   folder and its contents).
2. Repo **Settings → Pages → Build and deployment → Deploy from a branch**, select
   `main` / `/root`, save.
3. Your URL will be `https://<username>.github.io/<repo>/`.

Because GitHub Pages serves a project repo from a subpath, all asset paths in this app
are **relative** (`vendor/...`), so they resolve correctly under the subpath.

---

## Approach

- **OCR on-device.** OCR runs entirely in the browser (Tesseract WebAssembly, with the
  engine and English model bundled in `vendor/`), so there's no per-scan network round-trip
  and nothing for a firewall to block — the label and application never leave the machine.
- **Application-driven, with detection fallback.** When an application is supplied, its
  values are parsed straight from the submitted COLA PDF and used as the source of truth each
  label field is compared against. When a label arrives with **no application paired to it**,
  the tool falls back to reading every field from the image alone — the "detect from limited
  input" goal — so a label can still be screened on its own.
- **Field-specific logic**, because one rule does not fit every field:
  - **Brand name / class & type** — tolerant. Text is normalised (case, punctuation,
    spacing) and compared with fuzzy matching, so "STONE'S THROW" matches "Stone's Throw",
    minor OCR noise (e.g. `0` for `O`) still matches, and clipped or stylised characters in a
    long brand are tolerated without forcing false matches on the wrong brand.
  - **Alcohol content** — numeric. The percent value is parsed and compared; if proof
    appears, it is checked for consistency (proof ≈ 2 × ABV).
  - **Net contents** — unit-aware. Volumes are normalised to millilitres before comparison,
    across metric and imperial units and spelled-out forms.
  - **Origin / imports** — when the application marks a product imported (or the label shows
    import cues), the label must carry a U.S. importer statement ("Imported by …"); its
    absence is flagged. Domestic products skip this check.
  - **Health warning** — strict. The full statutory text must be present (OCR-noise
    tolerant), and "GOVERNMENT WARNING" must be in all capitals; title-case is flagged
    for review.
  - **Warning text size** — detected from the image alone. The warning's text height is
    compared against the rest of the label; a warning that's much smaller than the other
    text is flagged as buried / too small (the common way the warning gets hidden). No
    dimensions need to be entered. If a label height (inches) is provided, an approximate
    mm estimate is added as a note, but it is not required.
- **Three-state output** — match / needs review / mismatch — rather than a binary
  pass/fail, so borderline cases go to a human instead of being auto-rejected. In the Agent
  tool these feed the Failed / Passed — Imports / Passed routing, surfacing the cases that
  need attention first.

## Tools used

- [Tesseract.js](https://github.com/naptha/tesseract.js) v5.1.1 (OCR, WebAssembly) — bundled locally in `vendor/`
- `eng.traineddata` + `osd.traineddata` from [tessdata](https://github.com/tesseract-ocr/tessdata) (the standard LSTM models) — bundled locally
- [PDF.js](https://github.com/mozilla/pdf.js) v3.11.174 — reads the COLA application PDFs (form fields and text) — bundled locally
- ChatGPT (image generation) — used to create the synthetic test label images in `test-labels/`; not a runtime dependency
- [TTB.gov](https://www.ttb.gov) — current alcohol-labeling regulations (including the 27 CFR 16.21 health-warning text) used as the compliance reference; reference source, not a runtime dependency
- Claude (Anthropic) — development assistance: field-detection logic, debugging, and documentation; not a runtime dependency
- No frameworks, no build step — a single `index.html`

## Assumptions made

- **Validated against current TTB.gov regulations.** The field rules — mandatory
  health-warning wording, the proof/ABV relationship, net-contents and warning-size
  expectations — follow the regulations currently published on TTB.gov, rather than any
  specifics restated in the brief.
- **A lightweight first pass, by design.** The provided details pointed toward a lite,
  low-friction approach (the BETA): auto-screen the routine submissions and leave agents to
  touch only what needs judgment. The assumed end state is a fuller agent that pulls the Q/A
  directly from the application and reviews everything.
- **Application and label both come from the submitter.** Both are provided together at
  submission, so the application is used to validate the label.
- **Handle all major alcohol categories, separated by type.** The brief's example was
  distilled spirits, but the tool is assumed to cover beer, wine, mead, and spirits — and that
  separating results by type (plus imports) is the most useful way to review them.
- **App and label both viewable in the review area.** Assumed a reviewer wants to open each
  one next to its result.
- **Auditable from the results.** Assumed reviewers want to audit any completed label from
  within the results — open the label and application and print it — including ones that
  passed, not just the flagged cases.
- **An agent reviews every issue before the submitter is notified.** Assumed a human agent
  reviews any and all flagged label issues and approves the notice first — the applicant
  notification is drafted for agent review, never sent automatically.
- **Designed-in mismatches and defects.** Assumed it's worth seeding the test set with
  deliberate application-vs-label mismatches and label failures in several different forms, to
  demonstrate the tool's versatility at catching different kinds of discrepancy.
- **Applications are digitally-filled, saved PDFs.** The synthetic applications here stand in
  for real submissions; values are read from saved form fields, so an application should be
  filled and saved — not flattened or scanned from a printout.

## Known limitations (out of scope for the prototype)

- **Designed for simple labels.** The tool targets clean, straightforward labels; heavily
  stylized or densely designed labels are outside its intended scope.
- **OCR misreads on the harder fields.** In testing, **alcohol %, proof, and net contents**
  were the most error-prone — the OCR can grab the wrong number or unit, especially on
  decorative labels (ornate fonts, dividers, low contrast, rotated or curved text). Examples
  of each are included in the **Failed** results so the behaviour is visible; borderline reads
  route to **needs review** rather than being trusted, but a clean read isn't guaranteed on
  every label.
- **Bold / font weight** is not verified — OCR returns text, not font weight. Casing and
  wording of "GOVERNMENT WARNING" are checked; bold is not.
- **Type size** is detected *relatively* from the image (warning vs. the rest of the
  label), which catches a buried/too-small warning without any input. Certifying an exact
  millimetre size against the 1/2/3 mm minimum requires the label's true submitted
  dimensions; the optional inches field gives only a rough estimate.
- **Poor image quality** (angles, glare, low light) is not handled; clean, straight-on
  images are assumed.
- **English only** (one language model bundled).
- **Advisory, not a determination.** Results are decision support for triaging submissions; a
  human makes the official call.

## BUILD 15 — review UX (overhaul)
Detection engine unchanged from BUILD 14 (all lessons retained). Added the reviewer-facing layer:
- **Live count-up timer** while the label is read.
- **Results screen** with a green **SUCCESS** banner and a **1–10 label difficulty** score
  (1 = clean read, 10 = recommend human review), derived from per-field read confidence.
- **Quality check** button — lists every field the tool read correctly.
- **Defects** panel — lists flagged items and offers a **submitter notice** (drafted email
  preview; "Send" shows *Submitter notified*). Mockup only — nothing is actually sent.
- **Bulk verify** framed for ~5 labels at once (pass/review/fail table).

## BUILD 16 — screen-based UX (overhaul)
Detection engine unchanged from BUILD 14/15. Restructured the interface to cut choice overload:
- **Entry screen = two dropdowns:** **Single label** (auto-detect on, drag-and-drop, optional manual
  entry) and **Bulk** (auto-detect forced, handles up to 5 images). No image preview on entry.
- **Loading screen** with a live count-up timer between entry and results.
- **Single results** = its own success/failed page: SUCCESS/REVIEW/DEFECTS banner, 1-10 difficulty,
  and a dropdown holding the label image + a compact **row** table (Field / Detected / Result).
- **Bulk results** = one dropdown per file, tagged by filename, each showing its difficulty, label
  image, and the same compact row table.
- **Defect notice** (mockup) appears when a label has issues; Send shows "Submitter notified". Nothing
  is actually sent.
- Results render as tight rows, not tall stacked cards.

## BUILD 17 — application-driven verification + import check
- **Application (PDF) upload** in the single flow: drop the COLA application PDF and it parses
  the fields and pre-fills the expected values (brand, class/type, alcohol, net, origin).
- **Search-when-provided:** when an expected value is supplied, brand and class are now confirmed
  by SEARCHING the entire label read for that value (fuzzy), instead of guessing one line and
  scoring it. Fixes the false-Pass / wrong-line brand problem on stylized labels.
- **Import compliance check:** if the application origin (or label) indicates an import, the tool
  verifies a U.S. importer name/address is present and FLAGS imports that lack one.
- PDF parsing via self-hosted pdf.js (vendor/). Engine OCR pass unchanged.

## BUILD 17 — Troubleshooting (scan diagnostic)
- New "Troubleshooting" dropdown on the home screen: upload a label and see exactly what the
  OCR scan reads off it — the raw text document the rest of the tool works from — with a
  per-line confidence breakdown, the preprocessed image the engine actually saw, and the original.
  Scan-only: nothing is verified, no application needed. Useful for seeing why a field reads or fails.

## BUILD 24+ — detection refinements
- **Alcohol % capture.** The ABV rule anchors on the `%` sign and reads up to 5 characters to
  its left (allowing at most two spaces, stopping at the first non-number), so the *entire*
  percentage is captured instead of a partial or wrong grab.
- **Net contents / size capture.** Net contents anchors on the measurement unit (mL, L, cL,
  fl oz, and spelled-out forms), reads the number to its left, normalises to millilitres, and
  prefers standard bottle sizes; a secondary OCR pass re-reads the volume when the first pass
  is too sparse to find it.
- **Brand-name matching (blur / stylized fixes).** Brand is confirmed by searching the whole
  label read with fuzzy matching that tolerates OCR noise (e.g. `0`<->`O`) and blurred or
  clipped characters in long or stylized brand names — without false-matching the wrong brand.

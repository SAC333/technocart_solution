# Bug Fix Report

This document records every bug found across the two applications in this repository
(the Node.js **Book Management** app and the Python **Scribe** handwriting-OCR app),
the code **before** the change, the code **after**, and an explanation of why each fix
is correct.

---

## Summary

| # | App | File | Severity | Bug |
|---|-----|------|----------|-----|
| 1 | Node | `src/repositories/bookRepository.js` | High | Category filter used `!=` instead of `=` |
| 2 | Node | `src/services/stockService.js` | High | Decreasing stock actually increased it |
| 3 | Node | `src/controllers/bookController.js` | High | Update passed `price`/`published_year` in the wrong order |
| 4 | Node | `src/routes/books.js` | High | `/search` registered after `/:id`, so it was unreachable |
| 5 | Node | `frontend/js/app.js` | Medium | Search used the previous query (one keystroke behind) |
| 6 | Node | `src/repositories/statsRepository.js` | Medium | `available_copies` excluded single-copy titles |
| 7 | Python | `app/utils/files.py` | High | `read_result` opened in `w+` and truncated the file |
| 8 | Python | `app/services/pipeline.py` | High | OCR arguments swapped — ran on the wrong image |
| 9 | Python | `app/preprocessing/image_ops.py` | Medium | Resize forced 800×600, distorting aspect ratio |
| 10 | Python | `app/recognition/ocr.py` | High | `--psm 7` returned empty text for multi-line notes |

---

## Node.js — Book Management app

### Bug 1 — Category filter inverted

**File:** `nodejs/src/repositories/bookRepository.js`
**Function:** `findAll`

**Before**
```js
if (category) {
  conditions.push("category != ?");
  params.push(category);
}
```

**After**
```js
if (category) {
  conditions.push("category = ?");
  params.push(category);
}
```

**Explanation.**
Selecting a category in the UI sends `GET /api/books?category=Fiction`. The SQL built by
the repository used `category != ?`, which returns every book **except** the selected
category. Changing it to `category = ?` makes the filter inclusive.

*Observed before:* `category=Fiction` → 12 rows, categories `Science, History, Technology, Biography`.
*Observed after:* `category=Fiction` → 4 rows, all `Fiction`.

---

### Bug 2 — Decreasing stock increased the quantity

**File:** `nodejs/src/services/stockService.js`
**Function:** `adjustStock`

**Before**
```js
let next = book.quantity + delta;
if (delta < 0) {
  next = book.quantity + Math.abs(delta);
}
if (next < 0) {
  const error = new Error("Insufficient stock");
  error.status = 400;
  throw error;
}
return repository.updateQuantity(parsed, next);
```

**After**
```js
const next = book.quantity + delta;
if (next < 0) {
  const error = new Error("Insufficient stock");
  error.status = 400;
  throw error;
}
return repository.updateQuantity(parsed, next);
```

**Explanation.**
`book.quantity + delta` already handles a negative `delta` correctly. The extra
`if (delta < 0)` branch then overwrote the result with `book.quantity + Math.abs(delta)`,
turning a decrement into an increment. Removing the branch lets the plain addition work
for both signs; the negative check on the *result* still guards against going below zero.

*Observed before:* quantity `4`, `adjustment: -1` → `5`.
*Observed after:* quantity `4`, `adjustment: -1` → `3`; over-decrement still returns `400 Insufficient stock`.

---

### Bug 3 — Update swapped `price` and `published_year`

**File:** `nodejs/src/controllers/bookController.js`
**Function:** `update`
Service signature for reference — `nodejs/src/services/bookService.js`:
```js
function updateBook(id, title, author, isbn, category, publishedYear, price, quantity) { ... }
```

**Before**
```js
const book = bookService.updateBook(
  req.params.id,
  body.title,
  body.author,
  body.isbn,
  body.category,
  body.price,          // <-- passed into publishedYear
  body.published_year, // <-- passed into price
  body.quantity
);
```

**After**
```js
const book = bookService.updateBook(
  req.params.id,
  body.title,
  body.author,
  body.isbn,
  body.category,
  body.published_year,
  body.price,
  body.quantity
);
```

**Explanation.**
The sixth parameter is `publishedYear` and the seventh is `price`, but the controller
supplied them in the opposite order. The validators then rejected the data
(`price` is a non-integer, so `requiredYear` threw) or silently stored the wrong values.
Passing them in the declared order fixes the round-trip.

*Observed before:* PUT with `published_year: 2001, price: 9.99` → `400`.
*Observed after:* PUT → `200`, stored `published_year: 2001`, `price: 9.99`.

---

### Bug 4 — `/search` route unreachable

**File:** `nodejs/src/routes/books.js`
**Function:** `createBooksRouter`

**Before**
```js
router.get("/", controller.list);
router.get("/:id", controller.getById);
router.get("/search", controller.search);
```

**After**
```js
router.get("/", controller.list);
router.get("/search", controller.search);
router.get("/:id", controller.getById);
```

**Explanation.**
Express matches routes in registration order. `GET /api/books/search` matched `/:id`
first, so `getById` ran with `id = "search"`. `parseId("search")` returns `null`, which
became a `404`. Registering the static `/search` route before the dynamic `/:id` route
ensures it is reached.

*Observed before:* `GET /api/books/search?q=Dune` → `404 {"error":"Book not found"}`.
*Observed after:* `GET /api/books/search?q=Dune` → `200`, returns Dune.

---

### Bug 5 — Search results lagged one keystroke

**File:** `nodejs/frontend/js/app.js`
**Function:** `init`

**Before**
```js
let editingId = null;
let pendingQuery = "";

function init() {
  searchInput.addEventListener("input", () => {
    const query = pendingQuery;
    pendingQuery = searchInput.value.trim();
    loadBooks(query);
  });
```

**After**
```js
let editingId = null;

function init() {
  searchInput.addEventListener("input", () => {
    loadBooks(searchInput.value.trim());
  });
```

**Explanation.**
On each keystroke the handler read the *previous* `pendingQuery` value, then stored the
new one and searched with the old one. The results were always one character behind, and
the final query was never sent. Searching with the current input value fixes it; the
unused `pendingQuery` state was removed.

---

### Bug 6 — `available_copies` excluded single-copy books

**File:** `nodejs/src/repositories/statsRepository.js`
**Function:** `getStats`

**Before**
```sql
COALESCE(SUM(CASE WHEN quantity > 1 THEN quantity ELSE 0 END), 0) AS available_copies
```

**After**
```sql
COALESCE(SUM(CASE WHEN quantity > 0 THEN quantity ELSE 0 END), 0) AS available_copies
```

**Explanation.**
`quantity > 1` skipped any title that has exactly one copy in stock, under-counting the
dashboard's "Copies in stock" figure. `quantity > 0` sums all in-stock copies and still
ignores zero-quantity titles.

*Observed after:* `available_copies = 58` (was 56).

---

## Python — Scribe handwriting-OCR app

### Bug 7 — Reading a result truncated it (`w+`)

**File:** `python/app/utils/files.py`
**Function:** `read_result`

**Before**
```python
def read_result(result_id):
    if not is_result_id(result_id):
        raise FileNotFoundError(result_id)
    path = result_dir() / f"{result_id}.txt"
    with path.open("w+", encoding="utf-8") as handle:
        return handle.read()
```

**After**
```python
def read_result(result_id):
    if not is_result_id(result_id):
        raise FileNotFoundError(result_id)
    path = result_dir() / f"{result_id}.txt"
    with path.open("r", encoding="utf-8") as handle:
        return handle.read()
```

**Explanation.**
Mode `w+` opens the file for writing and **truncates it to zero bytes** before reading.
So every call returned `""` and destroyed the saved text. The download endpoint
(`app/routes/recognize.py`) calls `read_result`, meaning downloading a result wiped it
and produced an empty file. Mode `r` performs a read-only open.

---

### Bug 8 — OCR ran on the wrong image (swapped arguments)

**File:** `python/app/services/pipeline.py`
**Function:** `run_pipeline`
Reference signature — `app/recognition/ocr.py`:
```python
def prepare_for_ocr(original, preprocessed):
    return preprocessed
```

**Before**
```python
ocr_input = prepare_for_ocr(preprocessed, resized)
```

**After**
```python
ocr_input = prepare_for_ocr(resized, preprocessed)
```

**Explanation.**
`prepare_for_ocr(original, preprocessed)` returns its **second** argument (the
preprocessed image). The old call passed `(preprocessed, resized)`, so it returned the
resized *colour* image and OCR ran on that — completely bypassing the binarisation step.
Passing `(resized, preprocessed)` returns the processed binary image as intended.

*Verified:* `prepare_for_ocr(resized, preprocessed) is preprocessed` → `True` (2-D binary).

---

### Bug 9 — Resize distorted the image aspect ratio

**File:** `python/app/preprocessing/image_ops.py`
**Function:** `resize_image`

**Before**
```python
def resize_image(image, max_width=800, max_height=600):
    height, width = image.shape[:2]
    if width == max_width and height == max_height:
        return image
    return cv2.resize(image, (max_width, max_height))
```

**After**
```python
def resize_image(image, max_width=800, max_height=600):
    height, width = image.shape[:2]
    if width <= max_width and height <= max_height:
        return image
    scale = min(max_width / width, max_height / height)
    new_size = (max(1, round(width * scale)), max(1, round(height * scale)))
    return cv2.resize(image, new_size)
```

**Explanation.**
The old code forced every image to exactly 800×600, squashing wide images and stretching
tall ones; the early return only triggered when the input was *already* exactly 800×600.
The new version leaves images that already fit untouched and otherwise downscales
proportionally using a single scale factor, preserving the aspect ratio.

*Verified on the samples:* `1000×360 → 800×288`, `1100×760 → 800×553`,
`480×1200 → 240×600`, `1400×280 → 800×160` — all aspect-ratio-preserving, instead of a
forced `800×600` for every input.

---

### Bug 10 — `--psm 7` returned empty text for multi-line notes

**File:** `python/app/recognition/ocr.py`
**Function:** `tesseract_config`

**Before**
```python
def tesseract_config():
    return "--oem 3 --psm 7"
```

**After**
```python
def tesseract_config():
    return "--oem 3 --psm 11"
```

**Explanation.**
`--psm 7` tells Tesseract to "treat the image as a single text line". Multi-line or tall
notes do not match that model, so Tesseract returned an empty string. An empty result was
then written to disk and downloaded as an empty `.txt` — the exact symptom reported.
`--psm 11` (sparse text) finds text across varied layouts. `--oem 3` (default engine) is
unchanged.

*Verified on samples (`run_pipeline`):*

| Sample | Before (`--psm 7`) | After (`--psm 11`) |
|--------|--------------------|--------------------|
| `note_clear` | `'The quick brown fox jumps'` | `'The quick brown fox jumps'` |
| `note_lines` | `''` | `'Hello from the workshop Please review this note Thank you'` |
| `note_tall` | `''` | `'Tall sample page'` |
| `note_wide` | `'Wide sample line'` | `'Wide sample line'` |

---

## Verification

### Node.js

```powershell
cd nodejs
npm test          # Jest + Supertest
```

- Before: 12/12 passed, but the suite did **not** cover the category filter, stock
  decrement, update round-trip, or the `/search` route — which is why the bugs slipped
  through.
- After: 12/12 passed, plus targeted probes confirmed each fix (category returns only
  the selected category, decrement works, update round-trips, `/search` returns 200).

### Python

```powershell
cd python
.\env\Scripts\Activate.ps1
pip install -r requirements.txt
python -m pytest -q
python -c "from app.services.pipeline import run_pipeline; print(run_pipeline(open('samples/note_lines.png','rb').read())['text'])"
```

- Before: 8/8 passed, but the tests **mock** `pytesseract.image_to_string`, so they
  could not catch the `w+`, aspect-ratio, or psm bugs.
- After: 8/8 passed, plus real-OCR runs on all four samples return non-empty text.

> **Note:** The OCR tests replace the Tesseract call with a stub, so `--psm`/image-pipeline
> regressions are not visible to the automated suite. Manual verification with the bundled
> `samples/*.png` images is required for that path.

---

## How to test from the UI

See the walkthrough in the project conversation; in short:

- **Book app** (`http://localhost:3000`): use the category dropdown, the search box, the
  `+`/`-` stock buttons, the Edit dialog, and the dashboard stat cards.
- **Scribe app** (`http://localhost:8000`): upload `python/samples/note_lines.png`, click
  **Process**, confirm text appears, then click **Download text** and confirm the file is
  not empty.

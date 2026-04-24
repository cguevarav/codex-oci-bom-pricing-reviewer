---
name: oci-bom-pricing-reviewer
description: >
  Reviews Oracle Cloud Infrastructure (OCI) pricing configurations built in
  Excel spreadsheets. Use this skill whenever a user uploads or mentions an
  Excel file containing OCI prices, SKUs, cost estimates, or cloud service
  configurations. Trigger on phrases like "review my OCI pricing", "check this
  OCI estimate", "validate my cloud costs", "verify SKUs", or any time an xlsx
  file is shared in the context of Oracle Cloud pricing. The skill reads the
  Excel file, fetches live OCI prices from Oracle's public API, and produces a
  structured review report flagging SKU errors, price mismatches, unit mistakes,
  missing paired SKUs, and formula/math errors.
---

# OCI BoM/Pricing Reviewer Skill (Codex)

## Overview

This skill reviews human-authored Excel pricing sheets against **live OCI list
prices** (reachability to Oracle's Pricing API is mandatory). Sheets may be preferently in English otherwise a mix of languages. The format varies by
author — the skill must interpret the layout intelligently.

**Checks performed:**
1. SKU/part numbers exist in OCI (valid, not retired)
2. Unit prices match current OCI list prices (flag ALL differences, even small)
3. Units of measure are correct (OCPU/hr, GB/month, etc.)
4. Missing paired SKUs (e.g. E4 OCPU without its Memory SKU)
5. Row-level math errors (qty × unit price ≠ line total)
6. Sheet-level formula errors (subtotals, grand totals)

---

## Step 1 — Preflight: Verify the OCI Price API Is Reachable

**This skill requires live prices. Do not proceed without a successful
preflight.** Run this check *before* reading the Excel file so the user isn't
left waiting on a parse that can't be verified.

```python
import requests

OCI_API = "https://apexapps.oracle.com/pls/apex/cetools/api/v1/products/"

def preflight(currency="USD"):
    resp = requests.get(OCI_API, params={"currencyCode": currency, "limit": 1}, timeout=10)
    resp.raise_for_status()
    items = resp.json().get("items", [])
    if not items:
        raise RuntimeError("API reachable but returned no items")
    return True
```

**If preflight fails** (timeout, non-2xx, empty response, DNS error), stop
immediately and tell the user:

> ❌ Cannot run the OCI pricing review. The Oracle public price API
> (`apexapps.oracle.com/pls/apex/cetools/api/v1/products/`) is unreachable:
> [brief error]. This skill requires live prices and has no offline fallback.
> Please retry when the API is available, or check network/proxy access.

Do **not** attempt the review from cached or reference data. Do not read the
Excel file. Exit the skill.

---

## Step 2 — Read the Excel File

Use Codex's spreadsheet runtime when available. Read the workbook from the
local file path provided by the user in the current workspace, then use
openpyxl to parse all sheets. Look for columns containing:
- Part numbers / SKU codes (e.g. "B91628", "Part Number", "SKU", "Parte")
- Service/product descriptions (e.g. "Description", "Servicio", "Producto")
- Quantities (e.g. "Qty", "Quantity", "Cantidad", "Units")
- Unit prices (e.g. "Unit Price", "Precio Unitario", "Price/Unit")
- Line totals (e.g. "Total", "Amount", "Importe", "Subtotal")
- Units of measure (e.g. "Unit", "Unidad", "UOM")

**The format varies — do not assume fixed column positions.** Use fuzzy
matching on header names, considering both English and Spanish. If no headers
are found, scan for rows that look like pricing data (part number pattern +
numeric values).

OCI part numbers follow the pattern: letter(s) + digits, e.g. `B91628`,
`B88317`, `B93541`. Use regex `^[A-Z]\d{4,6}$` as a signal.

---

## Step 3 — Fetch Live Prices for the SKUs in the BoM (targeted)

Fetch **only** the part numbers that actually appear in the workbook. A real
BoM has tens of line items, not thousands — there is no reason to download
the entire catalog. The OCI API accepts a `partNumber` query parameter, so
one small call per SKU is the right primitive. Run those calls concurrently
with a small thread pool: the API is public and unauthenticated, so we stay
conservative to avoid being rate-limited or noisy.

```python
from concurrent.futures import ThreadPoolExecutor, as_completed

MAX_WORKERS = 8   # conservative: < 10, plays nice with Oracle's public API

def _fetch_one(pn, currency):
    resp = requests.get(
        OCI_API,
        params={"currencyCode": currency, "partNumber": pn},
        timeout=15,
    )
    resp.raise_for_status()
    items = resp.json().get("items", [])
    return pn, (items[0] if items else None)

def fetch_oci_prices_for(part_numbers, currency="USD"):
    """Fetch only the SKUs collected from the BoM, in parallel.
    Returns (found, missing) where found maps partNumber → API item."""
    unique = sorted(set(part_numbers))
    found, missing = {}, []
    with ThreadPoolExecutor(max_workers=MAX_WORKERS) as pool:
        futures = {pool.submit(_fetch_one, pn, currency): pn for pn in unique}
        for fut in as_completed(futures):
            pn, item = fut.result()   # exceptions propagate → abort the run
            if item is not None:
                found[pn] = item
            else:
                missing.append(pn)
    return found, missing
```

Collect the `part_numbers` set by scanning every sheet parsed in Step 2 for
cells matching the OCI SKU pattern (`^[A-Z]\d{4,6}$`) **or** values in a
column whose header fuzzy-matches "SKU" / "Part Number" / "Parte".

A SKU in `missing` is a Check A failure — the API returned no item for it
(retired or mistyped). If any worker raises (network error, non-2xx — **not**
an empty-result case, which is a real finding), let the exception propagate
and surface the same "cannot verify live prices" message as Step 1. Do not
review with a partial result set.

Each returned item includes:
- `partNumber` — the SKU (e.g. "B91628")
- `displayName` — human-readable name
- `metricName` — billing unit (e.g. "OCPU_PER_HOUR", "STORAGE_GIGABYTE_PER_MONTH")
- `currencyCode`
- `prices` — list of tiers; for list price use the entry where `model == "PAY_AS_YOU_GO"` and take `value`

Build the primary lookup dict from `found`:

```python
by_part_number = dict(found)  # partNumber → API item
```

**Fallback for rows with no part number.** If the workbook has priced rows
where no SKU could be extracted (description-only rows), the `partNumber=`
query cannot resolve them. In that case — and only then — fall back to a
single bulk catalog fetch *scoped to the service family hinted by the
description* (e.g. `?serviceCategory=Compute`) so the fuzzy-match in Step 4
has something to match against. If the sheet has part numbers on every
priced row (the common case), skip the fallback entirely.

---

## Step 4 — Match Sheet Rows to OCI SKUs

For each pricing row in the sheet:

1. **Row has a part number**: look up in `by_part_number` (the dict built
   from the targeted fetch in Step 3).
   - Present → proceed to checks
   - SKU appeared in the BoM but the API returned no item → ❌ INVALID SKU
     (retired or mistyped)

2. **Row has no part number**: fuzzy-match the description against the
   scoped fallback catalog fetched in Step 3 (use `difflib.get_close_matches`
   or `rapidfuzz`).
   - Confidence ≥ 0.85 → treat as matched, note it was inferred
   - Confidence < 0.85 → ⚠️ UNRESOLVED — cannot verify

3. **Never skip a row silently.** Always flag rows that cannot be matched.

**Tracking data for annotated output:** as you parse the sheet, keep the
exact cell coordinates (`sheet_name`, `row`, `column_letter`) for every
value you inspect — SKU, qty, unit price, total, hours, currency. Every
finding raised in Step 5 must carry the coordinates of the cell(s) it refers
to, so Step 6 can paint them in the annotated workbook.

---

## Step 5 — Run All Checks

### Check A — SKU Validity
- Part number not found in live API → ❌ INVALID SKU (retired or mistyped)
- Suggest the closest valid SKU if edit distance ≤ 2

### Check B — Price Mismatch
- Compare sheet unit price to API `PAY_AS_YOU_GO` price value
- Flag **all** differences, even $0.001:
  `⚠️ PRICE MISMATCH: Sheet=$X.XXXX | OCI List=$Y.YYYY (diff: ±Z%)`
- If the sheet uses a different currency than the fetched data, flag as:
  `⚠️ CURRENCY MISMATCH: sheet uses [CUR], prices fetched in USD — cannot compare directly`

### Check C — Unit of Measure
Compare the sheet's stated unit to the API `metricName`. Common mappings:

| metricName                    | Expected label variants                   |
|-------------------------------|-------------------------------------------|
| OCPU_PER_HOUR                 | OCPU/hr, OCPU per hour, OCPU hora         |
| GIGABYTE_PER_MONTH            | GB/month, GB/mes, GB/mo                   |
| GIGABYTE_PER_HOUR             | GB/hr, GB/hora                            |
| INSTANCE_PER_HOUR             | instance/hr, instancia/hora               |
| NODE_PER_HOUR                 | node/hr, nodo/hora                        |
| REQUESTS_PER_THOUSAND         | 1K requests, 1000 requests, solicitudes   |
| UNIT_PER_HOUR                 | unit/hr, unidad/hora                      |

Mismatch → ⚠️ UNIT MISMATCH: Sheet says "[X]" but OCI metric is [metricName]

### Check D — Missing Paired SKUs
Load pairing rules from `references/oci-sku-pairing-rules.md`.

Key rules:
- Any **E3 / E4 / E5 / X9 / A1 / A2 OCPU SKU** must have a corresponding
  **Memory SKU** for the same shape family in the same configuration block
- Bare metal GPU shapes include memory — no separate memory SKU needed
- Object Storage, Networking, and most PaaS services are single-SKU

If an OCPU SKU is present without its memory counterpart → ❌ MISSING PAIR

### Check E — OCPU : Memory Ratio
For every OCPU/Memory pair found in Check D, compute the ratio of GB of RAM
per OCPU for that configuration block:

```python
gb_per_ocpu = memory_qty / ocpu_qty
```

Supported flex range for standard shapes is **1:4 to 1:16** (GB per OCPU).
Anything outside this range is almost always a misconfiguration:

- `gb_per_ocpu < 4`  → ⚠️ RATIO TOO LOW: only X.X GB per OCPU (min 4)
- `gb_per_ocpu > 16` → ⚠️ RATIO TOO HIGH: X.X GB per OCPU (max 16)

The warning must reference **both** rows involved (the OCPU row and the
Memory row) so the engineer knows where to correct.

Exceptions — do not raise this warning for:
- Bare metal GPU families (`BM.GPU.*`) — fixed ratios, memory is included
- Dense IO shapes (`*.DenseIO.*`) — fixed non-flex ratios
- Exadata, DB bare metal, and any non-flex compute shape

### Check F — Row Math
```python
expected = round(qty * unit_price, 4)
if abs(expected - sheet_total) > 0.01:
    # flag ❌ MATH ERROR: expected $X.XX, sheet shows $Y.YY
```

### Check G — Subtotals / Grand Total
Sum all line totals per section, compare to subtotal/total cells.
Flag discrepancies > $0.01 as ❌ TOTAL ERROR.

### Check H — Monthly Hours Consistency (sheet-level)
Engineers commonly multiply hourly OCI prices by a monthly hour count. Only
three values are standard:

- **720** = 24 × 30 (rough month)
- **730** = OCI's official billing month (what the API uses)
- **744** = 24 × 31 (long month)

Determine the hours value each priced row uses. Sources, in priority order:
1. An explicit column named "Hours", "Horas", "Hrs/Mes", "Hours/Month", etc.
2. Implied value: for any row with an `*_PER_HOUR` metric and a monthly
   total, compute `implied_hours = monthly_total / (qty × unit_price)` and
   round to the nearest integer.

**Per-row rule:** any row whose hours value is **not** in {720, 730, 744}
→ ❌ NON-STANDARD HOURS: row uses X hours/month (expected 720, 730, or 744)

**Sheet-level rule:** if more than one distinct value from {720, 730, 744}
appears within the same sheet → ❌ MIXED MONTHLY HOURS, and include in the
report a summary table of every priced row and the hours value it uses, so
the correct vs. incorrect rows can be spotted immediately:

| Row | SKU    | Hours used | Source          |
|-----|--------|------------|-----------------|
| 5   | B91628 | 730        | "Horas/Mes" col |
| 12  | B91629 | 720        | implied         |
| 18  | B88316 | 744        | implied         |

If the sheet is consistent (a single value from the allowed set), note the
value chosen in the report's Notes section.

### Check I — Currency Consistency

Detect the currency of every priced cell using, in order:
1. An explicit "Currency" / "Moneda" column
2. Currency symbols in values (`$`, `€`, `£`, `MX$`, `R$`, `¥`) or ISO codes
   in cells or headers (USD, EUR, MXN, BRL, GBP, JPY, CAD, AUD, …)
3. Cell number-format currency codes from openpyxl

**Per-sheet rule:** determine the dominant currency (used by ≥80% of priced
rows). If two or more currencies appear in the same sheet
→ ❌ MIXED CURRENCY IN SHEET, list the offending rows with the currency each
uses.

**Workbook-level rule:** if different sheets have different dominant
currencies → ⚠️ CURRENCY VARIES BY SHEET, list each sheet with its dominant
currency and the first priced row, so the engineer can jump directly to the
source.

After detection, if the dominant workbook currency is supported by the OCI
API (USD, EUR, GBP, MXN, BRL, JPY, CAD, AUD, CHF, INR, and others — check
the API), **re-fetch prices in that currency** and compare directly. If the
currency is not supported, keep USD prices and emit
⚠️ CURRENCY NOT SUPPORTED BY API so Check B's price comparisons are clearly
flagged as non-authoritative.

---

## Step 6 — Output: Chat Report + Annotated Workbook

Produce **both** outputs every run. The chat report is for reading; the
annotated workbook is for fixing. Engineers choose whichever fits their flow.

### 6a — Chat report

Produce the review directly in the conversation as structured text.

```
## OCI Pricing Review — [filename] — [review date]
Prices verified against: OCI Public Price List (USD), fetched [timestamp]

### Summary
- Total rows reviewed: N
- ✅ Clean rows: N
- ❌ Errors: N   (must fix)
- ⚠️  Warnings: N  (review recommended)

---
### ❌ Errors

**Row 12 — B91628 (Compute Standard E4 OCPU)**
- PRICE MISMATCH: Sheet=$0.0252/OCPU-hr | OCI List=$0.0255/OCPU-hr (-1.2%)

**Row 13 — [no part number] matched to B91629 (confidence 91%)**
- MISSING PAIR: E4 OCPU (B91628) present but no E4 Memory SKU found

---
### ⚠️ Warnings

**Row 7 — B88317 (Block Volume Storage)**
- UNIT MISMATCH: Sheet says "GB/month" but OCI metric is GIGABYTE_PER_HOUR

---
### ✅ Clean Rows
Rows 3, 5, 8, 9, 14 — SKU valid, price correct, units correct, math correct.

---
### Notes
- Sheet currency appears to be [USD/MXN/other].
- [Any structural observations about the sheet.]
```

Keep the report concise. Always show OCI list price alongside sheet price so
the user can see the exact discrepancy.

### 6b — Annotated workbook

Alongside the chat report, produce an annotated copy of the input workbook
and save it next to the source file as:
`<source-directory>/<original-filename>-reviewed.xlsx`.

Using openpyxl, open a fresh copy of the original workbook and for every
finding emitted in Step 5:

- **Error (❌)** → fill the offending cell(s) with **red** (`FFC7CE`) and
  attach a cell comment with the finding text
- **Warning (⚠️)** → fill the offending cell(s) with **yellow** (`FFEB9C`)
  and attach a cell comment with the finding text
- If a single row has multiple findings, concatenate them into one comment
  per cell, newline-separated
- If a finding spans multiple cells (e.g. Check D missing-pair, Check E
  ratio out of range, Check H mixed hours), annotate **every** involved cell

```python
from openpyxl.styles import PatternFill
from openpyxl.comments import Comment

RED    = PatternFill(start_color="FFC7CE", end_color="FFC7CE", fill_type="solid")
YELLOW = PatternFill(start_color="FFEB9C", end_color="FFEB9C", fill_type="solid")

for finding in findings:
    ws = wb[finding.sheet]
    cell = ws[f"{finding.col}{finding.row}"]
    cell.fill = RED if finding.severity == "error" else YELLOW
    existing = cell.comment.text if cell.comment else ""
    cell.comment = Comment((existing + "\n" + finding.message).strip(), "OCI Review")
```

At the end, add a summary sheet named `__Review Summary__` at the front of
the workbook containing the same ❌/⚠️ tables from the chat report, so the
workbook is self-contained.

Mention the output path explicitly in the chat report as an absolute path:

> Annotated workbook: `/absolute/path/to/<filename>-reviewed.xlsx`

---

## Reference Files

- `references/oci-sku-pairing-rules.md` — Rules for which SKUs must appear together

**No offline price reference is used.** This skill validates only against live
prices from the OCI public API. If the API is unreachable, the review does not
run (see Step 1).

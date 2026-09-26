# marg-csv-import

![Python](https://img.shields.io/badge/python-3.8%2B-blue)
![License: MIT](https://img.shields.io/badge/license-MIT-green)

Nesayo is billing, stock and compliance software for Indian retail pharmacies that runs in the browser on any phone or computer ([nesayo.com](https://nesayo.com/?utm_source=github&utm_medium=readme&utm_campaign=marg_csv_import), [comparison with Marg ERP](https://nesayo.com/alternatives/marg-erp?utm_source=github&utm_medium=readme&utm_campaign=marg_csv_import)). This repository is a tiny, dependency-free Python library, maintained by the team behind it, that parses stock/inventory CSV exports from Indian pharmacy desktop software — **Marg ERP**, **eVitalRx**, **Vyapar** and similar — into clean, normalized Python dicts.

## Why

If you are migrating a pharmacy off desktop billing software, the first wall
you hit is the stock export: every tool names its columns differently
(`Item Name` vs `Product` vs `Medicine Name`), and expiry dates arrive in at
least five different formats (`07/27`, `Jul-27`, `31.07.2027`, `2027-07-31`,
`07-2027`). This library absorbs that mess so your import code sees one
consistent shape:

```python
{
    "name": "Paracetamol 500mg",
    "batch": "B123",
    "expiry_iso": "2027-07-31",
    "mrp": 25.5,
    "qty": 100.0,
    "hsn": "3004",
    "gst_percent": 5.0,
}
```

It is deliberately forgiving: title lines before the header are skipped,
blank and nameless footer rows are dropped, currency symbols and thousand
separators in numeric cells are handled, and malformed rows never crash the
import.

## Install

Stdlib only — no dependencies.

Clone this repository (use the "Code" button above for the URL), then:

```bash
cd marg-csv-import
pip install .
```

## Usage

### 1. Parse a whole export file

```python
from marg_csv_import import parse_csv

for item in parse_csv("marg_stock_export.csv"):
    print(item["name"], item["batch"], item["expiry_iso"], item["qty"])
```

### 2. Parse an in-memory upload (e.g. from a web request)

```python
import io
from marg_csv_import import parse_csv

uploaded_bytes = request_body  # bytes from your web framework
rows = list(parse_csv(io.StringIO(uploaded_bytes.decode("utf-8-sig"))))
print(f"Imported {len(rows)} stock items")
```

### 3. Use the pieces directly

```python
from marg_csv_import import detect_columns, parse_expiry

columns = detect_columns(["Medicine Name", "Batch No.", "Exp", "MRP", "Stock"])
# {'name': 0, 'batch': 1, 'expiry': 2, 'mrp': 3, 'qty': 4}

parse_expiry("Jul-27")      # '2027-07-31'  (last day of the month)
parse_expiry("31.07.2027")  # '2027-07-31'
parse_expiry("garbage")     # None
```

## Supported column aliases

Header matching is case-insensitive and ignores punctuation
(`Batch No.` == `batch_no` == `BATCH NO`).

| Canonical key | Recognized headers |
|---|---|
| `name` | item name, product, product name, medicine name, medicine, item, drug name, description, item description |
| `batch` | batch, batch no, batch number, batchno, batch code |
| `expiry` | exp, expiry, exp date, expiry date, exp dt, expdate |
| `mrp` | mrp, rate, price, sale rate, mrp rate, sale price |
| `qty` | qty, quantity, stock, stock qty, current stock, closing stock, balance qty |
| `hsn` | hsn, hsn code, hsn/sac, hsn no |
| `gst_percent` | gst, tax, gst %, gst rate, tax rate, tax %, igst, gst slab |

Note: `rate`, `sale rate` and `price` are mapped to `mrp`, and `tax` is
mapped to `gst_percent`. Some exports use "Rate" for a purchase or wholesale
price rather than MRP, and "Tax" for a tax amount rather than a percentage —
check a sample of your specific export before trusting these two columns
without review.

## Supported expiry-date formats

A pharmacy pack prints a MONTH, never a day ("EXP 08/2026"), and stock
printed that way is legally sellable through the LAST day of that month —
31 August, not 1 August. Every format below resolves to the last day of
the month, except a strict `YYYY-MM-DD` cell, which is assumed to already
be a real date from the source system and keeps its own day.

| Format | Example | Result |
|---|---|---|
| MM/YY | `07/27` | `2027-07-31` |
| MMM-YY / MMM YYYY | `Jul-27`, `DEC 2029` | `2027-07-31`, `2029-12-31` |
| DD.MM.YYYY | `31.07.2027` | `2027-07-31` (day dropped — see above) |
| ISO, month only | `2027-07` | `2027-07-31` |
| ISO, full date | `2027-07-15` | `2027-07-15` (day kept) |
| MM-YYYY | `07-2027`, `07/2027` | `2027-07-31` |

## CHANGELOG

See the repository's git commit history for exact dates. In order, oldest first:

- Initial release.
- `parse_expiry` resolved every input to the 1st of the month, which
  understates a pack's sellable life by up to 30 days. Now resolves to the
  last day of the month (mirroring Nesayo's own expiry-parsing rule), except
  a strict `YYYY-MM-DD` cell, which keeps its own day. Fixed the GST-rate
  example in this README (the 12% slab it showed no longer exists) and the
  `Repository` URL in `pyproject.toml`, which pointed at an unrelated
  account. Removed an unverifiable claim about which pharmacy's export this
  was validated against.

## Notes

- This library only parses. What you do with H/H1 schedule handling, GST
  filing, or batch-level FEFO logic is up to your application.

## Who built this

Built by the team behind [Nesayo](https://nesayo.com/?utm_source=github&utm_medium=readme&utm_campaign=marg_csv_import) — billing, stock and compliance software for Indian retail pharmacies, free forever for billing. Try the free medicine tools at
[nesayo.com/tools](https://nesayo.com/tools/?utm_source=github&utm_medium=readme&utm_campaign=marg_csv_import).

Contact: hello@nesayo.com · Site: https://nesayo.com

## License

MIT — see [LICENSE](LICENSE).

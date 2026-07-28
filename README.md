# Reports Generator

A single-page, dependency-free web app for turning Excel workbooks into summary reports.
Everything is one file — `index.html` — with raw HTML, CSS and JavaScript. Excel parsing is
handled by [SheetJS](https://sheetjs.com/) loaded from a CDN. No build step, no server, no
data ever leaves the browser.

## Tabs

1. **Casuals Detailed Report**
2. **FTE Detailed Report**
3. **Casual Period Summary**
4. **FTE Period Summary**

Each tab is independent — it keeps its own uploaded file, preview and generated report.

## What each tab does

- **Upload** an Excel workbook (`.xlsx` / `.xls`) by drag & drop or by browsing. The first
  sheet is read.
- **Preview** the parsed data in a table showing the first 100 rows.
- **Generate report** — computes the summary described below.
- **Download enhanced report (.xlsx)** — exports a workbook with two sheets:
  - `Summary` — report metadata plus the per-column statistics
  - `Data` — the full source data

## Report logic (placeholder)

All four tabs currently run the same placeholder analysis, pending the real rules for each
report type:

- Overall stats: row count, column count, number of numeric fields.
- Per numeric column: `count`, `sum`, `average`, `min`, `max`.

A column is treated as numeric when it has at least one value and every non-blank value in
it is a finite number.

## Running it

Open `index.html` in a browser, or serve the folder:

```bash
python3 -m http.server 8000
```

then visit <http://localhost:8000>.

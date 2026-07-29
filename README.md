# Reports Generator

A single-page, dependency-free web app that turns raw Excel timesheets into clean,
payroll-ready reports. Everything is one file — `index.html` — with raw HTML, CSS and
JavaScript. No build step, no server, no data ever leaves the browser.

Libraries loaded from CDN:

- [SheetJS](https://sheetjs.com/) — reads `.xlsx` and the older `.xls` format
- [ExcelJS](https://github.com/exceljs/exceljs) — writes the styled workbook with live formulas

## Tabs

1. **Casuals Detailed Report** — fully implemented (see below)
2. **FTE Detailed Report** — generic placeholder summary
3. **Casual Period Summary** — generic placeholder summary
4. **FTE Period Summary** — generic placeholder summary

Each tab is independent: it keeps its own uploaded file, preview and generated report.

## Casuals Detailed Report

Upload the casual employees timesheet by drag & drop or by browsing. The first sheet is
read and cleaned automatically.

### Cleanup

- The report title block above the real header is removed — the header lands on row 1.
- Merged cells are flattened. Vertical merges are filled down so no value is lost;
  horizontal merges leave their padding cells empty so they can be dropped.
- Columns with no header and no data are removed.
- Blank filler rows are removed.
- Headers are trimmed and their line breaks collapsed.
- `Employee Code` cells shaped like `2497070512  Employee Name: SHOHAG MIAH` are split
  into separate **Employee Code** and **Employee Name** columns.
- Text dates such as `7/1/2026` become real Excel dates; `HH:MM` durations become decimal
  hours.

### Column mapping

The date column, the actual-hours column and the required hours per day (default **12**)
are detected automatically and shown as editable controls before the report is generated.

### Added columns

| Column | Type | Meaning |
| --- | --- | --- |
| Required Hours / Day | calculated | The daily target, `12` by default |
| Hours Completion % | formula | `actual hours ÷ required hours` |
| Monthly Salary | **you fill in** | Left empty on purpose |
| Days in Month | formula | `DAY(EOMONTH(date,0))` → 28/29/30/31 |
| Daily Salary | formula | `Monthly Salary ÷ Days in Month` |
| Payment for Day | formula | `Daily Salary × Hours Completion %` |
| Incentive (+) | **you fill in** | Added to the day's payment |
| Deduction (-) | **you fill in** | Subtracted from the day's payment |
| Adjustment Note | **you fill in** | Why the incentive or deduction was applied |
| Net Payable | formula | `Payment for Day + Incentive − Deduction` |

The formulas are written as real Excel formulas, so the whole chain recalculates the
moment a monthly salary, incentive or deduction is typed into the downloaded file.

A day with no recorded hours counts as 0% completion, which pays 0 for that day.

### Exported workbook

Two sheets, styled in **Aptos** with no merged cells:

- `Casuals Detailed Report` — the full table, frozen header and first two columns,
  autofilter, per-column number formats, and colour-coded headers:
  green for cleaned source data, teal for calculated columns, amber for the columns
  you fill in. Completion below 100% is highlighted red, 100% or above green.
- `Summary` — report metadata, attendance totals, the cleanup that was applied, and
  payroll totals that sum the detail sheet.

## Other tabs

The remaining three tabs run the same generic placeholder analysis until their own rules
are defined: row/column/numeric-field counts, plus `count`, `sum`, `average`, `min` and
`max` per numeric column.

## Running it

Open `index.html` in a browser, or serve the folder:

```bash
python3 -m http.server 8000
```

then visit <http://localhost:8000>.

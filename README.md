# Reports Generator

A single-page, dependency-free web app that turns raw Excel timesheets into clean,
payroll-ready reports. Everything is one file — `index.html` — with raw HTML, CSS and
JavaScript. No build step, no server, no data ever leaves the browser.

Libraries loaded from CDN:

- [SheetJS](https://sheetjs.com/) — reads `.xlsx` and the older `.xls` format
- [ExcelJS](https://github.com/exceljs/exceljs) — writes the styled workbook with live formulas

## Tabs

1. **Casuals Detailed Report** — timesheet cleanup + payroll columns (see below)
2. **FTE Detailed Report** — Report Merge: four system exports into one attendance file
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

The date column, the actual-hours column, the status column (which drives overtime) and
the required hours per day (default **12**) are detected automatically and shown as
editable controls before the report is generated.

### Added columns

| Column | Type | Meaning |
| --- | --- | --- |
| Cost Center | **you fill in** | Left empty for manual entry |
| Required Hours / Day | calculated | The daily target, `12` by default |
| Hours Completion % | formula | `actual hours ÷ required hours`, uncapped so you can see overtime days |
| Monthly Salary | **you fill in** | Left empty on purpose |
| Days in Month | formula | `DAY(EOMONTH(date,0))` → 28/29/30/31 |
| Daily Salary | formula | `Monthly Salary ÷ Days in Month` |
| Payment for Day | formula | `Daily Salary × MIN(Completion %, 100%)` — never more than one full day |
| Excess Above Cap | formula | `Daily Salary × MAX(Completion % − 100%, 0)` — what the cap trimmed off |
| Day Off? | formula | `Yes` / `No`, so you can see exactly which rows qualify for OT |
| OT Hours | formula | On a worked day off: `1.5 × MIN(Total Hours, Required Hours)` |
| Manual OT Hours | **you fill in** | Overrides OT Hours on any row when it is not empty |
| OT Pay | formula | `OT Hours × (Daily Salary ÷ Required Hours)` |
| Incentive (+) | **you fill in** | Added to the day's payment |
| Deduction (-) | **you fill in** | Subtracted from the day's payment |
| Adjustment Note | **you fill in** | Why the incentive or deduction was applied |
| Net Payable | formula | `Payment for Day + OT Pay + Incentive − Deduction` |

The formulas are written as real Excel formulas, so the whole chain recalculates the
moment a monthly salary, incentive or deduction is typed into the downloaded file.

A day with no recorded hours counts as 0% completion, which pays 0 for that day.

The `Cost Center` column is skipped when the uploaded sheet already has a cost
centre/center column of its own, so no duplicate is added.

### Overtime

OT is credited when the status column matches one of the day-off words — `off, holiday`
by default, so `Day Off`, `Weekly Off` and `Holiday` all qualify — **and** hours were
actually worked that day. The words are editable in the mapping card, which also lists
every status value found in the sheet and marks the ones that will earn OT, so an
unrecognised wording is visible before the report is generated.

The worked hours are capped at the daily requirement before the 1.5 multiplier, so a
14-hour day off pays `1.5 × 12 = 18` OT hours, not 21.

`Manual OT Hours` wins whenever it holds a value, on any row, day off or not.

### Blank cells and typed-in values

Empty source cells are written as genuinely blank cells, not empty strings. This matters:
a cell holding an empty string makes Excel treat the whole column as text, and the older
formulas used `N()`, which returns `0` for anything textual — including a number typed as
text. Every formula now guards with `ISNUMBER()` instead, so a value typed into a blank
`Total Hours` cell immediately flows through completion %, payment, OT and the net.

### Exported workbook

Two sheets, styled in **Aptos** with no merged cells:

- `Casuals Detailed Report` — the full table, frozen header and first two columns,
  autofilter, per-column number formats, and colour-coded headers:
  green for cleaned source data, teal for calculated columns, amber for the columns
  you fill in. Completion below 100% is highlighted red, 100% or above green.
- `Summary` — report metadata, attendance totals, the cleanup that was applied, and
  payroll totals that sum the detail sheet.

## FTE Detailed Report — Report Merge

Ported from the `AIC-Attendance` project (`src/domain/merge.ts` + `src/pages/Merge.tsx`).
Drop all four period exports at once; each one is recognised from its own header shape:

| Report | System | Role in the merge |
| --- | --- | --- |
| **TAS** *(required)* | HR Works | Main attendance, one row per employee per day. `Employee Code` is usually the OLD number. |
| **Car Gate** | Speca Time & Space | Access events. Older exports carry names only and are matched by name tokens; newer ones carry the employee number, which wins. |
| **Leaves** | Oracle Fusion | Approved absences, keyed by the NEW employee number. |
| **Active list** *(required)* | Oracle Fusion | The Old No. ↔ New No. ↔ Name bridge that ties the other three together. |

What the merge does:

- **Best punches win** — earliest IN and latest OUT across TAS and the gate. Total, late,
  early, shortage and regular hours are recomputed whenever a punch changes, overnight
  shifts included.
- **Absences get resolved** — a day TAS calls `Absent` becomes `Present`, `Missing In` or
  `Missing Out` when gate punches prove attendance.
- **Approved leave replaces absence** — every approved leave type, expanded day by day and
  clipped to the report period.
- **12-hour gate exports** — Speca sometimes prints times with no AM/PM. When no timestamp
  in the file reaches hour 13, each event gets two candidates and the day is resolved
  against the schedule by picking the most plausible combination.
- **Employees missing from TAS** — anyone in the Active list with gate or leave evidence
  inside the period gets rows marked `Not in TAS` instead of vanishing.
- **Nothing is dropped silently** — unmatched gate badges and un-appliable leaves are
  listed on screen and in the workbook with the reason.

The exported workbook has an `Attendance` sheet (the 23 TAS-design columns plus
`Emp No. (New)`, `Leave Type`, `IN Source`, `OUT Source`; amber = punch from the gate,
green = leave applied) and a `Summary & Exceptions` sheet.

## Other tabs

The remaining two tabs run the same generic placeholder analysis until their own rules
are defined: row/column/numeric-field counts, plus `count`, `sum`, `average`, `min` and
`max` per numeric column.

## Running it

Open `index.html` in a browser, or serve the folder:

```bash
python3 -m http.server 8000
```

then visit <http://localhost:8000>.

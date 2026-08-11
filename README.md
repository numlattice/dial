# dial (दीर्ध लघु) - Split gains into STCG & LTCG

Command line app to split a trade CSV into a short-term file and a long-term file, using the calendar anniversary of the entry date.

---

## Usage

### Linux

* Download [dial.AppImage](https://github.com/numlattice/dial/releases/download/v1.0.0.0/dial.AppImage).
* Make it executable - **Right-click→Properties→Allow executing file as program**, or by running `chmod +x dial.AppImage` in a terminal.
* Run in terminal:

```bash
./dial.AppImage <input.csv> [--leap=clamp|roll]
```

### Windows

* Download [dial.zip](https://github.com/numlattice/dial/releases/download/v1.0.0.0_win/dial.zip).
* Unzip and open terminal
* Change directory to unzipped folder
* Run in terminal:

```bash
dial.exe <path of input.csv> [--leap=clamp|roll]
```

### Required columns

`Entry Date`, `Exit Date`. The program stops with an error if either is missing. All other columns are ignored and carried through untouched.


---

## The rule

A row is **long term** if the exit date falls on or after the same day of the same month in the year following the entry date. Anything earlier is **short term**.

| Entry Date | Exit Date | Result |
|---|---|---|
| 2025-06-10 | 2026-06-09 | short term |
| 2025-06-10 | 2026-06-10 | **long term** — the anniversary itself counts |
| 2025-06-10 | 2026-06-11 | long term |
| 2025-01-31 | 2026-01-31 | long term |
| 2023-03-01 | 2024-02-29 | short term — a leap day inside the span does not help |
| 2020-01-01 | 2026-01-01 | long term |

This is a calendar comparison, not a day count, so a leap year falling inside the holding period never shifts the boundary.

### The leap-day case

An entry on 29 February has no matching date the following year (the year after a leap year is never itself a leap year). Two conventions are available:

| Option | 2024-02-29 entry becomes long term on | Effect |
|---|---|---|
| `--leap=clamp` *(default)* | 2025-02-28 | slightly more lenient |
| `--leap=roll` | 2025-03-01 | stricter, requires one more day held |

Pick whichever matches the convention you are required to follow; it affects only rows with a 29 February entry date.

---

## Output files

The suffix is inserted before the file extension, and the directory is preserved:

```
data/trades.csv
  -> data/trades_stcg.csv           exit before the anniversary
  -> data/trades_ltcg.csv           exit on or after the anniversary
  -> data/trades_unclassified.csv   only created if some row's date is unreadable
```

Every output file gets a copy of the original header row, all original columns, and rows in their original order.

The third file exists so that nothing is silently lost or misfiled. Rows land there when the entry or exit date is blank or unparseable. If your data is clean, the file is never created.

### Example

```console
$ ./dial trades.csv
trades_stcg.csv : 4 rows (short term)
trades_ltcg.csv : 7 rows (long term)
trades_unclassified.csv : 1 rows (date unreadable)
Warning: 1 row(s) have an Exit Date before the Entry Date.
```

The warning about a backwards row is written to stderr. Such rows are still classified (as short term), because the classification is unambiguous, but the warning tells you the source data has a problem worth checking.

---

## Input file expectations

**Format.** A comma-separated CSV whose **first row is the header**.

**Column names.** Columns are found by name, not by position, so **column order does not matter**. Matching is case-insensitive and ignores extra spaces, so `Entry Date`, `entry date` and `ENTRY  DATE` are all the same column. If no exact match is found, the program falls back to the first header that *contains* the wanted name.

**Date format.** Accepted in any of these forms, with or without a trailing time:

```
2026-04-01      2026/04/01      01-04-2026      01/04/2026      2026-04-01 15:30:00
```

The order is decided by the first number: anything above 31 must be the year, so `2026-04-01` is read as year-month-day and `01-04-2026` as day-month-year. **Day-month-year is assumed for ambiguous dates** — `05/06/2026` is 5 June, not 6 May.

**Notes.**
The parser follows RFC 4180, so the following all work correctly:

- fields wrapped in quotes: `"BETA CORP, LTD"` stays one field
- a doubled `""` inside a quoted field means one literal `"`
- newlines inside a quoted field
- Windows (CRLF) or Unix (LF) line endings
- a UTF-8 byte-order mark at the start of the file (Excel adds one)
- blank lines, which are skipped

---

## Exit codes

`0` on success. `1` on any of: missing arguments, an unknown option, a file that cannot be opened or written, an empty file, or a missing `Entry Date` or `Exit Date` column.

The warning about an exit date preceding an entry date is written to stderr and does **not** change the exit code — the program finishes and produces output.

---

## Troubleshooting

**"No \"Entry Date\" column found."** — The header row is not the first row, the file is not comma-separated (a semicolon or tab export will not parse), or the column is genuinely named something else. Open the file in a text editor and check the first line.

**Rows near the anniversary landed on the wrong side.** — Likely an ambiguous `dd/mm` vs `mm/dd` file. Anything where both numbers are 12 or below is read as day-month-year. Re-export as `YYYY-MM-DD` if you can.

**Row counts do not add up to the source.** — Check for a `_unclassified.csv`; it exists to make unreadable rows visible rather than silently dropping them.

---

## Related

Both output files keep the original header and all columns, so they can be fed straight into [timahi](https://github.com/numlattice/timahi) to produce quarterly totals, required for filing income tax returns:

```bash
./dial trades.csv
./timahi trades_stcg.csv 2026 report_stcg.txt
./timahi trades_ltcg.csv 2026 report_ltcg.txt
```

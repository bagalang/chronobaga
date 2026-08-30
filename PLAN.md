# chronobaga — plan

> Civil date/time package for Baga, modelled on
> [chronotope/chrono](https://github.com/chronotope/chrono) `NaiveDate`.
> Accounting dates without locale-guessing.

**Package name (sandak):** `chronobaga`  
**Product name:** chronoBaga  
**Status:** **P0** (2026-08-30) · next: P1 zones / PG DATE  
**Roadmap role:** sibling of bagaDecimal — dates for bagabuch (invoices, VAT, periods)

---

## Why this package

`std/time` is `clock_gettime` (unix milliseconds). Accounting needs a
**civil calendar date**: leap years, months, VAT periods, invoice print.

| Need | Why not unix `i64` / JS `Date` |
|------|-------------------------------|
| Invoice issue / tax event / due | A day on the calendar, not an instant |
| VAT period, accounting month | `YYYY-MM`, first/last day |
| Bulgarian print (ЗДДС чл. 92) | `dd.mm.yyyy` from a canonical date |
| No `30/08` vs `08/30` | Slash dates are ambiguous — rejected |

**chrono** solves this with `NaiveDate` / `NaiveTime` / `NaiveDateTime`
plus parse/format. We follow that shape so algorithms (Hinnant
`days_from_civil`) and test vectors port cleanly.

---

## Format policy (locked)

| Role | Format | Example |
|------|--------|---------|
| Storage, JSON, UBL, SAF-T, PG TEXT | ISO 8601 date | `2026-08-30` |
| Bank / compact / some NAP files | Financial basic | `20260830` |
| VAT + accounting month | ISO year-month | `2026-08` |
| Invoice / VAT **display** | Bulgarian | `30.08.2026` |
| Invoice long form | Bulgarian long | `30 август 2026 г.` |

`dt_parse` accepts **only** ISO and financial. `dt_parse_bg` /
`dt_parse_invoice` exist because OCR and printed invoices arrive as
`dd.mm.yyyy`. After parse, store `dt_to_iso`.

Rejected: `30/08/2026`, `08/30/2026`, `30-08-2026`, 2-digit years.

---

## Representation (locked for P0)

```
struct Date { year: i64, month: i64, day: i64 }      // chrono NaiveDate
struct Time { hour: i64, minute: i64, second: i64 }  // NaiveTime, no ns
struct DateTime { … date fields + time fields }      // NaiveDateTime UTC
struct Period { year: i64, month: i64 }              // YYYY-MM
```

- Proleptic Gregorian, years **1..9999**.
- Serial day 0 = 1970-01-01 (unix epoch day), Hinnant algorithms.
- No leap seconds. Weekday is ISO (Mon=1 … Sun=7).
- Fallible paths return `DtResult` / `TmResult` / `TsResult` / `PerResult`.

---

## Scope map vs chrono

| chrono area | chronobaga | P0 | P1 |
|-------------|------------|----|-----|
| `NaiveDate` ymd / ordinal / weekday | `src/cal.baga` | ✅ | |
| `parse` ISO | `src/parse.baga` | ✅ | |
| format ISO + financial | `src/format.baga` | ✅ | |
| `+ Duration` days/months/years | `src/ops.baga` | ✅ | |
| Bulgarian civil (not in chrono) | `src/bg.baga` | ✅ | |
| year-month period | `src/period.baga` | ✅ | |
| `NaiveTime` / `NaiveDateTime` | `src/clock.baga` | ✅ | |
| JSON string ISO | `src/json.baga` | ✅ | |
| `Utc::now` | `src/now.baga` `!Time` | ✅ | |
| `DateTime<FixedOffset>` | | | offset parse |
| `Europe/Sofia` (IANA) | | | gap T4 |
| Postgres `DATE` | | | text bind |
| `strftime` / format strings | | | subset |

Out of scope v1: IANA tz database, leap seconds, ISO week dates,
Julian calendar, locale-guessing parse.

---

## Implementation phases

### P0 — accounting date path (ship bar)

1. `Date` + `dt_ymd`, leap, days-in-month, unix days, weekday, ordinal.
2. Parse: ISO `YYYY-MM-DD`, financial `YYYYMMDD`.
3. Format: ISO, financial, Bulgarian `dd.mm.yyyy` + long form.
4. Ops: cmp, add_days / add_months (clamp) / add_years, diff_days.
5. Period: `YYYY-MM` / `YYYYMM`, start/end, in_period, quarter.
6. Time/DateTime ISO, unix convert, `dt_today` (`!Time`).
7. JSON: always `"2026-08-30"`.
8. Tests + smoke + invoice-date example.

**Exit:** `sandak build` + `tests/chrono_test.baga` green.

### P1 — robustness

- Fixed offset (`+02:00` / `+03:00`) on DateTime parse.
- Postgres `DATE` text bridge (`src/pg`), same idea as bagadecimal NUMERIC.
- `dt_parse` reject more ISO-looking junk (`2026-8-30` unpadded).
- Optional ISO week date.

### P2 — zones (optional)

- Europe/Sofia DST table or a bundled tz subset.
- Not required for invoice **dates** (civil, not instants).

---

## Language probes (expected gaps)

| ID | Symptom | Severity | Path |
|----|---------|----------|------|
| T1 | No `u32` / `NaiveDate` packed internals | Low | ymd i64 fields |
| T2 | No operator overloading — `dt_add_days` not `d + 1` | Low | API naming |
| T3 | No `Option`/`Result` — ok/err structs | Medium | L3 when ready |
| T4 | No IANA tz / DST | Medium | P2; invoices are civil dates |
| T5 | `strftime` / format strings | Low | named functions |
| T6 | `clock_gettime` is UTC, not Sofia | Low | `dt_today` is UTC day |

Record hits in `gaps.md`.

---

## Architecture

```
app-product/chronobaga/
├── sandak.toml
├── README.md
├── PLAN.md
├── gaps.md
├── src/
│   ├── chrono.baga     # facade
│   ├── types.baga
│   ├── cal.baga
│   ├── parse.baga
│   ├── format.baga
│   ├── ops.baga
│   ├── bg.baga
│   ├── period.baga
│   ├── clock.baga
│   ├── json.baga
│   └── now.baga
├── examples/invoice_date.baga
└── tests/smoke.baga
```

---

## Success criteria (P0)

1. `cd app-product/chronobaga && sandak build` OK.
2. `dt_parse("2026-08-30")` / `dt_parse("20260830")` → same Date.
3. `dt_to_bg` → `30.08.2026`; `dt_to_period` → `2026-08`.
4. Leap / invalid day / add_months clamp goldens.
5. `gaps.md` lists T1–T6.

---

## Non-goals (explicit)

- Replacing `std/time` (monotonic / unix ms stay there).
- Guessing `01/02/03` locale.
- Shipping bagabuch domain (document types, ППДДС cells) inside this package.

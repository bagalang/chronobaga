# chronobaga — design notes

## Mapping from chrono

| chrono | chronobaga |
|--------|------------|
| `NaiveDate` | `Date { year, month, day }` |
| `NaiveDate::from_ymd_opt` | `dt_ymd` → `DtResult` |
| `NaiveDate::parse_from_str` / `%Y-%m-%d` | `dt_parse_iso` |
| `%Y%m%d` | `dt_parse_financial` / `dt_to_financial` |
| `%d.%m.%Y` | `dt_to_bg` / `dt_parse_bg` |
| `Datelike::weekday` | `dt_weekday` ISO Mon=1 |
| `checked_add_signed(Days)` | `dt_add_days` |
| `checked_add_months` | `dt_add_months` (clamp last day) |
| `num_days_from_ce` | `dt_num_days_from_ce` |
| `NaiveTime` | `Time` (second resolution) |
| `NaiveDateTime` | `DateTime` (UTC civil) |
| `Utc::now` | `dt_today` / `ts_now` (`!Time`) |

## Serial day

Howard Hinnant `days_from_civil` / `civil_from_days`. Unix day 0 =
1970-01-01. Rata Die / chrono `num_days_from_ce` = unix day + 719163
(0001-01-01 is day 1).

Years 1..9999 keep the era arithmetic non-negative under Baga/C
truncating division.

## Format policy

Wire and storage are **unambiguous**:

- ISO extended date: `YYYY-MM-DD`
- ISO basic / financial: `YYYYMMDD`
- ISO year-month: `YYYY-MM`

Bulgarian `dd.mm.yyyy` is a **presentation** of an already-valid `Date`.
Parsing it is allowed only through `dt_parse_bg` / `dt_parse_invoice`
because incoming invoices and OCR use it. Callers must write ISO back
to the database.

Slash dates are refused so `01/02/2026` cannot silently flip day/month.

## Month arithmetic

chrono `Months` on `NaiveDate` saturates the day to the last valid day
of the target month. We do the same: useful for due dates and
depreciation periods, honest for “31 January + 1 month”.

## Timezones

P0 DateTime is naive UTC (unix seconds). Bulgaria EET/EEST is T4.
Invoice **dates** in ЗДДС are calendar dates; they do not need a zone.
`created_at TIMESTAMPTZ` stays a database instant.

## JSON

Same rule as bagadecimal money: a date on the wire is a **string**,
never a JSON number (unix ms would invite JS `Date` and DST bugs).

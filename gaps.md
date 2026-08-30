# chronobaga — language & product gaps

Probe log for chronobaga (chrono-style civil dates).

## Anticipated (from PLAN)

### T1 — no packed `NaiveDate` internals

**Symptom.** chrono packs the date; we store `{year, month, day}` i64.

**Workaround.** Hinnant serial days for arithmetic; ymd for display.

**Severity.** Low.

**Verdict.** Accept. Accounting always wants the fields.

### T2 — no operator overloading

**Symptom.** Cannot write `d + Days(1)`.

**Workaround.** `dt_add_days` / `dt_add_months` / `dt_cmp`.

**Severity.** Low.

**Verdict.** Accept; same as bagadecimal.

### T3 — no `Option` / `Result` (L3)

**Symptom.** chrono returns `Option<NaiveDate>`; we return `DtResult { ok, err, value }`.

**Workaround.** `ok == 1` checks. Invalid dates never sit in a `Date` from public constructors.

**Severity.** Medium.

**Verdict.** Stand-in until L3.

### T4 — no IANA timezones / DST

**Symptom.** No `Europe/Sofia`. EET/EEST (+2/+3) is not in P0.

**Workaround.** Invoice/VAT dates are **civil** (`NaiveDate`). `dt_today` is the UTC day from `clock_gettime`. Offset parse is P1.

**Severity.** Medium for timestamps; low for documents.

**Verdict.** Documented. Do not pretend Sofia local.

### T5 — no `strftime`

**Symptom.** chrono `format("%d.%m.%Y")`. We have named functions.

**Workaround.** `dt_to_iso` / `dt_to_financial` / `dt_to_bg` / `dt_to_bg_long`.

**Severity.** Low.

**Verdict.** YAGNI for P0. Three formats cover bagabuch.

### T6 — `dt_today` is UTC, not local

**Symptom.** Around midnight in Sofia the UTC date may be the previous day.

**Workaround.** Pass the civil date from the client (already ISO `YYYY-MM-DD` in bagabuch forms). Server `dt_today` is a fallback.

**Severity.** Low for invoices (user picks the date).

**Verdict.** Documented.

## Open (product)

### T7 — unpadded ISO `2026-8-30`

Rejected by design (must be `YYYY-MM-DD`). Financial is 8 digits, also padded.

### T8 — Postgres `DATE` / `TIMESTAMPTZ` binary

P1. Today bagabuch stores dates as TEXT ISO; `created_at` is TIMESTAMPTZ from the DB clock.

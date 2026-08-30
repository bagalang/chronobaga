# chronoBaga

**Civil dates and times** for Baga — фактури, ДДС периоди, данъчно събитие —
по модела на [chrono](https://github.com/chronotope/chrono) `NaiveDate`.

`std/time` остава unix/monotonic clock. Този пакет е календарният ден.

| | |
|--|--|
| **sandak** | `chronobaga` **0.1.0** |
| **Layout** | `src/` modules |
| **Storage / JSON** | ISO `YYYY-MM-DD` или финансов `YYYYMMDD` |
| **Печат ДДС / фактури** | български `dd.mm.yyyy` |
| **Plan** | [PLAN.md](PLAN.md) · [gaps.md](gaps.md) |

This repository is the package. The compiler and `std` stay in the baga
language monorepo. Check this tree out as `app-product/chronobaga` there
(git submodule) so path deps and `-I app-product` keep working.

## Checkout

Inside a baga language clone:

```bash
git submodule update --init app-product/chronobaga
# or, first time from a fresh baga tree without the submodule recorded:
git clone git@github.com:bagalang/chronobaga.git app-product/chronobaga
```

`sandak.toml` keeps `std = { path = "../../std" }`.
`tests/chrono_test.baga` stays in baga.

```bash
cd app-product/chronobaga && BAGA=../../baga ../../sandak build
cd ../..
./baga -I . -I app-product tests/chrono_test.baga
./baga -I . -I app-product app-product/chronobaga/tests/smoke.baga
```

## Формати (заключено)

| Къде | Формат | Пример |
|------|--------|--------|
| DB TEXT, JSON, UBL, SAF-T | ISO | `2026-08-30` |
| Банкови файлове, компактно | финансов | `20260830` |
| ДДС / счетоводен месец | ISO year-month | `2026-08` |
| Фактура, ДДС дневник (екран/печат) | български | `30.08.2026` |

`dt_parse` **не** приема `30/08/2026` нито `08/30/2026`. За OCR на българска
фактура: `dt_parse_invoice` (ISO ∪ финансов ∪ `dd.mm.yyyy`) → пази `dt_to_iso`.

```baga
import "chronobaga/src/chrono.baga"

let d = dt_parse("2026-08-30")
// d.value == dt_parse("20260830").value
dt_to_iso(d.value)         // "2026-08-30"
dt_to_financial(d.value)   // "20260830"
dt_to_bg(d.value)          // "30.08.2026"
dt_to_bg_long(d.value)     // "30 август 2026 г."
dt_to_period(d.value)      // "2026-08"
```

## Счетоводен пример

```baga
let issue = dt_parse("2026-08-30").value
let due = dt_add_days(issue, 14).value     // 2026-09-13
let month = dt_to_period(issue)            // "2026-08"
let last = dt_period_end(dt_period_of(issue)).value  // 2026-08-31
print(dt_to_bg(issue))                     // фактура: 30.08.2026
```

## API snapshot (P0)

| Area | Functions |
|------|-----------|
| Construct | `dt_ymd`, `dt_unix_epoch`, `dt_from_unix_days`, `dt_from_unix_ms` |
| Calendar | `dt_is_leap`, `dt_days_in_month`, `dt_ordinal`, `dt_weekday` (ISO 1–7), `dt_quarter` |
| Parse | `dt_parse` (ISO \| financial), `dt_parse_iso`, `dt_parse_financial`, `dt_parse_bg`, `dt_parse_invoice` |
| Format | `dt_to_iso`, `dt_to_financial`, `dt_to_bg`, `dt_to_bg_long`, `dt_to_bg_or` |
| Ops | `dt_cmp/eq/lt/le/gt/ge`, `dt_min/max`, `dt_add_days/months/years`, `dt_diff_days` |
| Period | `dt_period_parse`, `dt_period_to_iso/financial/bg`, `dt_period_start/end`, `dt_in_period`, `dt_period_add` |
| Time | `tm_hms`, `tm_parse`, `tm_to_iso`, `ts_parse_iso`, `ts_to_iso`, `ts_from_unix` |
| JSON | `dt_to_json` / `dt_from_json` — always `"2026-08-30"` |
| Now | `dt_today` / `ts_now` (`!Time`, UTC) |

Fallible paths return `DtResult { ok, err, value }` (и `TmResult` / `TsResult` / `PerResult`).

`add_months` **clamp**-ва деня (31 януари + 1 месец → 28/29 февруари).

## Gaps (език)

- няма `d + 1` → `dt_add_days`
- няма `Option` → `DtResult`
- няма IANA `Europe/Sofia` (T4) — фактурната дата е граждански ден
- `dt_today` е UTC ден, не София

## License

[MIT](LICENSE) — Copyright (c) 2026 Dim Gigov.

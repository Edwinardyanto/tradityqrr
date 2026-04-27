# Shared Date Range Selector — Revenue Harian + Sold Products

**Date:** 2026-04-27
**Status:** Done
**Deployed:** 2026-04-27
**Branch:** feat/shared-date-range-selector (frontend + backend)

## Files Changed

**Frontend (tradityqrr):**
- `admin/index.html:1067-1099` — replaced individual Revenue Harian period buttons with a shared selector; added `id="cc-income-title"` + `id="cc-mix-sub"` for dynamic titles; renamed donut panel "Best Selling Product" → "Sold Products"
- `admin/index.html` (script) — replaced `adminIncomePeriod` state + `setAdminIncomePeriod()` setter with a shared `_adminRange` object + `setAdminRange(days, offset, label, btn)` setter; added `_lastTransactions` cache; added `loadSoldProductsDonut()` and `renderSoldProductsDonut()`; updated `loadAdminIncomeChart()` to send `?days=&offset=`; replaced inline donut block in `renderCCDashboard()` with cache-and-call

**Backend (tradity-qrr-server):**
- `src/index.js:1783` — `/admin/stats/daily` now accepts `?offset=N` (0 = window ends today, 1 = ends yesterday); whitelist updated to `[1, 7, 30, 90]` (added 1, kept 90 for the Traffic chart's existing 90D button); SQL queries gained an exclusive upper bound `< $2` so single-day windows actually constrain a single day; pre-seed loop iterates from `from` forward instead of from `today` backward

## Range Options (UI)

| Button | days | offset | Window |
|---|---|---|---|
| Today | 1 | 0 | Calendar day = today |
| Yesterday | 1 | 1 | Calendar day = yesterday |
| 7D (default for 7-day) | 7 | 0 | 7 calendar days ending today |
| 30D (default) | 30 | 0 | 30 calendar days ending today |

The selector lives on the Revenue Harian panel header (replacing the old 7D/30D/90D buttons). One state object, one setter, both charts re-fetch + re-render on change.

## Backend Range Math

```
to     = today − offset                  // inclusive last day
toExcl = to + 1 day                      // exclusive upper bound (midnight)
from   = to − (days − 1)                 // inclusive first day
SQL:   created_at >= from AND created_at < toExcl
```

Pre-seed loop now iterates `from + i` for `i in [0, numDays)` so the keys line up with the same window as the SQL filter, regardless of offset.

## Decisions Made

1. **Whitelist `[1, 7, 30, 90]`, not `[1, 7, 30]`.** The Traffic chart at the top of the admin Dashboard tab still has its own independent 7D/30D/90D toggle that calls `/admin/stats/daily?days=90`. Dropping 90 would silently fall back to 30 there. Keeping 90 in the whitelist preserves backward compat — the new shared selector only ever sends 1/7/30, so dropping 90 from the *UI* on the Revenue panel is sufficient. Flagged as a follow-up if we want the Traffic chart to also adopt the shared selector.

2. **`?offset=N` rather than `?date=YYYY-MM-DD`.** Cleaner extension of the existing `?days=` pattern, no need to parse and validate date strings, and the math composes naturally with `days` for arbitrary windows. Bounded to `[0, 365]` to refuse pathological inputs.

3. **Cache transactions client-side.** `loadSoldProductsDonut()` re-fetches `/admin/transactions` on every range change for parity with the Revenue chart's "re-fetch and re-render" semantics. `_lastTransactions` is cached so the initial render via `renderCCDashboard()` doesn't double-fetch.

4. **Calendar-day boundary on both charts.** Both now use `(today − offset)` as the day-aligned end of the window — Revenue Harian via SQL, Sold Products via client-side `setHours(0,0,0,0)` + `setHours(23,59,59,999)`. This fixes the inconsistency where the donut used a rolling 30·24h window while Revenue used calendar days.

5. **Re-render the Revenue title text dynamically.** `setAdminRange()` updates `#cc-income-title` to `Revenue harian · {label}` so the panel always shows what range it's reflecting. The previously-fixed text "Revenue harian · 30D" became misleading on any other selection.

## Known Limitations

- **Server vs client timezone**: the server runs in UTC (Railway); the admin browser is WIB (UTC+7). The frontend computes `today` from `new Date()` (local = WIB), the backend from `new Date()` (server = UTC). For "Today" specifically, the client's window starts 7h earlier than the server's. Pre-existing — the income SQL already has `DATE(completed_at + INTERVAL '7 hours')` for bucketing but date *comparisons* still use raw UTC. A proper fix would shift all comparisons by `+7h` server-side or send absolute UTC bounds from the client. Out of scope here.

- **Sold Products is filtered client-side.** It still relies on `/admin/transactions` returning all rows (the LIMIT cap was removed in an earlier commit). For a much larger production transactions table, server-side aggregation would be more efficient. Not a problem at current volume.

## Verification

- [x] Backend: `numDays`/`offset` math traced through for all 4 selector options
- [x] Backend: pre-seed loop produces the correct window for `offset > 0`
- [x] Frontend: no stale references to `adminIncomePeriod` or `setAdminIncomePeriod`
- [x] Frontend: all new symbols (`setAdminRange`, `_adminRange`, `_lastTransactions`, `loadSoldProductsDonut`, `renderSoldProductsDonut`, `cc-income-title`, `cc-mix-sub`) wired consistently
- [ ] Smoke-test on staging admin once merged: click each range button, verify both charts re-fetch + re-render with matching windows
- [ ] Verify against staging seed data: 127 granted orders in last 30 days; "30D" donut total should equal 127 (plus any baseline granted from `seed:staging` within the window); "Today" should match `SELECT COUNT(*) FROM transactions WHERE tv_access_status='granted' AND created_at::date = CURRENT_DATE`

# Sold Products Donut — filter by payment_status

**Date:** 2026-04-27
**Status:** Done (awaiting staging review)
**Branch:** feat/fix-donut-payment-filter

## Files Changed

- `admin/index.html` — `renderSoldProductsDonut()` filter changed from `tv_access_status === 'granted'` to `payment_status IN ('paid', 'settled')`. Same predicate as the Revenue Harian chart's server-side query.

## Why

"Sold Products" should reflect everything the customer actually paid for, not only the subset where the downstream TradingView grant succeeded. A failed TV grant is still a sale — the customer paid; the product moved. Aligning the donut with the Revenue chart's payment-based predicate also means the two panels can never disagree on what counts as a "sale."

## Required Backend Change

`payment_status` lives on `orders`, not `transactions`. The companion backend change on the same branch (`tradity-qrr-server` → `feat/fix-donut-payment-filter`) adds a `LEFT JOIN orders` to the `/admin/transactions` SELECT and includes `o.payment_status` in the response. Without that backend change, this filter would silently match nothing.

## Decisions Made

- **Strict equality on the two known values, not `!== 'unpaid'`.** Future payment_status values (e.g., a hypothetical `'refunded'` or `'partial'`) shouldn't silently count as sales. Explicit allowlist matches the Revenue chart's pattern and keeps the predicate auditable.
- **Excludes NULL.** The LEFT JOIN can yield NULL `payment_status` for orphan transactions with no matching order. Those are not auditable as sales, so they're dropped.
- **No change to date-window filter.** Calendar-day boundary from the shared range selector still applies on top of the payment filter.

## Behavioural Diff vs Before

| Before (`tv_access_status === 'granted'`) | After (`payment_status IN ('paid', 'settled')`) |
|---|---|
| TV grant succeeded | Customer paid, regardless of TV grant outcome |
| Excludes failed grants | Includes failed grants (paid but TV didn't go through) |
| Excludes pending/processing grants | Includes them if payment is confirmed |
| Excludes unpaid orders (correctly, by accident) | Excludes unpaid orders (correctly, by design) |

On staging seed data:
- Before: 127 granted only
- After: 127 granted + 39 failed = 166 (the 46 pending in seed have `payment_status='unpaid'` so they're correctly excluded)

## Verification

- [x] Read current donut filter logic
- [x] Read Revenue chart filter (`src/index.js:1838` and `:1865`) — confirmed `IN ('paid', 'settled')`
- [x] Filter rewritten to match exactly
- [ ] Smoke-test on staging: donut total for "30D" should equal `SELECT COUNT(*) FROM orders WHERE payment_status IN ('paid','settled') AND created_at >= now() - interval '30 days'` (modulo timezone alignment, see Known Limitations in the shared-date-range spec)

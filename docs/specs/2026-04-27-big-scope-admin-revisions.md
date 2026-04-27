# Big-Scope Admin Revisions

**Date:** 2026-04-27
**Status:** Done (awaiting staging review)
**Branch:** feat/big-scope-admin-revisions (frontend + backend)

## Scope

Four independent admin-side changes shipped together on the same feat branch:

1. Conv Rate calculation — replace the broken transactions/pending formula with an orders-based `paid / total` ratio
2. Pending-order amount fallback — show product price when `orders.amount = 0`
3. Orders tab — client-side ASC/DESC sort on Created At and Paid At columns
4. Traffic dashboard — new `/admin/stats/traffic` endpoint + site-wide summary cards on the Traffic tab

## #1 — Conv Rate

**Old (buggy):**
```
conv_rate = txCount / (txCount + pendingCount) * 100
```
`pendingCount` was a *subset* of `txCount` (the txCount query had no status filter), so the denominator double-counted. With 100 paid + 20 failed transactions the formula returned 83.3%; the truth was 80%.

**New:**
```sql
SELECT COUNT(*) FILTER (WHERE completed_at IS NOT NULL) AS paid,
       COUNT(*) AS total
  FROM orders WHERE created_at >= today − 29d
```
```
conv_rate = paid / total * 100
```

Tracks the funnel of "form submitted → payment confirmed" within the 30-day window — both numerator and denominator scoped by `orders.created_at`. The underlying counts ship in the response as `conv_paid_orders` / `conv_total_orders` so the FE can render a meaningful caption (`{paid} / {total} orders`) instead of the previous `{N} pending`.

## #2 — Amount fallback for pending orders

`/admin/orders` now `LEFT JOIN`s `products` and uses:
```sql
COALESCE(NULLIF(o.amount, 0), p.price, 0) AS amount
```
…so unpaid orders that landed via `order.created` without a `pg_payment_info.amount` field show the product's catalog price instead of `Rp 0`. `processPaymentPaid()` still backfills the real amount on `orders.amount` when the payment webhook arrives — this fallback only affects how the row is *displayed* before that.

**Note:** The user's original spec said `LEFT JOIN products p ON p.name = o.product_name`. The actual column is `products.product_name`, not `products.name`; corrected in the SQL.

No frontend change needed — `orders[i].amount` is already consumed verbatim. Verified by tracing through `renderOrderRows` at admin/index.html:2977.

## #3 — Sortable date columns on the Orders tab

Two clickable date column headers, independent toggle ASC ↔ DESC, arrow indicator on the active sort column.

- **Created At** (always visible) — `<th>` made clickable with `onclick="setOrderSort('created_at')"` and an arrow span
- **Paid At** (optional column) — `applyColumnPrefs()` now special-cases `col.key === 'completed_at'` and adds the same affordance when the column is enabled

Single sort state `_orderSort = { col, dir }` (default `created_at` / `desc`). `_sortOrders()` runs every time `renderOrderRows` is called, so the sort persists across re-renders, column-toggle re-renders, and re-fetches (filter changes / pagination). Sort is applied to the currently-displayed page (server-side pagination is unchanged).

## #4 — Site Traffic summary

**New backend endpoint `GET /admin/stats/traffic`** ([src/index.js:2122-2178](../tradity-qrr-server/src/index.js#L2122-L2178)) — single query, conditional aggregates per window, returns:

```json
{
  "today":        { "homepage": N, "checkout": N, "paid": N, "unique_sessions": N },
  "last_7_days":  { …same shape… },
  "last_30_days": { …same shape… },
  "conv_funnel": {
    "homepage_to_checkout": "X.X%",
    "checkout_to_paid":     "X.X%"
  }
}
```

`form_fill` is intentionally absent — the page value is rejected at `/track` and the `affiliate_clicks` `CHECK` constraint forbids it; reporting it would always be 0.

**Frontend** — new "Site Traffic" panel (cc-panel) above the existing affiliate table on the Traffic tab. Three metric cards:

| Card | Value |
|---|---|
| Visitors | unique homepage sessions in the period |
| Checkout rate | `checkout / homepage * 100` |
| Paid rate | `paid / checkout * 100` |

Period selector reuses `cc-period` / `period-btn` styles for visual consistency with the dashboard's shared range selector. Buttons: Today / 7D / 30D (default 30D). Period changes re-render from cached data — no re-fetch.

## Decisions Made

- **Conv Rate inputs exposed as `conv_paid_orders` / `conv_total_orders`** so the FE caption is data-driven, not hardcoded. Old `pending_count` / `pending_amt` fields kept in the response unchanged — other UI may depend on them and the user didn't ask to remove them.
- **Amount fallback is read-only on the orders endpoint.** Did NOT modify `recordOrder()` to write a fallback amount on insert. Fallback at SELECT time is the safer change — it doesn't pollute the `orders` table with derived data, and `processPaymentPaid()` keeps owning the "real amount" backfill on payment.
- **Sort state singular, not per-column.** "Independently sortable" interpreted as "click any date column to make it the active sort" — clicking the other one switches active column to it. Not two separate persistent toggles. Simpler state, matches the typical sortable-table convention.
- **Traffic period selector uses the same Today/7D/30D the backend returns** — no Yesterday option here, no `?days=` `?offset=` flexibility. The new endpoint hardcodes the three windows so the frontend can't ask for arbitrary ranges. Tradeoff: lower flexibility, much simpler implementation; matches the user's spec exactly.

## Known Limitations

- **`form_fill` removed from Traffic columns?** No — the existing affiliate-attributed table still shows a `Form Fill` column ([admin/index.html:1492](../admin/index.html#L1492)) with a value sourced from `affiliate_clicks` where `page='form_fill'`. That column always renders `0` (page value is rejected at `/track`) — pre-existing dead column, out of scope for this branch. Flagged here so it can be cleaned up separately.

- **Staging traffic tables are empty** (`site_clicks` count = 0, `affiliate_clicks` count = 0). Nothing seeds them. Site Traffic cards will all show `0` / `0.0%` until either (a) real visits accumulate on staging, or (b) we add traffic-table seed data. Verifiable on production where real traffic exists.

- **`orders.completed_at` for pending orders is NULL.** The new conv_rate denominator uses `created_at`, so brand-new orders enter the denominator immediately — they pull the rate down until they're paid. This is correct cohort-conversion behaviour but the ratio will look low for any window that includes recent unpaid orders. Documented here so it doesn't read as a bug.

## Files Changed

**Frontend (`tradityqrr`):**
- `admin/index.html` — Conv Rate caption update; Orders thead sort affordance; sort helpers + integration in `renderOrderRows` / `applyColumnPrefs`; Site Traffic HTML panel; `loadTrafficStats` / `renderTrafficStats` / `setTrafficPeriod`; hook into `loadAdminTraffic`

**Backend (`tradity-qrr-server`):**
- `src/index.js` — `/admin/stats/hero` ordersFunnel query + new conv_rate formula + `conv_paid_orders` / `conv_total_orders` response fields; `/admin/orders` SELECT amends with `LEFT JOIN products` + `COALESCE` fallback; new `/admin/stats/traffic` endpoint

## Verification

- [x] Backend syntax checked, no diagnostics
- [x] Frontend wiring traced — all new IDs resolve, all new functions reachable
- [ ] Smoke-test on staging: open admin dashboard, verify Conv Rate shows `{paid}/{total}` caption
- [ ] Smoke-test on staging: open Orders tab, click Created At header — rows reorder, arrow flips
- [ ] Smoke-test on staging: enable Paid At optional column, click its header — same behaviour
- [ ] Smoke-test on staging: open Traffic tab, click Today/7D/30D — cards update (numbers will all be 0 on staging until traffic exists)
- [ ] Smoke-test on production after merge: Sold Products donut total + Revenue chart total still agree

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

---

## Follow-up (same branch) — Dashboard restructure + Traffic tab removal

Funnel chart + per-affiliate table moved off the Traffic tab. Tab deleted. One shared range selector now drives Revenue Harian, Sold Products, and Funnel Traffic.

### Layout (Dashboard)

```
[Hero stats]
[Alert tiles]
[Revenue Harian (2fr)  |  Sold Products (1fr)]   ← existing row
[Funnel Traffic (1fr)  |  Reserved placeholder (1fr)]   ← NEW row
[Recent orders + Top affiliates]
[Failed orders]
```

The reserved-right cc-panel uses a new `.cc-panel-empty` style (centered "Coming soon", muted, min-height matched to the chart's 240px). New `.cc-row-1-1` grid utility added for 50/50 splits; collapses to single column at the same 1100px breakpoint as the other cc-row variants.

### Funnel chart

Stacked bar, 4 layers per day — Kunjungan (purple #7F77DD, bottom) → Checkout (green #1D9E75) → Form Diisi (amber #EF9F27) → Paid (orange #D85A30, top). Same Chart.js instance pattern as Revenue Harian (`responsive`, `maintainAspectRatio:false`, indexed tooltip showing all 4 layers). Legend below the canvas mirrors Revenue Harian's totals strip ("Kunjungan · 1,000 · …").

Subtitle dynamically tracks the shared range label: "Last 30 days · Kunjungan → Checkout → Form Diisi → Paid".

### Backend

`GET /admin/stats/funnel?days=N&offset=N` ([src/index.js:2191-2293](../tradity-qrr-server/src/index.js#L2191-L2293)) — replaces the previous `/admin/stats/traffic` endpoint at the same location. Same SQL pattern (4 parallel daily aggregations, pre-seeded zero map for continuous x-axis), but two changes:

1. **Adds `offset` param** (whitelist 0-365) so the dashboard's "Yesterday" button works (`days=1, offset=1`). Window math mirrors `/admin/stats/daily`: `to = today - offset`, `from = to - (days - 1)`, `toExcl = to + 1`.
2. **Paid layer buckets by `DATE(completed_at + INTERVAL '7 hours')`** to match the Revenue Harian WIB shift, so funnel.paid totals reconcile with the revenue chart's transaction count for the same day.

Response shape:
```json
{
  "days":   [{ "date": "2026-04-01", "homepage": 45, "checkout": 30, "form_fill": 18, "paid": 9 }, …],
  "totals": { "homepage": 1000, "checkout": 719, "form_fill": 344, "paid": 169 }
}
```

`days` is the array (per user spec) — slightly awkward given `?days=N` is also a query param, but matches the requested contract.

### Affiliate Traffic table

Moved verbatim from the deleted page-traffic to the bottom of `#page-affiliates`, wrapped in a `cc-panel` with title "Affiliate Traffic" and dynamic subtitle ("Kunjungan, checkout, dan konversi per affiliate · Last 30 days"). Columns unchanged (Affiliate / Ref Code / Pixel / Kunjungan / Checkout / Form Fill / Terbayar). Non-Affiliate row preserved.

`loadAffiliateTrafficTable()` replaces `loadTrafficAffiliateTable()` — fetches `/admin/traffic?days={_adminRange.days}` (no offset; `/admin/traffic` doesn't accept it, that's a known gap). Mounted on `showPage('affiliates')` and re-runs on dashboard range change if the table is currently in the DOM.

### Shared range selector

`setAdminRange()` now also calls `loadAdminFunnelChart()` and (if mounted) `loadAffiliateTrafficTable()`. Three charts re-fetch and re-render together when the user clicks Today / Yesterday / 7D / 30D.

### Deletions

- `<div class="page" id="page-traffic">` — entire section gone, including the dead detail panel (`traffic-detail-view`, `td-*-visit/checkout/formfill/paid`, `td-trend-chart`) which was never reachable from the table.
- Sidebar nav Traffic link, mobile bottom-nav Traffic item, legacy `<div class="nav-tab" onclick="showPage('traffic')">Traffic</div>`.
- `showPage()` tabs map: `traffic:6` removed, `promo:7` → `promo:6` (zero-indexed `.nav-tab` lookup).
- `if (name === 'traffic') loadAdminTraffic();` branch.
- JS funcs: `setTrafficPeriod`, `loadTrafficCharts`, `loadTrafficFunnelChart`, `loadAdminTraffic`, `loadTrafficDetail`, `closeTrafficDetail`, `_renderTrafficRow` (replaced by `_renderAffTrafficRow`), and the `_trafficDays` / `_trafficChart` / `_trafficLabels` state.

### Decisions

- **New endpoint path, not in-place rename.** `/admin/stats/traffic` → `/admin/stats/funnel`. Justification: response shape changed (`daily` → `days`, `days` integer dropped), and the only consumer is the new dashboard chart. Renaming surfaces the contract change; staging will 404 fast if a stale FE is deployed.
- **Affiliate Traffic table inside Affiliates tab, not a separate Affiliates sub-section.** User asked for "bottom of Affiliates tab" — placed it after the existing list/tree views, inside `#page-affiliates` so it shares the tab's nav state and refresh button is implicitly the page nav.
- **Reserved placeholder is a real cc-panel, not a hidden div.** Empty card with "Coming soon" preserves the 1:1 layout immediately and signals intent to the next contributor. Markup ready for future content drop-in.
- **Funnel `paid` layer uses WIB bucket, others don't.** Site_clicks and orders.created_at stay on UTC DATE() — matches the existing `/admin/stats/daily` pattern. Only the paid bucket gets the `+ INTERVAL '7 hours'` so it lines up with Revenue Harian. Mixed but intentional.
- **Affiliate traffic table re-fetches on Dashboard range change only if mounted.** Avoids spurious requests when the user is on Dashboard and never visits Affiliates. Re-runs on `showPage('affiliates')` regardless.

### Files Changed (follow-up)

- `admin/index.html` — `.cc-row-1-1` + `.cc-panel-empty` CSS; new Funnel chart + reserved placeholder row; `loadAdminFunnelChart()`; `setAdminRange()` extended; `loadAdminCharts()` parallelized; old Traffic tab + JS removed; affiliate traffic table relocated; nav entries removed.
- `src/index.js` — `/admin/stats/traffic` → `/admin/stats/funnel`, adds `offset` param, response key renamed `daily` → `days`, paid bucket uses WIB shift.

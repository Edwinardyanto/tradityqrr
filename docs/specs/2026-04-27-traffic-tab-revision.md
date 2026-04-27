# Traffic tab revision — Funnel chart + per-affiliate orders + Non-Affiliate row

**Date:** 2026-04-27
**Status:** Done
**Deployed:** 2026-04-27
**Branch:** feat/big-scope-admin-revisions

## Files Changed

**Backend (`tradity-qrr-server`):**
- `src/index.js` — `/admin/stats/traffic` restructured to return daily series + totals (was 3 fixed buckets); accepts `?days=N` (whitelist [1, 7, 30]). `/admin/traffic` now LATERAL-joins `affiliate_clicks` (visits/checkouts) AND `orders` (form_fills/paid) per affiliate, accepts `?days=N`, and adds a `non_affiliate` field to the response.

**Frontend (`tradityqrr`):**
- `admin/index.html` — Funnel Traffic chart panel replaces the Site Traffic 3-card panel above the per-affiliate table; canvas#admin-chart-traffic moved out of the hidden legacy div on the dashboard onto the visible Traffic tab; affiliate table thead drops `CR%` + `Aksi` (now 7 cols); tbody renderer drops the matching cells, appends a synthetic Non-Affiliate row from `data.non_affiliate`. Legacy chart code (`adminChartTraffic`, `adminTrafficPeriod`, `setAdminTrafficPeriod`, `loadAdminTrafficChart`) removed since the hidden canvas is gone; `loadAdminCharts` simplified.

## Three items, one cross-cutting set of changes

### Item 1 — Move 4-layer stacked bar chart to the Traffic tab

The chart code already existed (`loadAdminTrafficChart`) but rendered into a `display:none` canvas inside the dashboard's "Hidden legacy targets" div — invisible. Moved canvas to a visible panel on `page-traffic` titled **Funnel Traffic** with:

- Range selector (Today / 7D / 30D) reusing `cc-period` styles
- Stacked layers (top→bottom): **Paid · Form Diisi · Checkout · Kunjungan**
- Inline legend with totals per layer

Data source switched from `/admin/stats/daily` to the rewritten `/admin/stats/traffic`:
- `homepage` / `checkout` ← `site_clicks` (page filter)
- `form_fill` ← `orders.created_at` within range (NEW — not `transactions` like the old chart)
- `paid` ← `orders.completed_at` within range (NEW — bucketed by completion date)

Range change re-fetches both the chart endpoint AND the affiliate table endpoint via `Promise.all` so the two views never disagree.

### Item 2 — Affiliate table: drop CR% / Aksi, source Form Fill + Terbayar from orders

`<th>CR%</th>` and `<th>Aksi</th>` removed from `thead`; tbody renderer no longer emits matching cells; `colspan` updated from 9 → 7. The `loadTrafficDetail` function and the detail-panel HTML are left in place (orphaned, no entry point) so they can be re-wired later if needed.

Backend `/admin/traffic` SQL switched from a single GROUP BY on `affiliate_clicks` to LATERAL joins so `affiliate_clicks` (visits, checkouts) and `orders` (form_fills, paid) can each be aggregated independently per affiliate without cartesian-product issues:

```sql
SELECT a.id, a.name, …,
       COALESCE(c_q.visits, 0)    AS visits,
       COALESCE(c_q.checkouts, 0) AS checkouts,
       COALESCE(o_q.form_fills, 0) AS form_fills,
       COALESCE(o_q.paid, 0)       AS paid
  FROM affiliates a
  LEFT JOIN affiliate_pixels ap ON ap.affiliate_id = a.id
  LEFT JOIN LATERAL (
    SELECT COUNT(*) FILTER (WHERE page = 'homepage') AS visits,
           COUNT(*) FILTER (WHERE page = 'checkout') AS checkouts
      FROM affiliate_clicks
     WHERE ref_code = a.ref_code AND created_at >= $1
  ) c_q ON TRUE
  LEFT JOIN LATERAL (
    SELECT COUNT(*) AS form_fills,
           COUNT(*) FILTER (WHERE completed_at IS NOT NULL) AS paid
      FROM orders
     WHERE ref_code = a.ref_code AND created_at >= $1
  ) o_q ON TRUE
 ORDER BY visits DESC, a.name ASC
```

### Item 3 — Non-Affiliate synthetic row

Backend computes a separate `non_affiliate` bucket:
- `visits` / `checkouts` — `site_clicks` rows where the session_id has zero attribution in `affiliate_clicks` (`NOT EXISTS` subquery)
- `form_fills` / `paid` — `orders WHERE ref_code IS NULL`

Returned as `{ affiliates: [...], non_affiliate: { visits, checkouts, form_fills, paid } }`.

Frontend renders this as a synthetic 11th row at the bottom of the table with:
- Affiliate column: "Non-Affiliate" (italic + subtle background tint to mark it as a roll-up, not a real affiliate)
- Ref Code: `—`
- Pixel: `—`
- Same number columns

## Live verification (against current staging seed)

Simulated the new SQL against the Railway staging DB (read-only, no endpoint deploy needed):

```
/admin/stats/traffic?days=30 totals:
  homepage  : 499        (1 of 500 seed sessions fell outside the 30d window)
  checkout  : 200        (matches seed config)
  form_fill : 126        (orders.created_at in window — 1 of 127 outside)
  paid      : 126        (orders.completed_at in window)

/admin/traffic affiliates (top 5 by visits):
  TRXC02 | Affiliator 2  | v:32 co:9  ff:4 p:4
  TRXC10 | Affiliator 10 | v:27 co:10 ff:3 p:3
  TRXC04 | Affiliator 4  | v:25 co:8  ff:2 p:2
  TRXC01 | Affiliator 1  | v:23 co:8  ff:3 p:3
  TRXC05 | Affiliator 5  | v:21 co:10 ff:6 p:6

non_affiliate:
  visits     : 294
  checkouts  : 123
  form_fills :  76
  paid       :  76
```

Cross-check: `Σ(affiliate.form_fills) + non_affiliate.form_fills = 50 + 76 = 126 = chart's form_fill total` ✓. Same for paid. The funnel attribution adds up.

## Decisions Made

- **Daily-array response shape** instead of fixed-window buckets. The old `/admin/stats/traffic` returned three pre-aggregated buckets (today / last_7_days / last_30_days). The chart needs daily granularity for the x-axis, so the new shape is `{ daily: [{date, homepage, checkout, form_fill, paid}, ...], totals: {…} }` — same pattern as `/admin/stats/daily`, easier for the chart to consume directly.

- **`paid` bucketed by `completed_at`, not `created_at`.** A paid order shows up on the day payment confirmed, matching the dashboard Revenue chart's WIB-bucket pattern. Lets the chart accurately depict when conversions happen, not when the order was originally placed.

- **LATERAL subqueries on `/admin/traffic`** so the affiliate_clicks JOIN and the orders JOIN don't multiply each other. With ~30 affiliate_clicks rows and ~5 orders per affiliate, a naive `LEFT JOIN orders + LEFT JOIN affiliate_clicks + GROUP BY a.id` would produce 150 partial rows per affiliate before the GROUP BY collapses them.

- **`90` kept in `/admin/traffic` whitelist** for compatibility, even though the new selector only sends 1/7/30. Future-proofing.

- **Detail panel left orphaned, not deleted.** `loadTrafficDetail` function + `traffic-detail-view` HTML still in the file — easier to re-wire later than to delete and re-create. They consume zero runtime cost since nothing calls them now.

## Verification

- [x] Backend SQL traced for all three endpoint paths; LATERAL joins produce 1 row per affiliate, no cartesian products
- [x] Live SQL run against staging confirmed: chart totals = sum of (per-affiliate counts) + non_affiliate counts
- [x] Frontend: no stale references to removed symbols (`_trafficStatsData`, `loadTrafficStats`, `adminChartTraffic`, etc.)
- [x] Affiliate table colspan updated 9 → 7
- [ ] Smoke-test on staging admin Traffic tab: chart renders 30 stacked bars; period buttons toggle correctly; Non-Affiliate row appears below the 10 affiliates; numbers match the simulated values above

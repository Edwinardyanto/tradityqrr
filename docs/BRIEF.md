# TRXC QRR — Project Brief (Frontend)

**Repo:** Edwinardyanto/tradityqrr
**Production URL:** https://trxcqrr.com
**Staging URL:** https://staging.tradityqrr.pages.dev
**Host:** GitHub Pages (production, branch: main) + Cloudflare Pages (staging, branch: staging)
**Stack:** Static HTML/CSS/JS — no build step

---

## Current State

**Date:** 2026-04-27

### Pages
| File | Role |
|---|---|
| `index.html` | Public landing page — loads Meta Pixel, tracks traffic, injects affiliate pixels via `?ref=` |
| `checkout-basic.html` | Checkout — Rp 499.000 |
| `checkout-silver.html` | Checkout — Rp 1.199.000 |
| `checkout-gold.html` | Checkout — Rp 3.999.000 |
| `checkout-lifetime.html` | Checkout — Rp 20.999.000 |
| `checkout-test.html` | Checkout — Rp 10.000 (testing only) |
| `success-{basic,silver,gold,lifetime}.html` | Thank-you pages (static, no API calls) |
| `admin/index.html` | Admin console — SSE live feed, orders, affiliates, promo, daily stats |
| `affiliate/index.html` | Affiliate portal — registration + dashboard |

### Admin Dashboard layout (post-2026-05-06 waitlisting restructure)
1. **Hero stats** — Revenue, Transaksi, AOV, Conv Rate (orders-based, 30d)
2. **Alert tiles** — Failed grants, Pending affiliates, Payout ready
3. **Revenue Harian + Sold Products** — 2:1 row, both driven by the shared range selector
4. **Funnel Traffic + Order terbaru** — 1:1 row; left is the stacked bar (Kunjungan → Checkout → Form Diisi → Paid), right is the live recent-orders SSE feed (was a "Coming soon" placeholder until 2026-05-06)
5. **Waitlisting + Top affiliates** — 1.3:1 row; left lists `cap_waitlist` rows with status='waiting' (deferred revenue, NOT counted in hero), right is the 30d affiliate leaderboard
6. **Failed orders** — retry queue

The legacy **Traffic tab is gone**. The Funnel chart moved to the Dashboard (item 4) and the per-affiliate Kunjungan/Checkout/Form Fill/Terbayar table moved to the bottom of the **Affiliates tab** as an "Affiliate Traffic" panel (10 affiliate rows + Non-Affiliate aggregate row).

### Range selector behavior
A single Today / Yesterday / 7D / 30D selector at the top of the Revenue Harian panel controls **three charts simultaneously**:
- Revenue Harian (`/admin/stats/daily`)
- Sold Products donut (filtered client-side from `/admin/transactions`)
- Funnel Traffic (`/admin/stats/funnel`)

When the user is on the Affiliates tab, the same selector also reshapes the Affiliate Traffic table (`/admin/traffic`) on next mount.

### Funnel Traffic data sources
**Site-wide, not affiliate-scoped.** All four layers ignore `ref_code`:
| Layer | Source | Filter |
|---|---|---|
| Kunjungan | `site_clicks` | `page='homepage'` |
| Checkout | `site_clicks` | `page='checkout'` |
| Form Diisi | `orders` | (window only) |
| Paid | `orders` | `completed_at IS NOT NULL`, WIB-bucketed |

For affiliate-attributed numbers, see the Affiliate Traffic table on the Affiliates tab (sourced from `/admin/traffic`, which LATERAL-joins `affiliate_clicks` + `orders` per ref_code).

### Branch Status
- `main` → auto-deploys to production (GitHub Pages); fully synced with staging as of 2026-04-27
- `staging` → auto-deploys to Cloudflare Pages at the Staging URL above; talks to Railway staging backend; **fully operational and verified end-to-end**
- `feat/*` → local only

### Outstanding
- [ ] `PLACEHOLDER_META_PIXEL_ID` still in all pages — replace before go-live

---

## Recent Changes

| Date | Change |
|---|---|
| 2026-04-26 | Staging branch created and pushed |
| 2026-04-26 | docs/ folder initialized (BRIEF, ARCHITECTURE, DECISIONS, spec template) |
| 2026-04-26 | **Staging environment fully operational** — Cloudflare Pages live at staging URL; backend staging seed available via `npm run seed:staging` in tradity-qrr-server |
| 2026-04-27 | admin "Best Selling Product" donut → "Sold Products" with shared Today/Yesterday/7D/30D selector that also drives Revenue Harian (calendar-day boundary on both) |
| 2026-04-27 | Sold Products donut filter aligned to Revenue chart: now `payment_status IN ('paid', 'settled')` instead of `tv_access_status='granted'` |
| 2026-04-27 | Big-scope admin revisions: orders-based Conv Rate, sortable date columns on Orders tab, new Site Traffic summary cards on Traffic tab — see [spec](specs/2026-04-27-big-scope-admin-revisions.md) |
| 2026-04-27 | Traffic tab revision: Site Traffic 3-card panel replaced by **Funnel Traffic** stacked-bar chart (Today/7D/30D selector, 4 layers from site_clicks + orders); affiliate table drops CR%/Aksi columns, sources Form Fill + Terbayar from `orders.ref_code` (per affiliate); new Non-Affiliate row at the bottom — see [spec](specs/2026-04-27-traffic-tab-revision.md) |
| 2026-04-27 | **Dashboard restructure shipped to production**: Funnel Traffic chart moved from the (now-deleted) Traffic tab to the Dashboard, paired with a reserved 1:1 placeholder; per-affiliate traffic table relocated to the bottom of the Affiliates tab; shared range selector now drives Revenue Harian + Sold Products + Funnel Traffic together (and the Affiliate Traffic table when mounted). Traffic tab removed entirely from sidebar/mobile/legacy nav. See [spec follow-up](specs/2026-04-27-big-scope-admin-revisions.md) |
| 2026-05-06 | **Dashboard Waitlisting panel + Order terbaru relocation**: "Coming soon" placeholder retired; "Order terbaru" promoted into the slot next to Funnel Traffic; new "Waitlisting" panel surfaces `cap_waitlist` rows with status='waiting' as deferred revenue (NOT counted in hero metrics — only revenue once promoted). Reads `/admin/waitlist?status=waiting`. See [spec](specs/2026-05-06-dashboard-waitlisting-panel.md). |
| 2026-05-14 | **Waitlist sub-tab search + filter**: Subscribers-style filter bar (search box, status select, plan select, reset) on the admin Waitlist sub-tab. Single fetch + client-side filtering replaces the per-status server round-trip. See [spec](specs/2026-05-14-waitlist-search-filter.md). |

---

## Backend
Backend lives at `tradity-qrr-server-production.up.railway.app`. If domain changes, grep for it and update all pages.

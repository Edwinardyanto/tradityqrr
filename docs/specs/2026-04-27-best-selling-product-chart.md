# Best Selling Product Donut — Rename + Data Fix

**Date:** 2026-04-27
**Status:** Done
**Deployed:** 2026-04-27
**Branch:** feat/fix-best-selling-product

## Files Changed

- `admin/index.html` — three edits inside the Command Center dashboard:
  - L1090–1091 — panel title "Mix produk · 30D" → "Best Selling Product"; subtext "Berdasarkan jumlah order" → "Last 30 days"
  - L1095 — `<svg aria-label>` updated to match new title
  - L2304–2352 — donut data prep rewritten

## What Was Wrong

The donut panel claimed "30D" but the underlying `renderCCDashboard()` code applied no time filter. It also counted every transaction regardless of `tv_access_status`, so failed/pending TV-grant attempts inflated the mix. A dead `replace(/^TRXC QRR\s*/, '')` prefix-strip was a no-op against the current product names (`TRXC Basic`, `TRXC Pro`) and added noise. When zero qualifying transactions existed, the donut rendered an empty circle (because of the `|| 1` divide-by-zero guard) with no message.

## What Changed

1. Filter to `tv_access_status === 'granted'` — only successful sales count toward "best selling"
2. Filter to `created_at >= now − 30 days` — matches the visible "Last 30 days" label
3. Removed the `TRXC QRR ` prefix strip — display the canonical `product_name` as stored
4. Added empty-state branch — when `total === 0`, render a thin grey ring + "no data" text and a "No paid orders in the last 30 days." message in the legend list

## Decisions Made

- Used `tv_access_status === 'granted'` as the "successful sale" predicate rather than fetching `/admin/orders` with `payment_status=paid`. The transactions list was already in scope; switching endpoints would have broadened the change beyond what was asked.
- Kept the `slice(0, 4)` cap on shown products. The palette has 5 colours but the original UI displayed 4 — preserving that to avoid scope creep.

## Known Limitation (Not Fixed Here)

`/admin/transactions` returns `LIMIT 100`. If production has more than 100 transactions in 30 days, the chart undercounts. Filing this as future work; for staging seed data (5 transactions) it has no effect.

## Verification

- [x] Syntax checked — diff is clean
- [x] Logic traced for the 5-transaction seed: 2 granted `TRXC Pro` + 3 granted `TRXC Basic` → donut should show TRXC Basic 60%, TRXC Pro 40%
- [ ] Manual smoke test on staging admin (`https://staging.tradityqrr.pages.dev/admin/`) once merged into staging
- [ ] Visual check of empty-state by clearing transactions or seeding zero

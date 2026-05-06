# Dashboard Waitlisting Panel + Order terbaru Relocation

**Date:** 2026-05-06
**Status:** Done
**Branch:** feat/dashboard-waitlisting

## Files Changed

- `admin/index.html`
  - Removed `cc-panel-empty#cc-reserved-panel` ("Coming soon") from the Funnel Traffic row
  - Promoted "Order terbaru" into that slot (now alongside Funnel Traffic in row 1)
  - Added new "Waitlisting" panel in the slot Order terbaru vacated (alongside Top Affiliates in row 2)
  - Added `/admin/waitlist?status=waiting` to `loadDashboard()`'s parallel fetches; passed through to `renderCCDashboard`
  - New CSS: `.cc-status-waiting` badge (gold/amber theme matching the join-waitlist email accent) + `.cc-order-avatar-wait` for position-numbered avatars

- `docs/BRIEF.md` — updated dashboard layout section + retired "Reserved placeholder" outstanding item

## Decisions Made

- **Waitlist payments are deferred revenue, not revenue** — the new panel sub-title flags this explicitly ("Order menunggu slot terbuka — belum diakui sebagai revenue") so the operator can't confuse it with the live Order terbaru feed above. Hero metrics (Revenue, Conv Rate) intentionally still exclude waitlisted entries; counting them would overstate cash that's still refundable until promoted.
- **Footer shows full deferred total**, not the slice — even if only the top 6 entries render, the "Deferred: Rp X" footer sums across the entire waitlist so the operator sees the true outstanding obligation.
- **No SSE wiring needed** — `loadDashboard()` already re-fires on `order_paid` SSE events, which is exactly when a new waitlist entry would land (cap-full diversion happens inside `processPaymentPaid`). The panel re-fetches alongside everything else.
- **Position derivation client-side** — the endpoint returns rows ordered by `joined_at ASC`, so the array index + 1 is the position. Keeps the backend simple; no change to `/admin/waitlist`.

## Lessons Learned

- The "Order terbaru" panel was sourcing from `/admin/transactions` (i.e. the `transactions` table, populated only after a TV grant attempt). Waitlisted customers paid but don't get a transactions row until the promotion cron fires — which is exactly why "ada pembelian baru tapi panel ga update meski refresh" reproduces. Splitting the panels (live orders vs. waitlist) makes that gap visible instead of silent.

## Verification

- [x] HTML smoke checks (`cc-waitlist-list` container, `/admin/waitlist?status=waiting` API call, badge CSS, no leftover "Coming soon")
- [ ] Live preview against staging backend: panel populates when `cap_waitlist` has waiting rows
- [ ] Visual check on Cloudflare Pages staging deploy (https://staging.tradityqrr.pages.dev)
- [ ] Production smoke test after merge to main: deferred-revenue footer matches sum of waiting rows

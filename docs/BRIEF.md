# TRXC QRR — Project Brief (Frontend)

**Repo:** Edwinardyanto/tradityqrr
**Production URL:** https://trxcqrr.com
**Staging URL:** https://staging.tradityqrr.pages.dev
**Host:** GitHub Pages (production, branch: main) + Cloudflare Pages (staging, branch: staging)
**Stack:** Static HTML/CSS/JS — no build step

---

## Current State

**Date:** 2026-04-26

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
| `admin/index.html` | Admin console — SSE live feed, orders, affiliates, traffic, promo, daily stats |
| `affiliate/index.html` | Affiliate portal — registration + dashboard |

### Branch Status
- `main` → auto-deploys to production (GitHub Pages)
- `staging` → auto-deploys to Cloudflare Pages at the Staging URL above; talks to Railway staging backend
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

---

## Backend
Backend lives at `tradity-qrr-server-production.up.railway.app`. If domain changes, grep for it and update all pages.

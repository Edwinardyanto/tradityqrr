# Architecture — Frontend (tradityqrr)

## System Overview

```
Browser
  └─ GitHub Pages (trxcqrr.com)
       ├─ index.html          ← landing + affiliate pixel injection
       ├─ checkout-*.html     ← 5 tier-specific checkout pages
       ├─ success-*.html      ← 4 static thank-you pages
       ├─ admin/index.html    ← admin console (SSE + REST)
       └─ affiliate/index.html ← affiliate portal (Bearer token)

All dynamic data → Railway backend
  └─ tradity-qrr-server-production.up.railway.app
```

## Key Design Decisions

- **No build step.** Every page is a self-contained HTML file with inline `<style>` and `<script>`. Edits ship on commit+push.
- **Helpers duplicated per page.** `loadMetaPixel`, `loadGoogleAds`, `injectMetaPixels`, `getCheckoutUrl` — no shared JS file. When fixing a bug, fix it everywhere.
- **Affiliate ref flow:** `?ref=` → `sessionStorage['rqq_ref']` → appended to checkout URLs → passed to backend on `/checkout/create-order`.
- **Checkout tiers are near-duplicates.** Only `PRICE_LABEL`, `PRICE_AMOUNT`, CSS accent vars, success redirect URL, and og/title text differ between tiers.
- **Admin auth:** plain `x-admin-secret` header — no JWT.
- **Affiliate auth:** Bearer token stored in `localStorage['affiliate_token']`.

## Static Assets

`data/` — images only (`trxc_logo.jpg`, TradingView screenshot, indicator screenshot). No JS or build artifacts.

## Backup Copies

Files/folders suffixed `… 11.19.54` are timestamped backups of older versions — ignore when reading or editing.

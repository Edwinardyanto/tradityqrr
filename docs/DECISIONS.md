# Decision Log — Frontend

| Date | Decision | Reason |
|---|---|---|
| 2026-04-26 | No build step / inline-only JS+CSS | Simplicity; GitHub Pages serves static files directly; no CI required |
| 2026-04-26 | Helpers duplicated per page (no shared JS) | Avoids cache-busting complexity; each page fully self-contained |
| 2026-04-26 | Staging branch → Cloudflare Pages (pending) | Consistent with production-like preview; Netlify Drop used as temp workaround |
| 2026-04-26 | Multi-brand refactor (TRXC + ETC) deferred | Scope too large to mix with current work; separate task |

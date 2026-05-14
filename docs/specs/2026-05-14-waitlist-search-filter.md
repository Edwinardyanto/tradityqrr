# Waitlist sub-tab search & filter

**Date:** 2026-05-14
**Status:** Done
**Branch:** feat/waitlist-search-filter

## Files Changed

- `admin/index.html` — Waitlist sub-tab now mirrors Subscribers UX: search input + status select + plan select + reset button, with client-side filtering on a single fetch.

## Decisions Made

- **Client-side filtering, single fetch** — match the Subscribers pattern (`allSubscribers` cache + `getFilteredSubscribers`). Drops the per-status server round-trip and makes search/plan filtering snappy. Volume on `cap_waitlist` is small (hundreds, not tens of thousands) so the full-table fetch is cheap.
- **Default status = Waiting** — preserve existing UX where opening the sub-tab lands on the actionable subset; Reset also returns to `waiting`, not blank "Semua".
- **Filters kept: Search + Status + Plan** — TV/Telegram/Source filters from Subscribers omitted because they don't carry information for waitlist entries (no TV access until promotion, no Telegram link until promotion, all entries are paid orders).

## Lessons Learned

- The dashboard's Waitlisting panel still hits `/admin/waitlist?status=waiting` directly (separate from the admin sub-tab's `loadWaitlist`), so refactoring the sub-tab to drop the query param has no impact on the dashboard panel.
- The `populate*PlanFilter` helper pattern (preserve current value when repopulating from new data) used by Subscribers ports cleanly — refresh button no longer resets the operator's filter selection.

## Verification

- [x] Syntax checked (function references match, no orphan `loadWaitlist?status=` callers remain)
- [ ] Manual smoke test on staging: search by email/nama/phone/tv, switch status to Promoted/Refunded/Cancelled, switch plan, hit Reset → returns to Waiting view
- [ ] Verify Refresh button preserves active filters after re-fetch
- [ ] Verify `markWaitlistRefunded` / `rejectWaitlistRefund` still re-render correctly after action (they call `loadWaitlist()`)

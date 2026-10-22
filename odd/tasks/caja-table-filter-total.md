# Caja: total de mesa al filtrar

## Objective
When the cashier filters the queue by a single table (via the table filter buttons added in `caja-mesa-filter-pin-mask`), show the sum of all visible orders' totals for that table, so the cashier knows how much the whole table owes at a glance.

## Problem / why
The filter-by-table feature shows all orders for one table side by side, but the cashier still has to add up each card's total manually.

## Scope
Repo: `foodc-caja` (frontend only, no backend change — `total_amount` already comes from the API per order).

## Decisions
- The table total is shown ONLY when a table filter is active (not in the default "Todas" grouped view).
- Sum only orders currently visible for that table (same set rendered as cards — i.e. after the existing `visibleOrders`/pending-payment filtering).
- An order whose `total_amount` isn't a finite number (the existing `toCardModel` null case, when `order_details` is malformed and a summary fallback is used instead) is excluded from the sum; if any order in the filtered set was excluded this way, visually flag that the total may be incomplete rather than silently presenting a wrong number as if it were exact.

## Tasks
- [x] T1: Sum helper — `js/queue.js`: added `tableTotal(orders)`, returning `{ total: number, isComplete: boolean }`. Reuses the same finite-number parsing as `toCardModel`'s `totalAmount` (via the existing private `toFiniteNumber`), summing finite `total_amount` values and flipping `isComplete` to `false` when any order in the list lacked one. 4 unit tests added in `test/queue.test.mjs` (sum+complete, empty list, excluded non-finite entry, numeric-string amount parsed) — `npm test` green (217/217).
- [x] T2: Render — `js/views/queue-view.js`: when `selectedTable !== null`, renders `messages.queue.tableTotal(total)` as a status line right after the filter row and above the grid; nothing is rendered in the default "Todas" view (`selectedTable === null`). When `isComplete` is `false`, an extra line with `messages.queue.tableTotalIncomplete` is shown below it. New copy added to `js/messages.js` under `queue:` (`tableTotal`, `tableTotalIncomplete`), following the existing non-inline-string pattern. Wording for the incomplete case follows the same posture as `close.confirmSummaryUnavailable` (flag the total instead of presenting a possibly-wrong number as exact), not an invented new tone.
- [ ] T3: Manual verification — same production-testing constraint as before (do not test with fabricated data against the real prod backend from an automated session). Orchestrator will ask the user to verify this one on the real tablet since it directly reflects money totals.

## Checks
- Run `npm test` (node --test) after T1-T2, all green including new cases.

## Branch
`feat/caja-table-filter-total`

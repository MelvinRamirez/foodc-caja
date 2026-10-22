# Caja: filtro por mesa + ocultar PIN

## Objective
Add two UX/security improvements to the Caja screen:
1. Orders grouped/sorted by table, with filter buttons per active table.
2. Mask the PIN input so digits aren't shown in cleartext.

## Problem / why
- PIN input is `type="text"`, so the PIN is visible on screen while typing (security risk on a shared device).
- Orders list is FIFO-only; a cashier cannot quickly see all pending orders for one table.

## Scope
Repo: `foodc-caja` (frontend only; backend `table_number` field already exists, no API change needed).

## Decisions
- Default view (no table filter active): all orders, grouped/ordered by table number (not FIFO).
- Within a table group, preserve relative FIFO order (by `created_at`/server order).
- Table filter buttons shown only for tables that currently have visible (unpaid) orders.
- Clicking a table button filters the list to that table only; clicking again (or a "todas" button) clears the filter.

## Tasks
- [x] T1: PIN masking — `js/views/pin-view.js`: change input `type` from `text` to `password`, keep `inputmode="numeric"`, `maxlength`, sanitization logic unchanged. Done: single-attribute change (line 42), sanitization in `wireInteractivity` and `isValidPinInput`/`PIN_MIN`/`PIN_MAX` untouched.
- [x] T2: Derive active tables + sort/group — `js/queue.js`: added `activeTables(orders)` (distinct `table_number`, ascending numeric/alphabetical fallback) and `groupOrdersByTable(orders)` (groups by table, groups ascending, FIFO preserved within each group, new array, input untouched). `toCardModel`/`visibleOrders` unchanged. Tests added in `test/queue.test.mjs` (6 new cases: distinct/ascending, alpha fallback, empty, grouping/FIFO-within-group, no-mutation, empty).
- [x] T3: Filter UI + wiring — `js/views/queue-view.js`: added ephemeral `selectedTable` closure state (module-local, not in `js/store.js`/`js/actions.js` — no existing UI-filter-state pattern there to follow, and this is purely ephemeral per-session UI state with no persistence need, so a local variable in `mountQueueView` matches the simplicity already used for `cardNodes`). Renders a "Todas" + one-per-active-table button row above the grid (hidden when there are zero active tables), highlights the active button, filters/groups the grid accordingly, auto-clears the filter if the selected table empties out. Default view uses `groupOrdersByTable`. Existing "Pagada" click dispatch kept working (dispatch function now also routes `data-action="filter-table"` clicks, `pay` handling unchanged). New copy added to `js/messages.js` (`messages.queue.allTables`, `messages.queue.tableButton`).
- [ ] T4: Manual verification — PENDING. Code-reviewed the full diff line by line (orchestrator): logic matches spec, test coverage for the data-layer functions is solid, 213/213 green. Could NOT exercise it in a real browser from this session — `js/config.js` `API_URL` points at the real production backend, so loading the page would poll/display real orders and risks accidentally marking a real order as paid; this CLI environment also has no browser-automation tool available. Needs the same manual pass on the real tablet that closed K3 parte B.

## Checks
- Automated test suite exists (`npm test`, `node --test`). Ran it after T1-T3: 213/213 passing (207 pre-existing + 6 new for `activeTables`/`groupOrdersByTable`).
- Manual verification per T4 is still pending — do it on the tablet before/soon after deploy: confirm PIN shows dots, table buttons appear only for tables with orders, filtering and default grouped view work, "Pagada" still works per order.

## Branch
`feat/caja-mesa-filter-pin-mask`

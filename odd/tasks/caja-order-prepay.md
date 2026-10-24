# Caja order prepay (show prepaid orders in context, by payment status)

## Objective

The `FoodC` backend (sibling repo, branch `feat/caja-order-prepay`, already
implemented and merged-ready) now lets a diner pay for an order at the
moment it's placed, before the kitchen finishes it. It decoupled payment
from kitchen status: `Orders` rows now carry both `status`
(`Pendiente`/`Completado`, kitchen) and `payment_status`
(`Pendiente`/`Pagada`, money), independently. This task updates the
`foodc-caja` frontend to match that new backend contract and to show the
cashier both unpaid and already-paid orders in one screen, with a visible
kitchen-status hint on each.

## Breaking backend contract change this frontend must follow

`getCajaOrders`'s body param is renamed `status` -> `payment_status`
(values `Pendiente`/`Pagada`, default `Pendiente` when omitted). Every
returned order object now also carries `payment_status` alongside the
existing `status`. This frontend is currently on the OLD contract
(`js/api.js`'s `getCajaOrders(status)` sends `{action, status, pin}`) — it
will send a now-meaningless param and get back only the unpaid-regardless-
of-kitchen-status set, with no paid orders ever shown, once the new backend
is deployed. This task closes that gap.

## Current architecture (confirmed by reading the code, not guessing)

- `js/api.js` `getCajaOrders(status)`: today sends `{action:'getCajaOrders',
  status, pin}` (or no `status` key at all, meaning the OLD backend default
  "Completado"). `js/main.js`'s poll loop (`effects.js` `pollOnce`) calls it
  with NO argument (today: all unpaid-and-kitchen-done orders). A second,
  separate call, `main.js`'s `fetchPagadaSummary`, calls
  `getCajaOrders('Pagada')` only for the close-dialog preview
  (`close-view.js` `summarizePagadaPreview`), never for the main queue.
- The main queue (`js/views/queue-view.js` + `js/queue.js` +
  `js/views/order-card-view.js`) renders ONE grid of cards, grouped by
  table, each card always showing a "Pagada" button (`js/views/
  order-card-view.js` `createCardNode`) — because today every card the
  queue ever receives is, by construction, unpaid. There is currently no
  `status`/`payment_status` field read anywhere in `toCardModel` or the card
  view at all.
- `js/queue.js`'s optimistic-hide mechanism (`hiddenIds`/`visibleOrders`,
  consulted against `state.payment.pending`) hides a card by `order_id`
  while its `markOrderPaid` call is in flight/retrying. This is unrelated
  to the new `payment_status` field (it is a client-side "I just pressed
  pay, don't flicker" mechanism) and needs no change.
- `close-view.js`'s `summarizePagadaPreview` sums/counts every order in the
  `getCajaOrders('Pagada')` response as "what the close will archive."
  **This will become wrong once the backend's close guard requires BOTH
  `payment_status===Pagada` AND `status===Completado`** (a prepaid order
  still cooking is `Pagada` but not archived) — this preview must be
  corrected as part of this task, not left to silently overcount.

## Scope

1. **`js/api.js`**: rename `getCajaOrders(status)` ->
   `getCajaOrders(paymentStatus)`; body key `status` -> `payment_status`.
   `main.js`'s `fetchPagadaSummary` call site updates its argument name
   only (still passes the literal `'Pagada'` string — unchanged value,
   same semantics: "give me every currently-paid order").
2. **`js/effects.js` `pollOnce`**: the main poll must show BOTH unpaid and
   paid orders together (that's the actual ask — Caja sees both, in
   context). Fetch `ctx.api.getCajaOrders('Pendiente')` and
   `ctx.api.getCajaOrders('Pagada')` concurrently (`Promise.all`) each poll
   tick; the poll is a success only if BOTH resolve `kind:'success'` (merge
   `.data` arrays, dispatch the existing `actions.pollSucceeded` with the
   merged array — no new action needed); if either fails, the whole poll
   tick fails using the existing `mapPollFailureKind`/`pollFailed` path,
   preferring the more specific of the two failures if they differ (same
   precedence as the existing priority: `UNAUTHORIZED` > contract > other —
   use your judgment and document the choice in your report, this exact
   tie-break isn't specified further). Do not change `startPaymentRunner`
   (markOrderPaid flow) at all — it's independent of this listing change.
3. **`js/queue.js` `toCardModel`**: add `paymentStatus: order.payment_status`
   and `kitchenStatus: order.status` to the returned card model (plain
   passthrough of the two existing string fields the backend now always
   sends; no new validation logic — the existing `isWellFormedItem` etc.
   stay untouched).
4. **`js/views/order-card-view.js`**: when `cardModel.paymentStatus ===
   'Pagada'`, DO NOT render the "Pagada" pay button — the order is already
   paid. Instead render a small paid badge (e.g. text "✓ Pagada", reuse the
   existing Tailwind utility-class style of `order-card__notes` for
   positioning/visual weight, but its own distinct class
   `order-card__paid-badge`) AND a kitchen-status indicator next to the
   table number/timer in the header (e.g. "En cocina" when `kitchenStatus
   === 'Pendiente'`, "Lista" when `'Completado'`) using a new
   `order-card__kitchen-status` class. When `paymentStatus === 'Pendiente'`
   (the normal, today's case), render exactly as today — pay button, no
   paid badge. `updateCardNode` must also correctly add/remove/update the
   badge and kitchen-status indicator when a REUSED card's `cardModel`
   changes between polls (same pattern as its existing notes
   add/replace/remove branches).
5. **`js/messages.js`**: add the new UI copy under `payment.*` (e.g.
   `paidBadge: 'Pagada'` — note this is a DIFFERENT message key from the
   existing `payButton: 'Pagada'`, keep both, they serve different roles
   now) and `kitchenStatus.pending`/`kitchenStatus.done` (wording your
   call, consistent with the existing terse cashier-facing tone elsewhere in
   this file — e.g. "En cocina" / "Lista").
6. **`js/views/close-view.js` `summarizePagadaPreview`**: filter
   `apiResult.data` to `order.status === 'Completado'` before
   counting/summing — matching exactly what the backend's close guard will
   actually archive, so the "¿Seguro?" preview never overstates what's
   about to be closed.
7. **Tests**: update `test/api.test.mjs` (param rename), `test/effects.test.mjs`
   (the dual-fetch poll behavior, both-must-succeed rule, and the merge),
   `test/queue.test.mjs` (`toCardModel`'s two new fields), and
   `test/close.test.mjs` (`summarizePagadaPreview`'s new filter). There are
   no existing DOM-level tests for `order-card-view.js`/`queue-view.js`
   (confirmed: no such test files exist) — don't invent a new test
   framework/harness for them; a judgment call, not an oversight.
8. **Docs**: if this repo's `openspec/specs/caja-payment-queue/spec.md` (or
   equivalent) documents the single-fetch/single-status poll contract,
   update it to describe the dual-fetch/merged-queue behavior and the new
   card states. If no such spec exists in this repo, skip — don't invent
   one.

## Explicitly out of scope

- The `FoodC` backend repo itself — already done, this task only consumes
  its new contract.
- Any new screen/tab/toggle. One merged queue, badge-differentiated, per
  Melvin's confirmed design (2026-10-24 conversation): "mismo diseño de
  tarjeta, pero sin el botón de cobro... un badge chico tipo 'Pagada'... y
  un indicador del estado de cocina."
- Re-deploying/pointing `js/config.js`'s `API_URL` at the new backend — that
  happens after the backend's own Apps Script Web App redeploy, a separate
  manual step, not part of this frontend task.

## Tasks

- [x] T1: `js/api.js` — rename `getCajaOrders` param to `payment_status`;
      update `main.js`'s `fetchPagadaSummary` call site.
- [x] T2: `js/effects.js` — dual-fetch (`Pendiente` + `Pagada`) merged poll.
- [x] T3: `js/queue.js` — `toCardModel` carries `paymentStatus`/`kitchenStatus`.
- [x] T4: `js/views/order-card-view.js` — conditional pay button vs. paid
      badge + kitchen-status indicator, for both new-card and reused-card
      (`updateCardNode`) paths.
- [x] T5: `js/messages.js` — new copy keys.
- [x] T6: `js/views/close-view.js` — `summarizePagadaPreview` filters to
      kitchen-`Completado` before counting/summing.
- [x] T7: Tests updated across `api`/`effects`/`queue`/`close` test files;
      `npm test` passes with 0 failures. (Done incrementally alongside
      T1/T2/T3/T6's own commits; final full run: 228 passed, 0 failed.)
- [x] T8: Docs — update `caja-payment-queue` spec if it exists here and
      documents the single-fetch contract; skip if it doesn't exist.
      Confirmed no `openspec/` directory exists anywhere in this repo
      (`find . -iname "*caja-payment-queue*"` and `find openspec/specs`
      both empty) — skipped per the task file's explicit instruction, not
      an oversight.

## Verification

- `npm test` (Node's built-in test runner, `test/**/*.test.mjs`) passes, 0
  failures.

## TDD mode

No enforced TDD convention observed beyond the existing per-module
`*.test.mjs` suites already in this repo. Ordinary functional checks (full
test suite) at each task's close.

## Status

2026-10-24: Started. Branch `feat/caja-order-prepay` created off `main`.

2026-10-24: CLOSED. All T1-T8 complete. `npm test`: 228 passed, 0 failed, 0
skipped (started at 217, grew as T1/T2/T3/T6 each added coverage for their
own change). T8 (docs) skipped per its own instruction: no `openspec/`
directory exists anywhere in this repo, confirmed via `find`.

Commits (newest last):
- 22f5bad feat(api): rename getCajaOrders param to payment_status (T1)
- 2db0a1b feat(queue): poll both payment statuses and merge results (T2)
- a4d47c7 feat(queue): carry paymentStatus and kitchenStatus in toCardModel (T3)
- d03bb9f feat(messages): add paid badge and kitchen status copy (T5)
- 4d6d95e feat(card): show paid badge and kitchen status instead of pay button (T4)
- a9e04f9 fix(close): filter pagada preview to kitchen-completado orders (T6)

T2 failure tie-break judgment call: when the two concurrent
`getCajaOrders('Pendiente')`/`getCajaOrders('Pagada')` calls both fail and
differ, `pollFailureRank` in `js/effects.js` ranks a relock-triggering
UNAUTHORIZED (0) above `PIN_NOT_CONFIGURED` (1) above a contract-bug code
(2) above everything else/transient (3), and the poll reports whichever
failure has the lower (more specific) rank — reusing this poll's own
pre-existing single-fetch priority order instead of inventing a new one,
since the task explicitly left this unspecified.

`js/config.js`'s `API_URL` needed no change: it already pointed at the
stable `/exec` URL, which the backend redeploy kept unchanged (same
deployment id, new version `@10`).

## Follow-up visual tweaks (post-deploy, from Melvin's live testing)

2026-10-24: After testing the live prepay flow on the real cashier device,
Melvin asked for two card tweaks: "En cocina" on a red background with
white text (to stand out more than the neutral slate it had), and the
"Pagada" badge text/symbol doubled in size (`text-xs` -> `text-2xl`).
Implemented directly in `js/views/order-card-view.js`
(`renderKitchenStatus`/`renderPaidBadge`), commit `4547376`
("style(card): highlight pending kitchen status and enlarge paid badge").
`npm test` 228/228 unchanged (no DOM-level test asserts these exact
classes). Merged to `main`, pushed, GitHub Pages rebuilt and confirmed
serving the updated classes.

## Status

2026-10-24: **Done.** Merged to `main` in both `FoodC` and `foodc-caja`,
pushed, backend redeployed live (same `/exec` URL, version `@10`), frontend
rebuilt on GitHub Pages. End-to-end prepay flow confirmed working by Melvin
on the real cashier device, including the two follow-up visual tweaks.
Nothing outstanding. The only known follow-up item from the original scope
boundary (frontend contract rename) is itself this task — already closed.

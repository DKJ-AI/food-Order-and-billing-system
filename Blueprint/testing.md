# Testing Strategy & Acceptance Checklist

## 1. Testing layers

| Layer | Tool direction | What to verify |
|---|---|---|
| Backend unit | Pytest | Pricing, tax rounding, state machines, permission predicates, idempotency |
| Backend API | Pytest + HTTPX | Request/response contracts, errors, outlet scope, DB transactions |
| DB integration | PostgreSQL test DB | Constraints, migrations, row locks, unique indexes |
| Frontend unit/component | Vitest + React Testing Library | Components, form validation, status states, accessibility basics |
| Frontend e2e | Playwright | QR guest journey, kitchen, waiter, owner basic flows |
| Payment sandbox | Provider test mode or deterministic fake adapter | Create intent, success, fail, duplicate/replayed webhook, amount mismatch |
| Operational | Docker/CI smoke | Clean setup, health checks, migrations, build, backup/restore |

## 2. Critical end-to-end scenario

1. Seed one restaurant, one outlet, two tables, menu categories/items/options, and staff users for owner, waiter, kitchen.
2. Guest scans Table 08 QR and sees correct branding/table/menu.
3. Guest opens session; adds an available item with a valid option, allergy warning, and preference; places order.
4. Duplicate submit with same idempotency key returns the same order, not a duplicate.
5. Kitchen sees the order and dietary alert; accepts, starts preparation, and marks ready.
6. Guest receives live status update or sees it after polling/reconnect.
7. Guest calls waiter; waiter sees request, acknowledges, resolves; guest sees state.
8. Guest opens session bill; backend total matches order snapshots and configured tax/charge.
9. Guest starts checkout; new order is blocked while checkout is pending.
10. Fake/sandbox gateway sends signed capture webhook; backend verifies and marks payment paid once.
11. Duplicate webhook does not double-count payment. Guest sees receipt; session is closed.
12. Owner report for the outlet/date includes the paid sale with documented gross/net definitions.

## 3. Required negative/security tests

- Invalid, revoked, or cross-outlet QR token is rejected.
- Guest A cannot access Guest B's session/order/bill.
- Guest cannot send unit prices, totals, tax, status, or role to override server values.
- Unavailable item or invalid option cannot be ordered.
- Quantity/note input limits are enforced.
- Duplicate idempotency request returns consistent result.
- `CHECKOUT_PENDING` blocks order creation and duplicate active checkout.
- Payment webhook invalid signature, amount mismatch, currency mismatch, unknown provider order, and replay are rejected/handled safely.
- Browser redirect with `success=true` does not mark payment captured.
- Kitchen cannot access owner reports or another outlet's tickets.
- Waiter cannot change owner settings or mark kitchen status unless explicitly permitted.
- Staff session expiry/revocation blocks API and WebSocket access.
- Service request spam is rate-limited.
- Sensitive tokens/secrets are absent from app logs and WebSocket events.

## 4. UX test checklist

- Guest menu usable at 320 px width without horizontal overflow.
- Item unavailable, menu empty, network lost, invalid QR, closed ordering, and open-bill states are comprehensible.
- Dietary allergy and cross-contact warning visible before placing order.
- Cart controls usable by touch and keyboard.
- Duplicate place-order tap cannot duplicate order.
- Payment pending, failed, retry, and success states are distinct.
- Kitchen ticket remains visible after status action until explicit workflow removal.
- Waiter request priority and age are obvious.
- Owner metrics have loading/empty/error states and clear currency/date labels.
- Focus order, contrast, labels, and screen-reader status announcements are checked.

## 5. Done criteria for each feature

A feature is done only when:
- It follows the API/data contracts.
- It has server-side authorization/validation where applicable.
- Loading, empty, success, error, stale/offline states are covered.
- Unit/API/component tests pass.
- Type/lint/build checks pass for the affected package.
- No secrets, fake production data, or TODO-only functionality is left in the claimed scope.
- Agent completion report is recorded in `memory.md`.

## 6. Release severity

- **P0:** payment integrity issue, cross-tenant data leak, secret exposure, data loss, inability to place/fulfil orders.
- **P1:** incorrect bill, kitchen misses orders, staff cannot perform essential workflow, major outage without recovery.
- **P2:** non-critical UI defect, report/export issue with no financial data corruption, minor accessibility defect.

Do not launch live payments with an unresolved P0/P1 defect.

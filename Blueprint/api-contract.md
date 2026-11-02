# REST API Contract (Canonical MVP)

**Base path:** `/api/v1`  
**Content type:** JSON  
**Auth:** Guest routes use a secure guest-session cookie scoped to the session; staff routes require authenticated staff session/bearer token as selected by implementation. Payment webhooks use provider signature verification, not user auth.  
**Rule:** Names and payloads below are canonical. Agents must not invent parallel endpoints.

## 1. Common response rules

- Use UUID strings for internal IDs; human order/table labels are separate display fields.
- Server returns money as integer paise plus formatted currency text only as a convenience. Example: `amount_paise: 12500`, `currency: "INR"`.
- Times are ISO 8601 UTC strings.
- Mutating requests accept `Idempotency-Key` header where indicated.
- Every staff route enforces role + outlet scope.
- Paginated list response: `{ "items": [], "next_cursor": null }`.

### Error shape

```json
{
  "error": {
    "code": "ITEM_UNAVAILABLE",
    "message": "One or more items are unavailable.",
    "details": { "item_ids": ["uuid"] },
    "request_id": "uuid"
  }
}
```

Use stable machine-readable `code`; message is safe for display. Never return stack traces, secret values, or provider raw payloads.

## 2. Public guest APIs

### GET `/public/qr/{qr_token}/menu`
Returns restaurant branding, table display label, active categories/items/options, ordering state, and public notices. The raw QR token must be resolved and validated server-side; avoid exposing internal table IDs.

Example response (abbreviated):
```json
{
  "restaurant": {
    "name": "The Saffron Table",
    "slug": "the-saffron-table",
    "logo_url": null,
    "brand": { "primary": "#7A3E2D", "accent": "#D98B45" }
  },
  "outlet": { "name": "Salt Lake", "timezone": "Asia/Kolkata", "currency": "INR" },
  "table": { "label": "Table 08" },
  "ordering_enabled": true,
  "categories": [],
  "notices": []
}
```

### POST `/public/qr/{qr_token}/session`
Create or resume the active session for this table. Server sets a secure, HttpOnly, SameSite cookie (or an equivalent short-lived guest credential strategy documented by implementation). Must not create multiple open sessions for the same table.

Response:
```json
{
  "session": { "id": "uuid", "status": "OPEN", "table_label": "Table 08", "opened_at": "2026-11-02T10:00:00Z" }
}
```

### GET `/public/sessions/{session_id}`
Requires valid guest credential scoped to the session. Returns table/session info, orders, waiter-request summary, bill preview, and session status.

### POST `/public/sessions/{session_id}/orders`
Requires guest credential and `Idempotency-Key`.

Request:
```json
{
  "items": [
    {
      "menu_item_id": "uuid",
      "quantity": 2,
      "option_choice_ids": ["uuid"],
      "item_note": "Less spicy"
    }
  ],
  "dietary_requirements": [
    { "kind": "ALLERGY", "value": "Peanuts", "note": "Please confirm with staff" },
    { "kind": "PREFERENCE", "value": "No onion", "note": "If possible" }
  ],
  "customer_note": "Please bring water when convenient"
}
```

Response:
```json
{
  "order": {
    "id": "uuid", "order_number": "A-104",
    "status": "PLACED", "placed_at": "2026-11-02T10:02:00Z",
    "subtotal_paise": 50000, "tax_total_paise": 0, "total_paise": 50000,
    "items": []
  }
}
```

Prices shown are illustrative; real totals are computed from current server configuration. Include actual tax values when applicable. The sample does not imply a zero-tax default.

### POST `/public/sessions/{session_id}/service-requests`
Requires guest credential and idempotency/rate limiting.

Request:
```json
{ "request_type": "WATER", "note": "Two glasses, please" }
```

Response:
```json
{ "service_request": { "id": "uuid", "status": "OPEN", "created_at": "2026-11-02T10:03:00Z" } }
```

### GET `/public/sessions/{session_id}/bill`
Returns current session-level bill preview and `bill_version`/revision to detect updates.

Response shape:
```json
{
  "session_id": "uuid", "status": "OPEN", "bill_version": 3,
  "currency": "INR", "line_items": [],
  "subtotal_paise": 50000, "discount_paise": 0,
  "tax_lines": [], "service_charge_paise": 0,
  "rounding_paise": 0, "total_paise": 50000,
  "amount_paid_paise": 0, "balance_due_paise": 50000
}
```

### POST `/public/sessions/{session_id}/checkout`
Requires guest credential and `Idempotency-Key`. Server locks session, recalculates exact balance, creates a provider payment intent, and returns safe checkout information.

Request:
```json
{ "bill_version": 3 }
```

Response:
```json
{
  "payment": {
    "id": "uuid", "status": "PENDING", "amount_paise": 50000,
    "currency": "INR", "provider": "configured_provider",
    "checkout_url": "https://provider.example/checkout/opaque"
  }
}
```

Do not return secret keys or provider signing secrets. A stale bill version returns `BILL_CHANGED` with a fresh bill summary. While checkout is pending, reject new orders for MVP.

### GET `/public/sessions/{session_id}/payments/{payment_id}`
Returns backend-confirmed payment status. Guest credential must match session. Frontend polls/reloads after gateway redirect; webhook remains authoritative.

### GET `/public/sessions/{session_id}/receipt`
Returns receipt/invoice snapshot only after paid/authorized close; no receipt should be fabricated for pending/failed payments.

## 3. Staff authentication

### POST `/auth/login`
Request: `{ "email": "staff@example.com", "password": "..." }`  
Response: staff identity, allowed outlets/roles, and secure session/token according to implementation. Never return password hashes.

### POST `/auth/logout`
Invalidates the staff session/token where supported.

### GET `/staff/me`
Returns current staff identity, permissions, and allowed outlet IDs.

## 4. Kitchen APIs

### GET `/staff/kitchen/orders?outlet_id={uuid}&status=NEW,PREPARING,READY&station_id={optional}`
Returns authorized active tickets in queue order. Each ticket includes order number, table label, elapsed/placed time, items, options, notes, dietary/allergy alerts, and allowed actions.

### PATCH `/staff/orders/{order_id}/status`
Request: `{ "status": "PREPARING", "reason": null }`  
Server validates transition, outlet scope, and kitchen permission; records actor/time; emits `order.status_changed`.

Allowed normal progression: `PLACED → ACCEPTED → PREPARING → READY → SERVED`. Cancellation is separately permissioned and audited.

## 5. Waiter APIs

### GET `/staff/service-requests?outlet_id={uuid}&status=OPEN,ACKNOWLEDGED`
Returns oldest-first requests in authorized outlet.

### PATCH `/staff/service-requests/{request_id}`
Request: `{ "status": "ACKNOWLEDGED", "note": null }` or `{ "status": "RESOLVED", "note": "Water delivered" }`.

Server validates transition and role, records actor/time, and emits `service_request.updated`.

## 6. Owner/report APIs

All require OWNER/MANAGER permission and outlet scope.

### GET `/owner/reports/summary?outlet_id={uuid}&from=2026-11-01&to=2026-11-02`
Returns gross/net sales definitions, paid order count, average order value, outstanding balance, and open table sessions.

### GET `/owner/reports/sales-trend?outlet_id={uuid}&from=...&to=...&group_by=day`
Returns timestamp/period buckets with paid sales values.

### GET `/owner/reports/top-items?outlet_id={uuid}&from=...&to=...&limit=10`
Returns item snapshot name, quantity sold, sales amount.

### GET `/owner/reports/payment-mix?outlet_id={uuid}&from=...&to=...`
Returns provider/payment method totals and transaction counts.

### GET `/owner/reports/operations?outlet_id={uuid}&from=...&to=...`
Returns cancellations, refunds, order time/volume summaries, and other agreed operational metrics.

CSV export may use `GET /owner/reports/export?...` and return a streamed file. Keep export scope and columns documented.

## 7. Admin/menu APIs (implementation MVP)

Exact CRUD routes can be added by the owner-management task, but use these resource boundaries:
- `/owner/menu/categories`
- `/owner/menu/items`
- `/owner/tables` and `/owner/tables/{id}/qr` (create/rotate/revoke)
- `/owner/branding`
- `/owner/settings/tax`
- `/owner/staff`

All mutations require permissions, validation, and audit entries. Never expose QR token hashes.

## 8. Webhook

### POST `/webhooks/payments/{provider}`
- Verify provider-specific signature against raw request body before trusting event data.
- Validate merchant/provider order ID, amount, currency, payment ID, and expected local state.
- Persist provider event ID/idempotency marker; duplicate events return success without duplicate capture.
- Return provider-appropriate acknowledgement quickly; process safe retries.
- Do not require guest/staff auth. Protect using signature, rate limiting, and strict parsing.

## 9. Contract change process

1. Agent identifies missing/incorrect contract and provides example request/response plus affected tasks.
2. Orchestrator reviews data model, security and frontend impact.
3. Update this file and `memory.md` decision log before code change.
4. Notify dependent agents and update tests/types.

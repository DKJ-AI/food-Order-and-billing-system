# Security, Privacy & Operational Controls

## 1. Threat model

Protect against unauthorized staff access, cross-restaurant/outlet data leakage, QR token abuse, guest session guessing, forged order prices, duplicate commands, fake payment success, webhook replay, and accidental exposure of customer/payment data.

## 2. Identity and authorization

- Guest: no account; QR token validates a table, then a server-issued guest session credential scopes access to that table session.
- Staff: authenticated accounts; role and outlet membership checked on every request and WebSocket subscription.
- Roles: `OWNER`, `MANAGER`, `WAITER`, `KITCHEN`. Use deny-by-default permissions.
- Owner/manager may manage only assigned restaurant/outlet scope. Waiter handles service requests; kitchen handles kitchen ticket transitions. Avoid broad `is_admin` bypasses.
- Use secure, HttpOnly, SameSite cookies for browser sessions where possible; HTTPS only. If bearer tokens are used, document storage/expiry/rotation and XSS implications.
- Protect login with rate limits, generic errors, secure password hashing, and session revocation.

## 3. QR and guest session

- QR token must be cryptographically random and non-sequential; store a hash, not the raw token.
- Token is revocable and bound to exactly one active table/outlet.
- On QR scan, server creates/resumes a guest session; guest cookie is short-lived but must remain valid for an unpaid open bill. Define rotation/expiry carefully so guests are not locked out during payment.
- Guest credential can access only its session's menu/order/bill/service request endpoints.
- Rate-limit session creation, order placement, waiter requests, and checkout.
- A copied QR can be used by another person near the table; do not treat QR as proof of identity. Staff can revoke/rotate QR and close/reset a table with audit reason.

## 4. Input, pricing, and state validation

- Validate all IDs against the current restaurant/outlet/session scope.
- Ignore client-sent price, tax, discount, total, order status, payment status, staff role, and outlet authorization claims.
- Recompute prices using current active menu data at placement; snapshot all values in order rows.
- Validate option group min/max, duplicate options, quantities, item availability, note length, and dietary field length/allowed kinds.
- Use idempotency keys for order, waiter call, checkout, and provider events.
- Use DB transactions/row locks for checkout and payment transitions.
- Prevent invalid order/service/payment state transitions.

## 5. Payments

- Keep gateway secret keys only in server-side secret storage/env; never expose them in frontend bundle/logs.
- Create provider payment order on server using backend-computed amount/currency.
- Verify webhook signature using raw body and provider's official method; compare provider IDs, amount, currency, and local expected state.
- Use unique provider event/payment IDs and idempotent updates. A browser redirect is never authoritative.
- Do not store card number, CVV, UPI PIN, or raw payment credentials. Use provider-hosted checkout/tokenized flow.
- Redact webhook payloads and sensitive headers from logs.
- Refunds/cash adjustments require explicit permission and an audit reason.

## 6. Application and infrastructure

- HTTPS everywhere; secure headers; strict CORS allowlist; CSRF protection if cookie-authenticated state-changing endpoints need it.
- Parameterized ORM/database queries; output encoding; content-security policy appropriate to payment provider checkout.
- Validate uploaded logo/menu images by MIME, extension, size, and safe processing; store outside app filesystem in production.
- Secrets from environment/secret manager; `.env` excluded from Git; `.env.example` contains placeholders only.
- Least-privilege DB and hosting credentials; database backups encrypted and restore-tested.
- Keep dependencies updated through reviewed PRs and security advisories.
- Rate-limit login, QR session, waiter call, order submit, checkout, report export, and webhook endpoints.

## 7. Privacy and data minimization

- No customer account or personal data required for MVP. Do not request phone/email unless a defined feature needs it and consent/legal basis is established.
- Dietary/allergy details can be sensitive in context; collect only what is needed for the order, restrict staff visibility to those serving/preparing it, and define retention.
- Logs must avoid full guest tokens, passwords, payment data, and raw personal details.
- Provide owner-controlled retention settings only after legal/accounting requirements are reviewed.

## 8. Audit events

Record at minimum: staff login/security events as appropriate, staff/role changes, menu price and availability changes, QR token generation/revocation, tax/service-charge changes, order cancellation/adjustment, manual cash payment, checkout close override, payment/refund action, and report export where sensitive.

Audit record includes actor, outlet, action, entity, timestamp, and reason where relevant. Do not log secrets or full payment payload.

## 9. Launch blockers

- Live payment webhook signature verification not tested against provider sandbox.
- Cross-outlet access tests missing.
- QR revocation and session scope tests missing.
- Secrets committed or frontend-exposed.
- Tax/invoice configuration not reviewed by restaurant accountant.
- Backup restore procedure untested.

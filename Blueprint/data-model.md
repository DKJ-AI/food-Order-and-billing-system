# Canonical Data Model

**Purpose:** Shared schema contract for DB and API agents. Field names are proposed canonical names; implementation may add technical metadata but must not change business meaning without an ADR.

## 1. Conventions

- PostgreSQL; snake_case; UUID primary keys.
- Every tenant-owned operational record includes `restaurant_id` and/or `outlet_id` as appropriate. Enforce relationship consistency.
- All timestamps are timezone-aware UTC (`timestamptz`). Display and report using `outlet.timezone`.
- Store INR as integer paise (`BIGINT` where aggregate size warrants it). Never floating point.
- Use database enums or constrained strings consistently; document migrations.
- Soft-delete/hide catalog records where historical references exist. Do not delete historical orders/payments.
- Secret QR tokens are generated with cryptographic randomness; store only a secure hash when feasible.

## 2. Main entities

### `restaurants`
`id`, `name`, `slug` (unique), `logo_url`, `cover_image_url`, `tagline`, `description`, `brand_primary`, `brand_accent`, `created_at`, `updated_at`, `is_active`.

### `outlets`
`id`, `restaurant_id`, `name`, `timezone` (IANA name; e.g. `Asia/Kolkata`), `currency` (MVP `INR`), `address_json`, `phone`, `ordering_enabled`, `created_at`, `updated_at`, `is_active`.

### `dining_tables`
`id`, `restaurant_id`, `outlet_id`, `label` (e.g. `Table 08`), `area` (optional), `capacity` (optional), `is_active`, `created_at`, `updated_at`.

### `qr_tokens`
`id`, `restaurant_id`, `outlet_id`, `table_id`, `token_hash` (unique), `created_at`, `revoked_at`, `created_by_user_id` (nullable). Regenerate by revoking old token and issuing a new token. Only token plaintext is shown at creation/print time.

### `menu_categories`
`id`, `restaurant_id`, `outlet_id`, `name`, `description`, `sort_order`, `is_active`, `created_at`, `updated_at`.

### `menu_items`
`id`, `restaurant_id`, `outlet_id`, `category_id`, `name`, `short_description`, `description`, `image_url`, `base_price_paise`, `tax_rate_bps` (configurable basis points), `tax_inclusive` (boolean), `dietary_labels_json` (declared only), `allergens_json` (declared only), `is_available`, `sort_order`, `created_at`, `updated_at`, `archived_at`.

### `option_groups`
`id`, `restaurant_id`, `outlet_id`, `menu_item_id`, `name`, `is_required`, `min_select`, `max_select`, `sort_order`, `is_active`.

### `option_choices`
`id`, `restaurant_id`, `outlet_id`, `option_group_id`, `name`, `price_delta_paise`, `is_available`, `sort_order`.

### `staff_users`
`id`, `name`, `email` (normalized unique according to auth policy), `phone` (optional), `password_hash` or external auth subject, `is_active`, `created_at`, `updated_at`, `last_login_at`.

### `staff_memberships`
`id`, `restaurant_id`, `outlet_id`, `user_id`, `role` (`OWNER`, `MANAGER`, `WAITER`, `KITCHEN`), `is_active`, `created_at`. Add unique constraint on `(outlet_id, user_id, role)` or equivalent membership policy.

### `table_sessions`
`id`, `restaurant_id`, `outlet_id`, `table_id`, `session_number` or display code, `status` (`OPEN`, `CHECKOUT_PENDING`, `CLOSED`), `opened_at`, `checkout_started_at`, `closed_at`, `closed_by_user_id` (nullable), `guest_session_secret_hash` or secure session reference, `created_at`, `updated_at`.

Invariant: at most one non-closed session per table for MVP. Keep unpaid sessions open. The guest credential is scoped to this session and cannot access another table.

### `orders`
`id`, `restaurant_id`, `outlet_id`, `table_session_id`, `order_number` (human-readable, unique per outlet/business date as designed), `status` (`PLACED`, `ACCEPTED`, `PREPARING`, `READY`, `SERVED`, `CANCELLED`), `customer_note` (optional), `dietary_requirements_json` (structured allergy/preference entries), `subtotal_paise`, `tax_total_paise`, `discount_total_paise` (zero/no discounts in MVP), `service_charge_paise`, `total_paise`, `placed_at`, `accepted_at`, `preparing_at`, `ready_at`, `served_at`, `cancelled_at`, `cancel_reason`, `created_at`, `updated_at`, `created_by_guest_session_id`.

### `order_items`
`id`, `order_id`, `menu_item_id` (nullable if later archived), `item_name_snapshot`, `quantity`, `unit_base_price_paise`, `options_snapshot_json`, `item_note`, `declared_dietary_snapshot_json`, `declared_allergen_snapshot_json`, `tax_rate_bps_snapshot`, `tax_inclusive_snapshot`, `line_subtotal_paise`, `line_tax_paise`, `line_total_paise`, `created_at`.

Invariant: the snapshot fields are immutable after order placement except through an explicit audited correction workflow. Do not rely on current `menu_items` for past receipts.

### `service_requests`
`id`, `restaurant_id`, `outlet_id`, `table_session_id`, `table_id`, `request_type` (`ASSISTANCE`, `WATER`, `CUTLERY`, `ITEM_QUESTION`, `BILL`, `OTHER`), `note`, `status` (`OPEN`, `ACKNOWLEDGED`, `RESOLVED`), `created_at`, `acknowledged_at`, `acknowledged_by_user_id`, `resolved_at`, `resolved_by_user_id`, `resolution_note`.

### `payment_attempts`
`id`, `restaurant_id`, `outlet_id`, `table_session_id`, `provider` (e.g. configured provider key), `provider_order_id`, `provider_payment_id`, `provider_event_id` (unique when available), `idempotency_key`, `amount_paise`, `currency`, `status` (`CREATED`, `PENDING`, `CAPTURED`, `FAILED`, `CANCELLED`, `EXPIRED`, `PARTIALLY_REFUNDED`, `REFUNDED`), `provider_payload_safe_json` (redacted), `failure_code`, `failure_message_safe`, `created_at`, `updated_at`, `captured_at`, `refunded_at`.

Constraints: unique provider order/payment/event IDs where non-null; unique scoped idempotency key; amount/currency verified against the server-created intent.

### `invoices`
`id`, `restaurant_id`, `outlet_id`, `table_session_id`, `invoice_number`, `issued_at`, `subtotal_paise`, `tax_lines_json`, `discount_paise`, `service_charge_paise`, `rounding_paise`, `total_paise`, `amount_paid_paise`, `currency`, `seller_snapshot_json`, `buyer_snapshot_json` (optional/minimal), `line_items_snapshot_json`, `document_url` (optional), `created_at`.

Invoice snapshots are immutable. Invoice number uniqueness is scoped to the legally/operationally appropriate outlet/series. Confirm statutory invoice requirements with the restaurant's accountant before production.

### `tax_configs`
`id`, `restaurant_id`, `outlet_id`, `name`, `tax_rate_bps`, `price_includes_tax`, `is_active`, `effective_from`, `effective_to`, `created_by_user_id`, `created_at`.

### `audit_logs`
`id`, `restaurant_id`, `outlet_id` (nullable for tenant-level event), `actor_user_id`, `actor_type`, `action`, `entity_type`, `entity_id`, `before_json` (redacted), `after_json` (redacted), `reason`, `ip_hash_or_safe_metadata`, `created_at`.

## 3. Relationships (high level)

```text
Restaurant 1─N Outlets
Outlet 1─N DiningTables, MenuCategories, MenuItems, StaffMemberships
DiningTable 1─N QrTokens, TableSessions
TableSession 1─N Orders, ServiceRequests, PaymentAttempts, Invoices
Order 1─N OrderItems
MenuItem 1─N OptionGroups 1─N OptionChoices
StaffUser 1─N StaffMemberships
```

## 4. Required constraints and indexes

- Unique restaurant slug; unique `(outlet_id, table_id, status-active-session)` through a partial unique index for OPEN/CHECKOUT_PENDING.
- Unique QR `token_hash`; revoked tokens never authenticate.
- Unique human order number within defined outlet/series.
- Index orders by `(outlet_id, placed_at)`, `(outlet_id, status, placed_at)`, and `table_session_id`.
- Index open service requests by `(outlet_id, status, created_at)`.
- Index payment attempts by `(outlet_id, table_session_id, created_at)` and unique provider IDs.
- Index report date fields; use materialized/summary tables only after query performance requires them.
- Foreign keys enforce restaurant/outlet/table/session consistency; application checks are not enough.

## 5. Price and tax calculation requirements

- API accepts item ID, quantity, option-choice IDs, item note; it does not accept trusted unit price or total.
- Backend loads active price/options and validates outlet availability.
- Store base and option price snapshots in paise.
- Store tax rate basis points and price-inclusive flag snapshots for historical explanation.
- Calculation and rounding rule must be centralized in one tested service and documented. Use decimal/rational arithmetic or integer arithmetic with explicit rounding; never binary float.
- No blanket tax percentage in frontend. Use the configured tax lines returned by bill API.
- Support discounts/service charge only when enabled/configured; display each line transparently. Do not invent legal applicability.

## 6. Retention and privacy

- Keep financial/order records according to the restaurant's legal/accounting retention policy.
- Keep guest data minimal; MVP does not require guest name, phone, or email.
- Redact secrets, payment tokens, full webhook payloads, and personal data from audit/error logs.
- Define a retention/cleanup policy for expired guest session credentials and stale unactioned requests.

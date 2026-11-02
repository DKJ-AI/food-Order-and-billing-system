# Realtime Events & Recovery Contract

## 1. Purpose

Use WebSockets to reduce delay for kitchen tickets, waiter requests, guest order tracking, and bill/payment updates. REST remains the source of truth. Every screen must recover from disconnects by refetching state and using polling fallback.

## 2. Connections and authorization

Suggested endpoints:
- `WSS /api/v1/ws/guest` — guest credential scoped to one `table_session_id`.
- `WSS /api/v1/ws/staff` — authenticated staff session; server authorizes outlet and role before subscription.

Alternative single endpoint is acceptable only if the same authorization scopes are preserved. Never accept arbitrary client-supplied outlet IDs as proof of access. A guest cannot subscribe to another session. Kitchen cannot subscribe to owner reports/payment details.

## 3. Event envelope

```json
{
  "event_id": "uuid",
  "type": "order.created",
  "occurred_at": "2026-11-02T10:02:00Z",
  "restaurant_id": "uuid",
  "outlet_id": "uuid",
  "table_session_id": "uuid",
  "payload": {}
}
```

- `event_id` is unique; clients may ignore duplicate IDs.
- Keep payload minimal; send IDs + safe display data or instruct client to refetch.
- Never send payment secrets, full provider payload, staff password/session token, or unnecessary guest details.

## 4. Event types

| Event | Recipient | Payload minimum |
|---|---|---|
| `order.created` | Kitchen; guest session | order ID/number, table label, status, placed time, safe ticket summary |
| `order.status_changed` | Guest session; kitchen | order ID, order number, previous/new status, actor display role, changed time |
| `service_request.created` | Waiter outlet | request ID, table label, type, note, created time |
| `service_request.updated` | Waiter outlet; guest session | request ID, status, changed time, safe note |
| `session.bill_updated` | Guest session; authorized waiter/manager as needed | session ID, bill version, balance due, currency |
| `payment.updated` | Guest session; authorized manager | payment ID, status, amount, currency, timestamp |
| `menu.item_availability_changed` | Guest clients for affected outlet | item ID, availability, updated time |
| `connection.ready` | Connected client | server time, supported event version |

## 5. Client behavior

1. Fetch current state via REST before/when opening page.
2. Connect WebSocket with credential; show connected/reconnecting indicator.
3. On event, validate event type and invalidate/refetch relevant query; do not blindly trust stale event payload as full state.
4. On disconnect, reconnect with exponential backoff + jitter and a maximum retry interval.
5. While disconnected, poll relevant REST list/detail at a conservative interval (e.g. 5–10 seconds for active KDS; tune after pilot).
6. On reconnect, refetch full relevant state to close any event gap.
7. Deduplicate event IDs and handle out-of-order events by refetching server state.

## 6. Server behavior

- Authenticate connection at handshake and authorize every subscription.
- Limit connections per guest session/user/outlet and payload size.
- Heartbeat/ping and disconnect dead connections.
- Publish only after database commit.
- For initial one-worker deployment, in-process connection manager is acceptable. For multiple workers/instances, use Redis Pub/Sub or a transactional outbox/event relay so one worker can reach clients connected to another.
- If an event is lost, REST refetch restores correct state. Do not promise exactly-once delivery.

## 7. Reliability and security tests

- Guest cannot subscribe to another table session.
- Kitchen cannot join unauthorized outlet or owner-only channel.
- Reconnect causes state refresh and no missing ticket.
- Duplicate/out-of-order event does not regress displayed order state.
- API command works even when WebSocket is offline.
- Sensitive payment fields never appear in event payload.

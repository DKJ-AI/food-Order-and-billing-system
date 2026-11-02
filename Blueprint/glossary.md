# Project Glossary

| Term | Meaning |
|---|---|
| Restaurant | Tenant/brand that owns outlets and configuration |
| Outlet | Physical restaurant location; owns tables, orders, staff scope, timezone, tax config |
| Table | Physical dining table represented by a public label and one active QR token |
| QR token | Opaque revocable credential embedded in a table QR code |
| Guest session | Server-scoped guest access for one table session; not a customer account |
| Table session | Open dining visit at a table; aggregates multiple orders and one consolidated bill |
| Order | One submission of items to the kitchen within a table session |
| Order item | Historical line snapshot of item, options, price, notes, and tax basis |
| KDS | Kitchen Display System: live kitchen ticket queue |
| Service request | Guest call for waiter assistance, e.g. water, cutlery, bill |
| Bill | Current aggregate amount due for a table session |
| Payment attempt | One gateway checkout or authorized cash collection attempt |
| Invoice/receipt | Immutable financial snapshot created after successful payment/authorized close |
| Idempotency key | Client-supplied unique key so retrying a command does not duplicate its effect |
| Webhook | Server-to-server payment-provider event; verified before changing payment state |
| Tenant isolation | Ensuring one restaurant/outlet cannot access another's records |
| Paise | One hundredth of an Indian rupee; store ₹1.00 as 100 paise |
| Basis points (bps) | 1/100 of one percent; e.g. 500 bps represents 5% |

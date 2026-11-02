# Project Memory & Build Ledger

**Purpose:** Compact, current source of truth for what has been decided, built, tested, and remains. Agents should read this first, then only the linked task context.  
**Rule:** The orchestrator owns this file. Agents report results; they do not rewrite status for other agents.

## 1. Current project state

- Product: QR-first restaurant ordering, kitchen display, waiter calls, billing/payment, owner reports.
- Blueprint: documentation package created; application code not started.
- Current phase: Phase 0 — architecture and task planning.
- Current release target: MVP web/PWA for one restaurant brand, data model tenant/outlet-ready.
- Default currency: INR, stored in paise.
- Default timezone: configured per outlet; initial India deployment uses `Asia/Kolkata`.

## 2. Accepted architecture decisions

| ID | Decision | Reason / impact |
|---|---|---|
| D-001 | React + TypeScript + Vite frontend | One responsive role-based app |
| D-002 | Python FastAPI backend | Typed REST API and WebSocket support |
| D-003 | PostgreSQL + SQLAlchemy + Alembic | Transactional relational data and migrations |
| D-004 | Modular monolith for MVP | Lower operational and integration cost than microservices |
| D-005 | Table session aggregates multiple orders and owns the bill | Supports repeat ordering before checkout |
| D-006 | Money stored as integer paise | Prevents floating-point rounding errors |
| D-007 | Backend calculates price/tax/total; gateway webhook confirms payment | Prevents client tampering and false payment success |
| D-008 | Guest QR access without customer account; staff authenticated | Low-friction customer journey, protected operations |
| D-009 | Dietary/allergy messages are warnings requiring staff confirmation | No unsafe guarantee about cross-contact |
| D-010 | REST for commands/queries; WebSocket for live events with polling fallback | Clear, recoverable realtime model |
| D-011 | Payment gateway isolated behind provider adapter | Provider can be selected without rewriting order/bill logic |
| D-012 | Tax/service charge configurable by outlet; not hard-coded | Supports restaurant/accountant configuration |

## 3. Build status

Statuses: `NOT STARTED`, `IN PROGRESS`, `BLOCKED`, `DONE`, `NEEDS REVIEW`.

| Task | Owner/agent | Status | Output / commit / notes |
|---|---|---|---|
| DOC-01 | Architect | DONE | All blueprint markdown files |
| BASE-01 | Scaffold agent | DONE | Monorepo skeleton (apps/web, apps/api, docker-compose.yml, health tests) |
| BASE-02 | DB agent | NOT STARTED | SQLAlchemy models and Alembic baseline |
| BASE-03 | UI foundation agent | NOT STARTED | Theme, app shell, shared primitives |
| API-01 | Backend core agent | NOT STARTED | Public menu/session/order APIs |
| API-02 | Staff API agent | NOT STARTED | Auth, KDS, waiter requests, staff APIs |
| API-03 | Billing agent | NOT STARTED | Bill calculation, payment adapter/webhooks |
| API-04 | Reporting agent | NOT STARTED | Owner reporting APIs |
| UI-01 | Customer UI agent | NOT STARTED | QR, menu, cart, order tracking, bill UI |
| UI-02 | Kitchen UI agent | NOT STARTED | KDS |
| UI-03 | Waiter UI agent | NOT STARTED | Service request console |
| UI-04 | Owner UI agent | NOT STARTED | Dashboard and management UI |
| QA-01 | QA agent | NOT STARTED | Automated critical-path tests |
| OPS-01 | Deployment agent | NOT STARTED | Docker/deployment/runbook |
| INT-01 | Integration lead | NOT STARTED | End-to-end integration and release gate |

## 4. Open decisions (must be resolved before the relevant task)

| Question | Default for MVP | Resolve by |
|---|---|---|
| Payment gateway | Keep provider adapter; select one after merchant account/API access | Before API-03 |
| Tax rates and price inclusion | Owner-configured; accountant validates; no hard-coded assumptions | Before real billing pilot |
| Restaurant/station printer | Excluded; KDS screen first | Before any print feature |
| Split bills | Excluded; one full-session payment | Before billing contract expansion |
| Multiple outlets | Schema supports outlet scope; initial UI may deploy one outlet | Before multi-outlet pilot |
| Guest session expiry | Configurable; do not expire while bill unpaid | Before API-01 |

## 5. Known risks / guardrails

- Payment provider credentials and webhook signature verification are launch blockers for live payments.
- Internet/Wi-Fi loss must not silently lose a placed order; show stale/retry states and use idempotency keys.
- A QR image can be copied. The table token grants guest ordering access to that table, not staff access; provide token revocation and a staff-controlled session reset.
- Allergies require staff confirmation; the app cannot certify kitchen cross-contact safety.
- Tax/invoice rules depend on restaurant registration and applicable law. Owner configuration must be reviewed by a qualified accountant before production.
- Realtime should degrade gracefully to polling; do not make order placement dependent on an open WebSocket.

## 6. Latest handoff note

**Last update:** 2026-11-02  
**Completed:** BASE-01 Monorepo scaffold (apps/web, apps/api, Docker Compose, root scripts, health endpoints/pages, tests passing).  
**Next action:** Await owner review for BASE-01 before starting BASE-02 (Database models and migrations) or BASE-03 (UI foundation).

## 7. Agent completion report template

Each agent must return this compact report to the orchestrator:

```text
Task ID:
Status: DONE / NEEDS REVIEW / BLOCKED
Files created/changed:
What was implemented:
Tests run + results:
Contract deviations: NONE or exact proposal
Known limitations:
Next dependency:
```

Orchestrator updates the task status and appends a dated entry below.

## 8. Build log (append only)

- 2026-11-02 — DOC-01: Created the architecture and multi-agent blueprint. No application code has been generated or tested yet.
- 2026-11-02 — BASE-01: Created the monorepo skeleton (apps/web Vite+React+TS, apps/api FastAPI, docker-compose.yml for PostgreSQL, .gitignore, .env.example, root package.json, health endpoints and smoke tests). Vitest unit tests (1/1 passed) and Pytest health endpoint test (1/1 passed) verified. Frontend build succeeded.

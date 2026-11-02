# Task Breakdown & Copy-Ready AI Agent Prompts

**Goal:** Small, bounded implementation tasks. Give an agent only its task prompt and the listed context files. Do not ask an agent to read the whole repository.  
**Contract rule:** `api-contract.md` and `data-model.md` are canonical. Agents must not silently alter them.

## 1. How to assign work

**Free-plan clarification:** The roster below contains project roles, not 14 separate AI accounts. Reuse the same coding AI for different roles by changing the task prompt. Start with one worker; after Git and the scaffold are stable, use no more than two concurrent workers. Read `free-plan-workflow.md` before choosing tools.

- One task = one focused deliverable and one clear owner.
- Agents work in separate branches/worktrees. Never let two agents edit the same files concurrently.
- The orchestrator merges tasks in dependency order and updates `memory.md`.
- A task is not `DONE` until its acceptance checks pass and the agent returns the completion report in `memory.md` format.
- If a prompt requires a missing interface, agent reports the blocker; it does not invent a new contract.

## 2. Agent roster and file ownership

| Agent | Owns | Must not edit |
|---|---|---|
| Orchestrator / architect | `memory.md`, contract decisions, merge order | Feature implementation without explicit assignment |
| Scaffold agent | Root config, `apps/web`/`apps/api` skeleton, Docker Compose baseline | Feature source files owned by other agents |
| DB agent | Backend ORM models, Alembic migrations, DB constraints | API response semantics without contract change |
| UI foundation agent | Shared theme, primitives, app shell, route skeleton | Feature page implementations |
| Backend core agent | Public menu, QR session, cart/order services and routes | Payment-provider implementation; reports queries |
| Staff API agent | Staff auth/RBAC, KDS and waiter endpoints, WebSocket auth/dispatch | Public order pricing; payment callbacks |
| Billing agent | Bill calculator, payment adapter, payment/webhook routes | Menu/order creation semantics |
| Reporting agent | Owner reporting queries and report routes | Customer/staff UI |
| Customer UI agent | Guest QR/menu/cart/order/bill screens | Shared design tokens/API contracts |
| Kitchen UI agent | Kitchen display frontend | Shared foundation unless agreed |
| Waiter UI agent | Waiter frontend | Shared foundation unless agreed |
| Owner UI agent | Owner dashboard and management frontend | Shared foundation unless agreed |
| QA agent | Test suites, fixtures, test documentation | Production behavior except testability fixes approved by lead |
| Deployment agent | Docker, CI, deployment and operational docs | Application business logic |
| Integration lead | Cross-module wiring, end-to-end acceptance | Silent contract changes |

## 3. Dependency graph / execution waves

```text
DOC-01 (this blueprint)
   ├── BASE-01 Repository scaffold ──┬── BASE-02 Database models/migrations
   │                                ├── BASE-03 UI foundation
   │                                └── OPS-01 Local Docker/CI baseline
   ├── API-01 Public menu/session/order ──┬── UI-01 Customer UI
   │                                     ├── API-02 Staff APIs/realtime ── UI-02 KDS
   │                                     │                              └── UI-03 Waiter
   │                                     ├── API-03 Billing/payment ───── UI-01 bill integration
   │                                     └── API-04 Reports ───────────── UI-04 Owner
   └────────────────────────────────────────────────────────── QA-01 ── INT-01 release gate
```

Parallel after scaffold: `BASE-02` and `BASE-03`. Backend feature tasks can start once models and contracts are stable. Frontend feature agents can build against typed API mocks after the UI foundation; they must switch to real API before completion.

## 4. Common prompt rules

Append this short rule to every task prompt:

> Work only on the listed scope. Read only the named context files and directly imported code. Follow existing conventions. Do not change shared API/data contracts; report a proposed change with reason and affected tasks instead. Do not add unrelated dependencies. Run focused tests/type checks. Return changed-file list, test results, deviations, limitations, and next dependency. Do not claim tests passed unless executed.

## 5. Task cards and prompts

### DOC-01 — Blueprint and contract baseline
**Status:** Done.  
**Outputs:** All Markdown files in this blueprint.  
**Acceptance:** Every implementation task has a bounded prompt, ownership, dependencies, and source-of-truth docs.

### BASE-01 — Monorepo scaffold
**Status:** DONE  
**Owner:** Scaffold agent  
**Depends on:** DOC-01  
**Read:** `README.md`, `architecture.md`, `deployment.md`, `agent-workflow.md`  
**Owns:** Root configs, `apps/web` skeleton, `apps/api` skeleton, `.gitignore`, base `docker-compose.yml` only.

**Prompt:**
```text
Create the monorepo skeleton for the restaurant QR ordering MVP. Read README.md, architecture.md, deployment.md, agent-workflow.md only. Use React+TypeScript+Vite in apps/web and FastAPI in apps/api. Add root scripts, lint/type-check/test placeholders, .gitignore, Docker Compose for PostgreSQL, and minimal health endpoints/pages. Do not implement features or DB models. Keep setup steps in README. Run available checks and report exact results.
```
**Acceptance:** Web and API start locally; API health route works; Postgres service starts; commands are documented; no feature logic included.

### BASE-02 — Database models and migrations
**Owner:** DB agent  
**Depends on:** BASE-01  
**Read:** `data-model.md`, `architecture.md`, `security.md`  
**Owns:** `apps/api` database package, ORM models, Alembic migrations, DB tests.

**Prompt:**
```text
Implement the PostgreSQL schema from data-model.md using SQLAlchemy 2 and Alembic. Read data-model.md, architecture.md, security.md. Create models, relationships, constraints, indexes, enums, and one initial migration. Store money as integer paise; include tenant/outlet scope and immutable order snapshots. Add focused model/constraint tests. Do not build API routes or invent fields that change the spec; list any schema ambiguity before proceeding.
```
**Acceptance:** Fresh DB migration succeeds; model tests cover unique constraints, foreign keys, tenant/outlet links, order snapshots, and money types.

### BASE-03 — Frontend foundation and design system
**Owner:** UI foundation agent  
**Depends on:** BASE-01  
**Read:** `frontend.md`, `content.md`, `README.md`  
**Owns:** shared UI primitives, theme, app shell, route skeleton.

**Prompt:**
```text
Build the React/TypeScript frontend foundation from frontend.md. Read frontend.md, content.md, README.md. Set up route groups for guest, kitchen, waiter, owner and staff login; implement theme tokens, responsive AppShell, buttons, inputs, badges, cards, dialogs/drawers, toasts, loading/empty/error states, and accessible navigation. Use mock route guards only as UX scaffolding. Do not build feature pages or API logic. Add component tests and report them.
```
**Acceptance:** All route shells render; tokens are centralized; primitives are keyboard accessible; responsive smoke test passes.

### OPS-01 — Local developer environment and CI baseline
**Owner:** Deployment agent  
**Depends on:** BASE-01  
**Read:** `deployment.md`, `testing.md`, `README.md`  
**Owns:** CI workflow, Dockerfiles, local environment docs, health checks.

**Prompt:**
```text
Set up repeatable local development and CI for the existing scaffold. Read deployment.md, testing.md, README.md. Add production-minded Dockerfiles, env validation, API/web health checks, and CI jobs for backend tests, frontend lint/type-check/tests, and build. Never commit secrets. Keep changes limited to tooling and deployment files; do not change application behavior. Verify every command locally where possible.
```
**Acceptance:** Clean clone can be started using documented steps; CI runs on push/PR; no secrets in repo.

### API-01 — Public QR, menu, table session, and order APIs
**Owner:** Backend core agent  
**Depends on:** BASE-01, BASE-02  
**Read:** `architecture.md`, `data-model.md`, `api-contract.md`, `security.md`, `content.md`  
**Owns:** public API routers, menu/session/order services, DTOs, tests.

**Prompt:**
```text
Implement public guest APIs for QR menu lookup, opening/resuming a table session, reading session summary, and placing an order. Read architecture.md, data-model.md, api-contract.md, security.md, content.md. Match endpoint paths and JSON exactly to api-contract.md. Validate QR token, item availability/options, quantity, dietary fields, and session state. Recalculate totals server-side; snapshot item data; use idempotency. Add API tests. Do not implement payment or staff APIs.
```
**Acceptance:** Contract tests pass for valid/invalid QR, session reuse, menu availability, invalid options, duplicate idempotency key, and order snapshot creation.

### API-02 — Staff authentication, RBAC, KDS and waiter APIs
**Owner:** Staff API agent  
**Depends on:** BASE-02, API-01  
**Read:** `architecture.md`, `data-model.md`, `api-contract.md`, `realtime.md`, `security.md`  
**Owns:** staff auth/RBAC, kitchen/staff routers, realtime event dispatcher.

**Prompt:**
```text
Implement staff login/session auth, outlet-scoped role checks, kitchen order queue/status transitions, waiter service-request queue/acknowledge/resolve, and WebSocket event delivery. Read architecture.md, data-model.md, api-contract.md, realtime.md, security.md. Follow exact route/event contracts. Enforce permissions server-side and audit sensitive actions. Add role-isolation, state-transition, and WebSocket tests. Do not implement customer order creation or payment.
```
**Acceptance:** Kitchen sees only authorized outlet tickets; waiter can resolve requests; invalid transitions and cross-outlet access are rejected; live events are tested.

### API-03 — Bill calculation and payment integration
**Owner:** Billing agent  
**Depends on:** BASE-02, API-01  
**Read:** `architecture.md`, `data-model.md`, `api-contract.md`, `security.md`, `testing.md`  
**Owns:** billing service, payment adapter interface/implementation, checkout and webhook routers.

**Prompt:**
```text
Implement session-level bill calculation and payment lifecycle behind a provider adapter. Read architecture.md, data-model.md, api-contract.md, security.md, testing.md. Use integer paise, configured tax/charges, immutable order snapshots, transaction locking, idempotency, verified signed webhooks, and separate order/payment states. Keep provider-specific SDK calls isolated. If credentials/provider are unavailable, finish a tested fake adapter and clearly mark live integration blocked. Never trust browser payment success.
```
**Acceptance:** Bill totals reconcile; duplicate webhook is harmless; invalid signatures fail; failed payment reopens checkout; only verified capture marks paid/closes session; tests include race/retry cases.

### API-04 — Owner reports and summary APIs
**Owner:** Reporting agent  
**Depends on:** BASE-02, API-01, API-03  
**Read:** `architecture.md`, `data-model.md`, `api-contract.md`, `security.md`  
**Owns:** reporting query/service/router and report tests.

**Prompt:**
```text
Build owner/manager reporting APIs from api-contract.md. Read architecture.md, data-model.md, api-contract.md, security.md. Support outlet-scoped date range, gross/net sales, paid orders, average order value, item/category performance, hourly trend, payment mix, cancellations/refunds, and CSV export if contracted. Use outlet timezone and clearly defined metric semantics. Add aggregation tests and role/outlet isolation tests. Do not build frontend charts.
```
**Acceptance:** Metrics reconcile against seeded orders/payments; date boundaries respect outlet timezone; unauthorized roles/outlets are rejected.

### UI-01 — Customer QR, menu, cart, order tracking and bill UI
**Owner:** Customer UI agent  
**Depends on:** BASE-03; API-01 contract; API-03 contract for payment  
**Read:** `frontend.md`, `content.md`, `api-contract.md`, `realtime.md`  
**Owns:** guest route pages and customer feature components/services.

**Prompt:**
```text
Build the complete guest QR journey: branded table landing, menu/categories/search, item customization, cart, dietary/allergy form, order confirmation/tracking, waiter-call UI, consolidated bill, and payment status. Read frontend.md, content.md, api-contract.md, realtime.md. Follow contract types and safety copy exactly. Include loading/empty/error/offline states, keyboard/touch support, idempotent submit behavior, and responsive tests. Use mocks only until APIs are available; then connect real endpoints.
```
**Acceptance:** Phone-width journey completes with API integration; totals display server values; browser redirect alone never displays paid; dietary disclaimer is visible.

### UI-02 — Kitchen Display System frontend
**Owner:** Kitchen UI agent  
**Depends on:** BASE-03, API-02  
**Read:** `frontend.md`, `api-contract.md`, `realtime.md`, `content.md`  
**Owns:** `/kitchen` pages/components and KDS services.

**Prompt:**
```text
Build the tablet-first KDS described in frontend.md. Read frontend.md, api-contract.md, realtime.md, content.md. Show incoming tickets, table/order number, elapsed time, quantities, modifiers, item notes, dietary/allergy alerts, and valid status actions. Add station/status filters, connection/reconnect state, polling fallback, and no-auto-disappear behavior. Do not expose payments/reports. Add component and realtime behavior tests.
```
**Acceptance:** New ticket appears live or via fallback; ticket details are readable; status transitions match API; ready tickets remain visible until explicit action.

### UI-03 — Waiter service console
**Owner:** Waiter UI agent  
**Depends on:** BASE-03, API-02  
**Read:** `frontend.md`, `api-contract.md`, `realtime.md`  
**Owns:** `/waiter` pages/components and waiter services.

**Prompt:**
```text
Build the waiter console from frontend.md. Read frontend.md, api-contract.md, realtime.md. Prioritize new requests sorted oldest first, table/reason/time, acknowledge and resolve actions, optional internal note, and connection state. Show only role-permitted table context. Add filters, empty/error states, responsive layout, and tests for action states. Do not build kitchen or owner screens.
```
**Acceptance:** New requests appear live/polling; acknowledge/resolve update correctly; duplicate requests are visually prevented; controls are touch accessible.

### UI-04 — Owner dashboard and management UI
**Owner:** Owner UI agent  
**Depends on:** BASE-03, API-04  
**Read:** `frontend.md`, `content.md`, `api-contract.md`  
**Owns:** `/owner` dashboard, reports, and contracted management pages.

**Prompt:**
```text
Build the owner dashboard and MVP management screens from frontend.md. Read frontend.md, content.md, api-contract.md. Display gross/net sales with definitions, order count, average order value, open sessions, sales trends, top items/categories, hourly trend, payment mix, and cancellation/refund data. Add date/outlet filters, loading/empty/error states, accessible chart equivalents, and export action if contracted. Do not fabricate metrics or add fake production data.
```
**Acceptance:** API-backed metrics render and filter correctly; date/time labels are clear; no static mock values remain in production screens.

### QA-01 — Critical-path test suite
**Owner:** QA agent  
**Depends on:** API-01, API-02, API-03, UI-01, UI-02, UI-03  
**Read:** `testing.md`, `api-contract.md`, `security.md`, `frontend.md`  
**Owns:** integration/e2e tests, test fixtures, QA report.

**Prompt:**
```text
Create a critical-path automated test suite. Read testing.md, api-contract.md, security.md, frontend.md. Cover QR→menu→session→order→KDS status→waiter call→bill→payment webhook→receipt, plus duplicate submits/webhooks, failed payment, invalid QR, role/outlet isolation, unavailable items, and allergy warning display. Use deterministic gateway mocks. Do not weaken assertions to make tests pass; report reproducible failures.
```
**Acceptance:** Critical flows run in CI; tests are deterministic; failures identify the broken contract/module.

### INT-01 — Integration and MVP release gate
**Owner:** Integration lead  
**Depends on:** all implementation tasks  
**Read:** `README.md`, `memory.md`, `architecture.md`, `api-contract.md`, `testing.md`, `deployment.md`, `security.md`  
**Owns:** integration wiring, release checklist, unresolved issue report.

**Prompt:**
```text
Integrate completed modules without changing contracts silently. Read README.md, memory.md, architecture.md, api-contract.md, testing.md, deployment.md, security.md. Verify env/config, migrations, role guards, WebSocket fallback, payment webhook verification, report reconciliation, and customer-to-kitchen-to-payment flow. Run CI and a production-build smoke test. Fix only integration defects; propose any contract changes separately. Update memory.md with evidence and remaining blockers.
```
**Acceptance:** All critical-path tests pass; no P0/P1 security or payment defects; deployment checklist complete; owner accepts unresolved limitations.

## 6. Suggested context packs (minimize token use)

| Task family | Always read | Add only when needed |
|---|---|---|
| Frontend customer | `frontend.md`, `api-contract.md` | `content.md`, `realtime.md` |
| Frontend staff | `frontend.md`, `api-contract.md` | `realtime.md`, `security.md` |
| Backend public | `architecture.md`, `data-model.md`, `api-contract.md` | `security.md`, `content.md` |
| Backend staff | `architecture.md`, `data-model.md`, `api-contract.md` | `realtime.md`, `security.md` |
| Billing | `architecture.md`, `data-model.md`, `api-contract.md` | `security.md`, `testing.md` |
| Reports | `architecture.md`, `data-model.md`, `api-contract.md` | `security.md` |
| QA | `testing.md`, `api-contract.md` | `security.md`, `frontend.md` |
| Deployment | `deployment.md`, `testing.md` | `README.md` |

## 7. Task prompt budget rule

Keep task-specific prompts roughly 70–130 words, excluding the common rule. Never paste all documentation into a prompt. Attach the files or give the agent the exact path list. If the agent asks to inspect the whole project, direct it first to its task's context pack and ownership table.

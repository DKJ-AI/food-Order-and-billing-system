# Restaurant QR Ordering & Billing System

**Status:** Architecture and agent execution blueprint  
**Document version:** 1.0  
**Product type:** Multi-role, QR-first restaurant ordering, kitchen operations, billing, and reporting system  
**Initial market assumption:** India; INR; configurable tax and payment provider

## 1. Product goal

A guest scans a table QR code, sees a branded digital menu, places one or more orders, communicates dietary requirements, calls a waiter, tracks preparation, views the table bill, and pays. Kitchen staff receive live tickets. Waiters handle service requests. Owners/managers monitor sales and operations.

The first release is a responsive web app/PWA. Customers do not need to install an app or create an account.

## 2. MVP scope

### Must ship
- Table-specific QR entry with restaurant branding and table identification.
- Mobile-first menu: categories, item details, photos, prices, availability, variants/add-ons, search, cart.
- Guest table session; one table session can contain multiple orders.
- Order placement with item notes and session-level dietary requirements/allergy alerts.
- Live order status for the customer.
- Call-waiter request with a reason and optional note.
- Kitchen Display System (KDS): live incoming orders, table/order number, elapsed time, item notes and dietary alerts, status updates.
- Waiter console: open service requests and acknowledgement/resolution.
- Session bill, tax/charge calculation, digital payment integration, payment confirmation, receipt/invoice view.
- Owner dashboard: sales, order count, average order value, item/category performance, payment mix, cancellations/refunds, date filters.
- Staff authentication, role permissions, audit trail for sensitive actions.

### Not in MVP (design for extension)
- Native Android/iOS apps; customer loyalty; coupons; customer accounts; delivery; inventory/procurement; POS hardware integrations; multi-language content authoring; table reservation; split bills and item-level split payments; multi-gateway smart routing; advanced forecasting; franchise-level consolidation.

## 3. Fixed technology direction

| Layer | Decision | Reason |
|---|---|---|
| Customer/staff UI | React + TypeScript + Vite, responsive PWA | One codebase, role-based route groups, good ecosystem |
| UI styling | Tailwind CSS + accessible component primitives | Fast, consistent implementation; design tokens live in `frontend.md` |
| Server state | TanStack Query or equivalent | Caching, invalidation, polling fallback |
| Backend | Python + FastAPI | Clear typed API, async support, Python-friendly business logic |
| ORM/migrations | SQLAlchemy 2 + Alembic | Explicit models and versioned schema |
| Database | PostgreSQL | Transactions, reporting queries, reliable relational constraints |
| Realtime | FastAPI WebSockets; Redis Pub/Sub when multiple API workers are deployed | Live KDS and waiter updates; keep MVP deploy simple |
| Payments | Provider adapter; select Razorpay/Cashfree after merchant onboarding | Provider-specific code stays isolated |
| Hosting | Docker; managed PostgreSQL; HTTPS reverse proxy | Portable deployment |
| Tests | Pytest + HTTPX; Vitest + React Testing Library; Playwright smoke flows | API, UI, and end-to-end coverage |

Do not pin package versions in these design docs. At implementation time choose current mutually compatible stable versions, record them in lockfiles, and do not casually upgrade them mid-task.

## 4. Core product rules

1. **Backend is authoritative.** Never trust client price, tax, discount, total, role, payment status, or order status.
2. **Money is integer minor units.** Store INR as paise (`₹100.50` = `10050`). Never use floating-point money.
3. **A table session is the bill boundary.** One open session can contain multiple orders. The bill aggregates all billable orders in that session.
4. **Order history is immutable.** Copy item name, selected options, price, and tax basis into order-line snapshots when the order is placed. Later menu changes must not rewrite an old order.
5. **Order state and payment state are separate.** An order can be served while the table bill remains unpaid.
6. **Gateway webhook is payment truth.** Browser redirect/success screen is not proof of payment. Verify webhook signature and idempotency.
7. **Dietary warnings are operational alerts, not safety guarantees.** Never promise allergen-free food or no cross-contact. Staff must confirm with the guest.
8. **Every tenant-owned record is scoped.** Every query and mutation must enforce restaurant/outlet scope and role permission.
9. **QR codes are revocable opaque credentials.** Do not expose sequential table IDs as QR secrets.
10. **Tax and service charge are configurable.** No tax percentage is hard-coded into frontend code. The restaurant owner/accountant configures applicable values.

## 5. Main user journeys

- Customer: scan QR → create/resume table session → browse branded menu → customize items → add dietary notes → place order → track status → call waiter when needed → view consolidated bill → pay → see receipt.
- Kitchen: sign in → select outlet/station → receive new ticket → inspect item modifiers and warnings → mark preparing/ready → ticket remains visible until acknowledged/served according to KDS rules.
- Waiter: sign in → see new service requests → acknowledge → visit table → resolve request; also see table/order context permitted by role.
- Owner/manager: sign in → dashboard → filter date range → inspect sales and operations → export report → manage menu, availability, tables, branding, tax settings, and staff permissions according to role.

## 6. Documentation map

Read only the documents needed for a task; do not ask every agent to read the entire pack.

| File | Purpose | Who should read |
|---|---|---|
| `frontend.md` | UI/UX system, page layouts, interactions, responsive and accessibility rules | All frontend agents; design reviewer |
| `memory.md` | Current implementation status, decisions, open issues, next task | Orchestrator before/after every task |
| `tasks.md` | Small tasks, dependencies, ownership, copy-ready prompts | Orchestrator and assigned agent |
| `content.md` | Branding model, menu seed content, UX copy, empty/error states | Customer UI, owner UI, content agent |
| `architecture.md` | Runtime architecture, components, flows, boundaries | Backend lead, integration lead |
| `data-model.md` | Canonical entities, fields, invariants, indexes | Backend/database agents |
| `api-contract.md` | Canonical REST API shapes and error format | Every API/frontend agent |
| `realtime.md` | WebSocket events and fallback behavior | KDS, waiter, customer, backend agents |
| `security.md` | Authentication, QR, RBAC, payment and privacy controls | All backend and integration agents |
| `testing.md` | Test strategy, required scenarios, done criteria | Every implementation and QA agent |
| `deployment.md` | Local run, environment, deployment and operations | Scaffold, integration, deployment agents |
| `agent-workflow.md` | How to split work and prevent conflicting edits | Orchestrator and every AI agent |
| `agent-coordination.md` | Model-vs-agent explanation, beginner workflow, and dependency graph | Project owner and orchestrator |
| `free-plan-workflow.md` | Free-tier tools, role reuse, concurrency limits, and zero-API-spend workflow | Project owner and orchestrator |

## 7. Source-of-truth order

If documents or code conflict, follow this order:
1. Explicit accepted architecture decision recorded in `memory.md` / decision log.
2. `api-contract.md` for request/response behavior.
3. `data-model.md` for persistence and invariants.
4. `architecture.md` for module boundaries.
5. Feature-specific UX/content instructions.

An agent must not silently change a shared contract. Propose the change with impact, update the contract first, then implement it.

## 8. Definition of MVP complete

A diner can complete the full QR-to-payment journey on a phone; kitchen receives and updates the order live; waiter receives and resolves a call; owner sees the paid sale in reports; staff cannot access unauthorized outlets/roles; payment cannot be marked paid by a forged browser request; the system handles duplicate order/payment callbacks safely; and automated tests cover the critical flows.

## 9. Local Development & Setup

### Prerequisites
- Node.js (v18+) & npm
- Python 3.11+
- Docker & Docker Compose (for PostgreSQL)

### Initial Setup
1. **Clone & Configure Environment:**
   ```bash
   cp .env.example .env
   ```
2. **Backend Setup:**
   ```bash
   python -m venv .venv
   # Windows:
   .\.venv\Scripts\activate
   # Linux/macOS:
   source .venv/bin/activate

   pip install -r apps/api/requirements.txt
   ```
3. **Frontend Setup:**
   ```bash
   npm install --prefix apps/web
   ```
4. **Start Database (Docker):**
   ```bash
   docker compose up -d db
   ```

### Running Locally
- **Start FastAPI Backend:**
  ```bash
  uvicorn apps.api.app.main:app --reload --host 0.0.0.0 --port 8000
  ```
  Health endpoint: `http://localhost:8000/health`
- **Start React Frontend:**
  ```bash
  npm run dev --prefix apps/web
  ```
  Web UI: `http://localhost:5173`

### Running Tests
- **Backend Tests:**
  ```bash
  pytest apps/api/tests
  ```
- **Frontend Tests:**
  ```bash
  npm run test --prefix apps/web
  ```
- **Frontend Build Check:**
  ```bash
  npm run build --prefix apps/web
  ```


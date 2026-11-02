# Local Development, Deployment & Operations

## 1. Environments

- `local`: Docker Compose with PostgreSQL; fake payment adapter; test media/local fixtures.
- `staging`: production-like hosting, sandbox payment provider, test restaurant/outlet, HTTPS, restricted access.
- `production`: managed PostgreSQL, HTTPS, secret manager, real payment account/webhooks, backups and monitoring.

Never use real payment credentials or production guest data in local development.

## 2. Environment variables

Create `.env.example` with placeholders; never commit `.env`.

```dotenv
APP_ENV=local
APP_NAME=restaurant-order-system
API_HOST=0.0.0.0
API_PORT=8000
WEB_ORIGIN=http://localhost:5173
DATABASE_URL=postgresql+psycopg://app:app@localhost:5432/restaurant
SESSION_SECRET=replace-with-long-random-secret
GUEST_SESSION_SECRET=replace-with-long-random-secret
QR_TOKEN_PEPPER=replace-with-long-random-secret
PAYMENT_PROVIDER=fake
PAYMENT_PUBLIC_KEY=
PAYMENT_SECRET_KEY=
PAYMENT_WEBHOOK_SECRET=
REDIS_URL=
MEDIA_STORAGE=local
LOG_LEVEL=INFO
```

Production must use strong randomly generated secrets and a secret manager. Provider values vary by selected gateway. Do not put server secrets in `VITE_` frontend variables.

## 3. Local run (target workflow)

Exact command names are finalized by `BASE-01`; target flow:

1. Copy `.env.example` to `.env` and set local-only values.
2. Start database with Docker Compose.
3. Install backend and frontend dependencies using lockfiles.
4. Run Alembic migrations.
5. Seed demo restaurant/outlet/menu/staff/QR data via a development-only seed command.
6. Start FastAPI and Vite dev server.
7. Open web app; use printed development QR URL for guest flow; log in to staff routes with seed credentials.

Document exact commands after scaffold agent establishes scripts. Seed credentials must be clearly marked development-only and disabled in production.

## 4. Production topology

- HTTPS reverse proxy/load balancer.
- React static assets served via CDN/static hosting.
- FastAPI app container(s) behind proxy.
- Managed PostgreSQL with private network and automated backups.
- Optional Redis for cross-worker WebSocket pub/sub and throttling.
- Object storage/CDN for approved restaurant media.
- Payment provider sends webhook to public HTTPS API endpoint.
- Central logs/metrics/alerts with secrets and personal data redacted.

Start with one API instance if appropriate; add multiple workers/instances only after realtime fan-out is backed by Redis/outbox and DB connection limits are configured.

## 5. Deployment checklist

- [ ] Production env vars/secrets set; no secrets in frontend bundle.
- [ ] HTTPS, secure cookies, CORS allowlist, security headers, rate limits.
- [ ] Database provisioned privately; least-privilege DB account.
- [ ] Alembic migration tested on staging snapshot; backup before production migration.
- [ ] Payment provider sandbox tested; production webhook URL and signature secret verified.
- [ ] Owner's tax, invoice, service charge and currency settings reviewed by accountant.
- [ ] Restaurant branding, menu, availability, allergens, QR/table mapping verified by owner.
- [ ] Staff accounts/roles/outlet permissions configured; default credentials removed.
- [ ] QR test scan from iOS/Android cameras and weak Wi-Fi.
- [ ] Order/KDS/waiter/bill critical path tested on real devices.
- [ ] Backup restore tested; recovery owner and escalation contact identified.
- [ ] Monitoring and alerting for API errors, DB saturation, webhook failures, payment mismatch, and WebSocket disconnect spikes.
- [ ] Rollback plan documented.

## 6. Operations and monitoring

Monitor at minimum:
- API request volume/latency/error rate; health/readiness.
- DB connections, slow queries, storage, backup success.
- Orders created versus kitchen events published (outbox backlog if used).
- Oldest unacknowledged kitchen ticket and waiter request.
- Payment intents pending too long, webhook failure rate, amount mismatches, duplicate event count.
- WebSocket active connections/reconnect rates.
- Report query latency and export failures.

Provide safe admin procedures for rotating QR tokens, disabling ordering, disabling an unavailable item, and handling payment reconciliation. Sensitive manual actions must be audited.

## 7. Backups and recovery

- Automated encrypted PostgreSQL backups with retention based on business/legal needs.
- Periodic restore drill into non-production environment.
- Document recovery point and recovery time objectives with the restaurant owner/hosting budget.
- Back up media storage separately.
- Do not assume a successful backup job means a valid restore; test restoration.

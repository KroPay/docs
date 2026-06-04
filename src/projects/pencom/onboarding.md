---
icon: material/account-plus-outline
---

# eHub — Developer Onboarding

> How to get eHub running locally. Sections marked **⚠** flag known gotchas.

## 1. Prerequisites

### 1.1 Tooling

- **Node.js 20.x+** — use `nvm` or `fnm` to pin. Codebase uses ES2023 targets and NestJS v11.
- **Yarn** — the lockfile is `yarn.lock`. Do not use npm.
- **TypeScript / Nest CLI** — installed as devDependencies; `npx nest …` works without a global install.
- **Docker** — optional. The `docker-compose.yml` only uncomments `core` and `compliance`, with no datastore containers. Provision Postgres/Mongo/Redis externally.
- **ffmpeg / ffprobe** — bundled via `@ffmpeg-installer/ffmpeg` and `@ffprobe-installer/ffprobe`. Only needed for tutorial video upload processing in `api-gateway`.

### 1.2 Datastores

Provision locally:

- **PostgreSQL 14+** — six databases. Extension `uuid-ossp` is created by the first migration per service.
  ```
  pencom_core_db2
  pencom_payments
  pencom_compliance
  pencom_external_integrations
  pencom_audit
  (external-gateway reuses pencom_compliance)
  ```
- **MongoDB 6+** — two distinct URIs:
  - `notifications` history: `NOTIFICATIONS_DB_URI`
  - `core` OTP storage: `NOTIFICATIONS_MONGO_DB_URL` — **⚠ misleading name, consumed by `core` not `notifications`**
- **Redis 6+** — single instance for Bull queues (compliance) and magic-link cache (core).
- **Oracle** — optional for local dev. Missing `PENCOM_DB_*` does NOT crash on boot; Oracle-dependent features degrade at call time. Both `core` AND `compliance` open Oracle pools.
- **MinIO** — S3-compatible object store for file uploads. Configure `SPACES_*` env vars pointing at your local MinIO. Or use DigitalOcean Spaces / AWS S3 for dev.

**Quick Docker Compose snippet for local datastores:**

```yaml
services:
  postgres:
    image: postgres:14
    ports: ['5432:5432']
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
  mongo:
    image: mongo:6
    ports: ['27017:27017']
  redis:
    image: redis:7
    command: redis-server --requirepass yourpassword
    ports: ['6379:6379']
  minio:
    image: minio/minio
    ports: ['9000:9000', '9001:9001']
    command: server /data --console-address ":9001"
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin
```

Create the six Postgres databases manually after the container starts:

```bash
for db in pencom_core_db2 pencom_payments pencom_compliance pencom_external_integrations pencom_audit; do
  psql -U postgres -c "CREATE DATABASE $db;"
done
```

### 1.3 Third-party accounts needed for full functionality

- **Remita sandbox** — for payment flows. Demo credentials shipped in `.env.example`.
- **SendGrid** — for emails. Templates must match the IDs in `libs/shared/src/enums/email-template.enum.ts`.
- **Highlight.io** — optional error tracking.
- **PostHog** — optional, api-gateway only, disabled in `development`.
- **Termii** — optional; SMS currently has no producer.

---

## 2. Local Setup

```bash
# 1. Clone
git clone <repo>
cd pencom-project

# 2. Install dependencies
yarn install

# 3. Provision datastores (see §1.2)

# 4. Copy env template
cp .env.example .env
# Edit .env — see §3 below

# 5. Run a single service (watch mode)
yarn start:core:dev

# 6. Run all services (separate terminals or tmux)
yarn start:api-gateway:dev
yarn start:core:dev
yarn start:compliance:dev
yarn start:payments:dev
yarn start:notifications:dev
yarn start:external-integration:dev
yarn start:external-gateway:dev
yarn start:audit:dev
```

Migrations run automatically on first boot (`migrationsRun: true`). No manual migration step needed for a fresh database.

**Recommended startup order:** `core` → `compliance` → `payments` → `notifications` → `external-integrations` → `audit` → `api-gateway` → `external-gateway`

---

## 3. Key Environment Variables

Full reference in [Integrations & Env Vars](integrations.md#8-full-environment-variable-reference). Critical items for initial setup:

```bash
# Required at boot — service crashes without these:
INTERNAL_API_KEY=any-string-here
JWT_SECRET=your-secret

# Service URLs — UrlConfigService.getOrThrow() throws at boot if missing:
CORE_SERVICE_URL=http://localhost:4000
PAYMENT_SERVICE_URL=http://localhost:5000
COMPLIANCE_SERVICE_URL=http://localhost:6000
NOTIFICATIONS_SERVICE_URL=http://localhost:7000
EXTERNAL_INTEGRATIONS_SERVICE_URL=http://localhost:8000
AUDIT_SERVICE_URL=http://localhost:9000

# Database per service:
CORE_DB_HOST=localhost  CORE_DB_PORT=5432  CORE_DB_USER=postgres  CORE_DB_PASS=postgres  CORE_DB_NAME=pencom_core_db2
# ... (repeat for each service's DB_* vars)

# MongoDB:
NOTIFICATIONS_MONGO_DB_URL=mongodb://localhost:27017/pencom_core_otp
NOTIFICATIONS_DB_URI=mongodb://localhost:27017/pencom_notifications

# Redis:
REDIS_HOST=localhost  REDIS_PORT=6379  REDIS_PASSWORD=yourpassword
REDIS_URL=redis://:yourpassword@localhost:6379

# MinIO / S3:
SPACES_ENDPOINT=http://localhost:9000
SPACES_REGION=us-east-1
SPACES_ACCESS_KEY_ID=minioadmin
SPACES_SECRET_ACCESS_KEY=minioadmin
SPACES_BUCKET_NAME=ehub-local
SPACES_BUCKET_URL=http://localhost:9000/ehub-local
SPACES_ACL=public-read

# Login gating — required or all logins are blocked:
AUTH_PILOT_ALLOWLIST=[{"email":["you@example.com"]}]

ENVIRONMENT=development
FRONTEND_URL=http://localhost:3001
```

---

## 4. Commands

### Build

```bash
yarn build:core
yarn build:compliance
yarn build:api-gateway
yarn build:external-gateway
yarn build:external-integration
yarn build:audit
yarn build:notifications
yarn build:payments
```

### Run (dev watch mode)

```bash
yarn start:core:dev
yarn start:compliance:dev
yarn start:api-gateway:dev
yarn start:external-gateway:dev
yarn start:external-integration:dev
yarn start:audit:dev
yarn start:notifications:dev
yarn start:payments:dev
```

### Run (production builds)

```bash
yarn build:core && yarn start:core:prod
# same pattern for each app
```

### Tests

```bash
yarn test                    # all Jest specs
yarn test:watch
yarn test:cov
yarn jest path/to/file.spec.ts   # single file
yarn jest -t "test name"         # by name
```

### Lint / format

```bash
yarn lint     # eslint --fix across src, apps, libs, test
yarn format   # prettier --write across apps/ and libs/
```

### Database migrations

```bash
yarn gen:migration          # interactive: pick app → generate|run|revert → name
yarn migration:generate     # alias of the same helper
yarn migration:run

yarn migrate:payments       # direct typeorm-cli for payments
yarn generate:payments:migration
```

**⚠ Before generating a migration, register the new entity in `apps/<app>/src/typeorm.config.ts`** — the CLI uses explicit entity lists, not `autoLoadEntities`. Missing an entity produces an empty migration.

### Docker

```bash
docker-compose up --build    # only core + compliance are uncommented
```

---

## 5. Workflow Recipes

### Smoke-test gateway → core

```bash
# Terminal 1
yarn start:core:dev

# Terminal 2
yarn start:api-gateway:dev

# Terminal 3
curl http://localhost:3000/health
curl http://localhost:3000/ping/core
# → {ok: true, service: 'core', responseTimeMs: …}
```

### Test the OTP login flow

1. Add yourself to `AUTH_PILOT_ALLOWLIST`: `[{"email":["you@example.com"]}]`.
2. Configure SendGrid keys (or use `ENVIRONMENT=local` to get `plainOtp` in the API response).
3. `POST http://localhost:3000/auth` with `{employerCode: "EMP-XXXXXX", email: "you@example.com"}`.
4. Check email or API response for the OTP.
5. `POST http://localhost:3000/auth/verify` with `{otp: "…"}`.

### Generate a new migration

```bash
yarn gen:migration
# Pick app → "generate" → enter a name like "AddMyNewColumn"
# Output: apps/<app>/src/migrations/<timestamp>-AddMyNewColumn.ts
```

### Test a PFC CSV upload locally

1. Start `external-gateway`, `compliance`, Redis.
2. Ensure MinIO is running with a bucket configured.
3. `POST http://localhost:3010/api/v1/external-gateway/auth` to create a partner (returns `clientId`/`clientSecret`).
4. `POST /api/v1/external-gateway/auth/token` with `{clientId, clientSecret}` to get a JWT.
5. `POST /api/v1/external-gateway/compliance/upload/monthly-contribution/employee` — multipart `csv_file` + header `Idempotency-Key: <uuid>`.
6. Poll `GET /api/v1/external-gateway/compliance/upload/status/` with the same `Idempotency-Key` header.

### Replay a Remita webhook

```bash
curl -X POST http://localhost:3000/webhooks/payment/notification \
  -H 'Content-Type: application/json' \
  -d '{"rrr":"<known-rrr>","status":"00","message":"Successful","channel":"CARD"}'
# Response is plain text "Ok"
```

---

## 6. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Boot fails: `INTERNAL_API_KEY is not defined` | Missing env var | Set `INTERNAL_API_KEY` in `.env` |
| Boot fails: `Cannot read property 'getOrThrow'` on `*_SERVICE_URL` | Missing service URL env var | Set all `*_SERVICE_URL` vars even if the service isn't running locally (use placeholder `http://localhost:9999`) |
| Migrations don't pick up a new entity | Entity not in `apps/<app>/src/typeorm.config.ts` | Register the entity in that file before running `yarn gen:migration` |
| Redis `ECONNREFUSED` in compliance | `BullRedisConfigModule` reads `REDIS_HOST/PORT/PASSWORD`, not `REDIS_URL` | Set the discrete vars |
| All logins blocked with "invitation only" | `AUTH_PILOT_ALLOWLIST` missing or empty | Set `AUTH_PILOT_ALLOWLIST=[{"email":["youremail@example.com"]}]` |
| Emails not sending | SendGrid config or wrong `ENVIRONMENT` for template IDs | Check `SENDGRID_API_KEY`; verify `ENVIRONMENT` matches Dev vs Prod template set |
| `Payment.amount` ≠ what user sees on screen | By design — `Payment.amount` = Remita-billed total; response `amount` = display total | See Business Logic §7 |
| Penalty computed for unexpected month | Status frozen? Anchor wrong? Live path ignores `PENALTY_GRACE_DAYS` env. | Check `EmployerMonthlyPenalty.status` — once `PARTIALLY_PAID/PAID_IN_FULL`, recompute is skipped. Check `Employer.firstPaymentDate` (anchor). |
| Admin sessions never expire | Sliding expiry is computed but never persisted | Sessions hard-expire at original `expiresAt`. Cleanup cron runs every 10 min. |
| `entity_type NOT NULL` violation on audit insert | `AuditService.createAuditLog` drops derived fields before insert | Known bug — patch `apps/audit/src/audit.service.ts:39-57` to include `entityType` in the insert |
| File upload fails silently | `SPACES_*` env vars not set | Set all six `SPACES_*` vars (none are in `.env.example`) |
| Migration CLI targets wrong DB for payments | `typeorm.config.ts` reads `PAYMENTS_DB_*` (plural) but runtime uses `PAYMENT_DB_*` | Set both `PAYMENT_DB_*` and `PAYMENTS_DB_*` in `.env` |

---

## 7. Adding New Code

### Adding a new app

```bash
nest g app my-new-service
```

Then:

1. Add `build:my-new-service` and `start:my-new-service(:dev|:prod)` scripts to `package.json`.
2. Create `apps/my-new-service/src/typeorm.config.ts` if it owns a DB.
3. Verify the `assets` rule in `nest-cli.json` for migrations.
4. Add `*_SERVICE_URL` and `*_DB_*` env vars to `.env.example`; wire into `UrlConfigService` if other services need to call it.
5. Register `ApiKeyAuthGuard` as `APP_GUARD`.
6. Init Highlight in `main.ts` with a unique `HIGHLIGHT_<APP>_SERVICE_NAME`.

### Adding a module to an existing app

```bash
nest g mo my-module --project=<app-name>
# Always pass --project — without it, Nest generates under api-gateway (the default sourceRoot)
```

### Cross-service calls

Always inject `UrlConfigService` and `PencomInternalHttpClient`. Never hardcode URLs or skip the API key.

```ts
constructor(
  private readonly http: PencomInternalHttpClient,
  private readonly urls: UrlConfigService,
) {}

async listEmployers() {
  return this.http.get(`${this.urls.core}/admin/employers`);
}
```

### Adding an entity + migration

1. Create entity under `apps/<app>/src/entities/`.
2. Extend `BaseEntity` from `@app/database`.
3. Register in `apps/<app>/src/typeorm.config.ts`.
4. Register via `TypeOrmModule.forFeature([…])` in the consuming module.
5. `yarn gen:migration → <app> → generate → <name>`.
6. Inspect and trim the generated migration.

### Shared utilities

- Cross-cutting helper (≥2 apps) → `libs/shared/src/utils/<name>.ts`.
- Cross-service DTO → `libs/shared/src/dtos/<domain>/`.
- New message pattern constant → `libs/shared/src/patterns/<service>.patterns.ts` — note these are used as HTTP paths, not RPC topics.

### PR checklist

1. `yarn lint && yarn format` clean.
2. Tests added for non-trivial business logic.
3. Migrations included for schema changes; both `up` and `down` implemented.
4. New env vars added to `.env.example` and `docs/INTEGRATIONS.md`.
5. PR template filled: Summary / Details / References / Checks.

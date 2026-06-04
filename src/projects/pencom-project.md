# pencom-project

## Business Purpose

The Pencom project is a pension compliance management system built for Nigerian employers to manage their pension obligations with the National Pension Commission (PENCOM). It provides:

- **Employer registration** with PENCOM
- **Employee pension management** (RSA PIN tracking, PFA assignment)
- **Pension contribution tracking** and remittance scheduling
- **Compliance certificate generation** (PCC — Pension Clearance Certificates, GLI — Group Life Insurance)
- **Payment processing** via Remita for certificate fees and remittances
- **Discrepancy resolution** between employer and PENCOM records
- **Admin portal** for managing all employers, employees, and compliance workflows
- **Audit trail** for all significant actions

The system integrates with the official PENCOM Oracle database (on-premises) to sync authoritative pension data.

---

## Architecture

**Framework:** NestJS v11 (latest) — NestJS Monorepo  
**Pattern:** Microservices communicating over TCP (NestJS microservices transport)  
**Databases:** PostgreSQL (per-service), MongoDB (notifications), Oracle (PENCOM on-prem, read-only)  
**Cache / Queue:** Redis + BullMQ  
**Auth:** JWT + Passport  
**Payments:** Remita  
**Email:** SendGrid  
**SMS:** Termii  
**File storage:** AWS S3  
**Analytics:** PostHog  
**Observability:** Highlight.io  
**Queue UI:** @bull-board (BullMQ dashboard)  

### Monorepo Structure

```
pencom-project/
├── apps/
│   ├── api-gateway/          — Public HTTP entry point (:3000)
│   ├── external-gateway/     — External API gateway (:3010)
│   ├── core/                 — Core business logic (:4000)
│   ├── payments/             — Payment processing (:5000)
│   ├── compliance/           — Compliance modules (:6000)
│   ├── notifications/        — Email/SMS notifications (:7000)
│   ├── external-integrations/ — PENCOM Oracle sync (:8000)
│   └── audit/                — Audit trail service (:9000)
├── libs/
│   ├── database/             — Shared TypeORM config
│   ├── entities/             — Shared TypeORM entities
│   ├── interceptors/         — Shared request interceptors
│   ├── logger/               — Winston logger
│   ├── shared/               — Shared utilities and types
│   │   └── database-job-status/ — Shared job status tracking
│   └── url-config/           — Service URL configuration
└── scripts/
    └── migration.js          — Migration runner script
```

---

## Service Descriptions

### api-gateway (port 3000)

The primary entry point for all client requests (browser, mobile). It:
- Handles authentication (JWT validation)
- Routes requests to downstream microservices via TCP
- Enforces rate limiting (`@nestjs/throttler`)
- Logs requests (PostHog analytics interceptor)
- Serves health checks (`/health` endpoint via `@nestjs/terminus`)
- Streams tutorial videos
- Handles payment webhooks

Key controller groups: `auth`, `admin`, `employee`, `employer`, `employer-registration`, `compliance`, `discrepancy`, `notifications`, `file-upload`, `pfa`, `audit`, `tutorials`, `webhooks`, `external`.

### external-gateway (port 3010)

A separate HTTP gateway for external parties (e.g., other systems integrating with Pencom). Uses a separate `EXTERNAL_JWT_SECRET` for token validation.

### core (port 4000)

The primary business logic service. Handles:
- Employer and employee data management
- RSA PIN validation
- Contribution records
- Employee change requests
- Penalty calculations (configurable grace days and daily rate)
- Business rule enforcement

Owns the `pencom_core_db2` PostgreSQL database.

### payments (port 5000)

Handles all payment-related operations:
- Certificate fee payments via Remita
- Pension contribution remittances
- Payment status tracking
- Fee structure management (tiered by employee count + 7.5% tax)

Owns the `pencom_payments` PostgreSQL database.

### compliance (port 6000)

Handles compliance-specific workflows:
- **GLI** (Group Life Insurance) — tracking employer GLI compliance
- **PCC** (Pension Clearance Certificate) — application and issuance
- **PFC** (Pension Fund Custodian) uploads — processing PFC data files
- Discrepancy management between employer and PENCOM records
- Admin cleanup operations

Owns the `pencom_compliance` PostgreSQL database.

### notifications (port 7000)

Dedicated service for sending all outbound notifications:
- Email via SendGrid
- SMS via Termii
- Stores notification records in MongoDB

Owns the `pencom_notifications` MongoDB database.

### external-integrations (port 8000)

Bridges the gap between the on-premises PENCOM Oracle database and the Pencom system:
- Reads from PENCOM Oracle DB (`oracledb`)
- Syncs data into local `pencom_external_integrations` PostgreSQL database
- Exposes synced data to other services

Owns the `pencom_external_integrations` PostgreSQL database.

### audit (port 9000)

Immutable audit service. Every significant action in the system is recorded here. Other services send audit events to this service.

Owns the `pencom_audit_db` PostgreSQL database.

---

## Request Flow

### Employer Certificate Application

```
Employer logs in (browser → api-gateway)
  → POST /auth/login → JWT issued by api-gateway (validates via core)

Employer applies for PCC (Pension Clearance Certificate)
  → POST /compliance/pcc → api-gateway
  → api-gateway sends TCP message to compliance service
  → compliance checks employer eligibility (calls core via TCP)
  → compliance creates PCC application record
  → compliance sends TCP to notifications → employer emailed status

Payment for certificate
  → POST /payments/certificate-fee → api-gateway
  → api-gateway sends TCP to payments service
  → payments creates Remita invoice
  → Employer redirected to Remita payment page
  → Remita webhook → POST /webhooks/payment → api-gateway
  → api-gateway forwards to payments service
  → payments marks fee as paid
  → compliance auto-generates PCC
  → notifications sends certificate email
  → audit logs entire flow
```

### Admin Managing an Employer

```
Admin logs in → JWT with admin role

Admin views employer
  → GET /admin/employers → api-gateway → core service
  → Core queries pencom_core_db2
  → Response returned to admin frontend

Admin triggers PENCOM data sync
  → POST /admin/external/sync → api-gateway → external-integrations
  → external-integrations queries PENCOM Oracle DB
  → Data written to pencom_external_integrations PostgreSQL
  → Discrepancies identified and stored in compliance DB
  → Audit event logged
```

---

## Background Jobs (BullMQ)

BullMQ with Redis processes background jobs. The Bull Board dashboard is available via `@bull-board/express`.

Likely queues (inferred from domain):
- Penalty calculation jobs (scheduled, `PENALTY_GRACE_DAYS`, `PENALTY_DAILY_RATE`)
- PENCOM data sync jobs
- Notification delivery queue
- CSV/PFC file processing jobs

---

## Third-Party Integrations

| Service | Purpose | Package/Approach |
|---|---|---|
| Remita | Pension payment gateway | `axios` HTTP calls |
| PENCOM Oracle DB | Official pension data source | `oracledb` |
| SendGrid | Email notifications | `@sendgrid/mail` |
| Termii | SMS notifications | `axios` HTTP calls |
| AWS S3 | Document storage | `@aws-sdk/client-s3` |
| Redis | Job queues (BullMQ) + caching | `ioredis`, `bullmq` |
| PostHog | Product analytics | PostHog interceptor in api-gateway |
| Highlight.io | Error tracking | `@highlight-run/nest` |
| PDFKit | PDF generation (certificates) | `pdfkit` |
| xlsx | Excel file processing (PFC uploads) | `xlsx` |

---

## Environment Variables

Full details in `.env.example`. Key groups:

```bash
# Service ports and hosts
API_GATEWAY_SERVICE_HOST=    API_GATEWAY_SERVICE_PORT=3000
CORE_SERVICE_HOST=           CORE_SERVICE_PORT=4000
PAYMENT_SERVICE_HOST=        PAYMENT_SERVICE_PORT=5000
COMPLIANCE_SERVICE_HOST=     COMPLIANCE_SERVICE_PORT=6000
NOTIFICATIONS_SERVICE_HOST=  NOTIFICATIONS_SERVICE_PORT=7000
EXTERNAL_INTEGRATIONS_SERVICE_HOST= EXTERNAL_INTEGRATIONS_SERVICE_PORT=8000
AUDIT_SERVICE_PORT=9000

# Auth
JWT_SECRET=                  JWT_EXPIRES_IN=1d
INTERNAL_API_KEY=            # Service-to-service auth key
SESSION_TIMEOUT_MINUTES=15

# Each service's DB
CORE_DB_HOST=  CORE_DB_PORT=  CORE_DB_USER=  CORE_DB_PASS=  CORE_DB_NAME=
PAYMENT_DB_HOST=  ... PAYMENT_DB_NAME=pencom_payments
COMPLIANCE_DB_HOST=  ... COMPLIANCE_DB_NAME=pencom_compliance
EXTERNAL_INTEGRATIONS_DB_NAME=pencom_external_integrations
AUDIT_DB_NAME=

# Oracle (PENCOM)
PENCOM_DB_HOST=  PENCOM_DB_PORT=1521  PENCOM_DB_USER=  PENCOM_DB_PASSWORD=  PENCOM_DB_SERVICE_NAME=

# Remita
REMITA_BASE_URL=  REMITA_MERCHANT_ID=  REMITA_API_KEY=  REMITA_SERVICE_TYPE_ID=

# Certificate fees
CERTIFICATE_SERVICE_FEES='{...}'  # JSON config for fee tiers

# Comms
SENDGRID_API_KEY=  SENDGRID_FROM_NAME=  SENDGRID_FROM_EMAIL=
TERMII_API_KEY=  TERMII_BASE_URL=  TERMII_SENDER_ID=

# Infra
REDIS_URL=redis://<user>:<pass>@host:6379
ADMIN_EMAIL=                 # Default admin email
```

---

## Building and Running

```bash
cd pencom-project
cp .env.example .env
# Fill in all values

# Install dependencies
yarn install

# Build all services
yarn build

# Or build individually
yarn build:api-gateway
yarn build:core
yarn build:compliance
yarn build:payments

# Run in development (all together, hot reload)
yarn start:dev

# Or run individual services
yarn start:api-gateway:dev
yarn start:core:dev

# Production (run each in separate process / PM2)
yarn start:api-gateway:prod
yarn start:core:prod
yarn start:payments:prod
yarn start:compliance:prod
yarn start:notifications:prod
yarn start:external-integration:prod
yarn start:audit:prod
yarn start:external-gateway:prod
```

### Database Migrations

```bash
# Generate new migration
yarn migration:generate

# Run migrations
yarn migration:run

# Payments-specific
yarn migrate:payments
```

---

## Operational Notes

- **Service startup order matters:** `core` must be up before `api-gateway` can route to it. See [deployment.md](../deployment.md#recommended-startup-order).
- **PENCOM Oracle DB:** If the Oracle DB is unavailable, `external-integrations` will error but other services remain operational. The system can still use previously synced data.
- **Penalty calculations:** Configured via env vars (`PENALTY_GRACE_DAYS`, `PENALTY_DAILY_RATE`, `PENALTY_TRANSACTION_ISOLATION`). The `SERIALIZABLE` isolation level is used for penalty transactions to prevent race conditions.
- **Session timeout:** Configurable via `SESSION_TIMEOUT_MINUTES` (default: 15 min) with a `SESSION_WARNING_MINUTES` warning before expiry.
- **Bull Board:** BullMQ dashboard available at `/admin/queues` on the api-gateway (protected by admin auth). Useful for monitoring and retrying failed jobs.
- **PostHog analytics:** Tracking is applied via an interceptor on the api-gateway. All route hits are tracked.
- **PDF certificates:** PDFKit generates compliance certificates (PCC, GLI). Certificate files are stored in AWS S3.
- **OTP:** 5-minute expiry (`OTP_EXPIRY_MINUTES`), max 3 attempts (`MAX_OTP_ATTEMPTS`), 30-minute lockout (`LOCKOUT_DURATION_MINUTES`).
- **Highlight.io:** Each service has its own `HIGHLIGHT_SERVICE_NAME` to distinguish errors by service.

---

## Shutdown and Recovery

See [../shutdown-and-recovery.md](../shutdown-and-recovery.md) for full procedures.

The recommended process manager is **PM2**:
```bash
pm2 status               # Check all services
pm2 logs api-gateway     # Stream logs for a service
pm2 restart core         # Restart a service
pm2 stop all             # Stop everything gracefully
```

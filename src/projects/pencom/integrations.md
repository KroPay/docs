---
icon: material/connection
---

# eHub — Integrations & Environment Variables

> Every external system eHub connects to, and the full env var reference. Sections marked **⚠** flag misconfiguration risks.

## 1. Datastores

### 1.1 PostgreSQL (DB per service)

Six Postgres databases hosted on VPS 3. Each service owns its database exclusively.

| Service | DB Name | Env Prefix |
|---|---|---|
| core | `pencom_core_db2` | `CORE_DB_HOST/PORT/USER/PASS/NAME` |
| payments | `pencom_payments` | `PAYMENT_DB_HOST/PORT/USER/PASS/NAME` (**⚠ CLI reads `PAYMENTS_DB_*` (plural)**) |
| compliance | `pencom_compliance` | `COMPLIANCE_DB_HOST/PORT/USER/PASSWORD/NAME` |
| external-integrations | `pencom_external_integrations` | `EXTERNAL_INTEGRATIONS_DB_HOST/PORT/USER/PASSWORD/NAME` |
| audit | `pencom_audit` | `AUDIT_DB_HOST/PORT/USER/PASSWORD/NAME` |
| external-gateway | reuses `pencom_compliance` | reuses `COMPLIANCE_DB_*` |

Connection pattern (every service):

```ts
TypeOrmModule.forRootAsync({
  useFactory: (config) => ({
    type: 'postgres',
    host: config.get('XYZ_DB_HOST'),
    port: Number(config.get('XYZ_DB_PORT')),
    username: config.get('XYZ_DB_USER'),
    password: config.get('XYZ_DB_PASS'),
    database: config.get('XYZ_DB_NAME'),
    synchronize: false,
    autoLoadEntities: true,
    migrationsRun: true,
    ssl: null,
  }),
})
```

The `uuid-ossp` extension is created by the initial migration of each service.

### 1.2 Oracle ECRS DB (read-only)

The PENCOM ECRS Oracle database is the legacy source of truth. It is **hosted and managed by PenCom** — the eHub team has read-only access.

**Both `core` AND `compliance` open Oracle connection pools at boot.** Compliance cross-imports `PencomModule` from core, so compliance must also have all `PENCOM_DB_*` env vars set or it will fail to boot.

```bash
PENCOM_DB_HOST=
PENCOM_DB_PORT=1521
PENCOM_DB_USER=
PENCOM_DB_PASSWORD=
PENCOM_DB_SERVICE_NAME=
PENCOM_TABLE_PREFIX=      # ⚠ not in .env.example — used by pencomTable() util
PENCOM_CERTIFICATE_REQUEST_SUFFIX=  # ⚠ not in .env.example
```

All Oracle reads are **user-triggered** — no scheduled job or Bull worker queries Oracle. If `PENCOM_DB_HOST` is not set, the factory uses `retryAttempts: 0` and the connection is skipped; Oracle-dependent features degrade at call time rather than at boot.

### 1.3 MongoDB

Two separate MongoDB databases:

| Usage | Env var | Service |
|---|---|---|
| OTP storage (`otps` collection) | `NOTIFICATIONS_MONGO_DB_URL` | `core` — **⚠ misleading name, consumed by core not notifications** |
| Notification history (`pencom_notifications`) | `NOTIFICATIONS_DB_URI` | `notifications` |

### 1.4 Redis

Single Redis instance shared by `compliance` (Bull queues) and `core` (admin magic-link cache).

**⚠ `.env.example` ships `REDIS_URL` but the code reads individual components:**

```bash
REDIS_HOST=
REDIS_PORT=6379
REDIS_PASSWORD=
REDIS_URL=redis://:password@host:6379  # also set this
```

`BullRedisConfigModule` in compliance reads `REDIS_HOST/PORT/PASSWORD`, not `REDIS_URL`.

---

## 2. Object Storage — MinIO

MinIO is an S3-compatible object store running on VPS 4.

**SDK:** `@aws-sdk/client-s3` v3 (the same AWS SDK — MinIO is S3-compatible).

**Service:** `FileUploadService` (`libs/shared/src/file-upload/file-upload.service.ts`)

**⚠ None of these vars are in `.env.example`:**

```bash
SPACES_ENDPOINT=http://<minio-vps>:9000   # or HTTPS if TLS is configured
SPACES_REGION=us-east-1                   # MinIO accepts any region string
SPACES_ACCESS_KEY_ID=
SPACES_SECRET_ACCESS_KEY=
SPACES_BUCKET_NAME=
SPACES_BUCKET_URL=http://<minio-vps>:9000/<bucket>
SPACES_ACL=public-read
```

**Multipart threshold:** 10 MB. Parts of 10 MB. Aborts incomplete uploads on failure.

**Range reads:** `getFileRange(key, range)` for video streaming (206 Partial Content).

**Storage path keys** (whitelisted by `EStoragePath` in `apps/compliance/src/enums/storage.enums.ts`):

| Key | Used for |
|---|---|
| `EMPLOYER_CODE_REQUEST_DOCUMENT` | Employer registration documents |
| `CERTIFICATES` | PCC PDF certificates |
| `TUTORIAL_VIDEOS` | Admin/employer tutorial videos |
| `TUTORIAL_THUMBNAILS` | Video thumbnails |
| PFC upload namespace | CSV files from PFC partners |

**No presigned URLs** — all uploads/downloads route through `api-gateway` or `external-gateway`. No direct browser → MinIO.

---

## 3. Payment Provider — Remita

Source: `apps/payments/src/payments.service.ts`

Nigerian payment gateway for pension certificate fee collection.

```bash
REMITA_BASE_URL=https://demo.remita.net/remita/exapp/api/v1/send/api
REMITA_MERCHANT_ID=
REMITA_API_KEY=
REMITA_SERVICE_TYPE_ID=
REMITA_SERVICE_FEE=200     # fallback default
REMITA_PUBLIC_KEY=         # declared in .env.example, NOT used in code
```

**Endpoints used:**

| Operation | Endpoint |
|---|---|
| Create RRR | `POST {REMITA_BASE_URL}/echannelsvc/merchant/api/paymentinit` — JSONP response |
| Check status | `GET {REMITA_BASE_URL}/echannelsvc/{merchantId}/{rrr}/{hash}/status.reg` |
| Receive webhook | Remita POSTs to `https://<api-gateway>/webhooks/payment/notification` — **⚠ no signature verification** |

**Per-request auth:** `Authorization: remitaConsumerKey=<MERCHANT_ID>,remitaConsumerToken=<sha512Hash>`.

---

## 4. Communications

### 4.1 SendGrid (email)

Source: `apps/notifications/src/email/email.service.ts`

```bash
SENDGRID_API_KEY=
SENDGRID_FROM_EMAIL=no-reply@pencom.gov.ng
SENDGRID_FROM_NAME=PenCom
SENDGRID_REPLY_TO=   # ⚠ not in .env.example
```

Uses SendGrid Dynamic Templates. Template IDs differ between dev and prod, resolved by `getEmailTemplates(ENVIRONMENT)` — `EmailTemplateDev` vs `EmailTemplateProd` in `libs/shared/src/enums/email-template.enum.ts`.

**⚠ Setting the wrong `ENVIRONMENT` value will send with the wrong template ID.**

### 4.2 Termii (SMS)

Source: `apps/notifications/src/sms/sms.service.ts`

```bash
TERMII_API_KEY=
TERMII_BASE_URL=https://api.ng.termii.com
TERMII_SENDER_ID=PenCom
```

**⚠ Currently no producer calls `SmsService`** — the module exists but nothing triggers SMS sending. Safe to leave blank.

---

## 5. Observability

### 5.1 Highlight.io (APM / error tracking)

Wired in every app's `main.ts`.

```bash
HIGHLIGHT_PROJECT_ID=
HIGHLIGHT_SERVICE_VERSION=
HIGHLIGHT_ENVIRONMENT=development
HIGHLIGHT_DEBUG=false

# Per-service service name overrides:
HIGHLIGHT_API_GATEWAY_SERVICE_NAME=
HIGHLIGHT_CORE_SERVICE_NAME=
HIGHLIGHT_PAYMENTS_SERVICE_NAME=
HIGHLIGHT_COMPLIANCE_SERVICE_NAME=
HIGHLIGHT_NOTIFICATIONS_SERVICE_NAME=
HIGHLIGHT_AUDIT_SERVICE_NAME=
HIGHLIGHT_EXTERNAL_INTEGRATIONS_SERVICE_NAME=
HIGHLIGHT_EXTERNAL_GATEWAY_SERVICE_NAME=
```

**⚠ `.env.example` only declares `HIGHLIGHT_SERVICE_NAME`** — production must set per-service overrides.

### 5.2 PostHog (product analytics)

`api-gateway` only. Disabled when `ENVIRONMENT === 'development'`.

```bash
POSTHOG_KEY=
POSTHOG_HOST=https://us.i.posthog.com
```

Captures `api_request_succeeded` and `api_exception_captured` on every request.

---

## 6. Inter-Service Communication

### Internal HTTP (`PencomInternalHttpClient`)

Source: `libs/shared/src/pencom.http.ts`

Every outbound call automatically attaches `x-internal-api-key: <INTERNAL_API_KEY>`. Receiving services verify via global `ApiKeyAuthGuard`. **⚠ `compliance`, `payments`, and `external-integrations` do NOT register the guard** — they rely on network isolation alone.

**URL resolution via `UrlConfigService`:**

```bash
CORE_SERVICE_URL=http://<core-vps>:4000
PAYMENT_SERVICE_URL=http://<core-vps>:5000
COMPLIANCE_SERVICE_URL=http://<core-vps>:6000
NOTIFICATIONS_SERVICE_URL=http://<core-vps>:7000
EXTERNAL_INTEGRATIONS_SERVICE_URL=http://<core-vps>:8000
AUDIT_SERVICE_URL=http://<core-vps>:9000
```

`getOrThrow` is used — a missing `*_SERVICE_URL` throws at boot.

### Outbound HTTP call map (summary)

| Source | Calls | What for |
|---|---|---|
| api-gateway | core, compliance, payments, notifications, external-integrations, audit | All employer/admin/compliance routes |
| external-gateway | external-integrations, compliance | Partner auth, CSV job status |
| core | compliance, audit, notifications | PCC migration, audit logging, emails |
| compliance | core, payments, notifications, audit | Employer lookups, payment creation, email sending, audit logging |
| payments | core, audit | Employer enrichment, audit logging |

---

## 7. PFC Partner Integration

Three PenCom-licensed Pension Fund Custodians (PFCs) push contribution data to eHub via a secured REST API. eHub never pulls from PFCs — all uploads are PFC-initiated.

**Full integration flow:**

1. Admin creates partner: `POST /api/v1/external-gateway/auth` → `external-integrations /partners/create`. Returns `clientId`, `clientSecret` (plaintext, visible once only).
2. PFC authenticates: `POST /api/v1/external-gateway/auth/token` with `{clientId, clientSecret}` → 1h JWT.
3. PFC uploads CSV: `POST /api/v1/external-gateway/compliance/upload/monthly-contribution/employee` — multipart, `Idempotency-Key` header required, max 400 MB.
4. eHub stores to MinIO, creates `JobStatus` row, enqueues Bull job.
5. PFC polls: `GET /api/v1/external-gateway/compliance/upload/status/` with same `Idempotency-Key`.
6. When Bull worker finishes, compliance enqueues outbound webhook to PFC's configured `webhookUrl`, signed with HMAC-SHA256.

**Partner webhook config:** `PATCH /api/v1/external-gateway/auth/webhook-config` (authenticated).

---

## 8. Full Environment Variable Reference

### api-gateway

```bash
API_GATEWAY_SERVICE_HOST=localhost
API_GATEWAY_SERVICE_PORT=3000
API_GATEWAY_TCP_HOST=127.0.0.1
API_GATEWAY_TCP_PORT=3001

AUTH_PILOT_ALLOWLIST=    # ⚠ JSON array [{code?, email?[]}] — not in .env.example, blocks all logins if misconfigured
ADMIN_SUPPORT_PASSWORD=  # ⚠ not in .env.example — HTML KPI dashboard gate
```

### core

```bash
CORE_SERVICE_HOST=localhost
CORE_SERVICE_PORT=4000
CORE_SERVICE_TCP_PORT=4001
CORE_SERVICE_URL=http://localhost:4000

CORE_DB_HOST=  CORE_DB_PORT=5432  CORE_DB_USER=  CORE_DB_PASS=  CORE_DB_NAME=pencom_core_db2

# Oracle (both core AND compliance must have these):
PENCOM_DB_HOST=  PENCOM_DB_PORT=1521  PENCOM_DB_USER=  PENCOM_DB_PASSWORD=  PENCOM_DB_SERVICE_NAME=
PENCOM_TABLE_PREFIX=               # ⚠ not in .env.example
PENCOM_CERTIFICATE_REQUEST_SUFFIX= # ⚠ not in .env.example

# MongoDB (for OTPs — note: consumed by core, not notifications):
NOTIFICATIONS_MONGO_DB_URL=mongodb://host:27017/pencom_core_otp

# OTP & locking:
OTP_EXPIRY_MINUTES=15    # ⚠ declared twice in .env.example (5 then 15)
MAX_OTP_ATTEMPTS=3
LOCKOUT_DURATION_MINUTES=30
MAGIC_LINK_TTL_SECONDS=300
SESSION_TIMEOUT_MINUTES=15
SESSION_WARNING_MINUTES=1
ADMIN_OTP_RESEND_WINDOW_HOURS=  # ⚠ not in .env.example

# Admin bootstrap:
ADMIN_EMAIL=
ADMIN_ALLOWED_EMAILS=  # ⚠ not in .env.example — comma-separated
SUPER_ADMIN_EMAIL=     # ⚠ not in .env.example
SUPER_ADMIN_PHONE=     # ⚠ not in .env.example
SUPER_ADMIN_FIRST_NAME=  # ⚠ not in .env.example
SUPER_ADMIN_LAST_NAME=   # ⚠ not in .env.example
SUPPORT_EMAIL=           # ⚠ not in .env.example — default support@pencom.gov.ng

FRONTEND_URL=http://localhost:3001
```

### payments

```bash
PAYMENT_SERVICE_HOST=localhost
PAYMENT_SERVICE_PORT=5000
PAYMENT_SERVICE_TCP_PORT=5001
PAYMENT_SERVICE_URL=http://localhost:5000

PAYMENT_DB_HOST=  PAYMENT_DB_PORT=5432  PAYMENT_DB_USER=  PAYMENT_DB_PASS=  PAYMENT_DB_NAME=pencom_payments

# ⚠ CLI reads PAYMENTS_DB_* (plural) — set both for migration CLI to work:
PAYMENTS_DB_HOST=  PAYMENTS_DB_PORT=  PAYMENTS_DB_USER=  PAYMENTS_DB_PASS=  PAYMENTS_DB_NAME=

REMITA_BASE_URL=  REMITA_MERCHANT_ID=  REMITA_API_KEY=  REMITA_SERVICE_TYPE_ID=
CERTIFICATE_SERVICE_FEES={"ranges":[{"minEmployees":3,"maxEmployees":50,"fee":100000},{"minEmployees":51,"maxEmployees":100,"fee":150000},{"minEmployees":101,"maxEmployees":null,"fee":250000}],"taxPercent":7.5}
```

### compliance

```bash
COMPLIANCE_SERVICE_HOST=localhost
COMPLIANCE_SERVICE_PORT=6000
COMPLIANCE_SERVICE_TCP_PORT=6001
COMPLIANCE_SERVICE_URL=http://localhost:6000

COMPLIANCE_DB_HOST=  COMPLIANCE_DB_PORT=5432  COMPLIANCE_DB_USER=  COMPLIANCE_DB_PASSWORD=  COMPLIANCE_DB_NAME=pencom_compliance

REMITTANCE_DUE_DAY=28
PENALTY_GRACE_DAYS=7    # ⚠ only respected in backfill; live path hard-codes 11
PENALTY_TRANSACTION_ISOLATION=SERIALIZABLE
PENALTY_MONTHLY_DEFAULTING_RATE=0.02

# Redis (for Bull queues):
REDIS_HOST=  REDIS_PORT=6379  REDIS_PASSWORD=

# MinIO / S3:
SPACES_ENDPOINT=  SPACES_REGION=  SPACES_ACCESS_KEY_ID=  SPACES_SECRET_ACCESS_KEY=
SPACES_BUCKET_NAME=  SPACES_BUCKET_URL=  SPACES_ACL=public-read
```

### notifications

```bash
NOTIFICATIONS_SERVICE_HOST=127.0.0.1
NOTIFICATIONS_SERVICE_PORT=7000
NOTIFICATIONS_SERVICE_TCP_PORT=7001
NOTIFICATIONS_SERVICE_URL=http://localhost:7000

NOTIFICATIONS_DB_URI=mongodb://host:27017/pencom_notifications

SENDGRID_API_KEY=  SENDGRID_FROM_EMAIL=  SENDGRID_FROM_NAME=  SENDGRID_REPLY_TO=
TERMII_API_KEY=  TERMII_BASE_URL=  TERMII_SENDER_ID=
```

### external-integrations

```bash
EXTERNAL_INTEGRATIONS_SERVICE_HOST=127.0.0.1
EXTERNAL_INTEGRATIONS_SERVICE_PORT=8000
EXTERNAL_INTEGRATIONS_SERVICE_URL=http://localhost:8000

EXTERNAL_INTEGRATIONS_DB_HOST=  EXTERNAL_INTEGRATIONS_DB_PORT=  EXTERNAL_INTEGRATIONS_DB_USER=
EXTERNAL_INTEGRATIONS_DB_PASSWORD=  EXTERNAL_INTEGRATIONS_DB_NAME=pencom_external_integrations
```

### external-gateway

```bash
EXTERNAL_API_GATEWAY_SERVICE_HOST=localhost
EXTERNAL_API_GATEWAY_SERVICE_PORT=3010
EXTERNAL_JWT_SECRET=
# Also needs COMPLIANCE_DB_* for JobStatus writes
```

### audit

```bash
AUDIT_SERVICE_HOST=localhost
AUDIT_SERVICE_PORT=9000
AUDIT_SERVICE_URL=http://localhost:9000

AUDIT_DB_HOST=  AUDIT_DB_PORT=5432  AUDIT_DB_USER=  AUDIT_DB_PASSWORD=  AUDIT_DB_NAME=pencom_audit
```

### Shared (all services)

```bash
JWT_SECRET=
JWT_EXPIRES_IN=1d     # read but JwtModule hardcodes 1h — env has no effect
INTERNAL_API_KEY=     # required at boot — PencomInternalHttpClient throws if missing

ENVIRONMENT=development   # affects PostHog, email templates, OTP plaintext in response

REDIS_URL=redis://:password@host:6379   # also set REDIS_HOST/PORT/PASSWORD

POSTHOG_KEY=  POSTHOG_HOST=https://us.i.posthog.com

HIGHLIGHT_PROJECT_ID=  HIGHLIGHT_SERVICE_VERSION=  HIGHLIGHT_ENVIRONMENT=development  HIGHLIGHT_DEBUG=false
# Per-service: HIGHLIGHT_<APP>_SERVICE_NAME= for each of the 8 services
```

---

## 9. External System Summary

| System | Used by | Auth method | Notes |
|---|---|---|---|
| PostgreSQL (on-premises VPS 3) | All backend services | DB user/pass | DB-per-service |
| Oracle ECRS (PenCom on-prem) | core + compliance | DB user/pass | Read-only; user-triggered only |
| MySQL (regulator's historical certs) | **None — not implemented** | — | `mysql2` dep installed but unused |
| MongoDB (on-premises VPS 3) | core (otps) + notifications (history) | URI | Two distinct DBs |
| Redis (on-premises VPS 3) | compliance (queues) + core (cache) | password | Single host |
| MinIO (on-premises VPS 4) | api-gateway, external-gateway, compliance | access/secret key | S3-compatible |
| Remita | payments | SHA-512 per-request | Inbound webhook unsigned |
| SendGrid | notifications | API key | Dynamic templates, per-environment IDs |
| Termii | notifications | API key in body | **⚠ No producer — SMS not sent** |
| Highlight.io | every app | project ID | Server-side init in main.ts |
| PostHog | api-gateway only | project key | Disabled in development |

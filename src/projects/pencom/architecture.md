---
icon: material/layers-outline
---

# eHub — Architecture

> All facts derived from source code in `apps/`, `libs/`, `nest-cli.json`, `package.json`, `docker-compose.yml`, `.env.example`. Sections marked **⚠** flag known ambiguities.

## Tech Stack

| Layer | Choice |
|---|---|
| Language | TypeScript 5.7 |
| Framework | NestJS 11 (monorepo mode) |
| HTTP | Express via `@nestjs/platform-express`; `class-validator`/`class-transformer` for DTOs |
| Inter-service | HTTP via `PencomInternalHttpClient` (primary). NestJS TCP `@MessagePattern` exists but is only used internally within `notifications` for fire-and-forget email dispatch. |
| Persistence (primary) | PostgreSQL via TypeORM 0.3 — DB per service |
| Persistence (legacy) | Oracle via `oracledb` 6 — read-only `pencomConnection` in `core`; cross-app imported by `compliance` |
| Document store | MongoDB via Mongoose 8 — OTPs (`core`) and notification history (`notifications`) |
| Cache / queue broker | Redis (`ioredis`); `cache-manager-redis-yet` for admin magic-link cache |
| Job queues | Bull v3 (`@nestjs/bull`) — `employer-jobs`, `employee-jobs`, `employee-backfill-jobs`, `pencom-webhook` |
| Scheduling | `@nestjs/schedule` — compliance crons + core session cleanup |
| Auth (user) | JWT (`@nestjs/jwt`, `passport-jwt`) — extracted from cookie then `Authorization: Bearer`. Global `JwtAuthGuard` on `api-gateway` only. |
| Auth (service-to-service) | `INTERNAL_API_KEY` header enforced by global `ApiKeyAuthGuard` on backend services |
| Auth (external partners) | Separate JWT signed with `EXTERNAL_JWT_SECRET`; verified by `ExternalTokenAuthGuard` on `external-gateway` |
| File storage | S3-compatible (MinIO on-premises) via `@aws-sdk/client-s3` — multipart above 10 MB, range reads for video streaming |
| Email | SendGrid (`@sendgrid/mail`) via `notifications` service |
| SMS | Termii REST API via `notifications` service (**⚠ currently no producer calls `SmsService`**) |
| Payments | Remita REST API (JSONP responses) via `payments` service |
| Observability | Highlight.io (`@highlight-run/nest`) in every app; PostHog in `api-gateway` (non-development only) |
| Deployment | GitHub Actions → self-hosted runners (`prod-runner`, `staging-runner`) on on-premises VPS |

## Monorepo Layout

```
pencom-project/
├── apps/                    # 8 NestJS applications
│   ├── api-gateway/         # Public HTTP entry, JWT auth, routes to backend
│   ├── core/                # Employer/employee/auth domain, Oracle PENCOM bridge
│   ├── payments/            # Remita integration, payment state, certificate fees
│   ├── compliance/          # Contributions, penalties, PCC/GLI workflows, reports
│   ├── notifications/       # SendGrid + Termii fan-out, Mongo audit
│   ├── external-integrations/ # CAC company records + PFC partner credentials
│   ├── external-gateway/    # Partner-facing HTTP API — partner auth + CSV upload ingress
│   └── audit/               # Append-only audit-log API (Postgres)
├── libs/                    # Internal shared libraries
│   ├── shared/              # Guards, decorators, DTOs, PencomInternalHttpClient, FileUploadService
│   ├── shared/database-job-status/  # JobStatus + FailedBatch entities
│   ├── database/            # BaseEntity, BaseTypeOrmRepository, DatabaseModule
│   ├── url-config/          # Typed accessor over *_SERVICE_URL env vars
│   └── logger/              # LoggerModule.forRoot wrapper
├── monitoring/ansible/      # Prometheus + Grafana + node-exporter Ansible roles
├── .github/workflows/       # deploy-prod, deploy-staging, deploy-monitoring
├── nest-cli.json            # Monorepo project definitions
└── scripts/migration.js     # Interactive TypeORM CLI driver for per-app migrations
```

## Services and Ports

| Service | HTTP Port | TCP Port | Database Env Prefix | Owns DB? |
|---|---|---|---|---|
| api-gateway | `API_GATEWAY_SERVICE_PORT` (3000) | `API_GATEWAY_TCP_PORT` (3001) | — | No |
| core | `CORE_SERVICE_PORT` (4000) | `CORE_SERVICE_TCP_PORT` (4001) — declared but unused | `CORE_DB_*` + MongoDB + Oracle `PENCOM_DB_*` | Yes |
| payments | `PAYMENT_SERVICE_PORT` (5000) | `PAYMENT_SERVICE_TCP_PORT` (5001) — no handlers, dead | `PAYMENT_DB_*` | Yes |
| compliance | `COMPLIANCE_SERVICE_PORT` (6000) | `COMPLIANCE_SERVICE_TCP_PORT` (6001) — unused | `COMPLIANCE_DB_*` | Yes |
| notifications | `NOTIFICATIONS_SERVICE_PORT` (7000) | `NOTIFICATIONS_SERVICE_TCP_PORT` (7001) — active (self-emit) | `NOTIFICATIONS_DB_URI` (MongoDB) | Yes |
| external-integrations | `EXTERNAL_INTEGRATIONS_SERVICE_PORT` (8000) | — | `EXTERNAL_INTEGRATIONS_DB_*` | Yes |
| external-gateway | `EXTERNAL_API_GATEWAY_SERVICE_PORT` (3010) | — | reuses `COMPLIANCE_DB_*` | No |
| audit | `AUDIT_SERVICE_PORT` (9000) | — | `AUDIT_DB_*` | Yes |

## Process Topology (Request Flow)

```
┌───────────────┐                              ┌────────────────────┐
│  Web client   │  HTTPS                       │ External partner   │
│ (employer /   │ ──────────────┐              │ (PFC)              │
│  admin)       │               ▼              └──────────┬─────────┘
└───────────────┘     ┌──────────────────┐               │ Bearer JWT
                      │   api-gateway    │     ┌──────────▼─────────┐
                      │  JwtAuthGuard    │     │  external-gateway  │
                      │  ValidationPipe  │     │  ExternalTokenAuth │
                      │  Swagger, CORS   │     └──────────┬─────────┘
                      └─────────┬────────┘               │
                                │  HTTP (x-internal-api-key)       │
        ┌───────────────────────┼──────────────────────────────────┘
        │              ┌────────┼────────────────┐
        ▼              ▼        ▼                ▼
   ┌─────────┐  ┌────────────┐ ┌──────────┐  ┌──────────┐
   │  core   │  │ compliance │ │ payments │  │  audit   │
   └─┬───────┘  └─────┬──────┘ └──────────┘  └──────────┘
     │                │ HTTP + Bull queue (Redis)
     ▼                ▼
  Postgres      ┌──────────────┐         ┌───────────────────┐
  + Oracle  ────│ notifications│────────▶│ SendGrid / Termii │
  + Mongo        └──────────────┘         └───────────────────┘
```

Key points:

- **All synchronous inter-service calls are HTTP**, not TCP, despite message-pattern enum names suggesting otherwise. The enums like `CoreMessagePatterns.REGISTER_EMPLOYER` are used as HTTP path strings.
- **No central message bus.** No Kafka or RabbitMQ. Redis/Bull is used only for file-processing jobs and outbound webhooks.
- **`notifications` self-emits over in-process TCP** for fire-and-forget email dispatch (HTTP 202 → in-process TCP EventPattern → SendGrid). **⚠ A crash between the 202 and the handler loses the email.**
- **`compliance` runs scheduled jobs**: certificate expiry (hourly), GLI expiry (midnight), employer revaluation (midnight), approved evaluation re-sync (hourly), grace-period notifications (1am). Bull processors for PFC uploads also run here.

## Auth Model

Three independent authentication boundaries:

### 1. End-user JWT (`JWT_SECRET`)

- 1h access token; 7d refresh token on `path: /auth`.
- Issued by `api-gateway AuthController` (employer) and `AdminAuthController` (admin) after OTP verification.
- Verified by global `JwtAuthGuard` on `api-gateway` only, via `JwtStrategy`.
- Cookie first (`access_token`, httpOnly), then `Authorization: Bearer`.
- Opt-out per route with `@Public()`.

### 2. Internal API key (`INTERNAL_API_KEY`)

- Header: `x-internal-api-key`.
- Backend services (`core`, `notifications`, `audit`) register `ApiKeyAuthGuard` globally.
- **⚠ `compliance`, `payments`, and `external-integrations` do NOT register it** — rely on network isolation.
- `PencomInternalHttpClient` automatically attaches this header to every outbound call.

### 3. External partner JWT (`EXTERNAL_JWT_SECRET`)

- 1h expiry; issued by `external-integrations PartnerController.getToken`.
- Verified by `ExternalTokenAuthGuard` on `external-gateway`.
- JWT payload embeds the full partner `{id, name, code, organizationType}` — no per-request DB lookup.

### Admin RBAC on top of JWT

- `RolesGuard` + `@Roles(AdminRole.SUPER_ADMIN)` for super-admin-only endpoints.
- `JourneyPermissionsGuard` + `@RequireJourney([journeys], role?)` for fine-grained access. SUPER_ADMIN bypasses. Nine journey codes: `GLI`, `PCC`, `EMPLOYEE_MANAGEMENT`, `PAYMENT_MANAGEMENT`, `HELP_AND_SUPPORT`, four `EMPLOYER_CODE_REQUEST_*` variants.

## Data Strategy

- **PostgreSQL per service.** `synchronize: false`, `autoLoadEntities: true`, `migrationsRun: true`. Migrations run from compiled JS in `dist/.../migrations/` at boot.
- **Standalone CLI config.** `apps/<app>/src/typeorm.config.ts` drives `yarn gen:migration`. It lists entities explicitly — newly created entities missing from this file will produce empty migrations.
- **Cross-DB access from `compliance`.** Compliance opens a second named connection `EXTERNAL_INTEGRATIONS_DB` to read `IntegrationPartner` records. It also cross-app imports `PencomModule` from `core`, opening its own Oracle connection pool — meaning compliance must have all `PENCOM_DB_*` env vars set.
- **Soft deletes** via `@DeleteDateColumn deletedAt` on `BaseEntity`.
- **JSONB columns** for fast-changing compliance aggregates, metadata, and webhook payloads.
- **Natural-key deduplication** in contribution ingest. The key for `employee_monthly_contributions` is `(employer_code, rsa_pin, date_of_contribution, value_date, contribution_type)`, enforced as a partial unique index.
- **Oracle is read-only.** No INSERT/UPDATE/DELETE against Oracle anywhere in the codebase. Every Oracle call is user-triggered, not scheduled.

## CI/CD

| Workflow | Trigger | What it does |
|---|---|---|
| `deploy-prod.yaml` | Push to `main`, filter `apps/**` | Builds only changed apps via `dorny/paths-filter`, pushes Docker images, deploys via `prod-runner` (self-hosted runner on production VPS) |
| `deploy-staging.yaml` | Push to `staging`, filter `apps/**` | Same shape but uses `staging-runner` |
| `deploy-monitoring.yaml` | Push to `main` or `monitoring-deploy` branch, filter `monitoring/**` | Installs Ansible + sshpass on `prod-runner`, runs `monitoring/ansible/site.yml` to provision Prometheus + Grafana + node-exporter |

**Each app has its own `Dockerfile`** at `apps/<app>/Dockerfile`. Only changed apps are rebuilt per deployment.

## Cross-Cutting Concerns

| Concern | Mechanism | Notes |
|---|---|---|
| Validation | `ValidationPipe({ transform:true, whitelist:true })` | `api-gateway` only. **⚠ Other services don't register it** — `class-validator` decorators on their DTOs are not enforced at runtime. |
| Error mapping | `AllExceptionFilter` from `@app/shared` | Wraps everything in `ApiResponse.error`. **⚠ Unknown `HttpException` statuses map to 403, not 500.** |
| Logging | `LoggingInterceptor` from `@app/shared` | HTTP and TCP variants. **⚠ `api-gateway` registers it twice** (main.ts + module.ts). |
| APM | Highlight.io `HighlightInterceptor` + `H.init` | Every app's main.ts. Configure via `HIGHLIGHT_*` env vars. |
| Analytics | `PosthogInterceptor` | api-gateway only, disabled in `development`. |
| Throttling | `@nestjs/throttler` in `core` | Only applied to `auth/initiate` (4 attempts per 2 minutes). |
| Health | `@nestjs/terminus /health` | Every app, `@Public()`. |
| Audit | `apps/audit POST /logs` | Called via HTTP by `core`, `payments`, `compliance`. **⚠ `AuditService.createAuditLog` drops `entity_type` before insert — `entity_type` is NOT NULL in DB, so audit inserts likely fail.** |

## Known Ambiguities

1. **TCP transports declared but unused** in `api-gateway` and `payments`.
2. **`compliance` registers `core` entities** in `TypeOrmModule.forFeature` — either the compliance DB has those tables or it's dead code.
3. **`notifications` self-emit is non-durable** — crash between HTTP 202 and TCP handler loses the email.
4. **`audit.service.ts:39-57`** drops derived fields before insert; `entity_type` NOT NULL violation likely.
5. **Several admin endpoints lack guards** in `compliance` and `payments` — protected only at the gateway layer.
6. **Hard-coded values**: Convenience Fee ₦161.25 (payments), grace days 11 in live penalty path.
7. **`SPACES_*` env vars missing from `.env.example`** despite being required by `FileUploadService`.
8. **`payments` CLI reads `PAYMENTS_DB_*` (plural)** but runtime reads `PAYMENT_DB_*` — set both.
9. **`AUTH_PILOT_ALLOWLIST`** gates all logins but is undocumented in `.env.example`.
10. **MySQL not integrated** — `mysql2` installed but no connection, entity, or query exists.

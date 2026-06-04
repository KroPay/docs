# krogiving-backend

## Business Purpose

The KROGiving backend is the API server for the KROGiving crowdfunding platform. It enables:

- **Campaign creators** to create, manage, and publish fundraising campaigns
- **Donors** to donate to campaigns using Paystack (Nigerian cards, bank transfer, USSD)
- **Platform admins** to review campaigns, manage withdrawals, configure platform settings
- **Campaign organizers** to withdraw raised funds to their bank accounts

The platform supports multiple currencies, campaign media (images/video), commenting, donor leaderboards, and comprehensive admin controls.

---

## Architecture

**Framework:** NestJS v10, TypeScript  
**Databases:** MongoDB (Mongoose) + PostgreSQL (TypeORM) — dual database  
**Cache:** Redis (ioredis)  
**Auth:** JWT + Passport  
**Email:** SendGrid + MailerSend  
**SMS:** Termii + Telnyx  
**Payments:** Paystack  
**File storage:** DigitalOcean Spaces (AWS S3-compatible)  
**CMS:** Strapi (external)  
**Feature flags:** Split.io (frontend)  
**Observability:** Highlight.io  
**Scheduled tasks:** `@nestjs/schedule`  
**Events:** `@nestjs/event-emitter`  

### Module Structure

```
src/
├── admin/
│   ├── admin-auth/         — Admin authentication and user management
│   ├── admin-campaign/     — Campaign review, approval/rejection
│   ├── admin-donation/     — Donation management, CSV export/import
│   ├── admin-log/          — Admin action logs, withdrawal cap logs
│   ├── admin-payout/       — Payout approval with 2FA
│   ├── admin-permissions/  — Permission management
│   ├── admin-settings/     — Platform settings (withdrawal cap, etc.)
│   └── admin-user/         — User management by admins
├── campaign/               — Campaign CRUD, public listing
├── comment/                — Campaign comments
├── currency/               — Exchange rates, currency conversion
├── donation/               — Donation processing
├── fee/                    — Platform fee calculation
├── file-upload/            — DO Spaces file management
├── global-settings/        — Platform-wide settings (adjusted rates)
├── guards/                 — Auth guards
├── logger/                 — Winston logger
├── notification/
│   ├── email/              — Email notifications
│   └── sms/                — SMS notifications
├── otp/                    — OTP generation and validation
├── payment/                — Paystack payment orchestration
├── paystack/               — Paystack API client
├── profile/                — User profiles
├── remote/                 — Remote API client utilities
├── scheduler/              — Cron jobs / scheduled tasks
├── seed/                   — Database seeders
├── strapi/                 — Strapi CMS integration
├── users/                  — User CRUD and authentication
└── withdrawal/             — Withdrawal processing
```

### Database Split Rationale

| Data | Database | Why |
|---|---|---|
| Campaigns, users, donations, withdrawals | PostgreSQL | Relational data with strong consistency needs (financial) |
| Admin users, admin logs, campaign reviews, settings | MongoDB | Flexible schema for operational/audit data; admins frequently evolve |
| OTP codes | Redis | Short-lived; TTL expiry built in |

---

## Request Flow

### User Donation Flow

```
Donor opens campaign page (krogiving-frontend)
  → GET /v1/campaigns/:slug — fetch campaign details
  → POST /v1/donations/initialize — init Paystack transaction
  → Frontend redirects to Paystack checkout
  → Paystack processes payment
  → Paystack sends webhook → POST /v1/paystack/webhook
  → Backend verifies webhook signature
  → Donation marked as completed in PostgreSQL
  → Campaign raised amount updated
  → Donor receives email confirmation (SendGrid)
```

### Campaign Creation Flow

```
Creator logs in → JWT token
  → POST /v1/campaigns — create campaign (draft)
  → POST /v1/file-upload — upload campaign images to DO Spaces
  → PUT /v1/campaigns/:id — update campaign details
  → POST /v1/campaigns/:id/submit — submit for review
  → Admin receives notification
  → Admin reviews → PATCH /v1/admin/campaigns/:id/status (approve/reject)
  → Creator notified via email/SMS
  → Campaign published (visible to public)
```

### Withdrawal Flow

```
Campaign organizer requests withdrawal
  → POST /v1/withdrawal — submit withdrawal request
  → Admin reviews payout queue
  → Admin initiates 2FA → POST /v1/admin/payouts/two-factor/initialize
  → Admin enters TOTP code → POST /v1/admin/payouts/approve
  → Backend triggers Paystack transfer to organizer's bank
  → Status updated to completed
```

---

## Key API Sections

### Public APIs

| Resource | Description |
|---|---|
| `GET /v1/campaigns` | List active campaigns |
| `GET /v1/campaigns/:slug` | Campaign detail |
| `POST /v1/donations/initialize` | Start a donation |
| `POST /v1/paystack/webhook` | Paystack payment webhook |
| `POST /v1/otp/send` | Send OTP for auth |
| `POST /v1/users/register` | User registration |
| `POST /v1/users/login` | User login |

### Authenticated User APIs

| Resource | Description |
|---|---|
| `POST /v1/campaigns` | Create campaign |
| `GET /v1/campaigns/mine` | Own campaigns |
| `POST /v1/withdrawal` | Request withdrawal |
| `GET /v1/profile` | User profile |

### Admin APIs (prefix: `/v1/admin/`)

| Resource | Description |
|---|---|
| `POST /admin/auth/sign-in` | Admin login |
| `GET /admin/campaigns` | All campaigns |
| `PATCH /admin/campaigns/:id/status` | Approve/reject |
| `GET /admin/donations` | All donations |
| `POST /admin/payouts/approve` | Approve payout |
| `GET /admin/users` | All users |
| `GET /admin/logs` | Admin audit logs |
| `PUT /admin/settings/global-withdrawal-cap` | Set withdrawal cap |

---

## Scheduled Jobs

Defined in `src/scheduler/scheduler.module.ts`. Jobs run via `@nestjs/schedule` (cron expressions).

Likely scheduled tasks (inferred from domain):
- Currency rate refresh
- Campaign status auto-updates (expired campaigns)
- Stale OTP cleanup

---

## Third-Party Integrations

| Service | Purpose | SDK / Approach |
|---|---|---|
| Paystack | Payment processing and bank transfers | `axios` + webhook verification |
| DigitalOcean Spaces | File storage (S3-compatible) | `@aws-sdk/client-s3` |
| SendGrid | Transactional email | `@sendgrid/mail` |
| MailerSend | Email alternative | `mailersend` |
| Termii | SMS OTP | `axios` HTTP calls |
| Telnyx | SMS alternative | `axios` HTTP calls |
| Strapi | CMS content (blog, static content) | `axios` REST calls |
| Redis | OTP caching, app cache | `ioredis` |
| Highlight.io | Error tracking | `@highlight-run/nest` |

---

## Environment Variables

Full list in `.env.sample`. Key groups:

```bash
# Database
MONGO_URI=                     # MongoDB connection string
GIV_POSTGRES_HOST=             # PostgreSQL host
GIV_POSTGRES_PORT=             # PostgreSQL port
GIV_POSTGRES_USER=             # PostgreSQL username
GIV_POSTGRES_PASSWORD=         # PostgreSQL password
GIV_POSTGRES_DATABASE=         # Database name
GIV_POSTGRES_CA=               # SSL CA certificate content

# Auth
JWT_SECRET_KEY=                # JWT signing secret
EXPIRES_IN=                    # JWT expiry (e.g., "7d")

# Cache
REDIS_HOST=                    # Redis host
REDIS_PORT=                    # Redis port
TTL=                           # Default cache TTL

# Payments
PAYSTACK_SECRET_KEY=           # Paystack secret key
PAYSTACK_PUBLIC_KEY=           # Paystack public key

# Files
SPACES_ACCESS_KEY_ID=          # DO Spaces access key
SPACES_SECRET_ACCESS_KEY=      # DO Spaces secret key
SPACES_REGION=                 # DO Spaces region
SPACES_BUCKET_NAME=            # Bucket name
SPACES_ENDPOINT=               # DO Spaces endpoint URL
SPACES_BUCKET_URL=             # Public bucket URL

# Email
SENDGRID_API_KEY=              # SendGrid API key
SENDER_EMAIL=                  # From email address
SENDER_NAME=                   # From name

# SMS
TERMII_API_KEY=                # Termii API key
TELNYX_AUTH_TOKEN=             # Telnyx API token

# App
ENVIRONMENT=development        # or 'production'
FRONT_END_URL=                 # Frontend URL for email links
```

---

## Deployment

**Platform:** DigitalOcean App Platform  
**Trigger:** Push any git tag to the repo

CI workflow (`.github/workflows/deploy.yml`):
1. Receives tag push event
2. Calls DO App Platform API to redeploy `krogiving-backend` component
3. DO App Platform pulls latest code, rebuilds, and deploys

**Local dev:**
```bash
yarn install
cp .env.sample .env
docker-compose up -d       # starts MongoDB
yarn start:dev
```

**Docker (dev):** `docker-compose.yml` starts the app + MongoDB together.

---

## Operational Notes

- **Slug migration:** `SlugMigrationService` (injected in AppModule) handles any one-time data migrations for campaign slugs. It runs on startup and should be idempotent.
- **Migrations:** TypeORM migrations run automatically on startup (`migrationsRun: true`). Check `dist/migrations/*.js` are built before starting in production.
- **Admin permissions:** Fine-grained permission system. Permissions defined in MongoDB `Permission` collection. `AdminPermissionGuard` enforces per-route.
- **Dual DB caution:** When writing queries, be aware that some data is in MongoDB (admin schemas) and some in PostgreSQL (campaign, donation, user entities). Do not cross-query.
- **Paystack webhook:** Must be verified using the webhook signature before processing. Never process unverified webhooks.
- **Winston logging:** Rotating file logs via `winston-daily-rotate-file`. Check log files on the server for debugging.

---

## Shutdown and Recovery

See [../shutdown-and-recovery.md](../shutdown-and-recovery.md).

The App Platform handles restart automatically. For database recovery, see [../databases.md](../databases.md).

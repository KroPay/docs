---
icon: material/layers-outline
---

# KROGiving — Architecture

## Tech Stack

### krogiving-backend

| Layer | Choice |
|---|---|
| Framework | NestJS v10, TypeScript |
| Primary DB | PostgreSQL via TypeORM — campaigns, users, donations, withdrawals |
| Secondary DB | MongoDB via Mongoose — admin users, admin logs, reviews, settings |
| Cache | Redis (ioredis) — OTP codes |
| Auth | JWT + Passport — separate strategies for users and admins |
| Email | SendGrid + MailerSend |
| SMS | Termii + Telnyx |
| Payments | Paystack (webhooks + transfers) |
| File storage | DigitalOcean Spaces (AWS S3-compatible) |
| CMS | Strapi (external, REST API) |
| Observability | Highlight.io |
| Scheduled jobs | `@nestjs/schedule` |
| Events | `@nestjs/event-emitter` |

### krogiving-frontend

| Layer | Choice |
|---|---|
| Framework | React 18, Create React App, TypeScript |
| Styling | Tailwind CSS v3 + custom CSS |
| State | Zustand (auth), TanStack Query v5 (server state) |
| Routing | React Router DOM v6 |
| Forms | TanStack Form v0.33 |
| Payments | react-paystack |
| i18n | react-i18next |
| Rich text | CKEditor 5 Premium + Jodit |
| Video upload | Cloudinary (direct browser upload) |
| Feature flags | Split.io |
| Observability | Highlight.io (session replay) |

### giv-admin-new

| Layer | Choice |
|---|---|
| Framework | React 19, Vite 6, TypeScript |
| Styling | Tailwind CSS v4 + shadcn/ui (Radix UI) |
| State | Zustand (global), TanStack Query v5 (server state) |
| Routing | React Router DOM v7 |
| Forms | React Hook Form + Zod |
| Real-time | Socket.IO client |
| Charts | Recharts |

## Database Split Rationale

| Data | Database | Why |
|---|---|---|
| Campaigns, users, donations, withdrawals | PostgreSQL | Relational, financial consistency required |
| Admin users, admin logs, reviews, settings | MongoDB | Flexible schema; admin domain evolves frequently |
| OTP codes | Redis | Short-lived; TTL expiry built in |

## Module Structure — krogiving-backend

```
src/
├── admin/
│   ├── admin-auth/         — Admin authentication, JWT, OTP login
│   ├── admin-campaign/     — Campaign review, approval/rejection, messages
│   ├── admin-donation/     — Donation management, CSV export
│   ├── admin-log/          — Admin action audit logs, withdrawal cap logs
│   ├── admin-payout/       — Payout approval with TOTP 2FA
│   ├── admin-permissions/  — Permission definitions and enforcement
│   ├── admin-settings/     — Platform settings (withdrawal caps, rates)
│   └── admin-user/         — User management by admins
├── campaign/               — Campaign CRUD, public listing, slug management
├── comment/                — Campaign comments
├── currency/               — Exchange rates, currency conversion
├── donation/               — Donation initialization and verification
├── fee/                    — Platform fee calculation
├── file-upload/            — DO Spaces file management
├── global-settings/        — Platform-wide configurable settings
├── notification/
│   ├── email/              — SendGrid / MailerSend email notifications
│   └── sms/                — Termii / Telnyx SMS notifications
├── otp/                    — OTP generation and Redis-backed validation
├── payment/                — Paystack payment orchestration
├── paystack/               — Paystack API client
├── profile/                — User profiles
├── scheduler/              — Cron job definitions
├── seed/                   — Database seeders
└── withdrawal/             — Withdrawal processing
```

## Request Flows

### Donor Donation Flow

```
Donor opens campaign page (krogiving-frontend)
  → GET /v1/campaigns/:slug — fetch campaign details
  → POST /v1/donations/initialize — initialize Paystack transaction
  → react-paystack opens Paystack checkout modal
  → Paystack processes payment
  → Paystack sends webhook → POST /v1/paystack/webhook
  → Backend verifies webhook signature
  → Donation marked completed in PostgreSQL
  → Campaign raised amount updated
  → Donor receives email confirmation (SendGrid)
```

### Campaign Creation Flow

```
Creator logs in → JWT token
  → POST /v1/campaigns — create campaign (draft)
  → POST /v1/file-upload — upload images to DO Spaces
  → Cloudinary upload widget — upload video directly from browser
  → PUT /v1/campaigns/:id — update campaign details
  → POST /v1/campaigns/:id/submit — submit for admin review
  → Admin receives real-time Socket.IO notification (giv-admin-new)
  → Admin reviews in giv-admin-new dashboard
  → PATCH /v1/admin/campaigns/:id/status (approve/reject)
  → Creator notified via email/SMS
  → Campaign published
```

### Payout (Withdrawal) Flow

```
Campaign organizer requests withdrawal
  → POST /v1/withdrawal — submit request
  → Admin views payout queue in giv-admin-new
  → Admin initiates 2FA → POST /v1/admin/payouts/two-factor/initialize
  → Admin enters TOTP code → POST /v1/admin/payouts/approve
  → Backend triggers Paystack transfer to organizer's bank account
  → Status updated to completed
  → Organizer notified via email
```

### Admin Authentication Flow

```
Admin enters email
  → POST /v1/otp/send — OTP sent via email/SMS
  → Admin enters OTP
  → POST /v1/admin/auth/sign-in — returns JWT
  → GET /v1/admin/auth/permissions — permissions fetched and stored
  → JWT stored in Zustand; all subsequent requests use Bearer token
  → On 401: Axios interceptor attempts token refresh
```

## API Surface — krogiving-backend

### Public Endpoints

| Resource | Description |
|---|---|
| `GET /v1/campaigns` | List active campaigns |
| `GET /v1/campaigns/:slug` | Campaign detail |
| `POST /v1/donations/initialize` | Start a donation |
| `POST /v1/paystack/webhook` | Paystack payment webhook |
| `POST /v1/otp/send` | Send OTP for auth |
| `POST /v1/users/register` | User registration |
| `POST /v1/users/login` | User login |

### Authenticated User Endpoints

| Resource | Description |
|---|---|
| `POST /v1/campaigns` | Create campaign |
| `GET /v1/campaigns/mine` | Own campaigns |
| `POST /v1/withdrawal` | Request withdrawal |
| `GET /v1/profile` | User profile |

### Admin Endpoints (prefix: `/v1/admin/`)

| Resource | Description |
|---|---|
| `POST /admin/auth/sign-in` | Admin login |
| `GET /admin/campaigns` | All campaigns |
| `PATCH /admin/campaigns/:id/status` | Approve/reject campaign |
| `GET /admin/donations` | All donations |
| `GET /admin/donations/export/csv` | Export donations CSV |
| `POST /admin/payouts/approve` | Approve payout |
| `POST /admin/payouts/two-factor/initialize` | Init TOTP for payout |
| `GET /admin/users` | All users |
| `GET /admin/logs` | Admin audit logs |
| `PUT /admin/settings/global-withdrawal-cap` | Set withdrawal cap |

## Third-Party Integrations

| Service | Purpose | Component |
|---|---|---|
| Paystack | Payment processing and bank transfers | krogiving-backend |
| DigitalOcean Spaces | Image file storage (S3-compatible) | krogiving-backend |
| SendGrid | Transactional email | krogiving-backend |
| MailerSend | Email fallback | krogiving-backend |
| Termii | SMS OTP | krogiving-backend |
| Telnyx | SMS fallback | krogiving-backend |
| Strapi | CMS content (blog, static pages) | krogiving-backend + krogiving-frontend |
| Redis | OTP cache | krogiving-backend |
| Highlight.io | Error tracking | krogiving-backend + krogiving-frontend |
| Cloudinary | Campaign video upload | krogiving-frontend (direct upload) |
| Split.io | Feature flags / A/B testing | krogiving-frontend |

## Scheduled Jobs

Defined in `src/scheduler/scheduler.module.ts` using `@nestjs/schedule`:

- Currency exchange rate refresh
- Stale OTP cleanup
- Campaign status auto-updates (expired campaigns)

## Operational Notes

- **Webhook verification:** Paystack webhooks must be verified using the `x-paystack-signature` header before any processing. Never trust unverified webhooks.
- **Dual DB:** Admin data lives in MongoDB; user-facing financial data lives in PostgreSQL. Never cross-query.
- **Slug migration:** `SlugMigrationService` runs on startup to handle campaign slug backfills — it is idempotent.
- **TypeORM migrations:** `migrationsRun: true` — migrations run automatically on startup. Ensure `dist/migrations/*.js` are compiled before starting in production.
- **Admin permissions:** Fine-grained per-route. `AdminPermissionGuard` reads from MongoDB `Permission` collection.
- **CKEditor license:** The frontend uses `ckeditor5-premium-features` which requires a valid CKEditor commercial license key.
- **Console log guard:** Husky pre-commit hook (`scripts/check-console-log.js`) blocks commits with `console.log` statements.

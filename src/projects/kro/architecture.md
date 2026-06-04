---
icon: material/layers-outline
---

# KROTrust — Architecture

## Tech Stack

| Layer | Choice |
|---|---|
| Framework | NestJS v10, TypeScript |
| Database | PostgreSQL (DigitalOcean Managed DB) via TypeORM |
| Auth | JWT + Passport — separate strategies for users and admins |
| Email | SendGrid |
| SMS | Multi-provider manager — Twilio, Telnyx, Termii (failover between them) |
| Storage | DigitalOcean Spaces (AWS S3-compatible) |
| Real-time | WebSockets via `@nestjs/websockets` + Socket.IO |
| Observability | Highlight.io, OpenTelemetry (OTLP HTTP exporter) |
| 2FA | TOTP via `speakeasy` (for admin sensitive operations) |
| Scheduled jobs | `@nestjs/schedule` |
| Config | node-config pattern — `NODE_CONFIG_DIR` maps to a mounted `production.json` |

## Module Structure

```
src/
├── core/
│   ├── config/            — App config loader
│   ├── constants/         — App-wide constants, currency maps
│   ├── decorators/        — @UserContext, @RoleNames, @AuthenticationContext
│   ├── enums/             — Common enums (env, currencies, routes, errors)
│   ├── globalModules/
│   │   ├── auth/          — JWT strategies (user, admin), guards, token client
│   │   ├── captcha/       — reCAPTCHA verification
│   │   ├── encryption/    — bcrypt service
│   │   ├── sendGrid/      — Email service
│   │   ├── sms/           — SMS manager (Telnyx, Termii, WhatsApp failover)
│   │   └── twoFactor/     — TOTP 2FA (speakeasy)
│   └── guards/            — Roles guard, domain guard, user status guard
├── order/                 — Core domain: order lifecycle
├── payment/               — Payment processing (Paystack, Fincra, Providus)
├── management/            — User, seller, buyer management
├── kroBalance/            — Internal wallet / escrow
├── verificationCodes/     — Phone OTP verification
├── refund/                — Refund processing
└── utils/                 — DO Spaces utilities, exception filters
```

Each major module follows a consistent three-layer pattern:

```
module/
├── controller/   — HTTP layer (routes, DTOs, guards)
├── database/     — TypeORM repositories and entities
└── domain/       — Business logic services and interfaces
```

## Request Flow

```
HTTP Request
  → Nginx (reverse proxy on port 80/443)
  → NestJS HTTP pipeline
  → authentication.guard.ts or admin-auth.guard.ts
  → JWT strategy validates token
  → Roles guard checks user role
  → Controller method
  → Domain service (business logic)
  → TypeORM repository → PostgreSQL
  → Response
```

**Auth model:**

- Users authenticate via `passport-jwt` strategy.
- Admins have a separate JWT strategy (`admin-auth.strategy.ts`).
- Phone numbers verified via OTP codes in `verificationCodes` module.
- TOTP 2FA (via `speakeasy`) required for admin payout approvals.
- Magic links supported via `MagicLinkLog` table.

**WebSocket:** Real-time order status updates and notifications via Socket.IO.

## Core Domain — Orders

An **Order** is the central entity: a conditional payment agreement between a buyer and a seller.

### Order lifecycle

```
CREATED        — Buyer creates order with defined conditions
   ↓ Seller accepts
ACCEPTED
   ↓ Work begins
IN_PROGRESS
   ↓ Seller marks complete
COMPLETED
   ↓ Conditions verified, payment released
CLOSED
   ↓ (alternatively)
DISPUTED / ISSUE_REPORTED
   ↓ Resolution
REFUNDED
```

**Conditions** are predefined templates (from `conditions-seed`) or custom. All conditions must be marked fulfilled before payment is released.

**KRO Balance** holds payment in escrow via the `kroBalance` module until order closes.

## Payment Providers

| Provider | Location | Use case |
|---|---|---|
| Paystack | `src/payment/paymentAPI/paystack/` | Primary Nigerian payment gateway |
| Fincra | `src/payment/paymentAPI/fincra/` | Alternative payment provider |
| Providus | `src/payment/paymentAPI/providus/` | Bank transfer provider |

Payment webhooks are verified before updating order/balance state.

## Database Schema

Key tables (managed via TypeORM migrations in `/migrations/`):

| Table | Purpose |
|---|---|
| `users` | All platform users |
| `orders` | Order records with conditions and state |
| `conditions` | Condition definitions per order |
| `kro_balance` | Escrow wallet per user |
| `transactions` | Payment transaction records |
| `refunds` | Refund records |
| `magic_link_logs` | Magic link auth tokens |
| `verification_codes` | OTP records for phone verification |
| `payment_webhook_info` | Raw Paystack/Fincra/Providus webhook payloads |
| `two_factor_admin` | TOTP secrets for admin 2FA |
| `fee_configuration` | Platform fee rules |
| `user_roles` | Role definitions |

## Configuration

Config is file-based via node-config. The file name maps to `NODE_ENV` (e.g., `production.json`). The file is never committed — it is placed on the server and mounted as a Docker volume.

**Key config sections:**

```
database.*         — PostgreSQL host, port, user, password, DB name, SSL CA cert path
jwt.*              — JWT secrets and expiry
sendGrid.*         — API key and sender email
twilio.* / telnyx.* / termii.*  — SMS provider credentials
paystack.* / fincra.* / providus.*  — Payment gateway credentials
aws.*              — DO Spaces access key, secret, region, bucket
highlight.*        — Error tracking project ID
```

## Frontend (kro-frontend + kro-admin)

Both are React SPAs served as static sites behind Nginx.

- **kro-frontend** — User-facing app (orders, payments, profile, wallet).
- **kro-admin** — Admin dashboard (order management, user management, payouts).
- Both communicate exclusively with `kro-backend` over HTTPS.

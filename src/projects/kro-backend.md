# kro-backend

## Business Purpose

The KRO backend is the API server for KROTrust (krotrust.com), a peer-to-peer conditional payment and escrow platform. It allows:

- **Buyers** to place orders with conditions that must be met before payment is released
- **Sellers** to fulfill orders and receive payment once conditions are verified
- **Dispute resolution** when conditions are contested
- **Secure in-app balances** (KRO Balance wallet) for holding and releasing funds
- **Multi-provider payments** via Paystack, Fincra, and Providus
- **Notifications** via email (SendGrid), SMS (Twilio/Telnyx/Termii), and WhatsApp

---

## Architecture

**Framework:** NestJS v10, TypeScript  
**Database:** PostgreSQL (DigitalOcean Managed DB) via TypeORM  
**Auth:** JWT + Passport (multiple strategies: user JWT, admin JWT)  
**Email:** SendGrid  
**SMS:** Twilio, Telnyx, Termii (multi-client manager)  
**Storage:** AWS S3 / DigitalOcean Spaces  
**Observability:** Highlight.io, OpenTelemetry (OTLP HTTP exporter)  
**2FA:** TOTP via `speakeasy`  
**Real-time:** WebSockets via `@nestjs/websockets` + Socket.IO  
**Scheduled jobs:** `@nestjs/schedule`  

### Module Structure

```
src/
├── core/
│   ├── config/         — App config loader (node-config pattern)
│   ├── constants/      — App-wide constants and currency maps
│   ├── decorators/     — @UserContext, @RoleNames, @AuthenticationContext
│   ├── enums/          — Common enums (env, currencies, routes, errors)
│   ├── globalModules/
│   │   ├── auth/       — JWT strategies (user, admin), guards, token client
│   │   ├── captcha/    — reCAPTCHA verification
│   │   ├── encryption/ — bcrypt encryption service
│   │   ├── sendGrid/   — Email service
│   │   ├── sms/        — SMS manager (Telnyx, Termii, WhatsApp)
│   │   └── twoFactor/  — TOTP 2FA (speakeasy)
│   └── guards/         — Roles guard, domain guard, user status guard
├── order/              — Core business: order lifecycle management
├── payment/            — Payment processing (Paystack, Fincra, Providus)
├── management/         — User/seller/buyer management
├── kroBalance/         — Internal wallet / balance
├── verificationCodes/  — Phone OTP verification
├── refund/             — Refund processing
└── utils/              — DigitalOcean Spaces utilities, exception filters
```

Each major module (`order`, `payment`, `management`, `kroBalance`) follows a consistent layered pattern:

```
module/
├── controller/   — HTTP layer (routes, DTOs, guards)
├── database/     — TypeORM repositories and entities
└── domain/       — Business logic services and interfaces
```

---

## Request Flow

```
HTTP Request
  → Nginx (proxy_pass to backend:3000)
  → NestJS HTTP pipeline
  → Guard (authentication.guard.ts or admin-auth.guard.ts)
  → JWT strategy validates token
  → Roles guard checks user role
  → Controller method
  → Domain service (business logic)
  → Repository (TypeORM query to PostgreSQL)
  → Response
```

**Authentication:**
- Users authenticate via JWT (`passport-jwt` strategy)
- Admins have a separate JWT strategy (`admin-auth.strategy.ts`)
- Phone numbers are verified via OTP codes stored in `verificationCodes` module
- 2FA (TOTP via speakeasy) for sensitive operations (admin payout approvals)
- Magic links supported via `MagicLinkLog` table

**WebSocket events:** Real-time updates for order status changes, notifications.

---

## Payment Providers

The backend integrates three payment providers, selectable per transaction:

| Provider | Module Location | Use Case |
|---|---|---|
| Paystack | `src/payment/paymentAPI/paystack/` | Primary Nigerian payment gateway |
| Fincra | `src/payment/paymentAPI/fincra/` | Alternative payment provider |
| Providus | `src/payment/paymentAPI/providus/` | Bank transfer provider |

Payment webhooks are handled and verified before updating order status.

---

## Core Domain: Orders

An **Order** is the central entity. It represents a conditional payment agreement between a buyer and a seller.

Order lifecycle:
1. **Created** — Buyer creates order with conditions
2. **Accepted** — Seller accepts order terms
3. **In Progress** — Work/service in progress
4. **Completed** — Seller marks as done
5. **Closed** — Conditions verified, payment released to seller
6. **Disputed** / **Issue Reported** — Buyer disputes conditions
7. **Refunded** — Refund processed back to buyer

**Conditions** are predefined templates (`conditions-seed`) or custom. They must be marked fulfilled before payment releases.

**KRO Balance** holds payment in escrow until the order closes.

---

## Database Schema Overview

See [databases.md](../databases.md) for details. Key tables:

| Table | Created By |
|---|---|
| `fee_configuration` | `1000000000000` |
| `user_roles` | `1000000000002` |
| `users` | core |
| `orders` | `1000000000007` |
| `conditions` | `1000000000008` |
| `kro_balance` | `1000000000006` |
| `transactions` | payment module |
| `refunds` | `1721978170444` |
| `magic_link_logs` | `1724268172180` |
| `verification_codes` | `1725923355192` |
| `payment_webhook_info` | `1719379800300` |
| `two_factor_admin` | twoFactor module |

---

## Environment Configuration

Configuration is file-based (node-config pattern). The file name is based on `NODE_ENV` (e.g., `production.json`).

Config is loaded from: `NODE_CONFIG_DIR` env var (mapped to container at `/usr/src/app/local/`).

Key configuration keys (inferred from schema):
- `database.*` — PostgreSQL connection details
- `database.ssl` — Path to CA certificate file
- `jwt.*` — JWT secrets and expiry
- `sendGrid.*` — Email API key and sender
- `twilio.*` / `telnyx.*` / `termii.*` — SMS providers
- `paystack.*` / `fincra.*` / `providus.*` — Payment gateways
- `aws.*` — S3 / Spaces credentials
- `highlight.*` — Error tracking project ID

---

## Deployment Flow

1. Code merged to `main` in `kro-backend` repo
2. `kro-devops` CI (separate repo) SSHs to droplet and pulls devops config
3. Developer manually rebuilds: `docker-compose up --build -d backend`
4. Nginx continues serving; backend restarts with zero-downtime (container replacement)

**Container start command:** `npm run startProductionDocker`  
Which runs: `cross-env NODE_ENV=production cross-env NODE_CONFIG_DIR=/usr/src/app/local npm run start`

The `production.json` config file and DB CA cert are mounted via Docker volumes.

---

## Operational Notes

- **Multi-environment:** Supports `local`, `dev`, `stage`, `preprod`, `production` via `NODE_ENV`
- **Database migrations:** TypeORM migrations in `/migrations/` directory, must be run manually
  ```bash
  npm run typeorm -- migration:run -d <data-source>
  ```
- **SMS failover:** The SMS client manager (`sms-clients.manager.ts`) can switch between Telnyx, Termii, and WhatsApp-based OTP delivery
- **GeoIP:** `geoip-lite` is used to log request origin by geography
- **Rate limiting:** Not explicitly visible in packages — verify in guards
- **QR codes:** `qrcode` package used (likely for 2FA TOTP setup QR codes)
- **Phone number validation:** `libphonenumber-js` ensures E.164 format

---

## Shutdown and Recovery

See [shutdown-and-recovery.md](../shutdown-and-recovery.md) for full procedures.

Quick restart:
```bash
ssh root@188.166.145.68
cd /root/Kro/kro-devops/production
docker-compose restart backend
docker-compose logs -f backend
```

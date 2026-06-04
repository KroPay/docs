# Databases

## Summary

| Platform | Database | Type | ORM / Client | Purpose |
|---|---|---|---|---|
| KRO | Main DB | PostgreSQL | TypeORM | Orders, payments, users, balances |
| GIV | Main relational | PostgreSQL | TypeORM | Donations, withdrawals, GIV accounts |
| GIV | Main document | MongoDB | Mongoose | Campaigns, admins, OTP, notifications |
| GIV | Cache | Redis | ioredis | OTP TTL, caching |
| Pencom | Core | PostgreSQL | TypeORM | Employers, employees, contributions |
| Pencom | Payments | PostgreSQL | TypeORM | Payment records, Remita transactions |
| Pencom | Compliance | PostgreSQL | TypeORM | GLI, PCC, PFC compliance data |
| Pencom | External Integrations | PostgreSQL | TypeORM | PENCOM sync data |
| Pencom | Audit | PostgreSQL | TypeORM | Immutable audit trail |
| Pencom | Notifications | MongoDB | Mongoose | Notification records |
| Pencom | PENCOM Oracle | Oracle | oracledb | On-prem PENCOM read integration |

---

## KRO — PostgreSQL

**ORM:** TypeORM  
**Provider:** DigitalOcean Managed PostgreSQL  
**SSL:** Required in non-local environments (CA cert mounted in container)

### Migrations

Located in `kro-backend/migrations/`. Notable migrations:

| File | Description |
|---|---|
| `1000000000000-fee-configuration-table` | Initial fee config |
| `1000000000002-roles-seed` | Roles setup |
| `1000000000004-insert-admin` | Admin seed |
| `1000000000005-userRating-table` | User ratings |
| `1000000000006-kroBalance-table` | KRO balance tracking |
| `1000000000007-orders-table` | Orders |
| `1000000000008-conditions-table` | Order conditions |
| `1000000000009-conditions-seed` | Condition templates seed |
| `1688471728624-user-passcode` | User passcode auth |
| `1689696513332-users-kroHandle` | @kroHandle usernames |
| `1692292113129-transactions-internal-id` | Internal transaction IDs |
| `1721978170444-refund-table` | Refunds |
| `1724268172180-CreateMagicLinkLogTable` | Magic link auth logs |
| `1725923355192-AddPhoneNumberToVerificationCodes` | Phone verification |

### Key Modules

| Module | Purpose |
|---|---|
| `order` | Core business entity — buy/sell orders with conditions |
| `payment` | Payments via Paystack, Fincra, Providus |
| `management` | User management, roles |
| `kroBalance` | Internal wallet/balance per user |
| `verificationCodes` | OTP codes for phone verification |
| `refund` | Refund records |

---

## GIV — MongoDB (Mongoose)

**Provider:** DigitalOcean Managed MongoDB  
**URI:** `MONGO_URI` env var

### Collections (schemas)

| Schema | Location | Purpose |
|---|---|---|
| `Admin` | `src/admin/schemas/admin.schema.ts` | Admin users |
| `AdminLog` | `src/admin/schemas/admin-log.schema.ts` | Admin action logs |
| `CampaignReview` | `src/admin/schemas/campaign-review.schema.ts` | Campaign moderation |
| `Permission` | `src/admin/schemas/permission.schema.ts` | Admin permission roles |
| `Settings` | `src/admin/schemas/settings.schema.ts` | Platform settings |
| `WithdrawalCapLog` | `src/admin/schemas/withdrawal-cap-log.schema.ts` | Withdrawal cap change audit |

> **Note:** Campaigns, users (non-admin), and donations are stored in PostgreSQL, not MongoDB. MongoDB is used for admin-side data and logs in GIV.

---

## GIV — PostgreSQL (TypeORM)

**Provider:** DigitalOcean Managed PostgreSQL  
**SSL:** Required in non-development environments  
**Migrations:** Auto-run on startup (`migrationsRun: true`, files in `dist/migrations/*.js`)

### Key Entities

| Entity | Purpose |
|---|---|
| `ActivityLog` | User activity tracking |
| Campaign entities | Campaign creation and management |
| Donation entities | Donation records |
| Withdrawal entities | Payout/withdrawal requests |
| GIV account details | Bank account details for payouts |
| Payment entities | Payment transaction records |
| User entities | Platform users |

---

## GIV — Redis

**Client:** ioredis  
**Config:** `REDIS_HOST`, `REDIS_PORT`, `TTL`  
**Usage:** OTP code caching with TTL expiry, general caching via `@nestjs/cache-manager`

---

## Pencom — PostgreSQL (Multiple Databases)

Each microservice has its own isolated database. TypeORM handles migrations per service.

### Migration Commands

```bash
# Generate migration
yarn migration:generate

# Run migration
yarn migration:run

# Service-specific (e.g., payments)
yarn migrate:payments
```

### Core DB (`pencom_core_db2`)

Contains the main business entities: employers, employees, pension fund administrators (PFAs), RSA PINs, contribution records, change requests.

### Payments DB (`pencom_payments`)

Remita payment transactions, payment status tracking, certificate service fees.

Fee structure (from env config):
- 3–50 employees: ₦100,000
- 51–100 employees: ₦150,000
- 101+ employees: ₦250,000
- + 7.5% tax

### Compliance DB (`pencom_compliance`)

- **GLI** (Group Life Insurance) compliance records
- **PCC** (Pension Clearance Certificate) applications and approvals
- **PFC** (Pension Fund Custodian) upload processing
- Discrepancy records

### External Integrations DB (`pencom_external_integrations`)

Data fetched from the PENCOM Oracle database and stored locally for processing.

### Audit DB

Immutable audit trail. Schema defined in `apps/audit/src/entities/audit.entity.ts` and `apps/audit/src/migrations/1760110156398-AddAuditTable.ts`.

---

## Pencom — MongoDB (Notifications)

**URI:** `NOTIFICATIONS_DB_URI` (e.g., `mongodb://localhost:27017/pencom_notifications`)  
**Usage:** Notification records managed by the `notifications` microservice.

---

## Pencom — Oracle DB (PENCOM On-Prem)

**Connection:** `PENCOM_DB_HOST:1521`  
**Client:** `oracledb` npm package  
**Usage:** Read-only integration with the official PENCOM database. Data is pulled into the `external-integrations` PostgreSQL database for processing.

Environment variables:
- `PENCOM_DB_HOST`
- `PENCOM_DB_PORT` (default: 1521)
- `PENCOM_DB_USER`
- `PENCOM_DB_PASSWORD`
- `PENCOM_DB_SERVICE_NAME`

---

## Backup & Recovery Notes

- DigitalOcean Managed Databases (PostgreSQL, MongoDB, Redis) include automated daily backups managed by DigitalOcean. Restore via the DO console.
- On-premises Pencom databases must have backup procedures configured by the infrastructure team. Check with the DevOps team for backup schedules.
- Oracle PENCOM DB is managed by PENCOM and is read-only from the application's perspective.
- See [shutdown-and-recovery.md](./shutdown-and-recovery.md) for recovery procedures.

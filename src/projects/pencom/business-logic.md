---
icon: material/book-open-outline
---

# eHub — Business Logic

> Non-trivial rules, calculations, and workflows. All facts derived from source code. Sections marked **⚠** flag code that disagrees with itself or with documented behaviour.

## 1. Authentication & Session

### 1.1 Employer OTP flow

Source: `apps/core/src/auth/otp/otp.service.ts`, `auth/auth.service.ts`

- OTP = 6 numeric digits, stored **hashed** in MongoDB `otps` collection.
- Plaintext sent via SendGrid. Returned in API response in non-production (`process.env.NODE_ENV !== 'production'`).
- TTL: `OTP_EXPIRY_MINUTES` env var. **⚠ Declared twice in `.env.example`** (5 then 15) — last one wins.
- No TTL index on the `otps` collection — expired records accumulate.
- First login: OTP sent to user's email. After first successful verify, `User.isFirstLogin` is cleared and subsequent OTPs go to the authorized contact.

### 1.2 Admin OTP flow

Source: `apps/core/src/admin/services/admin-auth.service.ts`

- OTP stored as `AdminOtpSession` row in Postgres (not Mongo) with `otpHash`, `expiresAt`, `attemptCount=0`.
- Lockout: after `MAX_OTP_ATTEMPTS` (default 3) failed attempts → `isLocked=true`, `lockedUntil=now+LOCKOUT_DURATION_MINUTES`.
- Resend window skip: if `User.lastLoginAt` is within `ADMIN_OTP_RESEND_WINDOW_HOURS`, OTP is skipped and JWT returned directly.
- Replay protection: `isUsed=true` after first successful verify.

### 1.3 Admin magic links (onboarding / password reset)

- Token = `randomBytes(32).hex` (64-char hex).
- UserId encrypted AES-256-CBC, key = `scrypt(JWT_SECRET, 'salt', 32)`, random per-call IV.
- Stored in Redis as `admin_magic_link:<token>` for `MAGIC_LINK_TTL_SECONDS` (default 300 seconds = 5 minutes).

### 1.4 Admin session lifecycle

Source: `libs/shared/src/strategies/jwt.strategy.ts`, `apps/core/src/admin/services/session-cleanup.service.ts`

- `AdminSession` row created on successful OTP verify: `expiresAt = now + SESSION_TIMEOUT_MINUTES` (default 15).
- **Per-request check** when `userType=admin`: reject if session is missing, `isActive=false`, or expired. Update `lastActivityAt=now`.
- **⚠ Code computes `newExpiresAt` but never writes it back** — sliding expiry is not actually implemented.
- **Cleanup cron**: every 10 minutes, set `isActive=false` on rows where `expiresAt < now AND isActive=true`.

### 1.5 Employer onboarding state machine

```
NOT_STARTED
   ↓ registerEmployerAuth() / registerEmployerWithCode()
OTP_VERIFIED
   ↓ verifyRegistration()
AUTHORIZED_CONTACT_PENDING
   ↓ verifyAuthorizedContact()
COMPLETED  +  Employer.registrationStatus = 'approved'
```

- Path B (existing PENCOM employer code) can jump directly to `COMPLETED + approved` if Oracle confirms the employer.
- Discrepancy gate: `registerEmployerAuth` returns 400 if an OPEN `Discrepancy` row exists for the employer code.

### 1.6 Pilot allowlist

Source: `apps/api-gateway/src/auth/auth.controller.ts`

- `AUTH_PILOT_ALLOWLIST` env var is a JSON array `[{code?: string, email?: string[]}]`.
- Every `POST /auth` is checked: requests for an employer code or email not in the list are rejected with "invitation-only" message.
- **⚠ Not documented in `.env.example`.** If missing/empty, all logins may be blocked.

---

## 2. Certificate Service Fee Calculation

Source: `apps/core/src/services/employee.service.ts`

`CERTIFICATE_SERVICE_FEES` env var (JSON):

```json
{
  "ranges": [
    { "minEmployees": 3,   "maxEmployees": 50,  "fee": 100000 },
    { "minEmployees": 51,  "maxEmployees": 100, "fee": 150000 },
    { "minEmployees": 101, "maxEmployees": null, "fee": 250000 }
  ],
  "taxPercent": 7.5
}
```

Algorithm:

1. Load `Employer.numberOfEmployees`.
2. Find the range where `min ≤ count ≤ max` (treating `null` as +∞).
3. Return `{fee, taxPercent}`.

**⚠ Edge cases:**

- `max === undefined` does NOT match — must be explicit `null`. Misconfiguration → 500.
- Employer with `< 3` employees → 500 (no fallback range).
- `numberOfEmployees` is updated after employee migration, but not on ADDITION requests until they're approved.

---

## 3. Penalty Calculation

The most algorithmically complex feature.

Source: `apps/compliance/src/services/penalty.service.ts`

### 3.1 Configuration

| Env var | Default | Notes |
|---|---|---|
| `PENALTY_MONTHLY_DEFAULTING_RATE` | `0.02` (2%/month) | Active in live path |
| `PENALTY_TRANSACTION_ISOLATION` | `SERIALIZABLE` | Can downgrade to `READ_COMMITTED` |
| `PENALTY_GRACE_DAYS` | `11` | **⚠ Only honoured in backfill.** Live penalty service hard-codes 11. |
| `PENALTY_DAILY_RATE` | — | **⚠ Does NOT exist in code.** Daily rate is always derived from monthly rate. |
| `REMITTANCE_DUE_DAY` | `28` | Clamped 1–28 |

Daily rate derivation: `dailyRate = monthlyRate × 12 / 365`. With defaults: `≈ 0.000657534/day`.

### 3.2 Penalty formula (per employee row, COM contributions only)

```
days        = max(0, value_date − grace_end_date)         (calendar days)
penaltyBase = employee_contribution + employer_contribution      (AVC excluded)
share       = penaltyBase × (monthlyRate × 12 / 365) × days
total       = SUM(share) over all employees for that employer-month
```

- `due_date` = last calendar day of month.
- `grace_end_date` = `due_date + 11`.
- First chargeable day = `due_date + 12`.
- AVC fields (`employee_avc`, `employer_avc`) are excluded from `penaltyBase`.

### 3.3 Penalty state machine

```
GAP_DETECTED          → no COM row yet for this month
   ↓ handleComUpload
PENALTY_COMPUTED      → penalty_amount and days_in_default set
   ↓ handlePenUpload  (partial PEN payment)
PARTIALLY_PAID
   ↓ handlePenUpload  (PEN total ≥ penalty_amount)
PAID_IN_FULL
```

Once `PARTIALLY_PAID` or `PAID_IN_FULL`, recomputation is **skipped** — penalties freeze after payment starts.

### 3.4 Why `SERIALIZABLE` transactions?

Two PFCs can independently send COM or PEN rows for the same `(employer, month)` simultaneously. Combined with row-level pessimistic write lock on the penalty row, SERIALIZABLE prevents:

- Lost updates to `paid_amount` when two PEN uploads finalise the row.
- Inconsistent COM history during `penalty_amount` recompute.

### 3.5 Cron triggers

| Trigger | Frequency |
|---|---|
| CSV upload completion (`EmployeeJobProcessor.runPostStreamPenaltyLogic`) | Per upload |
| `EmployerRebuildService.rebuildEmployer` | Manual via `POST /admin/cleanup/rebuild/:code` |
| `BackfillCronService.runSweep` | Manual via `POST /penalty/recalculate-all` (**⚠ no guard**) |
| `GracePeriodNotificationCronService` | Daily at 1am — sends notifications only, does NOT recompute |

---

## 4. Contribution Deduplication

Source: `apps/compliance/src/migrations/1780180264632-UpdateContributionDuplicateKey.ts`

### Natural key for `employee_monthly_contributions`

```
(employer_code, rsa_pin, date_of_contribution, value_date, contribution_type)
```

Enforced at:

- DB layer: partial unique index `UQ_employee_contributions_monthly` (WHERE all four NOT NULL).
- Runtime: `employeeContributionNaturalKey()` + `orIgnore()` on bulk insert.

Bulk insert chunked at `floor(65535 / 17) = 3855` rows to stay under Postgres bound-parameter limit.

Crash-safe resume: `(jobId, sourceRowNumber)` protected by a separate partial unique index. CSV processor checkpoints every 10,000 rows.

---

## 5. Compliance Rule Engine

Source: `apps/compliance/src/config/compliance-rules.config.ts`

Current rule set (version `2.0`, last updated `2025-11-07`):

```
FULL_01 — "Fully Compliant"
conditions:
  - contri_status === true
  - gli_status === 'ACTIVE'
  - penalty_cleared === true   ⚠ currently commented out / stubbed
aggregation: all_must_pass
```

**⚠ `penalty_cleared` check is commented out** in `compliance-evaluation.service.ts` — an employer can be evaluated as "Fully Compliant" even with outstanding penalties.

---

## 6. PCC Certificate Workflow

Source: `apps/compliance/src/admin/services/pcc-approval.service.ts`

### State machine

```
PENDING
   ↓ Admin first review
REVIEWED_FIRST
   ↓ Admin second review
REVIEWED_SECOND
   ↓ Admin approve → Certificate issued
APPROVED
   ↓ Hourly cron (expiry_date < now)
EXPIRED

REVIEWED_SECOND
   ↓ Admin query
QUERIED ↔ EMPLOYER_RESPONDED
   ↓
REJECTED
```

Each transition writes a `PccActivity` row with `actor_id`, `actor_role`, `previous_status`, `new_status`.

### Certificate issuance (on APPROVE)

1. `CertificateGeneratorService` (pdfkit) renders a PDF.
2. `FileUploadService.saveFile(buffer, 'CERTIFICATES')` → MinIO.
3. Creates `Certificate` row:
   - `pcc_number = PCC/${YYYY}/${zero-padded-sequence}` (unique).
   - `status = Active`.
   - `expiry_date = today + N months` (typically 12).
   - `certificate_path` = MinIO URL.
   - `verification_url` = `${FRONTEND_URL}/verify-pcc/${token}`.
   - `digital_signature` = hash of certificate contents.
4. Sends `SEND_PCC_STATUS_UPDATE_EMAIL`.

---

## 7. Payment Lifecycle (Remita)

Source: `apps/payments/src/payments.service.ts`

### State machine

```
PENDING → SUCCESSFUL (via webhook OR validate)
PENDING → FAILED     (via validate returning terminal Remita error)
CANCELLED — declared but never reached ⚠
```

### Charge composition (CERTIFICATE_GENERATION)

| Layer | Amount |
|---|---|
| Service fee | From `CERTIFICATE_SERVICE_FEES` (e.g. ₦100,000) |
| VAT | 7.5% of service fee |
| Convenience Fee | **₦161.25 hardcoded** — no env override **⚠** |

- `Payment.amount` stores `serviceFee + appliedChargesTotal` (the Remita-billed total).
- Response `amount` field shows `serviceFee + chargesTotal` (display total including non-applied charges).
- **These two numbers are different. ⚠**

### Idempotency

- **On create**: if a PENDING `Payment` row exists for `(identifier, paymentType)`, return its existing RRR — no new Remita call.
- **On webhook**: lookup by `orderId` OR `rrr`. Re-receipt overwrites to SUCCESSFUL — idempotent.
- **⚠ No DB unique constraint** on `(identifier, paymentType)` — concurrent `POST /create` calls can create duplicate PENDING rows.

### Remita signing

- Init hash: `sha512(merchantId + serviceTypeId + orderId + remitaAmount + apiKey)`.
- Status hash: `sha512(rrr + apiKey + merchantId)`.
- Response is JSONP — parsed via regex `/jsonp\s*\((.*)\)/`. **⚠ Brittle** — fails on whitespace or alternative callback names.
- Webhook: **no signature verification**.

---

## 8. Scheduled Jobs Reference

All crons in `compliance` except `SessionCleanupService` in `core`.

| Job | Schedule | Purpose |
|---|---|---|
| `SessionCleanupService` | `*/10 * * * *` | Invalidate expired admin sessions |
| `CertificateExpiryCronService` | `0 * * * *` | Expire certificates; demote evaluations. **⚠ Not recorded in `cron_job_runs`** |
| `RevaluateAllEmployersCronService` | `0 0 * * *` | Re-run compliance evaluation for all approved employers |
| `ApprovedEvaluationStatusCronService` | `0 * * * *` | Restore `approved+NON_COMPLIANT` evaluations to `FULLY_COMPLIANT` |
| `GracePeriodNotificationCronService` | `0 1 * * *` | Send `SEND_PENALTY_INCURRED` emails. `maxPerRun=500`. |
| `GliExpiryService` | `0 0 * * *` | Mark expired GLI records |

Manually triggered (no `@Cron`):

- `BackfillCronService.runSweep()` — `POST /penalty/recalculate-all`. **⚠ No guard — any internal network caller can trigger a full sweep.**
- `MissingContributionsRepairService.run()` — `POST /admin/cleanup/repair-missing-contributions`.

---

## 9. Key Edge Cases

### Auth / sessions

- `User.email` normalised to lowercase on insert/update — case differences collide on the unique index.
- Refresh token cookie path is `/auth` — frontend must POST to a `/auth/*` URL to trigger refresh.
- Admin sliding-expiry is computed but never persisted — sessions expire at their original `expiresAt` regardless of activity.

### Payments

- `orderId = "ORD_" + Date.now()` — not collision-safe under burst load; no unique constraint.
- Remita webhook accepts both `channel` AND `channnel` (documented typo in Remita's API).
- `CANCELLED` payment status declared but never set by any code path.

### Compliance

- `PENALTY_GRACE_DAYS` env var only takes effect in backfill sweep. Live penalty computation hard-codes 11 days.
- `POST /penalty/recalculate-all` has no guard — any internal caller can trigger a full penalty sweep.
- `POST /compliance/run-daily-accrual` is JWT-only with no admin role — any employer can trigger it.

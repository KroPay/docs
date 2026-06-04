---
icon: material/database-outline
---

# eHub — Domain Model

> All entities derived from source code. Sections marked **⚠** flag ambiguities.

## BaseEntity (shared by all TypeORM entities)

`libs/database/src/entities/base.entity.ts`

| Column | Type | Notes |
|---|---|---|
| `id` | uuid | `@PrimaryGeneratedColumn('uuid')` |
| `createdAt` | timestamptz | `@CreateDateColumn` |
| `updatedAt` | timestamptz | `@UpdateDateColumn` |
| `createdBy` | varchar | nullable |
| `updatedBy` | varchar | nullable |
| `deletedAt` | timestamp | `@DeleteDateColumn` (soft delete) |

---

## 1. Core Service (`pencom_core_db2`)

### User → `user`

- `email` varchar **unique** (normalised to lowercase on insert/update)
- `passwordHash` nullable; `phoneNumber` unique nullable
- `firstName`, `lastName`
- `userType` enum: `EMPLOYER`, `ADMIN`
- `adminRole` enum `AdminRole` nullable — **⚠ only `SUPER_ADMIN` is declared**
- `onboardingStatus` enum: `NOT_STARTED → OTP_VERIFIED → AUTHORIZED_CONTACT_PENDING → COMPLETED`
- `isFirstLogin` bool; `isActive` bool; `lastLoginAt` timestamp nullable
- Relations: `@OneToOne Employer`, `@OneToMany Employer (registeredByAdmin)`

### Employer → `employers`

- `@OneToOne User`
- `employerCode` varchar **unique** nullable; `cacNumber` varchar **unique**
- `companyName`; `companyAddress` text; `tin` nullable
- `numberOfEmployees` int default 0
- `registrationStatus`: `pending_review | approved | rejected`
- `firstPaymentDate` date nullable — anchor for penalty gap detection
- `tourCompleted` bool; `complianceStatus` enum nullable
- Relations: `@OneToMany Employee`, `@OneToMany EmployerAuthorizedContact`

### Employee → `employees`

- `@ManyToOne Employer`
- `fullName`; `rsaPin` nullable; `pencomPin` nullable; `dateOfAppointment` date
- `consolidatedBHT` decimal(18,2)
- `migratedFromECRS` bool; `ecrsEmployeeMigrationDate` nullable
- `@ManyToOne PFA`
- `status` enum: `ACTIVE | REMOVED_PENDING_APPROVAL | REMOVED_APPROVED | PENDING_APPROVAL | REJECTED`
- `changePendingAdminApproval` bool

### EmployerAuthorizedContact → `employer_authorized_contacts`

- `employerCode` varchar(50) — FK to `employers.employerCode`
- `fullName`; `role`; `email` varchar(150) **unique**; `phoneNumber`
- `preferredChannel` varchar(10) — DB check: `'Email'` or `'SMS'`
- `consentGiven`, `isPhoneVerified`, `isEmailVerified` — bool default false

### EmployerRegistration → `employer_registrations`

- Registration funnel for employers without a PENCOM code.
- `companyName`, `emailAddress`, `organizationTypeCode`, `sectorCode`
- `additionalData` jsonb (org-type-specific fields)
- `status` enum `EmployerRegistrationStatus` default `PENDING_VERIFICATION`

### PFA → `pf_as`

- `name`, `code`, `shortName` — all unique nullable
- `restricted` bool default false
- Seeded from Oracle `TBL_PFA` via `PencomService.findPfaByAnyIdentifier`

### Discrepancy → `discrepancies`

- `employerCode`, `companyName`, `rcNumber`, `emailAddress`, `comment` text
- `status` enum `DiscrepancyStatus` default `OPEN`
- `@OneToMany DiscrepancyAdminNote`

### EmployeeChangeRequest → `employee_change_requests`

- `type` enum: `ADDITION | REMOVAL | EDIT` (**⚠ EDIT throws "not yet supported"**)
- `status` enum: `PENDING | APPROVED | REJECTED`
- `metadata` jsonb — full payload for admin review
- `@ManyToOne Employee`; `@ManyToOne Employer`

### Admin auth entities

**AdminOtpSession → `admin_otp_sessions`**

- `userId`, `email`, `otpHash`, `expiresAt`
- `attemptCount`, `isUsed`, `isLocked`, `lockedUntil`
- `purpose` enum: `LOGIN | PASSWORD_RESET`

**AdminSession → `admin_sessions`**

- `userId`, `token` text **unique**, `expiresAt`, `isActive`, `lastActivityAt`
- Index: `(userId, isActive)`

**AdminJourneyPermission → `admin_journey_permissions`**

- `userId`, `journey` (9 code enum), `role` (`FIRST_REVIEWER | SECOND_REVIEWER | APPROVER`)
- **Unique index: `(userId, journey)`**

### TutorialVideo → `tutorial_videos`

- `title`, `description`, `videoFileKey` (MinIO), `thumbnailFileKey`
- `targetHub` enum: `ADMIN_HUB | EMPLOYER_PLATFORM`
- `isPublished` bool; `durationSeconds`; `fileSizeBytes`
- `category` — **⚠ legacy column, always written null**; active field is `tutorialCategory` varchar

### Mongo: `Otp` → `otps` collection

- `code` (stored hashed); `recipient`, `email`, `expiresAt`
- `verified` bool default false
- **⚠ No TTL index — expired records accumulate**

---

## 2. Compliance Service (`pencom_compliance`)

### EmployeeMonthlyContribution → `employee_monthly_contributions` (hot table)

- `employer_code`, `rsa_pin`, `employee_name`, `pfa`
- `employee_contribution`, `employer_contribution` decimal(15,2)
- `employee_avc`, `employer_avc` decimal(15,2)
- `total_contribution`, `date_of_contribution`, `value_date`, `payment_date`
- `contribution_type` default `'COM'`
- `job_id`, `source_row_number` — for crash-safe resume
- `pfcId` — which integration partner uploaded this row

**Key indexes:**

- `(employerCode, valueDate)`
- `(rsaPin)`
- **`UQ_employee_contributions_monthly`** — partial unique `(employer_code, rsa_pin, date_of_contribution, value_date, contribution_type)` WHERE all NOT NULL — deduplication constraint
- Partial unique `(jobId, sourceRowNumber)` WHERE both NOT NULL — per-job idempotency

### EmployerMonthlyAggregate → `employer_monthly_aggregates`

- `employer_code`, `month` date (first of month), `contribution_type`
- `employee_count`, `total_contribution`, `avg_contribution`, `max_contribution`, `min_contribution`
- Unique: `(employer_code, month, contribution_type)`

### EmployerMonthlyPenalty → `employer_monthly_penalties`

- `employer_code`, `month` varchar(7) `'YYYY-MM'`
- `due_date`, `grace_end_date` (= due_date + 11)
- `status`: `GAP_DETECTED → PENALTY_COMPUTED → PARTIALLY_PAID → PAID_IN_FULL`
- `penalty_rate` — locked at first computation
- `days_in_default`, `penalty_amount`, `paid_amount`, `actual_contribution`
- `grace_notified_at`, `last_notified_paid_amount` — notification watermarks
- **Unique: `(employer_code, month)`**

### ComplianceEvaluation → `compliance_evaluation`

- `employer_id`, `employer_code`, `cac_number`
- `overall_status` enum `OverallComplianceStatus`
- `approval_status` enum `ComplianceApprovalStatus`
- `rule_results` jsonb; `summary` jsonb; `config_version`
- `is_latest` bool — only one row per employer has `is_latest=true`
- Relations: `OneToMany Certificate`, `OneToMany PccComment`, `OneToMany PccActivity`

### Certificate → `certificate`

- `evaluation_id` FK `ComplianceEvaluation` CASCADE
- `employer_id`, `employer_code`, `employer_name`, `rc_number`
- `pcc_number` **unique** (format `PCC/YYYY/NNNNNN`)
- `issue_date`, `expiry_date`
- `certificate_path` (MinIO URL)
- `status`: `Active | Expired | Revoked`
- `verification_url` **unique**
- `digital_signature`

### PccActivity → `pcc_activities`

- `activity_type`: `FIRST_REVIEW | SECOND_REVIEW | APPROVAL | REJECTION | QUERY | EMPLOYER_RESPONSE`
- `actor_id`, `actor_name`, `actor_role`; `previous_status`, `new_status`

### GroupLifeInsurance → `group_life_insurance`

- `employer_id`; `commencement_date`; `expiry_date`
- `number_of_lives_covered`; `insurance_company`; `policy_number`; `sum_assured`
- `status` enum `GroupLifeInsuranceStatus`
- `first_reviewed_by/at`, `second_reviewed_by/at`, `approved_by/at`

### WebhookAttempt → `webhook_attempts`

- `entityId`, `event`, `payload` jsonb
- `attemptCount` default 0; `status` (`PENDING → SENT/FAILED`)

### CronJobRun → `cron_job_runs`

- `job_name`, `schedule_expression`
- `started_at`, `finished_at`, `duration_ms`
- `status`, `records_processed`, `gaps_detected`
- Used for both scheduled cron observability and tracking long-running repair runs

### JobStatus → `job_status` (shared with external-gateway)

Stored in the compliance DB. Written by `external-gateway` upload ingress, read by both.

- `idempotency_key` **unique**
- `owner_id` uuid (pfcId); `owner_type`; `s3_key`
- `status` enum: `PENDING | PROCESSING | COMPLETED | FAILED | PARTIAL_SUCCESS`
- `job_id` (Bull queue job ID)
- Checkpoint fields: `lastProcessedRow`, `totalRows`, `processedRows`, `validRows`, `invalidRows`
- `meta` jsonb; `error_details` jsonb

---

## 3. Payments Service (`pencom_payments`)

### Payment → `payment`

- `identifier` varchar — business key (e.g. compliance `evaluationId`)
- `employerCode` varchar
- `remitaRrr` varchar; `orderId` varchar NOT NULL (`ORD_${Date.now()}`)
- `amount` numeric(18,2) — **stores billed total (serviceFee + applied charges), NOT display total ⚠**
- `status` enum: `pending | successful | failed | cancelled` (**⚠ `cancelled` never set by code**)
- `paymentType` enum: `certificate_generation` (only value)
- `metadata` jsonb — `{email, phoneNumber, description, charges, totalAmount, remitaAmount}`
- `paymentChannel`; `failureReason`; `verifiedDate`

**⚠ No indexes on any lookup column. ⚠ No unique constraint on `orderId` or `(identifier, paymentType)`.**

---

## 4. External-Integrations Service (`pencom_external_integrations`)

### CacCompany → `cac_companies`

- `cacNumber` **unique**; `companyName`; `companyType`; `sector`
- `email`, `phone`, `tin`, `directorName`; `employerCode` nullable **unique**
- **No actual outbound CAC API call** — this is local CRUD data, seeded manually or from Oracle.

### IntegrationPartner → `integration_partners`

- `apiKey` varchar(64) **unique** (auto-generated)
- `clientId` varchar(32) **unique** (auto-generated: `client_<16hex>`)
- `clientSecretHash` varchar(128) — bcrypt(plaintext, 10). **Plaintext returned once on create, never stored.**
- `name`; `organizationType`; `isActive` bool
- `whitelistedIps` simple-array — **⚠ check commented out, never enforced**
- `webhookUrl`, `webhookSecret`, `webhookEvents` — for outbound eHub → PFC webhooks

---

## 5. Audit Service (`pencom_audit`)

### AuditLog → `audit_logs`

- `service`, `action`, `outcome` enum (`SUCCESS | FAILURE | PENDING | REVERTED`)
- `entityType` varchar **NOT NULL** — **⚠ `AuditService.createAuditLog` never sets this before insert**
- `entityId`; `correlationId`; `actorType`; `userId`
- `beforeSnapshot`, `afterSnapshot`, `metadata` — all jsonb
- **No retention/archival** — table grows unbounded.

---

## 6. Notifications Service (MongoDB)

### Notifications → `pencom_notifications` collection

- `recepientId` — **⚠ typo persists across schema, DTOs, controllers, events**
- `context` enum `NotificationContext`; `notificationType` enum: `email | sms`
- `sourceService` enum; `sourceEvent` string
- `status` enum: `pending | sent | failed` (default `pending`)
- `dynamicData` Mixed; `referenceId` string indexed
- `retryCount` — **⚠ never incremented**

### OTPs → `otps` collection (in core's MongoDB, not notifications')

- See Core Service section above. Note: env var is `NOTIFICATIONS_MONGO_DB_URL` but consumed by **core**, not notifications — **⚠ misleading name**.

---

## 7. Oracle Entities (read-only, `pencomConnection`)

Two processes open Oracle pools: `core` and `compliance` (via cross-app import of `PencomModule`). All Oracle reads are user-triggered — no cron or worker queries Oracle.

| Entity | Oracle Table | Status |
|---|---|---|
| `PencomEmployer` | `TBL_EMPLOYER_CODE_MST` | Active — employer lookup during auth, registration, discrepancy |
| `PencomContributorBiodata` | `BIODATA1` | Active — bulk employee migration on first login and admin employer view |
| `PencomPfa` | `TBL_PFA` | Active — PFA fallback during registration |
| `PencomCertificateRequest3` | `CERTIFICATE_REQUEST3` | Active — historical PCC migration (compliance only) |
| `PencomCertificate` | `ECRSPCC.CERTIFICATE_REQUESTS` | **⚠ Registered but no repository is injected — dead entity** |

---

## 8. Cross-Service Relationships

No database-level foreign keys exist between services. All cross-service references are loose string keys:

- `Employer.employerCode` — universal join key. Appears in `employee_monthly_contributions`, `employer_monthly_penalties`, `compliance_evaluation`, `certificate`, `cac_companies`, and `payment`.
- `Compliance.payment_id` references `payments.payment.id` — resolved via HTTP.
- `Notifications.recepientId` — typically a `User.id` or authorized-contact ID; not enforced by DB constraint.
- `AuditLog.entityId` — whatever the producing service provides; no FK.

---

## 9. Key Enums

| Enum | Values |
|---|---|
| `OnboardingStatus` | `NOT_STARTED`, `OTP_VERIFIED`, `AUTHORIZED_CONTACT_PENDING`, `COMPLETED` |
| `EmployeeStatus` | `ACTIVE`, `REMOVED_PENDING_APPROVAL`, `REMOVED_APPROVED`, `PENDING_APPROVAL`, `REJECTED` |
| `ChangeRequestType` | `ADDITION`, `REMOVAL`, `EDIT` (EDIT throws) |
| `PaymentStatus` | `PENDING`, `SUCCESSFUL`, `FAILED`, `CANCELLED` (CANCELLED never set) |
| `EJobStatus` | `PENDING`, `PROCESSING`, `COMPLETED`, `FAILED`, `PARTIAL_SUCCESS` |
| `ComplianceApprovalStatus` | `PENDING`, `REVIEWED_FIRST`, `REVIEWED_SECOND`, `APPROVED`, `QUERIED`, `EMPLOYER_RESPONDED`, `REJECTED`, `EXPIRED` |
| `Journey` (RBAC) | `GROUP_LIFE_INSURANCE`, `PCC_APPLICATION`, `EMPLOYEE_MANAGEMENT`, `PAYMENT_MANAGEMENT`, `HELP_AND_SUPPORT`, 4× `EMPLOYER_CODE_REQUEST_*` |
| `AdminRole` | `SUPER_ADMIN` (only declared value) |

---
icon: material/bank-outline
---

# Pencom — eHub

> **eHub** is the customer-facing product name. The codebase uses *PenCom* branding throughout (`pencom-project`, `PencomService`, etc.). This documentation uses both names interchangeably.

## What is eHub?

eHub is a NestJS microservices backend operated by Nigeria's National Pension Commission (PenCom) to manage pension compliance operations for Nigerian employers. It is the digital interface between employers and PenCom's official pension systems.

The system bridges to two pre-existing on-premises PenCom datastores:

- **PENCOM ECRS Oracle database** — legacy source of truth for employer master data, contributor biodata, PFA list, and historical PCC certificates. Read-only.
- **Regulator's historical compliance certificates MySQL database** — referenced in stakeholder context but **not yet integrated** in the codebase.

## Business Purpose

| Domain | What eHub does |
|---|---|
| Employer registration | Onboard new employers with PENCOM, validate via Oracle ECRS |
| Employee management | Track employees by RSA PIN, manage additions/removals, sync from PENCOM ECRS |
| Contributions | Ingest monthly contribution CSVs from Pension Fund Custodians (PFCs), store and aggregate |
| Penalty management | Calculate, track, and notify penalties for late/missing contributions |
| Compliance certificates | Issue Pension Clearance Certificates (PCC) and Group Life Insurance (GLI) certificates |
| Payment processing | Handle PCC fee payments via Remita; track payment state |
| Partner API | Receive PFC contribution data uploads via a secured external REST API |
| Audit trail | Append-only audit log across all services |

## Infrastructure — 4-VPS On-Premises

eHub runs entirely on-premises via Docker containers across four dedicated VPS nodes:

```
┌─────────────────────────────────────────────────────┐
│  VPS 1 — Internet-Facing (Ingress)                  │
│  Nginx (TLS termination + reverse proxy)            │
│  ├── api-gateway          :3000                     │
│  └── external-gateway     :3010                     │
└────────────────────┬────────────────────────────────┘
                     │ Internal network
┌────────────────────▼────────────────────────────────┐
│  VPS 2 — Core Services                              │
│  ├── core                 :4000                     │
│  ├── payments             :5000                     │
│  ├── compliance           :6000                     │
│  ├── notifications        :7000                     │
│  ├── external-integrations :8000                   │
│  └── audit                :9000                     │
└────────────────────┬────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────┐
│  VPS 3 — Databases                                  │
│  ├── PostgreSQL (6 databases)                       │
│  ├── MongoDB (2 databases)                          │
│  └── Redis                                          │
└─────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────┐
│  VPS 4 — Object Storage                             │
│  └── MinIO (S3-compatible)                          │
└─────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────┐
│  Existing PenCom On-Prem (not managed by team)      │
│  └── Oracle ECRS DB  (read-only, pencomConnection)  │
└─────────────────────────────────────────────────────┘
```

Deployment is via GitHub Actions using self-hosted runners (`prod-runner`, `staging-runner`) installed on the VPS nodes. Only changed apps are rebuilt per deploy thanks to `dorny/paths-filter`.

See [Infrastructure](../../infrastructure.md) for the full topology and [Deployment](../../deployment.md) for deploy procedures.

## Service Overview

| Service | Port | Role | Database |
|---|---|---|---|
| api-gateway | 3000 | Public HTTP entry point, JWT auth, routes to backend | None |
| external-gateway | 3010 | Partner-facing HTTP API for PFC CSV uploads | Reuses compliance DB |
| core | 4000 | Employer/employee domain, Oracle bridge, admin auth | PostgreSQL `pencom_core_db2` + MongoDB (OTPs) |
| payments | 5000 | Remita payment state, fee calculation | PostgreSQL `pencom_payments` |
| compliance | 6000 | Contributions, penalties, PCC/GLI workflows | PostgreSQL `pencom_compliance` |
| notifications | 7000 | SendGrid email + Termii SMS fan-out | MongoDB `pencom_notifications` |
| external-integrations | 8000 | CAC company CRUD, PFC partner credential management | PostgreSQL `pencom_external_integrations` |
| audit | 9000 | Append-only audit log | PostgreSQL `pencom_audit` |

## Key Operational Notes

- **Migrations run on boot** — every backend service applies `migrationsRun: true`. Concurrent boots can race the migration runner; prefer sequential restarts.
- **Oracle dependency is soft** — missing `PENCOM_DB_*` env vars do not crash on boot; Oracle calls degrade at runtime. Other services remain operational.
- **Both `core` AND `compliance`** open Oracle connection pools. Both must have `PENCOM_DB_*` set.
- **File storage uses MinIO** — configure `SPACES_*` env vars pointing at the internal MinIO endpoint. No presigned URLs; all file I/O routes through the gateway services.
- **Redis is shared** by compliance (Bull job queues) and core (admin magic-link cache). `REDIS_HOST`/`REDIS_PORT`/`REDIS_PASSWORD` must be set.
- **Internal API key** (`INTERNAL_API_KEY`) is required at boot — `PencomInternalHttpClient` throws if missing.

## Documentation Index

| Doc | What it covers |
|---|---|
| [Architecture](architecture.md) | Tech stack, monorepo layout, service topology, auth model, CI/CD, known ambiguities |
| [Business Logic](business-logic.md) | OTP flows, fee calculation, penalty formula, PCC state machine, payment lifecycle |
| [Domain Model](domain-model.md) | Every TypeORM entity, Mongoose schema, index, and cross-service relationship |
| [Integrations & Env Vars](integrations.md) | Every external system, how to connect, full env var reference |
| [Developer Onboarding](onboarding.md) | Local setup, commands, troubleshooting guide |

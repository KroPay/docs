---
icon: material/shield-lock-outline
---

# KROTrust

KROTrust (krotrust.com) is a peer-to-peer conditional payment and escrow platform for Nigerian buyers and sellers.

## Business Purpose

KROTrust solves the trust problem in online commerce: a buyer places funds in escrow, a seller fulfils the agreed conditions, and payment releases only when conditions are verified. Either party can raise a dispute if conditions aren't met.

Core capabilities:

- **Conditional payments** — buyer defines conditions that must be met before payment releases
- **Escrow (KRO Balance)** — funds held in-platform until order closes
- **Multi-provider payments** — Paystack, Fincra, Providus
- **Dispute resolution** — structured process when conditions are contested
- **Refund processing** — funds returned to buyer when refund conditions are met
- **Real-time notifications** — WebSocket order status updates

## System Components

| Component | Repo | Role |
|---|---|---|
| **kro-backend** | `kro-backend/` | NestJS API server — all business logic, payments, auth |
| **kro-devops** | `kro-devops/` | Docker Compose stacks, Nginx config, CI/CD pipeline |
| **kro-frontend** | `kro-frontend/` | React SPA — user-facing web app |
| **kro-admin** | `kro-admin/` | React SPA — admin dashboard |

## Infrastructure

KROTrust runs on a single **DigitalOcean Droplet** (`188.166.145.68`). All four components run as Docker containers managed by `kro-devops/production/docker-compose.yml`. Nginx handles TLS termination and routes traffic to the appropriate container.

**Database:** DigitalOcean Managed PostgreSQL (SSL required, CA cert mounted in backend container).

See [Architecture](architecture.md) for the technical deep-dive and [Deployment](deployment.md) for operational procedures.

## Quick Navigation

| Doc | What it covers |
|---|---|
| [Architecture](architecture.md) | Tech stack, module structure, order domain, payment providers, request flow |
| [Deployment](deployment.md) | Docker Compose, Nginx, CI/CD, deploy procedures, TLS renewal |

# Infrastructure

## Overview

Three distinct infrastructure environments exist across the two product lines:

| System | Provider | Type | Access |
|---|---|---|---|
| KRO | DigitalOcean | Managed Droplet (VPS) | SSH to root@188.166.145.68 |
| GIV (KROGiving) | DigitalOcean | App Platform (PaaS) | DO Console / API |
| Pencom | On-Premises | Internal servers | VPN / internal network |

---

## KRO — DigitalOcean Droplets

### Servers

| Role | IP | Environment |
|---|---|---|
| Production / Pre-prod | `188.166.145.68` | Docker containers via kro-devops |
| Stage (inactive) | `64.226.94.111` | Commented out in CI pipeline |

### Container Architecture

All services run as Docker containers managed via `docker-compose` in the `kro-devops` repository.

**Production stack (`kro-devops/production/docker-compose.yml`):**

```
nginx (stable-alpine)          — reverse proxy, port 80/443
  └── backend (kro-backend)    — NestJS API, port 3000
  └── frontend (kro-frontend)  — React SPA
  └── admin-frontend (kro-admin) — Admin React SPA
```

**Local dev stack (`kro-devops/local/docker-compose.yml`):**
- `frontend` on port 8088
- `backend-db` (Postgres 11) on port 5432
- `backend` on port 3000
- `admin-frontend` on port 8089

### Nginx

Domain routing via Nginx with Let's Encrypt TLS (`/etc/letsencrypt/live/krotrust.com/`):

| Domain | Target |
|---|---|
| `api.krotrust.com` | `http://backend:3000` (NestJS API) |
| `app.krotrust.com` | `http://frontend` (user SPA) |
| `admin.krotrust.com` | `http://admin-frontend` (admin SPA) |
| `app2.krotrust.com` | `http://frontend` (alias) |

All HTTP traffic on port 80 is redirected to HTTPS (301).

WebSocket upgrades are enabled for `api.krotrust.com` (real-time notifications).

### TLS Renewal

Certificates are managed by Certbot / Let's Encrypt. Stored at `/etc/letsencrypt/` on the host and mounted into the Nginx container.

### Databases (KRO)

| Database | Type | Provider | Notes |
|---|---|---|---|
| KRO main DB | PostgreSQL | DigitalOcean Managed DB | SSL required (CA cert mounted in container) |
| WhatsApp verifications | MongoDB schema (in PostgreSQL via TypeORM) | — | — |

The CA certificate file is mounted at `/usr/src/app/kro-prod-db-cluster-ca-certificate.crt` in the backend container.

### CI/CD (KRO)

**Repository:** `kro-devops/.github/workflows/main.yml`

Trigger: push to `main` branch

Steps:
1. SSH into production droplet (`188.166.145.68`) using `NEW_PREPROD_DROPLET_SSH_PRIVATE_KEY` secret
2. `cd /root/Kro/kro-devops && git pull origin main`
3. `docker system prune -f` (clear disk space)

> **Note:** The stage deployment (to `64.226.94.111`) is commented out in the workflow. Currently only production is deployed automatically.

**Application rebuild:** After pulling updated devops config, you must manually rebuild and restart containers:
```bash
cd /root/Kro/kro-devops/production
docker-compose up --build -d
```

---

## GIV (KROGiving) — DigitalOcean App Platform

### Overview

KROGiving runs fully managed on DigitalOcean App Platform. There are no servers to SSH into; scaling, TLS, and routing are managed by DigitalOcean.

**Components deployed to App Platform:**
- `krogiving-backend` — NestJS backend service
- `krogiving-frontend` — React frontend static site
- `giv-admin-new` — Admin dashboard static site

### CI/CD (GIV)

**Repository:** `krogiving-backend/.github/workflows/deploy.yml`

Trigger: push any git tag (`*`)

Steps:
1. Calls DigitalOcean App Platform API via `curl`
2. Triggers re-deployment of the `krogiving-backend` component
3. Uses secrets: `STAGING_APP_PLATFORM_ID`, `DIGITALOCEAN_ACCESS_TOKEN`

> **Note:** Only the backend is deployed via CI. The frontend apps may be deployed separately or via DO App Platform's built-in GitHub integration.

### Databases (GIV)

| Database | Type | Provider |
|---|---|---|
| MongoDB | Document DB | DigitalOcean Managed MongoDB |
| PostgreSQL | Relational DB | DigitalOcean Managed PostgreSQL |
| Redis | Cache / queues | DigitalOcean Managed Redis |

### File Storage (GIV)

DigitalOcean Spaces (S3-compatible) for campaign media:
- `SPACES_ACCESS_KEY_ID` / `SPACES_SECRET_ACCESS_KEY`
- `SPACES_REGION`, `SPACES_BUCKET_NAME`, `SPACES_ENDPOINT`

Cloudinary is used for video uploads (via `useCloudinaryVideoUpload` hook in frontend).

---

## Pencom — On-Premises

### Overview

The Pencom/eHub system runs entirely on-premises across four dedicated VPS nodes. All Docker containers are deployed via GitHub Actions to self-hosted runners on each VPS. There is no cloud provider dependency for eHub compute or databases.

### VPS Topology

```
┌─────────────────────────────────────────────────────┐
│  VPS 1 — Internet-Facing (Ingress)                  │
│  Exposed to the public internet                     │
│                                                     │
│  ├── Nginx (TLS termination, reverse proxy)         │
│  ├── api-gateway          :3000                     │
│  └── external-gateway     :3010                     │
└────────────────────┬────────────────────────────────┘
                     │ Internal network only
┌────────────────────▼────────────────────────────────┐
│  VPS 2 — Core Services                              │
│  Not internet-accessible; internal network only     │
│                                                     │
│  ├── core                 :4000                     │
│  ├── payments             :5000                     │
│  ├── compliance           :6000                     │
│  ├── notifications        :7000                     │
│  ├── external-integrations :8000                   │
│  └── audit                :9000                     │
└────────────────────┬────────────────────────────────┘
                     │ Internal network only
┌────────────────────▼────────────────────────────────┐
│  VPS 3 — Databases                                  │
│  Not internet-accessible; internal network only     │
│                                                     │
│  ├── PostgreSQL                                     │
│  │   ├── pencom_core_db2          (core)            │
│  │   ├── pencom_payments          (payments)        │
│  │   ├── pencom_compliance        (compliance +     │
│  │   │                             external-gateway)│
│  │   ├── pencom_external_integrations               │
│  │   └── pencom_audit             (audit)           │
│  ├── MongoDB                                        │
│  │   ├── pencom_notifications     (notifications)   │
│  │   └── pencom_core_otp          (core OTPs)       │
│  └── Redis                                          │
│      └── Bull queues + magic-link cache             │
└─────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────┐
│  VPS 4 — Object Storage                             │
│                                                     │
│  └── MinIO (S3-compatible object store)             │
│      ├── PCC certificates (PDFs)                    │
│      ├── Employer registration documents            │
│      ├── PFC contribution CSVs                      │
│      └── Tutorial videos + thumbnails               │
└─────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────┐
│  Existing PenCom On-Prem (managed by PenCom)        │
│                                                     │
│  └── Oracle ECRS DB                                 │
│      Read-only access by core + compliance          │
└─────────────────────────────────────────────────────┘
```

### Service Ports

| Service | VPS | HTTP Port | Role |
|---|---|---|---|
| api-gateway | VPS 1 | 3000 | Public internet — employer & admin portal |
| external-gateway | VPS 1 | 3010 | Public internet — PFC partner API |
| core | VPS 2 | 4000 | Internal — employer/employee domain, Oracle bridge |
| payments | VPS 2 | 5000 | Internal — Remita payment state |
| compliance | VPS 2 | 6000 | Internal — contributions, penalties, PCC/GLI |
| notifications | VPS 2 | 7000 | Internal — email/SMS fan-out |
| external-integrations | VPS 2 | 8000 | Internal — CAC CRUD, PFC partner credentials |
| audit | VPS 2 | 9000 | Internal — append-only audit log |

Inter-service calls are HTTP with `x-internal-api-key` header. TCP microservice transport is declared on some services but only used internally within `notifications` for fire-and-forget email dispatch.

### CI/CD (Pencom)

Deployment via GitHub Actions with self-hosted runners installed on the VPS nodes.

| Workflow | Trigger | Runner | What it does |
|---|---|---|---|
| `deploy-prod.yaml` | Push to `main`, filter `apps/**` | `prod-runner` | Builds only changed apps via `dorny/paths-filter`, pushes Docker images, deploys |
| `deploy-staging.yaml` | Push to `staging`, filter `apps/**` | `staging-runner` | Same shape, staging environment |
| `deploy-monitoring.yaml` | Push to `main` or `monitoring-deploy`, filter `monitoring/**` | `prod-runner` | Runs Ansible playbook to provision Prometheus + Grafana + node-exporter |

Each app has its own `Dockerfile` at `apps/<app>/Dockerfile`.

### Databases (Pencom)

| Database | Type | Host | Service |
|---|---|---|---|
| `pencom_core_db2` | PostgreSQL | VPS 3 | core |
| `pencom_payments` | PostgreSQL | VPS 3 | payments |
| `pencom_compliance` | PostgreSQL | VPS 3 | compliance + external-gateway |
| `pencom_external_integrations` | PostgreSQL | VPS 3 | external-integrations |
| `pencom_audit` | PostgreSQL | VPS 3 | audit |
| `pencom_notifications` | MongoDB | VPS 3 | notifications |
| `pencom_core_otp` | MongoDB | VPS 3 | core (OTP storage) |
| Redis | Redis | VPS 3 | compliance (Bull queues) + core (magic-link cache) |
| Oracle ECRS | Oracle | PenCom on-prem | core + compliance (read-only) |

### File Storage (Pencom)

MinIO on VPS 4, accessed via the `@aws-sdk/client-s3` SDK (MinIO is S3-compatible). Configure with `SPACES_*` env vars pointing at the MinIO endpoint.

| Bucket path | Contents |
|---|---|
| `CERTIFICATES/` | PCC PDF certificates |
| `EMPLOYER_CODE_REQUEST_DOCUMENT/` | Employer registration documents |
| `TUTORIAL_VIDEOS/` | Admin/employer tutorial videos |
| `TUTORIAL_THUMBNAILS/` | Video thumbnails |
| PFC upload namespace | Monthly contribution CSVs from PFC partners |

### External Integrations (Pencom)

| Service | Purpose | Notes |
|---|---|---|
| Remita | Pension certificate fee payment processing | Nigerian payment gateway |
| PENCOM Oracle ECRS | Official employer/employee/PFA data source | Read-only; PenCom-managed |
| SendGrid | Email notifications (OTPs, certificates, alerts) | Via `notifications` service |
| Termii | SMS notifications | **⚠ Currently unused — no producer** |
| Highlight.io | Error tracking + APM | Every service's `main.ts` |
| PostHog | Product analytics | api-gateway only, dev disabled |

### Monitoring

Prometheus + Grafana stack provisioned via Ansible (`monitoring/ansible/`), deployed via `deploy-monitoring.yaml`.

- **Node-exporter** collects host-level metrics from the VPS nodes.
- **⚠ No application-level Grafana dashboards exist** — only host metrics via node-exporter.

---

## Network Security Notes

- VPS 1 is the only internet-facing node. VPS 2, 3, and 4 communicate only over the internal network.
- Nginx on VPS 1 handles TLS termination. All external traffic is HTTPS.
- All inter-service HTTP calls carry an `INTERNAL_API_KEY` header. Backend services on VPS 2 (`core`, `notifications`, `audit`) verify it via `ApiKeyAuthGuard`. Other services rely on network isolation.
- JWT is used for end-user authentication (employer/admin). Separate JWT secret for external PFC partner tokens.
- Database connections do not use SSL (internal network only). Oracle connections are read-only.
- KRO database connections use SSL with CA certificate verification (DigitalOcean managed database).

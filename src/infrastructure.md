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

The Pencom system runs entirely on internal/on-premises infrastructure. No cloud provider details are documented in the repositories.

### Service Ports

Each microservice listens on its own HTTP and TCP port:

| Service | HTTP Port | TCP Port |
|---|---|---|
| api-gateway | 3000 | 3001 |
| external-gateway | 3010 | — |
| core | 4000 | 4001 |
| payments | 5000 | 5001 |
| compliance | 6000 | 6001 |
| notifications | 7000 | 7001 |
| external-integrations | 8000 | — |
| audit | 9000 | — |

Services communicate with each other over TCP using NestJS microservices (not HTTP-to-HTTP calls).

### Databases (Pencom)

| Database | Type | Service | Notes |
|---|---|---|---|
| `pencom_core_db2` | PostgreSQL | core | Main business data |
| `pencom_payments` | PostgreSQL | payments | Payment records |
| `pencom_compliance` | PostgreSQL | compliance | GLI, PCC, PFC compliance data |
| `pencom_external_integrations` | PostgreSQL | external-integrations | Integration data |
| `pencom_notifications` | MongoDB | notifications | Notification records |
| `pencom_audit_db` | PostgreSQL | audit | Audit trail |
| PENCOM Oracle DB | Oracle | external-integrations | On-prem PENCOM read integration |

### External Integrations (Pencom)

| Service | Purpose |
|---|---|
| Remita | Pension payment processing |
| PENCOM Oracle DB | Official pension data source (on-prem) |
| SendGrid | Email notifications |
| Termii | SMS notifications |
| AWS S3 | Document storage |
| PostHog | Product analytics |
| Highlight.io | Error tracking |
| Redis (BullMQ) | Background job queues |

---

## Network Security Notes

- All production services use TLS (HTTPS / SSL)
- Database connections use SSL with CA certificate verification (KRO)
- Internal service communication in Pencom is TCP-based (not exposed to internet)
- JWT is used for API authentication across all platforms
- An `INTERNAL_API_KEY` header is used for service-to-service calls in Pencom

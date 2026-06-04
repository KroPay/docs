# Deployment

## KRO Platform

### Production Deployment

**Infrastructure:** DigitalOcean Droplet at `188.166.145.68`  
**Trigger:** Automatic on push to `main` branch of `kro-devops`

#### Automated CI Steps (GitHub Actions)

1. SSH into droplet using `NEW_PREPROD_DROPLET_SSH_PRIVATE_KEY` secret
2. Pull latest devops config: `git pull origin main`
3. Prune Docker: `docker system prune -f`

> After the CI pipeline runs, a **manual step** is required to rebuild application containers. The CI only updates the devops configuration, not the application code.

#### Deploying Application Code Changes

When `kro-backend`, `kro-frontend`, or `kro-admin` changes:

```bash
# SSH into droplet
ssh root@188.166.145.68

# Navigate to the production compose directory
cd /root/Kro/kro-devops/production

# Rebuild and restart containers
docker-compose up --build -d

# Verify containers are running
docker-compose ps
docker-compose logs -f backend
```

#### First-Time Setup / Full Deploy

```bash
ssh root@188.166.145.68

# Clone all repos under /root/Kro/
cd /root/Kro
git clone <kro-backend-repo>
git clone <kro-frontend-repo>
git clone <kro-admin-repo>
git clone <kro-devops-repo>

# Place production config
cp /path/to/production.json /root/Kro/kro-devops/production/production.json
cp /path/to/kro-prod-db-cluster-ca-certificate.crt /root/Kro/kro-devops/production/

# Start stack
cd /root/Kro/kro-devops/production
docker-compose up --build -d
```

#### Required Files (not in git)

| File | Location on Droplet | Purpose |
|---|---|---|
| `production.json` | `/root/Kro/kro-devops/production/production.json` | Runtime config (DB credentials, secrets) |
| `kro-prod-db-cluster-ca-certificate.crt` | `/root/Kro/kro-devops/production/` | PostgreSQL SSL CA cert |

#### TLS Certificate Management

Certificates are managed by Certbot/Let's Encrypt stored in `/etc/letsencrypt/`.

To renew:
```bash
certbot renew
# Then restart nginx:
docker-compose -f /root/Kro/kro-devops/production/docker-compose.yml restart nginx
```

### Stage Environment

The stage deployment (droplet `64.226.94.111`) is currently **commented out** in the CI pipeline. To deploy to stage, manually:

```bash
ssh root@64.226.94.111
cd /root/Kro/kro-devops
git pull origin main
cd stage
docker-compose up --build -d
```

### Local Development

```bash
cd kro-devops/local
docker-compose up --build -d
```

Services will be available at:
- Frontend: http://localhost:8088
- Admin: http://localhost:8089
- API: http://localhost:3000

---

## GIV (KROGiving) Platform

### Production Deployment

**Infrastructure:** DigitalOcean App Platform (fully managed)  
**Trigger:** Push any git tag to `krogiving-backend`

#### How Deployment Works

1. Developer creates and pushes a git tag (e.g., `v1.2.3`) to `krogiving-backend`
2. GitHub Actions workflow fires and calls the DigitalOcean App Platform API
3. DO App Platform pulls the latest code and rebuilds the container
4. Zero-downtime deployment is handled by App Platform

```bash
# Create and push a release tag
git tag v1.2.3
git push origin v1.2.3
```

Required GitHub secrets:
- `STAGING_APP_PLATFORM_ID` — DO App Platform app ID
- `DIGITALOCEAN_ACCESS_TOKEN` — DO API token

#### Frontend Deployment

`krogiving-frontend` and `giv-admin-new` are static React apps deployed to DigitalOcean App Platform. Deployment is likely configured in the DO App Platform console with GitHub integration (auto-deploys on push to `main`).

To trigger a manual deploy via DO API:
```bash
curl -X POST "https://api.digitalocean.com/v2/apps/<APP_ID>/deployments" \
     -H "Authorization: Bearer $DIGITALOCEAN_ACCESS_TOKEN" \
     -H "Content-Type: application/json" \
     -d '{"components": [{"name": "krogiving-frontend"}]}'
```

### Local Development

```bash
# Backend
cd krogiving-backend
cp .env.sample .env
# Fill in .env values
docker-compose up  # starts MongoDB locally
yarn start:dev

# Frontend
cd krogiving-frontend
cp .env.development.local.example .env.development.local
# Fill in API URL and Paystack key
yarn start

# Admin
cd giv-admin-new
cp .env.example .env
yarn dev
```

---

## Pencom Platform

### Overview

Pencom runs on on-premises infrastructure. Each microservice is deployed independently. There is no CI/CD pipeline defined in the repositories.

### Building

```bash
cd pencom-project

# Build all services
yarn build

# Or build individual services
yarn build:api-gateway
yarn build:core
yarn build:compliance
yarn build:payments
yarn build:audit
yarn build:notifications
yarn build:external-integration
```

### Starting Services

Each service runs as a separate Node.js process:

```bash
# Production
yarn start:api-gateway:prod       # :3000
yarn start:core:prod              # :4000
yarn start:payments:prod          # :5000 (via node dist/apps/payments/src/main)
yarn start:compliance:prod        # :6000
yarn start:notifications:prod     # :7000
yarn start:external-integration:prod  # :8000
yarn start:audit:prod             # :9000
yarn start:external-gateway:prod  # :3010

# Development (with watch)
yarn start:api-gateway:dev
yarn start:core:dev
# etc.
```

### Database Migrations

```bash
# Generate a new migration
yarn migration:generate

# Run pending migrations
yarn migration:run

# Payments service migrations specifically
yarn migrate:payments
```

### Environment Configuration

Copy `.env.example` to `.env` and fill in all values:

```bash
cp .env.example .env
```

Key sections in `.env`:
- API Gateway, Core, Payments, Compliance, Notifications, Audit service hosts/ports
- Each service's own DB credentials
- PENCOM Oracle DB credentials
- Remita (payment gateway) credentials
- SendGrid, Termii credentials
- Redis URL
- JWT secrets

### Recommended Startup Order

Services communicate via TCP. Start in this order:

1. Infrastructure: PostgreSQL, MongoDB, Redis, Oracle DB connection
2. `core` service (most other services depend on it)
3. `payments`, `compliance`, `notifications`, `audit`, `external-integrations`
4. `api-gateway` (public entry point)
5. `external-gateway` (external API entry point)

---

## Environment Variables Reference

### KROGiving Backend (`.env.sample`)

| Variable | Description |
|---|---|
| `MONGO_URI` | MongoDB connection string |
| `SPACES_*` | DigitalOcean Spaces (S3) credentials |
| `PAYSTACK_*` | Paystack API keys and URLs |
| `SENDGRID_API_KEY` | SendGrid email API key |
| `SENDER_EMAIL` / `SENDER_NAME` | Email sender identity |
| `FRONT_END_URL` | Frontend base URL for email links |
| `ENVIRONMENT` | `development` or `production` |
| `TERMII_*` | Termii SMS API credentials |
| `GIV_POSTGRES_*` | PostgreSQL connection (host, port, user, pass, db) |
| `GIV_POSTGRES_CA` | SSL CA certificate content |
| `JWT_SECRET_KEY` | JWT signing secret |
| `EXPIRES_IN` | JWT expiry |
| `REDIS_HOST` / `REDIS_PORT` / `TTL` | Redis connection |
| `TELNYX_*` | Telnyx SMS/WhatsApp credentials |

### KRO Backend

Configuration is loaded from a JSON file (`production.json`) mapped into the container. Environment-specific configuration is handled via `NODE_ENV` and `NODE_CONFIG_DIR`.

### Pencom (`.env.example`)

See [infrastructure.md](./infrastructure.md) for the full list. Key variables:
- Service host/port for each microservice
- Database credentials for each service's DB
- `JWT_SECRET`, `JWT_EXPIRES_IN`
- `INTERNAL_API_KEY` (service-to-service auth)
- `PENCOM_DB_*` (Oracle DB credentials)
- `REMITA_*` (payment gateway)
- `REDIS_URL`
- `SESSION_TIMEOUT_MINUTES`

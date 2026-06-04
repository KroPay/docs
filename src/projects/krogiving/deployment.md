---
icon: material/rocket-launch-outline
---

# KROGiving — Deployment

## Infrastructure

| Resource | Details |
|---|---|
| Platform | DigitalOcean App Platform |
| Trigger | Git tag push to each repo |
| Database | DigitalOcean Managed PostgreSQL (SSL required) |
| Cache | DigitalOcean Managed MongoDB |
| Redis | DigitalOcean Managed Redis |
| File storage | DigitalOcean Spaces (S3-compatible) |
| Video storage | Cloudinary (direct browser upload) |

## App Platform Components

Each component is deployed independently on DigitalOcean App Platform:

| Component | Type | Build command | Output |
|---|---|---|---|
| `krogiving-backend` | Web service | `yarn build` | Runs `dist/main.js` |
| `krogiving-frontend` | Static site | `react-scripts build` | `build/` |
| `giv-admin-new` | Static site | `tsc -b && vite build` | `dist/` |

## CI/CD Pipeline

**Trigger:** Push a git tag to the repo (e.g., `v1.2.3`)

**Workflow file:** `.github/workflows/deploy.yml`

What happens:

1. GitHub Actions detects the tag push event.
2. Calls the DigitalOcean App Platform API to trigger a redeployment of the component.
3. DO App Platform pulls the latest tagged code, rebuilds, and deploys with zero-downtime replacement.

There is no automatic image rebuild for the backend — DO App Platform handles the build environment entirely.

## Deploying a New Version

```bash
# Tag and push to trigger a deploy
git tag v1.2.3
git push origin v1.2.3
```

Monitor the deploy in the DigitalOcean App Platform console under the app → Activity tab.

## Environment Variables

### krogiving-backend

Set in DigitalOcean App Platform → Component → Environment Variables.

```bash
# Databases
MONGO_URI=                     # MongoDB connection string
GIV_POSTGRES_HOST=             # PostgreSQL host
GIV_POSTGRES_PORT=             # PostgreSQL port (default 5432)
GIV_POSTGRES_USER=             # PostgreSQL user
GIV_POSTGRES_PASSWORD=         # PostgreSQL password
GIV_POSTGRES_DATABASE=         # Database name
GIV_POSTGRES_CA=               # SSL CA certificate content (multiline)

# Auth
JWT_SECRET_KEY=                # JWT signing secret
EXPIRES_IN=                    # JWT expiry (e.g., "7d")

# Cache / Redis
REDIS_HOST=                    # Redis host
REDIS_PORT=                    # Redis port
TTL=                           # Default cache TTL (seconds)

# Payments
PAYSTACK_SECRET_KEY=           # Paystack secret key
PAYSTACK_PUBLIC_KEY=           # Paystack public key

# File storage (DO Spaces)
SPACES_ACCESS_KEY_ID=          # DO Spaces access key
SPACES_SECRET_ACCESS_KEY=      # DO Spaces secret key
SPACES_REGION=                 # DO Spaces region (e.g., "fra1")
SPACES_BUCKET_NAME=            # Bucket name
SPACES_ENDPOINT=               # DO Spaces endpoint URL
SPACES_BUCKET_URL=             # Public bucket URL

# Email
SENDGRID_API_KEY=              # SendGrid API key
SENDER_EMAIL=                  # From email address
SENDER_NAME=                   # From display name

# SMS
TERMII_API_KEY=                # Termii API key
TELNYX_AUTH_TOKEN=             # Telnyx API token

# App
ENVIRONMENT=production         # Environment name
FRONT_END_URL=                 # Frontend URL for email links (https://krogiving.com)
```

### krogiving-frontend

```bash
REACT_APP_API_BASE_URL=                  # Backend API URL
REACT_APP_PAYSTACK_PUBLIC_KEY=           # Paystack public key (frontend)
REACT_APP_SPLIT_API_KEY=                 # Split.io API key
REACT_APP_HIGHLIGHT_PROJECT_ID=          # Highlight.io project ID
REACT_APP_CLOUDINARY_CLOUD_NAME=         # Cloudinary cloud name
REACT_APP_CLOUDINARY_UPLOAD_PRESET=      # Cloudinary unsigned upload preset
```

### giv-admin-new

```bash
VITE_API_BASE_URL=             # Backend API URL (e.g., https://api.krogiving.com)
```

## Local Development

### krogiving-backend

```bash
# Install dependencies
yarn install

# Copy and populate env file
cp .env.sample .env

# Start local dependencies (MongoDB)
docker-compose up -d

# Start in dev mode (watch + reload)
yarn start:dev
```

TypeORM migrations run automatically on startup via `migrationsRun: true`. Ensure `dist/migrations/` is built if starting without watch mode.

### krogiving-frontend

```bash
yarn install
cp .env.development.local.example .env.development.local
# Set REACT_APP_API_BASE_URL to local backend or staging API

yarn start           # Local dev with .env.development.local
yarn start:production # Local dev pointing at production API
yarn build           # Production build
yarn test            # Run Jest test suite
```

### giv-admin-new

```bash
yarn install
cp .env.example .env
# Set VITE_API_BASE_URL

yarn dev             # Local dev server
yarn build           # Production build (tsc + vite)
```

## Operational Procedures

### Add a new environment variable

1. Go to DigitalOcean App Platform → App → Component → Environment Variables.
2. Add the variable and save.
3. DO App Platform will automatically restart the component.
4. Update `.env.sample` / `.env.example` in the repo (never put real values in git).

### Check service health

```bash
# Check App Platform component status
# → DO Console → App → Overview tab (shows component health + recent deploys)

# Or hit the health endpoint directly
curl -f https://api.krogiving.com/health || echo "Health check failed"
```

### View logs

Access logs from the DigitalOcean App Platform console:

- App → Component → Logs tab (live streaming)
- Or use the `doctl` CLI: `doctl apps logs <app-id> --component krogiving-backend --follow`

### Database migrations

TypeORM migrations run automatically on startup. To run manually:

```bash
# Using doctl to run a one-off command on App Platform
doctl apps create-deployment <app-id>

# Or via the backend's npm scripts if you have local DB access
yarn typeorm migration:run -d dist/database/data-source.js
```

### Force a redeployment (no code change)

```bash
# Via DO App Platform API or console
doctl apps create-deployment <app-id>
```

## Secrets Reference

| Secret | Location | Description |
|---|---|---|
| `PAYSTACK_SECRET_KEY` | DO App Platform env vars | Paystack secret — never in git |
| `JWT_SECRET_KEY` | DO App Platform env vars | JWT signing secret |
| `MONGO_URI` | DO App Platform env vars | MongoDB connection string with credentials |
| `GIV_POSTGRES_PASSWORD` | DO App Platform env vars | PostgreSQL password |
| `GIV_POSTGRES_CA` | DO App Platform env vars | PostgreSQL SSL CA cert (multiline) |
| `SPACES_SECRET_ACCESS_KEY` | DO App Platform env vars | DO Spaces secret key |
| `SENDGRID_API_KEY` | DO App Platform env vars | SendGrid API key |

## Shutdown and Recovery

### Graceful shutdown

Scale the component down to 0 instances via the DO App Platform console:

- App → Component → Settings → Instance Count → set to 0

This stops all traffic to that component. The database and other components remain running.

### Recovery

Scale the component back to the desired instance count. DO App Platform will pull the last deployed image and restart.

### Database recovery

DigitalOcean Managed PostgreSQL, MongoDB, and Redis all have automatic daily backups. Restore via the DigitalOcean console:

- Databases → Select database → Backups → Restore from backup

### Frontend recovery

Static sites on App Platform can be re-deployed instantly by pushing a new tag or triggering a manual deployment from the console.

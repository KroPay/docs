---
icon: material/rocket-launch-outline
---

# KROTrust — Deployment

## Infrastructure

| Resource | Details |
|---|---|
| Server | DigitalOcean Droplet — `188.166.145.68` |
| Orchestration | Docker Compose via `kro-devops/production/docker-compose.yml` |
| TLS | Let's Encrypt, Certbot — `/etc/letsencrypt/live/krotrust.com/` |
| Database | DigitalOcean Managed PostgreSQL (SSL, CA cert required) |
| Stage server | `64.226.94.111` — currently inactive (commented out in CI) |

## Docker Services (Production)

All services are on the `production_network` Docker network with `restart: unless-stopped`.

| Service | Image | Exposed port | Description |
|---|---|---|---|
| `nginx` | `nginx:stable-alpine` | 80, 443 | Reverse proxy + TLS termination |
| `backend` | Built from `kro-backend` repo | 3000 (internal) | NestJS API |
| `frontend` | Built from `kro-frontend` repo | — (proxied by Nginx) | React user SPA |
| `admin-frontend` | Built from `kro-admin` repo | — (proxied by Nginx) | React admin SPA |

**Backend mounted volumes:**

```yaml
- ./production.json:/usr/src/app/local/production.json
- ./kro-prod-db-cluster-ca-certificate.crt:/usr/src/app/kro-prod-db-cluster-ca-certificate.crt
- ./scripts/wait-for-it.sh:/wait-for-it.sh
```

`production.json` and the CA cert are **never in git** — they must exist on the server.

## Nginx Configuration

Domain routing (`production/nginx/default.conf`):

| Domain | Backend |
|---|---|
| `api.krotrust.com` | `http://backend:3000` |
| `app.krotrust.com` | `http://frontend` |
| `admin.krotrust.com` | `http://admin-frontend` |
| `app2.krotrust.com` | `http://frontend` (alias) |

All HTTP traffic (port 80) is redirected to HTTPS (301). WebSocket upgrades enabled on `api.krotrust.com`. Max body size: 20MB. Proxy read timeout: 600s.

## CI/CD Pipeline

**File:** `kro-devops/.github/workflows/main.yml`  
**Trigger:** Push to `main` branch of `kro-devops`

What the CI pipeline does:

1. Sets up SSH using the `NEW_PREPROD_DROPLET_SSH_PRIVATE_KEY` secret.
2. SSH into `188.166.145.68`.
3. `cd /root/Kro/kro-devops && git pull origin main`.
4. `docker system prune -f`.

**What it does NOT do automatically:** rebuild application Docker images or restart containers. After CI, a manual rebuild is required.

## Deploying a New Version

```bash
# 1. SSH into server
ssh root@188.166.145.68

# 2. Pull latest application code
cd /root/Kro/kro-backend && git pull origin main

# 3. Rebuild and restart
cd /root/Kro/kro-devops/production
docker-compose up --build -d backend

# 4. Check logs
docker-compose logs -f backend
```

The backend container start command is:

```bash
cross-env NODE_ENV=production cross-env NODE_CONFIG_DIR=/usr/src/app/local npm run start
```

## Operational Procedures

### Add a new environment variable

1. Update `production.json` on the server (SSH in, edit directly).
2. Restart: `docker-compose restart backend`.
3. Update `.env.example` in the repo (never put real values in the repo).

### Renew TLS certificates

```bash
ssh root@188.166.145.68
certbot renew
cd /root/Kro/kro-devops/production
docker-compose restart nginx
```

### Check service health

```bash
ssh root@188.166.145.68
cd /root/Kro/kro-devops/production
docker-compose ps
docker-compose logs --tail=100 backend
curl -f https://api.krotrust.com/health || echo "Health check failed"
```

### Free up disk space

```bash
bash /root/Kro/kro-devops/scripts/server-cleanup.sh
# Or manually:
docker system prune -f
```

## Database Migrations

TypeORM migrations must be run manually:

```bash
# SSH into server, then inside the backend container:
docker exec -it <backend-container> npm run typeorm -- migration:run -d <data-source>
```

## Local Development

Uses `kro-devops/local/docker-compose.yml`:

| Service | Port |
|---|---|
| `frontend` | 8088 |
| `backend-db` (Postgres 11) | 5432 |
| `backend` | 3000 |
| `admin-frontend` | 8089 |

The backend uses `wait-for-it.sh` to wait for `backend-db:5432` before starting.

## Secrets Reference

| Secret | Location | Description |
|---|---|---|
| `NEW_PREPROD_DROPLET_SSH_PRIVATE_KEY` | GitHub Secrets | SSH key for CI to access the droplet |
| `production.json` | On droplet: `/root/Kro/kro-devops/production/production.json` | Full app config including DB credentials, API keys |
| `kro-prod-db-cluster-ca-certificate.crt` | On droplet | PostgreSQL SSL CA cert for DigitalOcean Managed DB |
| TLS certs | On droplet: `/etc/letsencrypt/` | Let's Encrypt certificates (auto-mounted into Nginx) |

## Shutdown and Recovery

### Graceful shutdown

```bash
ssh root@188.166.145.68
cd /root/Kro/kro-devops/production
docker-compose stop        # graceful stop (sends SIGTERM)
# or
docker-compose down        # stop + remove containers
```

### Recovery after server restart

Docker services have `restart: unless-stopped` — they will auto-restart after a server reboot. If they don't:

```bash
cd /root/Kro/kro-devops/production
docker-compose up -d
```

### Database recovery

DigitalOcean Managed PostgreSQL has automatic daily backups. Restore via the DigitalOcean console under Databases → Backups.

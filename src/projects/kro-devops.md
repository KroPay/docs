# kro-devops

## Business Purpose

This repository contains all infrastructure-as-code and deployment configuration for the KRO platform (krotrust.com). It manages:

- Docker Compose stacks for local development, staging, and production
- Nginx reverse proxy configuration
- Deployment automation scripts
- CI/CD pipeline (GitHub Actions)

**This is the single source of truth for how KRO services are wired together and deployed.**

---

## Repository Structure

```
kro-devops/
├── local/
│   ├── docker-compose.yml      — Local development stack
│   └── .gitignore
├── stage/
│   ├── docker-compose.yml      — Stage environment stack
│   └── nginx/default.conf      — Stage Nginx config
├── production/
│   ├── docker-compose.yml      — Production stack
│   ├── nginx/default.conf      — Production Nginx config
│   └── .env.example            — Template for production secrets
├── production-only-nginx/
│   ├── docker-compose.yml      — Nginx-only compose (if app containers are external)
│   └── nginx/default.conf
├── scripts/
│   ├── wait-for-it.sh          — Bash script: wait for TCP service to be ready
│   ├── server-cleanup.sh       — Disk cleanup script
│   └── kro-prod-deploy.sh      — Manual deploy helper script
└── .github/
    └── workflows/main.yml      — CI/CD: deploy on push to main
```

---

## Docker Services

### Production (`production/docker-compose.yml`)

| Service | Image / Build | Port | Description |
|---|---|---|---|
| `nginx` | `nginx:stable-alpine` | 80, 443 | Reverse proxy + TLS termination |
| `backend` | Built from `kro-backend` | 3000 | NestJS API |
| `frontend` | Built from `kro-frontend` | — | React SPA (proxied by Nginx) |
| `admin-frontend` | Built from `kro-admin` | — | Admin React SPA (proxied by Nginx) |

All services share a Docker network: `production_network`.  
All services use `restart: unless-stopped` (auto-restart on reboot).

**Mounted volumes for backend:**
- `./scripts/wait-for-it.sh` → `/wait-for-it.sh` — TCP readiness check utility
- `./production.json` → `/usr/src/app/local/production.json` — App configuration
- `./kro-prod-db-cluster-ca-certificate.crt` → `/usr/src/app/kro-prod-db-cluster-ca-certificate.crt` — DB SSL cert

### Local Development (`local/docker-compose.yml`)

| Service | Port | Description |
|---|---|---|
| `frontend` | 8088 | React SPA |
| `backend-db` | 5432 | PostgreSQL 11 |
| `backend` | 3000 | NestJS API |
| `admin-frontend` | 8089 | Admin React SPA |

The backend waits for `backend-db:5432` to be ready (via `wait-for-it.sh`) before starting.

### Stage (`stage/docker-compose.yml`)

Similar to production but with stage-specific configuration. The stage droplet IP is `64.226.94.111` (currently inactive in CI).

---

## Nginx Configuration

### Production Domains (`production/nginx/default.conf`)

| Domain | Protocol | Backend |
|---|---|---|
| `api.krotrust.com` | HTTP (301→HTTPS) + HTTPS | `http://backend:3000` |
| `app.krotrust.com` | HTTP (301→HTTPS) + HTTPS | `http://frontend` |
| `admin.krotrust.com` | HTTP (301→HTTPS) + HTTPS | `http://admin-frontend` |
| `app2.krotrust.com` | HTTP (301→HTTPS) + HTTPS | `http://frontend` (alias) |

**TLS:** Let's Encrypt certificates at `/etc/letsencrypt/live/krotrust.com/`  
**WebSocket support:** `api.krotrust.com` proxies WebSocket upgrades (for Socket.IO real-time features)  
**Max body size:** 20MB (for file uploads)  
**Proxy read timeout:** 600 seconds (for long-running API calls)  

---

## CI/CD Pipeline

**File:** `.github/workflows/main.yml`  
**Trigger:** Push to `main` branch  

```yaml
Steps:
1. Set up SSH using NEW_PREPROD_DROPLET_SSH_PRIVATE_KEY secret
2. SSH into 188.166.145.68
3. cd /root/Kro/kro-devops && git pull origin main
4. docker system prune -f
```

**What this does NOT do automatically:**
- It does not rebuild application Docker images
- It does not restart containers
- It only updates the devops configuration files on the server

**After CI runs, manually:**
```bash
ssh root@188.166.145.68
cd /root/Kro/kro-devops/production
docker-compose up --build -d
```

---

## Utility Scripts

### `scripts/wait-for-it.sh`

Standard wait-for-it script. Blocks until a TCP port is open.

Usage: `/wait-for-it.sh backend-db:5432 -- npm run start`

### `scripts/server-cleanup.sh`

Cleans up disk space on the server (old Docker images, logs). Run manually when disk is full.

### `scripts/kro-prod-deploy.sh`

Helper script for manual production deployments. Review its contents before running.

---

## Secrets Required

| Secret | Where | Description |
|---|---|---|
| `NEW_PREPROD_DROPLET_SSH_PRIVATE_KEY` | GitHub Secrets | SSH private key for droplet access |
| `production.json` | On droplet (not in git) | Full app config (DB credentials, API keys) |
| `kro-prod-db-cluster-ca-certificate.crt` | On droplet (not in git) | PostgreSQL SSL CA certificate |
| TLS certificates | On droplet at `/etc/letsencrypt/` | Let's Encrypt TLS certs |

---

## Operational Procedures

### Deploy a New Version

```bash
# 1. Ensure kro-devops/main is up to date (CI handles this on push)
# 2. SSH into server
ssh root@188.166.145.68

# 3. Go to production directory
cd /root/Kro/kro-devops/production

# 4. Pull latest application code
# (each app has its own repo pulled separately)
cd /root/Kro/kro-backend && git pull origin main

# 5. Rebuild and restart
cd /root/Kro/kro-devops/production
docker-compose up --build -d backend

# 6. Check logs
docker-compose logs -f backend
```

### Add a New Environment Variable

1. Update `production.json` on the server
2. Restart the backend: `docker-compose restart backend`
3. Update `.env.example` in the repo (never put real values in the repo)

### Renew TLS Certificates

```bash
ssh root@188.166.145.68
certbot renew
cd /root/Kro/kro-devops/production
docker-compose restart nginx
```

### Check Service Health

```bash
ssh root@188.166.145.68
cd /root/Kro/kro-devops/production
docker-compose ps
docker-compose logs --tail=100 backend
curl -f https://api.krotrust.com/health || echo "Health check failed"
```

---

## Cross-References

- Backend API: [kro-backend.md](./kro-backend.md)
- Infrastructure: [../infrastructure.md](../infrastructure.md)
- Shutdown & Recovery: [../shutdown-and-recovery.md](../shutdown-and-recovery.md)

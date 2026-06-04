# Shutdown and Recovery Procedures

## KRO Platform (DigitalOcean Droplet)

### Graceful Shutdown

```bash
ssh root@188.166.145.68
cd /root/Kro/kro-devops/production

# Stop all containers gracefully
docker-compose down

# Or stop individual services
docker-compose stop backend
docker-compose stop frontend
docker-compose stop admin-frontend
docker-compose stop nginx
```

### Emergency Stop

```bash
docker-compose kill
```

### Restart a Single Service

```bash
docker-compose restart backend
# Check health
docker-compose logs -f backend
```

### Full Recovery After Server Restart

If the droplet is rebooted, containers will auto-restart because they are configured with `restart: unless-stopped`. Verify:

```bash
docker-compose ps
# All should show 'Up' status
```

If containers did not auto-restart:

```bash
cd /root/Kro/kro-devops/production
docker-compose up -d
```

### Database Recovery (KRO)

The PostgreSQL database is a DigitalOcean Managed Database.

**To restore from backup:**
1. Log into the DigitalOcean console
2. Navigate to Databases → KRO production DB
3. Go to Backups → Select backup → Restore

> DigitalOcean performs daily automated backups. Point-in-time recovery is available based on the plan.

**If the DB cluster is unreachable:**
1. Check DO status page for outages
2. Check that the SSL certificate (`kro-prod-db-cluster-ca-certificate.crt`) is valid
3. Verify `production.json` DB credentials are correct
4. Check Nginx/backend logs: `docker-compose logs backend`

### Certificate Renewal Recovery

If TLS certificates expire:

```bash
ssh root@188.166.145.68
certbot renew --force-renewal
docker-compose -f /root/Kro/kro-devops/production/docker-compose.yml restart nginx
```

---

## GIV (KROGiving) — DigitalOcean App Platform

### Shutdown (Pause/Disable)

The App Platform does not have a one-button shutdown. To disable:
1. Log into the DigitalOcean console
2. Navigate to Apps → KROGiving app
3. Go to Settings → Disable app (or destroy deployment)

**Alternative: scale to 0 instances** (if supported by your plan) via the DO API:
```bash
curl -X PUT "https://api.digitalocean.com/v2/apps/<APP_ID>/deployments/<DEPLOY_ID>/cancel" \
     -H "Authorization: Bearer $DIGITALOCEAN_ACCESS_TOKEN"
```

### Recovery / Redeploy

To trigger a full redeploy:
```bash
# Push a new tag (triggers CI)
git tag v<version>
git push origin v<version>

# Or trigger via API
curl -X POST "https://api.digitalocean.com/v2/apps/<STAGING_APP_PLATFORM_ID>/deployments" \
     -H "Authorization: Bearer $DIGITALOCEAN_ACCESS_TOKEN" \
     -H "Content-Type: application/json" \
     -d '{"force_build": true}'
```

### Database Recovery (GIV)

DigitalOcean Managed MongoDB and PostgreSQL both have automated daily backups.

**MongoDB restore:**
1. DO Console → Databases → GIV MongoDB cluster
2. Backups → Restore

**PostgreSQL restore:**
1. DO Console → Databases → GIV PostgreSQL cluster
2. Backups → Restore

**Redis:** Redis is a cache layer only. If Redis data is lost, the application will regenerate cache. OTP codes already in flight will be invalidated (users will need to request new OTPs). No restore needed.

### Rollback a Deployment

In the DO Console under Apps → Deployments, select a previous successful deployment and click "Rollback".

---

## Pencom Platform (On-Premises)

### Shutting Down Services

Stop in reverse startup order to minimize in-flight request errors:

```bash
# 1. Stop public-facing entry points first
pkill -f "dist/apps/api-gateway"
pkill -f "dist/apps/external-gateway"

# 2. Stop downstream services
pkill -f "dist/apps/compliance"
pkill -f "dist/apps/payments"
pkill -f "dist/apps/audit"
pkill -f "dist/apps/notifications"
pkill -f "dist/apps/external-integrations"

# 3. Stop core last
pkill -f "dist/apps/core"
```

If using a process manager (PM2 recommended):
```bash
pm2 stop all
pm2 delete all
```

### Starting Up Services

```bash
cd /path/to/pencom-project

# Recommended startup order
node dist/apps/core/src/main &
sleep 5

node dist/apps/payments/src/main &
node dist/apps/compliance/src/main &
node dist/apps/notifications/src/main &
node dist/apps/audit/src/main &
node dist/apps/external-integrations/src/main &
sleep 5

node dist/apps/api-gateway/main &
node dist/apps/external-gateway/src/main &
```

Using PM2 (recommended for production):
```bash
pm2 start dist/apps/core/src/main.js --name core
pm2 start dist/apps/payments/src/main.js --name payments
pm2 start dist/apps/compliance/src/main.js --name compliance
pm2 start dist/apps/notifications/src/main.js --name notifications
pm2 start dist/apps/audit/src/main.js --name audit
pm2 start dist/apps/external-integrations/src/main.js --name ext-integrations
pm2 start dist/apps/api-gateway/main.js --name api-gateway
pm2 start dist/apps/external-gateway/src/main.js --name ext-gateway
pm2 save
```

### Database Recovery (Pencom)

> Contact the infrastructure team for backup schedules and restore procedures for on-premises PostgreSQL databases.

**Standard PostgreSQL restore:**
```bash
# Stop the service that uses the database first
pm2 stop core

# Restore from dump
psql -h <CORE_DB_HOST> -U <CORE_DB_USER> -d pencom_core_db2 < backup.sql

# Restart service
pm2 start core
```

**MongoDB restore (notifications):**
```bash
mongorestore --uri="$NOTIFICATIONS_DB_URI" --drop ./backup/pencom_notifications
```

### Redis Recovery (Pencom)

Redis is used for job queues (BullMQ) and caching. If Redis is lost:
1. In-flight jobs in BullMQ queues will be lost
2. Re-trigger any critical background jobs manually after Redis restarts
3. Cache will regenerate automatically

### PENCOM Oracle DB Unavailability

The Oracle DB (`external-integrations` service) is read-only from the Pencom app. If it becomes unavailable:
- The `external-integrations` service will degrade gracefully (check error handling)
- Data already synced into `pencom_external_integrations` PostgreSQL will remain usable
- New sync operations will fail and should be retried once Oracle is available
- Contact the PENCOM IT department for Oracle DB issues

---

## General Incident Checklist

When a system goes down, work through these steps in order:

1. **Identify scope** — Is it a single service, a database, or the entire server?
2. **Check logs** — Look at application logs for error messages
   - KRO: `docker-compose logs -f <service>`
   - GIV: DO App Platform console → Logs
   - Pencom: `pm2 logs <service-name>`
3. **Check infrastructure** — Is the server/container running?
   - KRO: `docker-compose ps`
   - GIV: DO App Platform → Overview (deployment status)
   - Pencom: `pm2 status`
4. **Check databases** — Can the application connect to its database?
5. **Check third-party services** — Paystack, SendGrid, Termii status pages
6. **Rollback if needed** — Deploy the last known good version
7. **Communicate** — Notify affected parties of the issue and ETA

---

## Critical Secrets and Where They Live

> **Never commit these to git.**

| System | Secret | Where to Find |
|---|---|---|
| KRO | `production.json` | Stored on droplet at `/root/Kro/kro-devops/production/production.json` |
| KRO | DB CA cert | Stored on droplet at `/root/Kro/kro-devops/production/kro-prod-db-cluster-ca-certificate.crt` |
| GIV | `.env` | DigitalOcean App Platform environment variables (console) |
| GIV | Paystack keys | DigitalOcean App Platform env vars |
| Pencom | `.env` | On server — ask the infrastructure team |
| Pencom | Oracle DB credentials | On server — ask the infrastructure team |
| CI/CD (KRO) | `NEW_PREPROD_DROPLET_SSH_PRIVATE_KEY` | GitHub repo secrets |
| CI/CD (GIV) | `DIGITALOCEAN_ACCESS_TOKEN`, `STAGING_APP_PLATFORM_ID` | GitHub repo secrets |

# Engineering Handover

Complete engineering documentation for the KROTrust, KROGiving (GIV), and Pencom systems.

---

## Products at a Glance

<div class="grid cards" markdown>

-   :material-bank-transfer: **KROTrust**

    ---

    Peer-to-peer conditional payment and escrow platform. Buyers and sellers agree to conditions before funds are released.

    **Stack:** NestJS · PostgreSQL · Docker · DigitalOcean Droplet

    [:octicons-arrow-right-24: KRO Backend](./projects/kro-backend.md) &nbsp; [:octicons-arrow-right-24: DevOps](./projects/kro-devops.md)

-   :material-hand-heart: **KROGiving (GIV)**

    ---

    Crowdfunding and charitable donations platform. Campaign organizers raise funds; donors give via Paystack.

    **Stack:** NestJS · MongoDB · PostgreSQL · React · DigitalOcean App Platform

    [:octicons-arrow-right-24: Backend](./projects/krogiving-backend.md) &nbsp; [:octicons-arrow-right-24: Frontend](./projects/krogiving-frontend.md) &nbsp; [:octicons-arrow-right-24: Admin](./projects/giv-admin-new.md)

-   :material-shield-check: **Pencom**

    ---

    Nigerian pension compliance management system. Employers manage pension obligations with PENCOM via a microservices platform.

    **Stack:** NestJS Monorepo · 8 Microservices · PostgreSQL · Oracle · Redis · On-Premises

    [:octicons-arrow-right-24: Pencom Project](./projects/pencom-project.md)

</div>

---

## Core Documentation

| Document | What it covers |
|---|---|
| [Architecture](./architecture.md) | System diagrams, component relationships, technology stacks |
| [Infrastructure](./infrastructure.md) | Servers, cloud providers, CI/CD pipelines, networking, TLS |
| [Databases](./databases.md) | All databases, schemas, migrations, backup procedures |
| [Deployment](./deployment.md) | Step-by-step deploy instructions, environment variables |
| [Shutdown & Recovery](./shutdown-and-recovery.md) | Graceful shutdown, emergency procedures, disaster recovery |

---

## Production Quick Reference

### URLs

| URL | Service |
|---|---|
| `app.krotrust.com` | KRO user application |
| `api.krotrust.com` | KRO backend API |
| `admin.krotrust.com` | KRO admin panel |

### Servers

| System | Host | Access |
|---|---|---|
| KRO | `188.166.145.68` | SSH with `NEW_PREPROD_DROPLET_SSH_PRIVATE_KEY` |
| GIV | DigitalOcean App Platform | DO Console |
| Pencom | On-premises | Internal network / VPN |

### Deploy a Change

| System | How to deploy |
|---|---|
| KRO (app code) | SSH to droplet → `docker-compose up --build -d` |
| KRO (devops config) | Push to `main` branch → CI pulls to server automatically |
| GIV backend | Push a git tag → GitHub Actions triggers DO App Platform redeploy |
| GIV frontend / admin | DO App Platform GitHub integration (auto on `main` push) |
| Pencom | Manual build + PM2 restart on on-premises server |

---

!!! warning "Before you touch production"
    Read [Shutdown & Recovery](./shutdown-and-recovery.md) to understand the blast radius of any change. Pay special attention to the [Critical Secrets](./shutdown-and-recovery.md#critical-secrets-and-where-they-live) section — secrets are not in git.

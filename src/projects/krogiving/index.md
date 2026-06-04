---
icon: material/hand-heart-outline
---

# KROGiving

KROGiving is a Nigerian crowdfunding platform that enables individuals and organizations to create fundraising campaigns and receive donations from supporters locally and internationally.

## Business Purpose

| Domain | What KROGiving does |
|---|---|
| Campaign management | Creators launch campaigns with targets, durations, rich media, and descriptions |
| Donations | Donors give via Paystack (card, bank transfer, USSD); multi-currency supported |
| Withdrawals | Campaign organizers request payouts; admins approve with TOTP 2FA |
| Admin review | Staff review and approve/reject campaigns before they go live |
| Notifications | Email (SendGrid/MailerSend) and SMS (Termii/Telnyx) for all key events |
| CMS content | Blog and static content managed via Strapi |

## System Components

| Component | Repo | Role |
|---|---|---|
| **krogiving-backend** | `krogiving-backend/` | NestJS API — all business logic, payments, auth |
| **krogiving-frontend** | `krogiving-frontend/` | React 18 SPA — public donor and campaign creator app |
| **giv-admin-new** | `giv-admin-new/` | React 19 + Vite admin dashboard — internal staff tool |

## Infrastructure

KROGiving runs on **DigitalOcean App Platform**. Each component is deployed as a separate App Platform component. Git-tag-triggered CI/CD calls the DO App Platform API to redeploy the relevant component.

**Databases:**

- **PostgreSQL** (DigitalOcean Managed) — campaigns, users, donations, withdrawals
- **MongoDB** (DigitalOcean Managed) — admin users, admin logs, reviews, settings
- **Redis** — OTP code cache (TTL-based expiry)

**File storage:** DigitalOcean Spaces (S3-compatible) for campaign images; Cloudinary for campaign videos (uploaded directly from browser).

## Quick Navigation

| Doc | What it covers |
|---|---|
| [Architecture](architecture.md) | Tech stack per component, module structure, request flows, API surface, integrations |
| [Deployment](deployment.md) | DO App Platform setup, CI/CD, environment variables, operational procedures |

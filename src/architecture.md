# System Architecture

## Overview

This workspace contains two independent product lines and one client project, each with its own stack and infrastructure:

| Product | Purpose | Infrastructure |
|---|---|---|
| **KRO** | Peer-to-peer conditional payment / escrow platform | DigitalOcean Droplets (Docker + Nginx) |
| **KROGiving (GIV)** | Crowdfunding and charitable donations platform | DigitalOcean App Platform |
| **Pencom** | Nigerian pension compliance management system | On-Premises infrastructure |

---

## KRO Platform

KRO (krotrust.com) is a trust/escrow platform where two parties (buyer and seller) agree to terms before money is released. Orders are created with conditions; funds are held until conditions are met or a dispute is resolved.

### Components

```
┌──────────────────────────────────────────────────────────────┐
│                      DigitalOcean Droplet                    │
│                                                              │
│  ┌──────────┐   ┌──────────────┐   ┌───────────────────┐   │
│  │  Nginx   │──▶│ kro-backend  │──▶│  PostgreSQL (DO)  │   │
│  │ (proxy)  │   │  NestJS/10   │   │  (managed DB)     │   │
│  └──────────┘   └──────────────┘   └───────────────────┘   │
│       │                                                      │
│       ├──▶ app.krotrust.com  → kro-frontend (React)         │
│       ├──▶ api.krotrust.com  → kro-backend  (:3000)         │
│       └──▶ admin.krotrust.com → kro-admin  (React)          │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

**Repositories:**
- `kro-backend` — NestJS v10, TypeScript, PostgreSQL via TypeORM
- `kro-devops` — Docker Compose configs and Nginx for local/stage/production
- `kro-frontend` — (not in this workspace) React frontend at app.krotrust.com
- `kro-admin` — (not in this workspace) React admin at admin.krotrust.com

**Payment providers:** Paystack, Fincra, Providus

**Notifications:** SendGrid (email), Twilio/Telnyx/Termii (SMS), WhatsApp

**Observability:** Highlight.io (error tracking + session replay), OpenTelemetry traces

---

## KROGiving (GIV) Platform

KROGiving is a crowdfunding platform where campaign organizers raise funds from donors. Donations are processed through Paystack and eventually withdrawn to the campaign organizer's bank account.

### Components

```
┌─────────────────────────────────────────────────────────────────────┐
│                    DigitalOcean App Platform                         │
│                                                                      │
│  ┌───────────────────┐     ┌──────────────────────────────────┐     │
│  │ krogiving-frontend │     │      krogiving-backend           │     │
│  │  React CRA / TSX   │────▶│  NestJS v10, TypeScript          │     │
│  │  (user-facing)     │     │                                  │     │
│  └───────────────────┘     │  ┌────────────────────────────┐  │     │
│                             │  │  MongoDB (Mongoose)        │  │     │
│  ┌───────────────────┐     │  │  Admin, campaigns, logs    │  │     │
│  │   giv-admin-new   │     │  └────────────────────────────┘  │     │
│  │  React + Vite     │────▶│                                  │     │
│  │  (admin panel)    │     │  ┌────────────────────────────┐  │     │
│  └───────────────────┘     │  │  PostgreSQL (TypeORM)      │  │     │
│                             │  │  Donations, withdrawals    │  │     │
│                             │  └────────────────────────────┘  │     │
│                             │                                  │     │
│                             │  ┌────────────────────────────┐  │     │
│                             │  │  Redis                     │  │     │
│                             │  │  OTP / caching             │  │     │
│                             │  └────────────────────────────┘  │     │
│                             └──────────────────────────────────┘     │
│                                                                      │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  DigitalOcean Spaces (S3-compatible)                          │  │
│  │  Campaign images, user uploads                                │  │
│  └───────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

**Repositories:**
- `krogiving-backend` — NestJS v10, dual-database (MongoDB + PostgreSQL), Redis
- `krogiving-frontend` — React CRA v18, Paystack payments, i18n, Split.io feature flags
- `giv-admin-new` — React v19 + Vite, admin dashboard for the GIV platform

**Integrations:** Paystack, Strapi CMS, SendGrid, Termii/Telnyx (SMS), DigitalOcean Spaces, Cloudinary (video), Highlight.io

---

## Pencom Platform

Pencom is a Nigerian pension compliance management system that interfaces with the National Pension Commission (PENCOM). It allows employers to register, manage employees, track pension contributions, and generate compliance certificates.

### Components

```
┌───────────────────────────────────────────────────────────────────┐
│                    NestJS Monorepo (pencom-project)               │
│                                                                   │
│  ┌─────────────────┐   TCP/Microservices   ┌─────────────────┐  │
│  │   api-gateway   │◀────────────────────▶│     core        │  │
│  │  :3000 (HTTP)   │                       │  :4000 (HTTP)   │  │
│  │  External entry │                       │  Business logic │  │
│  └─────────────────┘                       └─────────────────┘  │
│         │                                          │             │
│         ├──▶ compliance   :6000 (GLI, PCC)         │             │
│         ├──▶ payments     :5000 (Remita)           │             │
│         ├──▶ audit        :9000                    │             │
│         ├──▶ notifications :7000                   │             │
│         ├──▶ external-integrations :8000           │             │
│         └──▶ external-gateway :3010               │             │
│                                                   │             │
│  Shared libs: @app/database, @app/entities,       │             │
│               @app/shared, @app/logger            │             │
└───────────────────────────────────────────────────────────────────┘

Databases:
  - PostgreSQL  (core, compliance, payments, audit, external-integrations)
  - MongoDB     (notifications)
  - Oracle      (PENCOM on-prem oracle DB — read-only integration)
  - Redis       (caching, job queues via BullMQ)
```

**Key integrations:** Remita (payments), PENCOM Oracle DB (on-prem), SendGrid, Termii, AWS S3, PostHog analytics, Highlight.io

---

## Cross-Cutting Concerns

| Concern | KRO | GIV | Pencom |
|---|---|---|---|
| Auth | JWT + Passport | JWT + Passport | JWT + Passport |
| Email | SendGrid | SendGrid + MailerSend | SendGrid |
| SMS | Twilio / Telnyx / Termii | Termii / Telnyx | Termii |
| File storage | AWS S3 / DO Spaces | DO Spaces + Cloudinary | AWS S3 |
| Error tracking | Highlight.io | Highlight.io | Highlight.io |
| Tracing | OpenTelemetry | — | — |
| Job queues | NestJS Schedule | NestJS Schedule | BullMQ + Redis |
| 2FA | TOTP (speakeasy) | TOTP | — |

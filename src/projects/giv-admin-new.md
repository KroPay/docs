# giv-admin-new

## Business Purpose

The GIV Admin Dashboard is the internal tool used by KROGiving staff to manage the crowdfunding platform. Admins use it to:

- Review and approve/reject fundraising campaigns before they go live
- Monitor donations across all campaigns
- Manage payouts/withdrawals to campaign organizers
- Manage admin users and their permissions
- Configure platform-wide settings (withdrawal caps, exchange rate adjustments)
- View audit logs of admin actions
- Send messages to campaign organizers
- Track donation remittance status

---

## Architecture

**Framework:** React 19 + Vite 6 + TypeScript  
**Styling:** Tailwind CSS v4 + shadcn/ui (Radix UI components)  
**State:** Zustand (global), TanStack Query v5 (server state)  
**Routing:** React Router DOM v7  
**Forms:** React Hook Form + Zod validation  
**Real-time:** Socket.IO client (notifications)  
**Charts:** Recharts  

### Directory Structure

```
src/
├── components/
│   ├── admin/          — Admin user management dialogs & tables
│   ├── auth/           — Login form, OTP form
│   ├── layout/         — MainLayout, sidebar, nav
│   ├── settings/       — Global withdrawal cap, 2FA setup
│   └── ui/             — shadcn/ui primitives (button, dialog, table…)
├── config/
│   ├── api.ts          — All API route constants
│   └── routes.ts       — Frontend route constants
├── hooks/              — TanStack Query hooks for all data fetching
├── lib/
│   ├── api.ts          — Axios instance with auth interceptors
│   └── websocket.ts    — Socket.IO setup
├── middleware/
│   └── auth.tsx        — Route guard (redirects unauthenticated)
├── pages/              — Page components per route
│   ├── auth/login.tsx
│   ├── dashboard.tsx
│   ├── campaign/       — Campaigns list, details, review, review logs
│   ├── donations.tsx
│   ├── donations/remitted.tsx
│   ├── payouts.tsx
│   ├── users.tsx
│   ├── admin/          — Admin user management pages
│   ├── logs.tsx
│   ├── notifications.tsx
│   └── settings.tsx
├── services/           — API service functions (campaign, donation)
└── store/              — Zustand stores (user, metrics, notifications)
```

---

## Request Flow

```
User opens browser
  → React Router loads page component
  → auth.tsx middleware checks JWT in store/localStorage
  → If authenticated, render page
  → Page component calls TanStack Query hook (e.g., useCampaigns)
  → Hook calls API service via Axios instance in lib/api.ts
  → Axios sends request to krogiving-backend API
  → Response updates TanStack Query cache
  → Page renders data
```

**Authentication flow:**
1. Admin enters email → `POST /v1/otp/send`
2. Admin enters OTP → `POST /v1/admin/auth/sign-in` → receives JWT
3. JWT stored in Zustand store
4. All subsequent requests include `Authorization: Bearer <token>` header
5. On 401 response, Axios interceptor attempts token refresh at `/v1/users/refresh-token`

**Real-time notifications:**
- Socket.IO connects to krogiving-backend WebSocket server
- New notifications appear in the notification drawer without refresh

---

## Key Pages and Features

| Page | Route | Description |
|---|---|---|
| Login | `/auth/login` | OTP-based admin login |
| Dashboard | `/` | Metrics overview and charts |
| Campaigns | `/campaigns` | List all campaigns with filters |
| Campaign Details | `/campaigns/:id` | Full campaign info |
| Campaign Review | `/campaigns/:id/review` | Approve/reject campaign |
| Review Logs | `/campaigns/review-logs` | History of all reviews |
| Donations | `/donations` | All donations across platform |
| Remitted Donations | `/donations/remitted` | Donations marked as sent |
| Payouts | `/payouts` | Withdrawal requests, approve with 2FA |
| Users | `/users` | All platform users |
| Admins | `/admin` | Manage admin accounts |
| Logs | `/logs` | Admin audit log |
| Notifications | `/notifications` | In-app notification center |
| Settings | `/settings` | Global withdrawal cap, 2FA setup |

---

## API Endpoints Consumed

All endpoints are on `krogiving-backend`. Prefix: `/v1/`

| Feature | Method | Endpoint |
|---|---|---|
| Send OTP | POST | `/v1/otp/send` |
| Admin sign-in | POST | `/v1/admin/auth/sign-in` |
| Get permissions | GET | `/v1/admin/auth/permissions` |
| List campaigns | GET | `/v1/admin/campaigns` |
| Campaign metrics | GET | `/v1/admin/campaigns/metrics` |
| Campaign statistics | GET | `/v1/admin/campaigns/:id/statistics` |
| Update campaign status | PATCH | `/v1/admin/campaigns/:id/status` |
| Campaign review logs | GET | `/v1/admin/campaigns/:id/review-logs` |
| Send campaign message | POST | `/v1/admin/campaigns/:id/message` |
| Update withdrawal cap | PATCH | `/v1/admin/campaigns/:id/withdrawal-cap` |
| All donations | GET | `/v1/admin/donations` |
| Remitted donations | GET | `/v1/admin/donations/remitted` |
| Export donations CSV | GET | `/v1/admin/donations/export/csv` |
| All payouts | GET | `/v1/admin/payouts` |
| Approve payout | POST | `/v1/admin/payouts/approve` |
| Init 2FA for payout | POST | `/v1/admin/payouts/two-factor/initialize` |
| All users | GET | `/v1/admin/users` |
| List admins | GET | `/v1/admin/auth/admins` |
| Create admin | POST | `/v1/admin/auth/admins` |
| Admin logs | GET | `/v1/admin/logs` |
| Global withdrawal cap | GET/PUT | `/v1/admin/settings/global-withdrawal-cap` |

---

## Deployment

**Platform:** DigitalOcean App Platform (or self-hosted static build)  
**Build command:** `yarn build` → `tsc -b && vite build`  
**Output:** `dist/` directory (static files)

**Environment variables (`.env`):**
- `VITE_API_BASE_URL` — Backend API URL (e.g., `https://api.krogiving.com`)

**Local dev:**
```bash
yarn install
cp .env.example .env
# Set VITE_API_BASE_URL
yarn dev
```

---

## Dependencies

| Dependency | Purpose |
|---|---|
| `react-router-dom` v7 | Client-side routing |
| `@tanstack/react-query` v5 | Server state management |
| `zustand` v5 | Global client state |
| `axios` | HTTP client |
| `socket.io-client` | WebSocket for real-time notifications |
| `react-hook-form` + `zod` | Form management and validation |
| `recharts` | Charts and data visualization |
| `@radix-ui/*` | Accessible UI primitives (via shadcn/ui) |
| `tailwindcss` v4 | Utility-first CSS |
| `date-fns` | Date formatting |
| `lucide-react` | Icon library |
| `sonner` | Toast notifications |

---

## Operational Notes

- **Authentication:** OTP-based (no password). Email → OTP → JWT. 2FA (TOTP) required for approving payouts.
- **Permissions:** Admin roles control which pages/actions are visible. Permissions fetched at login from `/v1/admin/auth/permissions`.
- **Withdrawal cap:** A global per-campaign withdrawal cap can be set. Logs of changes are tracked in the `WithdrawalCapLog` collection.
- **Export:** Donations can be exported as CSV. Currency-converted donation data can be imported back.
- **Real-time:** Admin receives real-time notifications via Socket.IO (e.g., new campaign submissions).

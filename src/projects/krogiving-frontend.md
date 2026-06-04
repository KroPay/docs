# krogiving-frontend

## Business Purpose

The KROGiving frontend is the public-facing web application for the KROGiving crowdfunding platform. It allows:

- **Donors** to discover campaigns, donate via Paystack, and track their giving history
- **Campaign organizers** to create and manage fundraising campaigns, upload media, set targets and durations
- **Anonymous visitors** to browse campaigns, read campaign details, and make donations without creating an account

The platform supports international donations (multiple currencies), social sharing, campaign comments, rich-text content, and a blog powered by Strapi CMS.

---

## Architecture

**Framework:** React 18, Create React App, TypeScript  
**Styling:** Tailwind CSS v3 + custom CSS  
**State:** Zustand (auth state), TanStack Query v5 (server state)  
**Routing:** React Router DOM v6  
**Forms:** TanStack Form v0.33  
**Payments:** react-paystack + paystack  
**Internationalization:** react-i18next + i18next-browser-languagedetector  
**Analytics / Feature Flags:** Highlight.io (session replay), Split.io  
**Rich Text Editor:** CKEditor 5 (premium) + Jodit  
**Video:** Cloudinary upload, react-player  
**Phone Numbers:** google-libphonenumber / libphonenumber-js  
**Testing:** Jest + React Testing Library  

### Directory Structure

```
src/
├── common/
│   ├── assets/         — Lottie animations
│   ├── constants/      — App-wide constants
│   ├── context/        — React contexts (auth, snackbar)
│   ├── data/           — Country/state data
│   ├── enums/          — Use case enums
│   ├── hooks/          — Shared hooks (cloudinary upload, geolocation, debounce)
│   ├── io/
│   │   ├── client/kroClient.ts  — Axios HTTP client with auth interceptors
│   │   └── endpoints.ts         — All API endpoint constants
│   ├── layout/         — KroLayout, KroNavBar, KroFooter, WhatsApp button
│   ├── models/         — TypeScript user model
│   ├── services/       — State management (states.ts)
│   ├── store/          — Zustand auth store
│   └── utils/          — Currency formatting, date utils, fee calc, etc.
└── features/
    ├── campaign/
    │   ├── domain/
    │   │   ├── enums/      — Input types, upload card types
    │   │   ├── models/     — Campaign, donation, payment models
    │   │   ├── repositories/ — API call functions
    │   │   └── services/   — Campaign service layer
    │   └── presentation/
    │       ├── components/ — UI components for campaign creation and display
    │       └── pages/      — Campaign listing, detail, creation wizard pages
    └── [other features...]
```

### Architecture Pattern

The frontend follows a feature-based architecture with domain/presentation separation:

```
features/<name>/
├── domain/           — Pure business logic: models, API calls, services
└── presentation/     — React components and pages
```

This pattern keeps API concerns separate from rendering concerns.

---

## Request Flow

### Donor Making a Donation

```
1. User visits campaign page (krogiving-frontend)
   → GET /campaigns/:slug from krogiving-backend
   → Campaign details rendered

2. User clicks "Donate"
   → Donation form: amount, name, email, currency

3. User submits donation
   → POST /donations/initialize → Paystack transaction reference returned
   → react-paystack opens Paystack payment modal

4. User completes payment in Paystack modal
   → Paystack calls onSuccess callback

5. Frontend verifies payment
   → POST /donations/verify → krogiving-backend verifies with Paystack
   → Success screen shown with confetti animation

6. Donor receives email confirmation (backend sends via SendGrid)
```

### Campaign Creator Creating a Campaign

```
1. User signs up / logs in
   → POST /users/register or /users/login → JWT returned
   → JWT stored in Zustand auth store

2. Create campaign wizard:
   → Step 1: Category and type selection
   → Step 2: Campaign information (title, description via CKEditor)
   → Step 3: Media upload (images → DO Spaces, video → Cloudinary)
   → Step 4: Target amount and duration
   → POST /campaigns → campaign created in draft state

3. Submit for review
   → PATCH /campaigns/:id/submit

4. Admin reviews and approves (in giv-admin-new)
   → Creator notified via email

5. Campaign goes live
```

---

## Key Features

| Feature | Description |
|---|---|
| Multi-currency donations | Donors can donate in their local currency; conversion tracked by backend |
| Paystack integration | Nigerian card, bank transfer, USSD payments via react-paystack |
| Campaign creation wizard | Multi-step form with CKEditor rich text, image gallery, video |
| Cloudinary video upload | Large video files uploaded directly to Cloudinary |
| Social sharing | Campaign links shareable via react-share |
| Campaign comments | Donors and organizers can comment on campaigns |
| i18n | Multi-language support via react-i18next |
| Feature flags | Split.io used for A/B testing / feature rollouts |
| Session replay | Highlight.io for debugging user sessions |
| Avatar generation | DiceBear for user avatars |
| Image cropping | react-cropper for profile and campaign images |
| Blog / CMS | Strapi CMS content rendered via @strapi/blocks-react-renderer |
| QR codes | QR code generation for campaign sharing (inferred from backend) |

---

## Third-Party Integrations

| Service | Purpose | Package |
|---|---|---|
| Paystack | Payment processing | `react-paystack`, `paystack` |
| Cloudinary | Video upload | `useCloudinaryVideoUpload` hook |
| Split.io | Feature flags | `@splitsoftware/splitio-react` |
| Highlight.io | Error tracking + session replay | `highlight.run`, `@highlight-run/react` |
| Strapi | CMS for blog/static content | `@strapi/blocks-react-renderer` |
| DiceBear | Random avatar generation | `@dicebear/core`, `@dicebear/collection` |
| GSAP | Animations | `gsap` |

---

## Environment Variables

Environment files used:
- `.env.development.local` — local development
- `.env.production.local` — production

Key variables (inferred from code patterns):
```bash
REACT_APP_API_BASE_URL=        # Backend API URL
REACT_APP_PAYSTACK_PUBLIC_KEY= # Paystack public key (for frontend)
REACT_APP_SPLIT_API_KEY=       # Split.io API key
REACT_APP_HIGHLIGHT_PROJECT_ID= # Highlight.io project ID
REACT_APP_CLOUDINARY_CLOUD_NAME= # Cloudinary cloud name
REACT_APP_CLOUDINARY_UPLOAD_PRESET= # Cloudinary unsigned upload preset
```

---

## Deployment

**Platform:** DigitalOcean App Platform (static site)  
**Build command:** `react-scripts build`  
**Output:** `build/` directory

**Scripts:**
```bash
yarn start           # Local dev with .env.development.local
yarn start:production # Local dev with .env.production.local
yarn build:staging   # Build without env override
yarn build           # Production build
yarn test            # Run Jest test suite
```

**Pre-commit hooks (Husky):**
- ESLint auto-fix
- Prettier auto-format
- Console.log check (prevents accidental console.log commits)

**Commit conventions:** Conventional commits enforced via `@commitlint/cli`.

---

## Operational Notes

- **API client:** `src/common/io/client/kroClient.ts` — Axios instance with auth token injection and error handling
- **Auth flow:** JWT stored in Zustand. On 401, token refresh is attempted. Unauthenticated users are redirected to login for protected routes.
- **Fee calculator:** `src/common/utils/fee_calculator.ts` — Client-side fee preview before donation submission
- **Currency formatting:** `src/common/utils/currencyFormater.ts` — Formats amounts per currency
- **Console log guard:** `scripts/check-console-log.js` blocks commits with `console.log` statements (except in test files)
- **CKEditor premium:** The app uses `ckeditor5-premium-features` which requires a valid CKEditor license
- **WhatsApp button:** A floating WhatsApp contact button appears on all pages (`whatsappContainer.tsx`)

---

## Testing

```bash
yarn test           # Run all Jest tests
yarn test --coverage # With coverage report
```

Test setup: `jest.config.ts`, `jest.setup.ts`, `babel.config.js`.  
Uses `@testing-library/react` with `jsdom` environment.

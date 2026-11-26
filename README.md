# Student Messenger

**Student Messenger** is a fullstack academic cohort communication platform combining a cloud-hosted web dashboard, Supabase PostgreSQL with Row-Level Security (RLS), Vercel Serverless APIs, an individual message queue, and a **local Node.js TypeScript Playwright worker** that drives WhatsApp Web through a dedicated persistent Chrome profile.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│  Web Dashboard (React 19 + TypeScript + Tailwind CSS + Supabase Auth)   │
│  Overview • Students • Groups • Campaigns • Compose • Queue • Worker    │
└───────────────────────────────┬─────────────────────────────────────────┘
                                │ Bearer JWT + Realtime Subscriptions
                                ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  Vercel Serverless API Routes (/api/*.js)                               │
│  Verifies JWT & executes user-scoped queries under Postgres RLS         │
└───────────────────────────────┬─────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  Supabase PostgreSQL (students, groups, student_groups, campaigns,      │
│  message_queue, workers, worker_logs) + RLS + Realtime                  │
└───────────────────────────────▲─────────────────────────────────────────┘
                                │ Service Role Key (scoped to WORKER_OWNER_USER_ID)
                                │ Claims pending queue rows, syncs heartbeats & desired_state
┌───────────────────────────────┴─────────────────────────────────────────┐
│  Local Node.js TypeScript Worker (worker/src/*.ts)                      │
│  Playwright launchPersistentContext() → Dedicated Chrome Profile        │
│  WhatsApp Web Session • Humanized Throttling • Pause/Resume/Stop        │
└─────────────────────────────────────────────────────────────────────────┘
```

> **Note on Framework Choice:** This repository runs on Vite + React 19 + TypeScript + Vercel Serverless Functions (`/api/*.js`), pre-wired for direct deployment on Vercel with Supabase and shared TypeScript modules across `shared/`, `src/`, and `worker/`.

---

## Key Features

- **Supabase Authentication**: Email/password sign-up & sign-in, Google OAuth support, protected routes, and JWT-verified backend API routes.
- **Row-Level Security (RLS)**: Strict per-user tenant isolation (`auth.uid() = user_id`) on all 7 PostgreSQL tables.
- **Student Management**: Full CRUD directory, E.164 international phone normalization & validation, opt-out protection, group tagging, search/filtering, and bulk CSV import.
- **Group Management**: Color-coded student cohorts with interactive membership management.
- **Message Composer & Recipient Selection**: Target by Groups, Individual Students, Mixed, or All Active Students with automatic deduplication, opt-out exclusion, template variables (`{{first_name}}`, `{{full_name}}`, `{{grade}}`, `{{group}}`, `{{student_code}}`, `{{guardian_name}}`), and live WhatsApp bubble preview.
- **Campaign Management**: Save drafts or enqueue immediately, live delivery progress bars, campaign pause/resume/cancel, and humanized throttle windows.
- **Individual Message Queue & History**: Per-recipient queue records with status filtering, priority boost, pause/resume, cancel, single & bulk retry of failed messages, delivery audit detail modal, and CSV export.
- **Local Node.js TypeScript Playwright Worker**:
  - Dedicated persistent Chrome user data directory (preserves WhatsApp Web login across restarts).
  - Live dashboard **Pause**, **Resume**, and **Stop** controls via `workers.desired_state`.
  - QR authentication detection (`whatsapp_status = 'qr_required'`) surfaced live in the dashboard.
  - Exponential retry backoff, stale processing lock recovery, automatic browser context recovery after consecutive failures, and `DRY_RUN=true` testing mode.

---

## Project Folder Structure

```
.
├── api/                         # Vercel Serverless API routes (JWT-verified + RLS-scoped)
│   ├── _auth.js                 # CORS, Supabase JWT auth & server helpers
│   ├── db-client.js             # Pre-configured Supabase service client
│   ├── db-wake.js               # Database resilience helper
│   ├── dashboard.js             # Aggregated KPI & overview summary
│   ├── students.js              # Student CRUD & bulk CSV import
│   ├── groups.js                # Group CRUD & membership sync
│   ├── campaigns.js             # Campaign creation, queueing, pause/resume/cancel
│   ├── queue.js                 # Individual queue & history operations
│   ├── workers.js               # Worker telemetry, logs, pause/resume/stop & lock recovery
│   └── seed-sample.js           # One-click sample cohort seeder for new accounts
├── shared/                      # Shared TypeScript types & pure business logic
│   ├── types.ts                 # Shared domain types for frontend, worker & tests
│   └── template.ts              # Phone E.164 normalization, templates, CSV parser, helpers
├── src/                         # React + TypeScript + Tailwind frontend
│   ├── components/              # Layout, ProtectedRoute, Modal, StatusBadge, EmptyState
│   ├── contexts/                # AuthContext (Supabase session provider)
│   ├── lib/                     # supabase.ts, googleAuth.ts, api.ts
│   └── pages/                   # Login, Dashboard, Students, Groups, Compose, Campaigns, Queue, History, WorkerMonitor
├── worker/                      # Local Node.js TypeScript Playwright WhatsApp Worker
│   ├── tsconfig.json
│   └── src/
│       ├── index.ts             # CLI entry point & SIGINT/SIGTERM handling
│       ├── config.ts            # Env loader & persistent Chrome profile config
│       ├── supabaseClient.ts    # Queue claiming, heartbeats, retries & campaign sync
│       ├── whatsappClient.ts    # Playwright persistent Chrome & WhatsApp Web automation
│       └── runner.ts            # Main polling loop, pause/resume/stop & crash recovery
├── supabase/migrations/         # Versioned PostgreSQL migrations, RLS & Realtime setup
├── tests/                       # Automated Vitest unit & integration test suites
├── docs/                        # Dedicated Windows, Supabase & Vercel guides
├── .env.example                 # Web & API environment template
├── .env.worker.example          # Local worker environment template
└── package.json
```

---

## Quick Start

### 1. Install Dependencies
```bash
npm install
```

### 2. Configure Environment Variables
```bash
cp .env.example .env
cp .env.worker.example .env.worker
```
Fill in your Supabase project URL, anon key, service role key, and `WORKER_OWNER_USER_ID`.

### 3. Apply Database Migrations
Run the SQL files in `supabase/migrations/` in order inside your Supabase SQL Editor (see `docs/SUPABASE_SETUP.md`).

### 4. Run the Web Application Locally
```bash
npm run dev
```

### 5. Start the Local Playwright WhatsApp Worker
```bash
# First run opens the dedicated persistent Chrome profile so you can scan WhatsApp Web QR:
npm run worker:start

# Or verify queue polling & heartbeats safely without sending real WhatsApp messages:
DRY_RUN=true npm run worker:start
```

### 6. Run Type Checks, Linting & Automated Tests
```bash
npm run typecheck
npm run lint
npm test
npm run build
```

---

## Detailed Guides

- **Windows Installation & Persistent Chrome Setup**: See [`docs/WINDOWS_INSTALL.md`](docs/WINDOWS_INSTALL.md)
- **Supabase Database, Auth, RLS & Realtime Setup**: See [`docs/SUPABASE_SETUP.md`](docs/SUPABASE_SETUP.md)
- **Vercel Production Deployment**: See [`docs/VERCEL_DEPLOYMENT.md`](docs/VERCEL_DEPLOYMENT.md)
#   S T U D E N T M A N A G M E N T _ S Y S T E M  
 
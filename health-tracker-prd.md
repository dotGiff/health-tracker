# Peptide & Weight Tracker — Product Requirements Document

**Version:** 1.2  
**Date:** May 22, 2026  
**Status:** Draft

**Changelog v1.2**
- Retooled to TypeScript NX monorepo
- Frontend: Next.js 15 (App Router) replacing vanilla JS PWA
- Backend: NestJS + Fastify adapter replacing FastAPI
- ORM: Drizzle replacing SQLModel/Alembic
- Auth: hand-rolled JWT with NestJS Guards + Passport JWT strategy
- Shared libs: `@tracker/types`, `@tracker/calculations`

**Changelog v1.1**
- Added multi-user support with per-user data isolation
- Added custom health metrics system (ad-hoc, user-defined)
- Updated dashboard to support dynamic multi-metric chart overlays with per-metric y-axis scaling
- Moved weight tracking into the custom metrics system as a first-class built-in metric

---

## 1. Overview

A multi-user Progressive Web App for tracking peptide injection protocols and arbitrary health metrics over time. Each user manages their own peptides, injection logs, and a fully custom set of health metrics — which can be defined and added at any point in their health journey without any schema changes.

The app visualizes how peptide concentration accumulates and decays in the body (based on configurable half-lives) alongside any combination of health metrics on a single chart, with independent y-axis scaling per metric so that vastly different units (lbs, inches, score 1–10, mg/dL, etc.) can be overlaid meaningfully.

Accessible from any device via browser. Installable on iOS home screen as a PWA. All data stored server-side in a persistent SQLite database.

---

## 2. Goals

- Support multiple independent user accounts, each with fully isolated data
- Log any number of user-defined health metrics (weight, waist, blood pressure, side effects, mood, etc.) on any date, independently of injections
- Support multiple peptides per user, each with a configurable name and half-life
- Automatically calculate and store estimated peptide concentration in system at every injection point
- Visualize any combination of metrics and peptide concentrations on a single chart, each with its own independently scaled y-axis
- Provide weekly trend summaries and a recent entries log on the dashboard
- Allow new metrics to be defined at any time without affecting historical data for other metrics

## 3. Non-Goals

- Native iOS/Android app distribution via App Store
- Calorie or macro tracking (unless added as a custom metric by the user)
- Integration with external health platforms (Apple Health, Fitbit, etc.) — v1
- Shared data or social features between users

---

## 4. Tech Stack

| Layer | Technology |
|---|---|
| Monorepo | NX |
| Frontend | Next.js 15 (App Router) |
| Charts | Chart.js |
| Frontend Hosting | Vercel |
| Backend | NestJS + Fastify adapter |
| ORM | Drizzle ORM |
| Database | SQLite (on Fly.io persistent volume) |
| Backend Hosting | Fly.io |
| Auth | Hand-rolled JWT — NestJS Guards + Passport JWT strategy + bcrypt |
| Shared Libs | `@tracker/types`, `@tracker/calculations` |
| Backups | Litestream → Fly.io Tigris (continuous replication) |

---

## 5. Architecture

```
[Browser / iOS PWA]
[Next.js 15 App Router on Vercel]
        │
        │  HTTPS REST API (JWT Bearer token)
        ▼
[NestJS + Fastify on Fly.io]
        │
        │  Drizzle ORM
        ▼
[Fly.io Persistent Volume]
   └── tracker.db
        │
        │  Litestream continuous replication
        ▼
[Fly.io Tigris Object Storage]
   └── tracker.db backups
```

- NX monorepo with two apps (`web`, `api`) and two shared libs (`@tracker/types`, `@tracker/calculations`)
- `web` is a Next.js 15 App Router app deployed to Vercel; all pages are React Server Components by default, with client components opted in via `"use client"` only where needed (chart rendering, forms, interactive UI)
- `api` is a NestJS app using the Fastify adapter, deployed to a single Fly.io machine (never auto-scaled — required for SQLite file consistency)
- SQLite file lives on a Fly.io volume mounted at `/data/tracker.db`; accessed exclusively by the `api` app via Drizzle
- Litestream runs as a sidecar process in the same Fly.io container, continuously replicating to Tigris
- `@tracker/types` contains shared TypeScript interfaces and Zod schemas used by both `web` and `api`
- `@tracker/calculations` contains the half-life decay math as a pure TypeScript library, imported by `api` at write time; also importable by `web` if client-side projections are ever needed
- All concentration calculations run server-side (in `api`) at write time; results stored in the database
- All queries in `api` are scoped to the authenticated user's `userId`

---

## 6. User Model & Authentication

### 6.1 User

| Field | Type | Notes |
|---|---|---|
| `id` | integer PK | |
| `email` | string unique | |
| `password_hash` | string | bcrypt |
| `display_name` | string | |
| `role` | enum | `admin`, `user` |
| `created_at` | datetime | |
| `is_active` | boolean | admins can deactivate accounts |

### 6.2 Auth Flow

- Registration is **invite-only**: only an `admin` user can create new accounts (via a protected `POST /admin/users` endpoint or a CLI script). No public sign-up form.
- Login returns a **JWT access token** (1-hour expiry) signed with `JWT_SECRET` + a **refresh token** (30-day expiry) stored as an httpOnly cookie
- Access token is sent as a Bearer token in the `Authorization` header on every API request
- NestJS auth implementation:
  - `PassportStrategy(Strategy, 'jwt')` validates the Bearer token on every protected route
  - `JwtAuthGuard` extends `AuthGuard('jwt')` and is applied globally; public routes use a `@Public()` decorator to opt out
  - `AuthService` handles login, token issuance, refresh token rotation, and bcrypt comparison
  - Refresh tokens are stored hashed in a `refresh_tokens` table with `userId`, `tokenHash`, and `expiresAt`; rotating on each use (old token invalidated, new one issued)
- All data queries in every service are automatically filtered by `req.user.id` injected by the Guard
- Optional: Google OAuth as a second login method — v2

---

## 7. Data Models

All models include a `user_id` FK to `User`. All queries are scoped to the authenticated user.

### 7.1 Peptide

| Field | Type | Notes |
|---|---|---|
| `id` | integer PK | |
| `user_id` | FK → User | |
| `name` | string | e.g. "Retatrutide" |
| `half_life_days` | float | e.g. 7.0 |
| `unit` | string | default "mg" |
| `color` | string | hex color for chart |
| `created_at` | datetime | |
| `archived` | boolean | soft delete |

### 7.2 MetricDefinition

Defines a user-created health metric. Created once; entries are logged against it indefinitely. Can be added at any point in the user's health journey — existing data for other metrics is completely unaffected.

| Field | Type | Notes |
|---|---|---|
| `id` | integer PK | |
| `user_id` | FK → User | |
| `name` | string | e.g. "Weight", "Waist", "Energy Level", "Nausea" |
| `unit` | string | e.g. "lbs", "in", "score 1–10", "mg/dL" |
| `data_type` | enum | `numeric`, `boolean`, `scale_1_10` |
| `color` | string | hex color for chart |
| `is_builtin` | boolean | true for Weight (pre-seeded per user, cannot be deleted) |
| `display_order` | integer | user-controlled sort order |
| `created_at` | datetime | |
| `archived` | boolean | soft delete; hides from active UI but preserves all history |

**Built-in metrics:** When a new user account is created, `Weight (lbs)` is automatically seeded as a `MetricDefinition` with `is_builtin = true`. This keeps weight first-class without special-casing it elsewhere in the data model.

**Data types:**
- `numeric` — any float value (weight, waist circumference, blood pressure, lab values)
- `boolean` — yes/no (e.g. "Experienced nausea today?")
- `scale_1_10` — integer 1–10 (e.g. "Energy level", "Hunger level", "Sleep quality")

### 7.3 MetricEntry

A single logged value for any metric on any date.

| Field | Type | Notes |
|---|---|---|
| `id` | integer PK | |
| `user_id` | FK → User | |
| `metric_id` | FK → MetricDefinition | |
| `date` | date | |
| `value_numeric` | float | used for `numeric` and `scale_1_10` types |
| `value_boolean` | boolean | used for `boolean` type |
| `notes` | string | optional free text |
| `created_at` | datetime | |

One row per metric per date. If a user logs weight and waist on the same day, that is two `MetricEntry` rows with different `metric_id` values. Sparse by design — a metric only has entries on dates the user actually logged it.

### 7.4 InjectionEntry

| Field | Type | Notes |
|---|---|---|
| `id` | integer PK | |
| `user_id` | FK → User | |
| `peptide_id` | FK → Peptide | |
| `date` | date | |
| `dose` | float | in peptide's unit |
| `notes` | string | optional |
| `created_at` | datetime | |

### 7.5 ConcentrationSnapshot

Pre-computed concentration at each injection event, per peptide. Recalculated server-side at write time. Stored so the frontend never needs to compute decay math.

| Field | Type | Notes |
|---|---|---|
| `id` | integer PK | |
| `user_id` | FK → User | |
| `peptide_id` | FK → Peptide | |
| `date` | date | |
| `dose` | float | dose injected this date |
| `in_system` | float | estimated total concentration after this injection |

**Calculation logic:**
When a new `InjectionEntry` is saved, the server:
1. Fetches all prior snapshots for that peptide, ordered by date
2. Walks forward from the most recent snapshot to the new injection date, applying continuous exponential decay: `remaining = last_in_system × (0.5 ^ (days_elapsed / half_life_days))`
3. Adds the new dose: `in_system = remaining + new_dose`
4. Saves the new snapshot
5. Recalculates all future snapshots if the entry is backdated (full downstream recalculation)

---

## 8. API Endpoints

All endpoints require Bearer token authentication except where noted. All data is automatically scoped to the authenticated user.

### Auth
| Method | Path | Description |
|---|---|---|
| POST | `/auth/login` | Email + password → access token + refresh token |
| POST | `/auth/refresh` | Refresh token → new access token |
| POST | `/auth/logout` | Invalidate refresh token |

### Admin
| Method | Path | Description |
|---|---|---|
| GET | `/admin/users` | List all users (admin role required) |
| POST | `/admin/users` | Create a new user account (admin only) |
| PATCH | `/admin/users/{id}` | Update or deactivate a user (admin only) |

### Peptides
| Method | Path | Description |
|---|---|---|
| GET | `/peptides` | List user's active peptides |
| POST | `/peptides` | Create a peptide |
| PATCH | `/peptides/{id}` | Update name, half-life, color, unit |
| DELETE | `/peptides/{id}` | Archive a peptide |

### Metrics
| Method | Path | Description |
|---|---|---|
| GET | `/metrics` | List user's metric definitions (active + archived) |
| POST | `/metrics` | Create a new metric definition |
| PATCH | `/metrics/{id}` | Update name, unit, color, display order |
| DELETE | `/metrics/{id}` | Archive a metric (preserves all historical entries) |
| GET | `/metrics/{id}/entries` | List entries; accepts `?from=&to=` |
| POST | `/metrics/{id}/entries` | Log a value for a metric on a date |
| PATCH | `/metrics/{id}/entries/{entry_id}` | Edit an entry |
| DELETE | `/metrics/{id}/entries/{entry_id}` | Delete an entry |

### Injections
| Method | Path | Description |
|---|---|---|
| GET | `/injections` | List entries; accepts `?peptide_id=&from=&to=` |
| POST | `/injections` | Log an injection (triggers snapshot recalculation) |
| PATCH | `/injections/{id}` | Edit (triggers recalculation) |
| DELETE | `/injections/{id}` | Delete and recalculate downstream snapshots |

### Dashboard
| Method | Path | Description |
|---|---|---|
| GET | `/dashboard` | Returns metric series, concentration snapshots, weekly summary, recent entries. Accepts `?from=&to=&metrics=id,id&peptides=id,id` to control which series are included |

---

## 9. Frontend Screens

### 9.1 Login
- Email + password form
- "Sign in with Google" button (optional v1)
- No public registration link

### 9.2 Dashboard (Home)

**Stats Bar**
- One card per active metric: current value + change from last entry
- One card per active peptide: current dose + estimated in-system concentration

**Main Chart**
- X-axis: shared date axis
- One line per selected metric, each with its own independently scaled y-axis
- One line per selected peptide (in-system concentration), each with its own axis
- Dose shown as a dashed step line per peptide, sharing the peptide's concentration axis
- Dose change events marked with a labeled dot on the line
- **Axis scaling:** each series is auto-scaled to its own min/max range. A waist metric moving 28–36 inches and a weight metric moving 200–260 lbs both render with full vertical resolution — they are never forced onto a shared scale
- Each y-axis is color-coded to match its series
- Axes alternate left/right positioning; maximum 4 simultaneous axes before a warning is shown
- **Chart controls:**
  - Date range selector: 7d / 30d / 90d / 6mo / 1yr / All
  - Metric toggle checkboxes: show/hide any metric or peptide series
  - Default visible series persisted in localStorage per user

**Weekly Summary Panel**
- Rolling 4-week table
- Columns: week ending date → one column per tracked numeric metric (avg + delta) → one column per peptide (avg dose + avg in-system)
- Trend arrows (↑↓→) with color coding per metric per week

**Recent Entries Log**
- Unified chronological feed: metric entries and injection entries interleaved
- Each row: date, metric/peptide name, value + unit, notes snippet
- Inline edit and delete

### 9.3 Log Entry (Modal)
Floating "+" button opens a modal with two tabs:

**Metric Tab**
- Metric selector (dropdown of active metrics)
- Date picker (defaults to today)
- Value input — control adapts to data type:
  - `numeric` → number field with unit label
  - `scale_1_10` → segmented control 1–10
  - `boolean` → toggle switch
- Optional notes
- Save

**Injection Tab**
- Peptide selector
- Date picker (defaults to today)
- Dose input with unit label
- Optional notes
- Save

### 9.4 Metrics (Settings sub-page)

List of all metric definitions grouped: Built-in → Active → Archived.

For each: name, unit, data type badge, color swatch, entry count, edit and archive buttons, drag handle for display order.

**Create / Edit Metric form:**
- Name (text)
- Unit (free-form text — e.g. "lbs", "in", "mg/dL")
- Data type (select: Numeric / Scale 1–10 / Yes/No)
- Color picker

### 9.5 Peptides (Settings sub-page)

List of all peptides, active + archived.

**Create / Edit Peptide form:**
- Name
- Half-life in days
- Unit (select: mg / mcg / IU)
- Color picker

### 9.6 Account Settings
- Change display name, email, password
- Export all data as CSV (metric entries + injections + snapshots, one file each)
- Danger zone: delete all my data (preserves account)

### 9.7 Admin Panel *(admin role only)*
- User list: email, display name, created date, active status
- Create user: email, display name, temporary password
- Deactivate / reactivate toggle

---

## 10. Chart Overlay & Scaling Design

This is the core UI challenge. Key requirements and decisions:

**Problem:** Users can overlay metrics with completely incompatible scales simultaneously — body weight (200–260 lbs), waist (32–40 in), energy level (1–10), blood glucose (80–140 mg/dL), and peptide concentration (0–10 mg) may all be on the chart at once.

**Solution: Per-series independent y-axes**
- Each visible series gets its own y-axis, auto-scaled to that series' min/max with 10% padding
- Axes are positioned alternating left/right (left: weight, waist; right: peptide 1, energy)
- Chart.js supports up to N y-axes natively via the `scales` config; each dataset references its axis by ID
- Chart components live in `apps/web/components/charts/` and are all `"use client"` since Chart.js requires browser APIs
- Maximum 4 axes displayed simultaneously; a soft warning appears if the user tries to enable more ("Hide a metric to add another")

**Axis presentation:**
- Each axis tick label is colored to match its series line
- Unit appended to axis label (e.g. "lbs", "in", "mg")
- Grid lines only drawn for the first left-side axis to avoid visual noise

**Sparse data handling:**
- Metrics are not logged every day. Chart.js `spanGaps: true` connects dots across missing dates for trend visibility
- A toggle allows the user to switch to "dots only" mode to see actual logged dates without the interpolated line

---

## 11. PWA Requirements

- `manifest.json`: app name, icons (192px, 512px), `display: standalone`, `theme_color`
- Service worker: cache-first for static assets; network-first for API calls
- Offline: cached dashboard readable; log forms show offline warning
- iOS Safari: manual "Add to Home Screen" prompt shown on first visit

---

## 12. Deployment

### Frontend (Vercel)
- `apps/web` in the NX monorepo, auto-deploys from `main`
- Vercel NX integration detects the Next.js app automatically
- Env var: `NEXT_PUBLIC_API_BASE_URL`
- Next.js server actions and server components call the API directly server-to-server where possible, avoiding unnecessary client round-trips

### Backend (Fly.io)
- `apps/api` in the NX monorepo
- Single machine, never auto-scaled (SQLite consistency)
- Persistent volume at `/data` (1GB minimum)
- Litestream sidecar in the same container: continuous replication to Fly Tigris
- Fly secrets: `JWT_SECRET`, `LITESTREAM_ACCESS_KEY_ID`, `LITESTREAM_SECRET_ACCESS_KEY`
- Health check: `GET /health`

### Database Migrations
- Managed with **Drizzle Kit** (`drizzle-kit push` for dev, `drizzle-kit migrate` for production)
- Migration files committed to repo under `apps/api/drizzle/`
- Migrations run automatically on container start via an entrypoint script before NestJS boots

---

## 13. Repo Structure

```
peptide-tracker/                      # NX monorepo root
├── apps/
│   ├── web/                          # Next.js 15 App Router
│   │   ├── app/
│   │   │   ├── (auth)/
│   │   │   │   └── login/page.tsx
│   │   │   ├── (app)/
│   │   │   │   ├── layout.tsx        # Auth-gated shell
│   │   │   │   ├── dashboard/page.tsx
│   │   │   │   ├── settings/
│   │   │   │   │   ├── metrics/page.tsx
│   │   │   │   │   ├── peptides/page.tsx
│   │   │   │   │   └── account/page.tsx
│   │   │   │   └── admin/page.tsx
│   │   │   └── api/                  # Minimal — auth cookie handling only
│   │   ├── components/
│   │   │   ├── charts/               # "use client" Chart.js wrappers
│   │   │   ├── forms/                # "use client" log entry forms
│   │   │   └── ui/                   # Shared UI primitives
│   │   ├── lib/
│   │   │   └── api-client.ts         # Typed fetch wrapper (Bearer token injection)
│   │   ├── public/
│   │   │   ├── manifest.json
│   │   │   └── sw.js
│   │   └── next.config.ts
│   │
│   └── api/                          # NestJS + Fastify
│       ├── src/
│       │   ├── main.ts               # Fastify adapter bootstrap
│       │   ├── app.module.ts
│       │   ├── auth/
│       │   │   ├── auth.module.ts
│       │   │   ├── auth.service.ts   # Login, token issuance, bcrypt
│       │   │   ├── auth.controller.ts
│       │   │   ├── jwt.strategy.ts   # Passport JWT strategy
│       │   │   ├── jwt-auth.guard.ts # Global guard
│       │   │   └── public.decorator.ts
│       │   ├── users/
│       │   ├── peptides/
│       │   ├── metrics/
│       │   ├── injections/
│       │   ├── dashboard/
│       │   └── admin/
│       ├── drizzle/
│       │   ├── schema.ts             # Drizzle table definitions
│       │   └── migrations/
│       ├── drizzle.config.ts
│       ├── Dockerfile
│       └── fly.toml
│
├── libs/
│   ├── types/                        # @tracker/types
│   │   └── src/
│   │       ├── metric.ts
│   │       ├── peptide.ts
│   │       ├── injection.ts
│   │       └── index.ts
│   └── calculations/                 # @tracker/calculations
│       └── src/
│           ├── half-life.ts          # Decay math, pure functions
│           ├── half-life.spec.ts
│           └── index.ts
│
├── nx.json
├── tsconfig.base.json
└── package.json
```

---

## 14. Open Questions / Future Scope

- **Data import endpoint:** A `POST /import` endpoint for bulk-loading historical CSVs — needed for migrating the existing retatrutide + weight data
- **Notifications:** iOS PWA push is limited; could use email reminders for missed injection days — v2
- **Axis pinning:** Let users manually lock y-axis min/max per metric for consistent cross-date-range comparison — v2
- **Metric templates:** One-click "add common metric" suggestions (Sleep Quality, Nausea, Blood Pressure, etc.) — v2
- **Correlation view:** Scatter plot of any metric A vs. metric B over a shared time window — v2
- **Google OAuth:** Skip for v1; straightforward to add later
- **Litestream:** Included from day one — too important for a single-file SQLite deployment to defer

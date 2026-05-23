# Implementation Plan — Health Tracker

You are implementing the Health Tracker app: a multi-user Progressive Web App for tracking supplement dosing and arbitrary health metrics over time.

Before writing any code:
- Read `CONTEXT.md` for canonical domain language. Use it everywhere — issue titles, variable names, API paths, comments.
- Read `docs/adr/` for architectural decisions that must not be reversed.
- Fetch the full issue before starting a slice: `gh issue view <number> --comments`.

The monorepo structure:

```
apps/web        Next.js 15 App Router → Vercel
apps/api        NestJS + Fastify → Fly.io (single machine, never auto-scaled — ADR-0003)
libs/types      @tracker/types   — shared TypeScript interfaces + Zod schemas
libs/calculations  @tracker/calculations — half-life decay math, pure functions
```

Full spec: `peptide-tracker-prd.md`. Work through the phases below in order. Within each phase, listed issues can be implemented in parallel.

---

## Phase 1 — Foundation

**Blocked by: nothing. Human required.**

### #1 Monorepo scaffold + infra _(HITL)_

A human must provision external services before this phase can be verified:

- Fly.io machine + persistent volume (≥1GB mounted at `/data`)
- Vercel project linked to `apps/web`
- Fly Tigris bucket for Litestream replication
- Secrets: `JWT_SECRET`, `LITESTREAM_ACCESS_KEY_ID`, `LITESTREAM_SECRET_ACCESS_KEY`, `NEXT_PUBLIC_API_BASE_URL`

The agent's job: scaffold the NX monorepo, write all config (fly.toml, Dockerfile, next.config.ts, drizzle.config.ts, nx.json, tsconfig.base.json), implement `GET /health`, create the Next.js placeholder page, configure Drizzle with the migration entrypoint script, and stub `@tracker/types` and `@tracker/calculations` with empty exports.

---

## Phase 2 — Auth gate

**Blocked by: #1. Sequential — must complete before anything else.**

### #2 Auth — seed CLI + login flow

Implements the `users` and `refresh_tokens` Drizzle schemas, the `npm run seed:admin` CLI, `POST /auth/login|refresh|logout`, the global JWT guard, `@Public()` decorator, and the Next.js login page. Every subsequent phase depends on authenticated requests — nothing else can land until this merges.

---

## Phase 3 — Parallel workstreams

**Blocked by: #2. All four can start the moment #2 merges.**

Assign to separate agents or implement in any order — no dependencies between them.

### #3 Admin — user management
Admin role guard, `GET/POST/PATCH /admin/users`, Next.js admin panel. `POST /admin/users` must seed the built-in Weight Metric for each new user (coordinate with #6 schema if not yet merged — stub the seeding call and wire it up once #6 lands).

### #4 Supplements CRUD
`supplements` Drizzle schema, `GET/POST/PATCH/DELETE /supplements` (returns active + archived by default; `?archived=false` to filter), Next.js Supplements settings page. The `PATCH` endpoint should return 422 if `half_life_days` is changed while Dose Logs exist — full recalculation is wired in #5.

### #6 Metrics CRUD
`metric_definitions` Drizzle schema — `data_type` is `numeric|boolean` only; `numeric` metrics have optional `min`/`max` (no `scale_1_10` type — ADR-0002). Built-in Weight Metric seeded on user creation. `GET/POST/PATCH/DELETE /metrics`, Next.js Metrics settings page (Built-in / Active / Archived groups, drag-reorder).

### #12 Account settings + data export
`PATCH /users/me`, `GET /export/metric-logs|dose-logs|snapshots` (CSV downloads), `DELETE /users/me/data`. Next.js Account Settings page with profile update, export buttons, and danger zone.

---

## Phase 4 — Core tracking

**#5 blocked by #4. #7 blocked by #6. They don't block each other — start each as soon as its Phase 3 blocker merges.**

### #5 Dose logging + Active Dose Snapshots
Implement `@tracker/calculations` first — `decayAmount` and `recalculateSnapshots` pure functions with full unit tests. Then `dose_logs` + `active_dose_snapshots` Drizzle schema, `GET/POST/PATCH/DELETE /dose-logs` with snapshot recalculation pipeline. Backdated inserts and deletes must trigger downstream recalculation. Half-life edit on `PATCH /supplements/{id}` triggers **full** recalculation from the first Dose Log (ADR-0001); frontend shows a confirmation warning before submitting. Log Entry modal — Dose tab.

### #7 Metric logging
`metric_logs` Drizzle schema, `GET/POST/PATCH/DELETE /metrics/{id}/logs`. Value input in the Log Entry modal adapts to data type: number field for numeric, bounded input or segmented control if min/max set, toggle for boolean. Validation enforces min/max on the API.

---

## Phase 5 — Data + dashboard foundation

**Blocked by: #5 AND #7 (both must be merged). Start both simultaneously once both Phase 4 issues are done.**

### #8 Data import — CSV bulk-load
`POST /import` — multipart CSV upload for `dose_logs.csv` and `metric_logs.csv`. Matches supplements and metrics by name; creates missing ones. Triggers full Active Dose Snapshot recalculation for affected supplements after import. Implement this before the dashboard so real historical data is present for testing visualization.

### #9 Dashboard — stats bar + recent log
`GET /dashboard` (stats bar + recent log sections). Dashboard page: auth-gated app shell, stats bar (one card per active metric with current value + delta; one card per active supplement with last dose + current Active Dose), unified chronological recent log feed with inline edit/delete.

---

## Phase 6 — Dashboard visualization

**Blocked by: #9. Both can start the moment #9 merges — they extend the same endpoint independently.**

### #10 Dashboard — main chart
Extend `GET /dashboard` with `metricSeries` and `supplementSeries` time-series data. Chart.js multi-axis chart (`"use client"`): one auto-scaled y-axis per visible series, alternating left/right, color-coded to series. Supplement Active Dose as solid line; dose as dashed step line with a dot marker at every injection. Date range selector (7d / 30d / 90d / 6mo / 1yr / All). Series toggle checkboxes — archived series in a separate group, off by default. `spanGaps: true` default with a "dots only" toggle. Soft warning at 5th axis. Default visible series persisted in localStorage.

### #11 Dashboard — weekly summary
Extend `GET /dashboard` with `weeklySummary`: rolling 4 weeks, per-metric average + delta, per-supplement average dose + average Active Dose, trend direction. Render as a table below the chart with color-coded trend arrows (↑ green / ↓ red / → grey).

---

## Phase 7 — PWA

**Blocked by: #10 AND #11 (both must be merged).**

### #13 PWA — manifest, service worker, offline support
`public/manifest.json` (standalone display, icons 192px + 512px). Service worker: cache-first for static assets, network-first for API calls. Offline: cached dashboard readable, offline banner on Log Entry modal. iOS "Add to Home Screen" dismissible instructions banner on first visit.

---

## Standing rules for every phase

- **Domain language**: names in code must match `CONTEXT.md`. Use Supplement, Dose Log, Active Dose Snapshot, Metric, Metric Log. Never "peptide", "injection", "concentration", "MetricDefinition", or "InjectionEntry".
- **User scoping**: every API query must filter by `req.user.id`. There is no cross-user data access.
- **Single machine**: the API must never be configured to auto-scale (ADR-0003). SQLite lives on a single Fly.io volume.
- **Recalculation**: any write to `dose_logs` (insert, update, delete) or a `half_life_days` change must recalculate Active Dose Snapshots. The recalculation is synchronous at write time (ADR-0001).
- **No scale_1_10**: this data type does not exist. Bounded numeric scales use `data_type = numeric` with `min`/`max` (ADR-0002).
- **Fetch the issue**: always run `gh issue view <number> --comments` for full acceptance criteria before starting a slice.

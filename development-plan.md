# Personal Trainer Management — Phased Development Plan

> Project: 334-personal-trainer-management · Created: 2026-05-30
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and the three `data-model-suggestion-*.md` files. The database schema is anchored on **Data Model Suggestion 1 (Entity-Centric Normalized Relational)** because the project targets the underserved mid-market team segment (3–20 trainers) with shared client rosters and AI-driven cross-client analytics — exactly the use case that document recommends. JSONB is used selectively (assessments, medical intake, wearable raw payloads) per the hybrid insights from Suggestion 2, and an append-only `audit_log` plus event-shaped wearable/exercise logging captures the timeline benefits highlighted in Suggestion 3 without paying the full event-sourcing complexity tax.

---

## Core Requirements (synthesis)

- **What it does**: An AI-native, open-source platform for independent personal trainers and small training teams to deliver workout programmes, track client progress and biometrics, schedule sessions, message clients, and run billing — with AI treated as a continuous adaptation loop rather than a one-shot draft generator.
- **Who uses it**: (1) Independent personal trainers, (2) small training teams of 3–20 trainers (the underserved mid-market), (3) clients consuming programmes via a mobile/web portal.
- **Key differentiators**: genuine adaptive programming from wearable + logged data; normalised cross-vendor wearable layer; LLM check-in sentiment analysis; AI weekly progress narratives; churn prediction; natural-language business co-pilot (MCP server). Open-source — no comparable OSS incumbent exists.
- **Deployment model**: Self-hostable web application (Docker Compose) with a JSON REST API documented in OpenAPI 3.1, a trainer dashboard SPA, and a client portal (responsive web first; native mobile via the same API later).
- **Integration surface**: Stripe (billing), Terra API (wearable aggregation + webhooks), Apple HealthKit / Health Connect (native relay later), Google Calendar, LLM provider (Anthropic Claude), MCP server for the co-pilot.
- **Standards compliance**: OpenAPI 3.1, OAuth 2.0 (RFC 6749/6750) + JWT (RFC 7519) for auth, OpenID Connect for SSO, RFC 8288 pagination, PCI DSS v4.0 (tokens only, never PAN), GDPR/HIPAA-aware data handling, Open mHealth / IEEE P1752 for wearable normalisation, HL7 FHIR R4 export (Patient/Observation, Exercise Vital Sign LOINC 89574-8), OWASP Top 10, WCAG 2.2.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Language (backend) | TypeScript (Node 22 LTS) | The product is integration- and API-heavy (Stripe, Terra, Calendar, webhooks, an SPA, and an MCP server). One language across backend, frontend, and the MCP server reduces context-switching and shares DTO/Zod types end-to-end. |
| API framework | Fastify 5 + `@fastify/swagger` | High-throughput, first-class JSON Schema → auto-generated OpenAPI 3.1 (a launch requirement for Zapier/Make/n8n community connectors). Plugin model isolates integrations cleanly. |
| Validation / typing | Zod + `zod-to-openapi` | Single source of truth for runtime validation and the OpenAPI schema; shared with the frontend. |
| ORM / DB access | Drizzle ORM | Type-safe SQL close to the normalized schema in Data Model 1; migration tooling; no heavy runtime. Raw SQL escape hatch for time-series wearable queries. |
| Database | PostgreSQL 16 | Data Model 1 relies on UUIDs, `JSONB`, `TEXT[]`, GIN indexes, partial indexes, and a partitioned `audit_log` — all native to Postgres. Multi-tenant row isolation via `team_id` + Row-Level Security. |
| Time-series | Postgres + monthly partitions on `wearable_data` | Avoids a second datastore for the MVP; wearable volume is moderate per client and partitioning keeps trend queries fast. |
| Cache / queue broker | Redis 7 + BullMQ | Async work: Terra webhook ingestion, Stripe webhook processing, reminder dispatch, LLM calls (adaptation, narratives, churn scoring), recurring billing. Decouples slow/external work from request latency. |
| LLM provider | Anthropic Claude via official SDK, with prompt caching | All AI features (adaptive prescription, sentiment, narratives, churn, co-pilot). Prompt caching cuts cost on the large system prompts that carry coaching guardrails (NASM OPT / ACE scope of practice). |
| AI co-pilot transport | MCP server (`@modelcontextprotocol/sdk`) | Per `standards.md`, expose client/programme/wearable/billing as MCP resources + tools so the business co-pilot queries real context without bespoke prompt plumbing. |
| Frontend (trainer + client) | React 19 + Vite + TanStack Query + Tailwind + shadcn/ui | SPA trainer dashboard and responsive client portal share the typed API client. shadcn/ui gives accessible (WCAG 2.2) primitives fast. |
| Payments | Stripe (Customers, PaymentMethods, Subscriptions, Invoices, Webhooks) | Industry-standard; PCI DSS scope minimised by storing only Stripe tokens (`payment_methods.processor_token`). |
| Wearables | Terra API (primary) + native HealthKit/Health Connect relay (later) | Terra normalises 500+ providers via one webhook integration and is GDPR/HIPAA/SOC2 compliant — the lowest-friction path to the "normalised wearable layer" differentiator. |
| Auth | OAuth 2.0 / OIDC, JWT access + rotating refresh tokens; `@fastify/jwt`; argon2 password hashing | RFC 6749/6750/7519 + OIDC from `standards.md`; supports SSO for multi-business trainers and white-label apps. |
| File storage | S3-compatible (MinIO in dev, S3/R2 in prod) | Progress photos, exercise videos, meal-plan docs, generated PDF reports. Signed URLs; never public buckets. |
| Email / SMS / push | Provider-abstracted `Notifier` interface (Resend/SendGrid, Twilio, web push) | Reminders, check-in sequences, welcome flows — pluggable for self-hosters. |
| Testing | Vitest (unit/integration) + Supertest (HTTP) + Playwright (E2E) + Testcontainers (real Postgres/Redis) | Vitest matches the Vite/TS stack; Testcontainers gives real-DB integration tests; Playwright covers the two web surfaces. |
| Code quality | ESLint + Prettier + `tsc --noEmit` + Biome (optional) | Enforced in CI; type checking is a Definition-of-Done gate. |
| Package manager / monorepo | pnpm workspaces + Turborepo | Shared packages (`@ptm/db`, `@ptm/core`, `@ptm/sdk`) consumed by api, web, mcp. |
| Containerisation | Docker + docker-compose (api, worker, web, postgres, redis, minio, mcp) | Self-hosting is a stated deployment mode; one `docker compose up` brings up the full stack. |
| Observability | pino (structured logs) + OpenTelemetry traces + `/healthz` | Operational baseline for a multi-tenant SaaS. |

### Project Structure

```
personal-trainer-management/
├── package.json                 # pnpm workspace root
├── pnpm-workspace.yaml
├── turbo.json
├── docker-compose.yml
├── Dockerfile.api
├── Dockerfile.worker
├── Dockerfile.web
├── Dockerfile.mcp
├── .env.example
├── packages/
│   ├── db/                      # Drizzle schema, migrations, seed, RLS policies
│   │   ├── src/schema/          # one file per domain (teams, clients, programmes, ...)
│   │   ├── src/migrations/
│   │   ├── src/rls.ts
│   │   └── src/seed.ts
│   ├── core/                    # domain logic, framework-agnostic
│   │   ├── src/programmes/      # template→client copy, compliance scoring
│   │   ├── src/billing/         # invoice/subscription state machines
│   │   ├── src/scheduling/      # availability, conflict detection
│   │   ├── src/wearables/       # Open mHealth normalisation, FHIR export
│   │   ├── src/ai/              # prompt templates, adaptation, narratives, churn, sentiment
│   │   └── src/notify/          # Notifier interface + providers
│   ├── sdk/                     # generated TS API client (from OpenAPI) for web
│   └── config/                  # shared eslint/tsconfig/zod helpers
├── apps/
│   ├── api/                     # Fastify HTTP API
│   │   ├── src/plugins/         # auth, db, redis, swagger, error-handler, rls-context
│   │   ├── src/routes/          # /auth /clients /exercises /programmes /sessions /billing ...
│   │   ├── src/integrations/    # stripe, terra, google-calendar webhook handlers
│   │   └── src/app.ts
│   ├── worker/                  # BullMQ processors (jobs/)
│   │   └── src/jobs/            # terra-ingest, stripe-events, reminders, ai-*, billing-cycle
│   ├── web/                     # React SPA (trainer dashboard + client portal)
│   │   └── src/{trainer,client,shared}/
│   └── mcp/                     # MCP server exposing PTM resources/tools to the co-pilot
└── tests/
    ├── fixtures/                # seed JSON, sample Terra/Stripe payloads, FIT/Open mHealth samples
    └── e2e/                     # Playwright specs
```

---

## Phase 1: Foundation & Multi-Tenant Data Layer

### Purpose
Establish the monorepo, the PostgreSQL schema from Data Model 1, multi-tenant isolation via Row-Level Security, and the Dockerised dev environment. After this phase the full schema migrates cleanly, RLS prevents cross-team data leaks, and `docker compose up` yields a running (empty) API with `/healthz` and an auto-generated OpenAPI document.

### Tasks

#### 1.1 — Monorepo & toolchain bootstrap
**What**: Initialise the pnpm/Turborepo workspace, shared tsconfig/eslint, and the Docker Compose stack.

**Design**:
- `pnpm-workspace.yaml` lists `packages/*` and `apps/*`.
- `packages/config` exports `tsconfig.base.json` (strict, `noUncheckedIndexedAccess`), shared ESLint flat config, and a `z` re-export.
- `docker-compose.yml` services: `postgres` (16, healthcheck `pg_isready`), `redis` (7), `minio` (+ `createbuckets` init), `api`, `worker`, `web`, `mcp`. Named volumes for pg/minio data.
- `.env.example` documents every variable: `DATABASE_URL`, `REDIS_URL`, `JWT_SECRET`, `JWT_REFRESH_SECRET`, `S3_ENDPOINT`/`S3_BUCKET`/`S3_KEY`/`S3_SECRET`, `STRIPE_SECRET_KEY`/`STRIPE_WEBHOOK_SECRET`, `TERRA_API_KEY`/`TERRA_DEV_ID`/`TERRA_SIGNING_SECRET`, `ANTHROPIC_API_KEY`, `ANTHROPIC_MODEL` (default `claude-sonnet-4-5`), `APP_BASE_URL`, `DEFAULT_TIMEZONE`, `DEFAULT_CURRENCY`.

**Testing**:
- `Unit: env loader with all required vars present → typed Config object`.
- `Unit: env loader missing JWT_SECRET → throws with "JWT_SECRET" in message`.
- `Integration: docker compose up → postgres, redis, minio reach healthy state` (smoke script, marked real/optional).

#### 1.2 — Database schema & migrations (`packages/db`)
**What**: Encode the 22 tables from Data Model 1 as Drizzle schema with all enums, constraints, and indexes.

**Design**:
- One schema file per domain: `teams.ts`, `clients.ts`, `exercises.ts`, `programmes.ts` (programmes/workouts/workout_exercises), `logging.ts` (exercise_logs/workout_completions), `progress.ts` (progress_entries/wearable_data/assessments), `nutrition.ts`, `sessions.ts`, `billing.ts` (packages/client_packages/payment_methods/invoices), `messaging.ts`, `integrations.ts` (integrations/audit_log).
- Mirror the DDL exactly: UUID PKs `default gen_random_uuid()`, `CHECK` constraints as Drizzle enums/`check()`, `TEXT[]` columns, `JSONB` defaults, and every named index including partial indexes (`idx_clients_churn WHERE status='active'`, `idx_client_packages_billing WHERE status='active'`, `idx_exercise_logs_pr WHERE is_personal_record`).
- `wearable_data` declared `PARTITION BY RANGE (recorded_at)`; migration creates monthly partitions + a `create_wearable_partition(month)` helper. `audit_log` partitioned `BY RANGE (ce_time)`.
- New columns beyond Data Model 1 needed for auth: add `trainers.password_hash TEXT`, `trainers.refresh_token_family UUID`, and a `client_portal_credentials` table (`client_id`, `password_hash`, `last_login_at`) — clients authenticate separately from trainers.
- `pnpm db:migrate`, `pnpm db:seed`, `pnpm db:reset` scripts.

**Testing**:
- `Integration (Testcontainers): run all migrations on fresh Postgres → 24 tables + expected indexes exist` (query `pg_indexes`).
- `Integration: insert trainer with role='wizard' → CHECK violation`.
- `Integration: insert client referencing non-existent team_id → FK violation`.
- `Integration: insert wearable_data for current month → lands in correct partition`.

#### 1.3 — Row-Level Security & tenant context
**What**: RLS policies isolating every team-scoped table, plus a request-scoped Postgres session setting carrying the caller's `team_id`.

**Design**:
- `ALTER TABLE ... ENABLE ROW LEVEL SECURITY` on all tables with a `team_id`.
- Policy template: `USING (team_id = current_setting('app.current_team')::uuid)`. Tables without direct `team_id` (e.g. `workouts`, `workout_exercises`, `exercise_logs`) get policies that join to the owning `programme`/`client`.
- API DB plugin runs `SET LOCAL app.current_team = $teamId` at the start of each authenticated transaction.
- A privileged `migrator` role bypasses RLS for migrations/seeds.

**Testing**:
- `Integration: session set to team A, SELECT clients → only team A rows`.
- `Integration: team A context, attempt UPDATE of team B client by id → 0 rows affected`.
- `Integration: no app.current_team set → SELECT returns 0 rows (fail-closed)`.

#### 1.4 — Fastify app skeleton, error model & OpenAPI
**What**: Bootable API with health checks, structured logging, a uniform error envelope, and auto-generated OpenAPI 3.1.

**Design**:
- Plugins: `db` (Drizzle + pool), `redis`, `swagger` (`@fastify/swagger` + `@fastify/swagger-ui` at `/docs`, spec at `/openapi.json`), `error-handler`, `pino` logger with request IDs.
- Error envelope: `{ error: { code: string, message: string, details?: unknown, requestId: string } }`. Codes: `VALIDATION_ERROR` (400), `UNAUTHORIZED` (401), `FORBIDDEN` (403), `NOT_FOUND` (404), `CONFLICT` (409), `RATE_LIMITED` (429), `INTERNAL` (500). Maps OWASP A01/A03 concerns to consistent, non-leaky responses.
- Pagination helper emits RFC 8288 `Link` headers (`rel="next"`/`"prev"`) and a `{ data, page: { cursor, limit, total? } }` body for list endpoints.
- `/healthz` (liveness) and `/readyz` (checks DB + Redis).

**Testing**:
- `Integration: GET /healthz → 200 {status:"ok"}`.
- `Integration: GET /readyz with DB down → 503`.
- `Unit: error handler given ZodError → 400 VALIDATION_ERROR with field paths in details`.
- `Integration: GET /openapi.json → valid OpenAPI 3.1 document (validated against schema)`.

---

## Phase 2: Identity, Teams & Access Control

### Purpose
Trainers can register a team, authenticate, and manage trainer accounts with roles; clients get separate portal credentials. This phase makes every later endpoint authorisable and underpins the multi-tenant model. Implements OAuth2/JWT (RFC 6749/6750/7519) and OWASP A01 (broken access control) controls.

### Tasks

#### 2.1 — Trainer auth (register, login, refresh, logout)
**What**: Email/password auth issuing JWT access tokens and rotating refresh tokens.

**Design**:
- Endpoints:
  - `POST /auth/register` `{ teamName, fullName, email, password, timezone?, currency? }` → creates `teams` row + `trainers` row with `role='owner'`, returns `{ accessToken, refreshToken, trainer }`.
  - `POST /auth/login` `{ email, password }` → tokens.
  - `POST /auth/refresh` `{ refreshToken }` → new pair; rotates `refresh_token_family`, detects reuse → revokes family (401).
  - `POST /auth/logout` → revokes current family.
- Access JWT claims: `{ sub: trainerId, team: teamId, role, type:'trainer', exp }`, 15 min TTL. Refresh: opaque, hashed at rest, 30-day TTL.
- argon2id hashing (memory 64MB, t=3). Rate-limit login to 5/min/IP (`@fastify/rate-limit` backed by Redis) — OWASP brute-force mitigation.

**Testing**:
- `Integration: register → team+owner created, valid JWT returned`.
- `Integration: login wrong password → 401, no token`.
- `Integration: refresh with rotated (reused) token → 401 and family revoked`.
- `Unit: argon2 verify of correct/incorrect password → true/false`.
- `Integration: 6th login attempt in a minute → 429`.

#### 2.2 — Trainer management & RBAC
**What**: Owners/head trainers invite, list, update, and deactivate trainers; role-based route guards.

**Design**:
- `POST /trainers` (invite: creates trainer, emails set-password link), `GET /trainers`, `PATCH /trainers/:id`, `DELETE /trainers/:id` (soft: `is_active=false`).
- `requireRole(...roles)` preHandler. Permission matrix: `owner` = all; `head_trainer` = manage trainers (except owner) + all clients; `trainer` = own clients only; `intern` = read own clients; `admin` = billing + scheduling, no programme edits.
- Client ownership enforced via `clients.trainer_id` in addition to RLS team scope.

**Testing**:
- `Integration (role=trainer): POST /trainers → 403 FORBIDDEN`.
- `Integration (role=owner): POST /trainers → 201, invite email queued`.
- `Integration (role=trainer): GET /clients → only clients where trainer_id = self`.
- `Unit: requireRole('owner') with role='intern' → throws FORBIDDEN`.

#### 2.3 — Client portal authentication
**What**: Separate credential store and token type for clients accessing their portal.

**Design**:
- `POST /portal/auth/login` `{ email, password }` against `client_portal_credentials`; JWT `type:'client'`, claim `client: clientId`, `team: teamId`.
- Client tokens authorise only `/portal/*` routes; a guard rejects `type:'client'` on trainer routes and vice-versa.
- Onboarding sets the password via a signed, single-use link (Phase 6 welcome sequence reuses this).

**Testing**:
- `Integration: client token on GET /clients (trainer route) → 403`.
- `Integration: trainer token on GET /portal/me → 403`.
- `Integration: client login → token scoped to own client_id only`.

---

## Phase 3: Exercise Library & Programme Builder (Core Value — Part 1)

### Purpose
The heart of the product: a per-team exercise library (system-seeded + custom uploads) and a programme builder modelling workouts, ordered exercises, supersets/circuits/drop-sets, and full prescriptions (sets, reps, weight, %1RM, RPE, tempo, rest). Templates can be authored once and assigned to many clients. After this phase trainers can build and assign real programmes.

### Tasks

#### 3.1 — Exercise library CRUD + media
**What**: Manage exercises with categories, muscle groups, equipment, coaching cues, and video/thumbnail uploads.

**Design**:
- `GET /exercises` (filters: `category`, `muscle_group`, `equipment`, `q` full-text on name; uses GIN indexes), `POST /exercises`, `PATCH /exercises/:id`, `DELETE /exercises/:id` (block delete if referenced by a `workout_exercises` row → 409).
- Media: `POST /exercises/:id/media` returns a presigned S3 PUT URL; client uploads directly; callback stores `video_path`/`thumbnail_path`.
- System exercises (`is_system=true`, seeded ~150 common movements with cues) are read-only to trainers but copyable.

**Testing**:
- `Integration: POST exercise with category='strength' → 201`.
- `Integration: POST with category='dance' → 400 VALIDATION_ERROR`.
- `Integration: GET /exercises?muscle_group=chest → only chest exercises`.
- `Integration: DELETE exercise used in a workout → 409 CONFLICT`.
- `Integration: POST media → presigned URL with correct content-type constraint`.

#### 3.2 — Programme & workout structure
**What**: CRUD for programmes, their workouts, and ordered workout-exercises with grouping and prescriptions.

**Design**:
- `POST /programmes` `{ name, programmeType, clientId?, durationWeeks, isTemplate }` (clientId null + isTemplate true = reusable template).
- Nested authoring: `POST /programmes/:id/workouts`, `POST /workouts/:id/exercises` with body matching `workout_exercises` (group_type, group_id, sets, reps, weight_kg, weight_pct_1rm, rpe, tempo `"3-1-2-0"`, rest_seconds, duration_seconds, distance_metres, sort_order).
- Reorder endpoint `PATCH /workouts/:id/exercises/order` `{ orderedIds: uuid[] }`.
- Validation rules (in `packages/core/programmes`): exercises sharing a `group_id` must share `group_type` and be contiguous in `sort_order`; `reps` accepts `"8-12" | "AMRAP" | "30s"`; tempo matches `/^\d+-\d+-\d+-\d+$/`. Aligns programme structure with NASM OPT phasing via `programme_type`.

**Testing**:
- `Unit: superset with mismatched group_type members → validation error`.
- `Unit: tempo "3-1-2" → validation error; "3-1-2-0" → valid`.
- `Integration: create programme → add workout → add 3 exercises → GET returns nested tree in sort_order`.
- `Integration: reorder exercises → persisted sort_order matches request`.

#### 3.3 — Template → client assignment (deep copy)
**What**: Assign a template to a client by deep-copying programme → workouts → workout_exercises and binding to the client.

**Design**:
- `POST /programmes/:templateId/assign` `{ clientId, startDate }` → transactional deep copy: new `programmes` row (`client_id` set, `is_template=false`, `status='active'`), copied workouts and workout_exercises with new UUIDs, preserving structure and prescriptions. Generates `workout_completions` rows (`status='scheduled'`) per `scheduled_date` derived from `start_date` + `week_number`/`day_number`.
- Pure copy function `assignTemplate(template, clientId, startDate)` in `packages/core` for unit testing without a DB.

**Testing**:
- `Unit: assignTemplate copies N workouts and M exercises with fresh ids, identical prescriptions`.
- `Integration: assign template → client programme + scheduled workout_completions created; template row unchanged`.
- `Integration: assign to client in another team → 404 (RLS hides template/client mismatch)`.

---

## Phase 4: Client Management, Logging & Compliance (Core Value — Part 2)

### Purpose
Trainers manage client records (profile, goals, medical intake, consent) and clients log workouts and progress. Compliance scoring (prescribed vs. actual) and personal-record detection make the prescription-execution loop measurable — the data foundation every AI feature later consumes.

### Tasks

#### 4.1 — Client CRUD, lifecycle & consent
**What**: Manage clients with status lifecycle, goals, medical intake, and GDPR consent fields.

**Design**:
- `POST /clients`, `GET /clients` (filters: `status`, `tag`, `trainerId`), `GET /clients/:id`, `PATCH /clients/:id`.
- Status state machine: `lead → onboarding → active → paused → churned → archived` (and `active ↔ paused`); transitions validated; illegal transition → 409.
- `medical_intake` JSONB validated by a Zod schema `{ injuries[], medications[], conditions[], physicianClearance: boolean, parQCompleted: boolean }` (PAR-Q; supports ACE scope-of-practice boundaries).
- GDPR: `POST /clients/:id/consent` records `consent_status`, `consent_given_at`; `DELETE /clients/:id?mode=erase` performs right-to-erasure (anonymise PII, retain aggregate metrics) vs `mode=archive`.

**Testing**:
- `Unit: status transition churned→active → rejected; paused→active → allowed`.
- `Integration: POST client → 201 with status='lead'`.
- `Integration: erase client → PII nulled, exercise_logs retained but de-identified`.
- `Unit: medical_intake missing parQCompleted → validation error`.

#### 4.2 — Workout logging & completion
**What**: Clients (portal) and trainers record per-set performance and workout-level completion.

**Design**:
- `POST /portal/workout-completions/:id/start` → `status='in_progress'`.
- `POST /portal/workout-completions/:id/logs` body = array of `exercise_logs` rows (set_number, reps_completed, weight_kg, rpe_actual, duration/distance).
- `POST /portal/workout-completions/:id/finish` `{ duration_minutes, mood_rating, energy_rating, client_notes }` → computes `compliance_pct` and sets `status` (`completed` if ≥ threshold else `partial`).
- `computeCompliance(prescribed, logged)` in `packages/core`: `min(1, sets_logged / sets_prescribed)` averaged across the workout's exercises → percentage.

**Testing**:
- `Unit: computeCompliance, 3/4 exercises fully logged → 75%`.
- `Integration: finish workout with all sets logged → status='completed', compliance=100`.
- `Integration (client token): start another client's workout → 403`.

#### 4.3 — Personal-record detection
**What**: On each new log, detect and flag PRs against the client's history for that exercise.

**Design**:
- `detectPR(exerciseId, clientId, newLog, history)` → returns `{ isPR, prType }` where `prType ∈ {weight, reps, volume, time, distance}`. Volume = `reps × weight`; time/distance per exercise category.
- Runs synchronously on log insert; sets `exercise_logs.is_personal_record` and `pr_type`. Surfaced via `GET /clients/:id/personal-records` (uses `idx_exercise_logs_pr`).

**Testing**:
- `Unit: new log heavier than all history → isPR=true, prType='weight'`.
- `Unit: same weight more reps → isPR=true, prType='reps'`.
- `Unit: lighter, fewer reps → isPR=false`.
- `Integration: log a PR → flag persisted and appears in personal-records list`.

#### 4.4 — Progress entries & compliance dashboard
**What**: Body metrics/photo tracking and an aggregate compliance dashboard with at-risk flags.

**Design**:
- `POST /clients/:id/progress` (entry_type weight/body_fat/measurements/photo; photos via presigned S3); `GET /clients/:id/progress?type=weight` returns a time series.
- `GET /clients/:id/compliance` and team-wide `GET /dashboard/compliance` returning per-client `{ completionRate30d, lastActivityAt, atRisk }`. `atRisk` heuristic (pre-AI): completion < 50% over 14d OR no login in 10d. This is the deterministic baseline that Phase 7 churn AI augments.

**Testing**:
- `Integration: post 3 weight entries → GET returns ordered series`.
- `Unit: atRisk heuristic, 40% completion + 12d inactive → atRisk=true`.
- `Integration: dashboard lists clients sorted by risk`.

---

## Phase 5: Scheduling, Messaging & Billing

### Purpose
The business-operations backbone: session scheduling with reminders, trainer↔client messaging with automated sequences, and Stripe-backed packages, subscriptions, and invoicing. After this phase a trainer can run their practice end-to-end without AI.

### Tasks

#### 5.1 — Sessions & scheduling
**What**: Book, confirm, reschedule, cancel sessions with conflict detection and automated reminders.

**Design**:
- `POST /sessions` `{ trainerId, clientId, sessionType, startsAt, endsAt, location?, workoutId? }`; `PATCH /sessions/:id` (status transitions: `scheduled→confirmed→in_progress→completed`, plus `cancelled/no_show/late_cancel`).
- Conflict detection in `packages/core/scheduling`: reject overlapping sessions for the same trainer (409 with conflicting session id). Late-cancel rule: cancellation within configurable window sets `cancellation_fee_cents`.
- On create/confirm, enqueue reminder jobs (24h + 2h before) on BullMQ; reminder dispatch via `Notifier`. Optional Google Calendar two-way sync (`integrations.provider='google_calendar'`).
- `GET /sessions?from=&to=&trainerId=` calendar feed.

**Testing**:
- `Unit: overlapping session same trainer → conflict; back-to-back (no overlap) → ok`.
- `Integration: create session → two reminder jobs enqueued`.
- `Integration: late cancel within window → cancellation_fee set`.

#### 5.2 — Messaging & automation sequences
**What**: Threaded trainer↔client messaging plus scheduled automated sequences (welcome, check-in, re-engagement).

**Design**:
- `POST /messages`, `GET /messages?clientId=` (thread), read receipts (`read_at`). Channels: in_app (default), email, sms, push via `Notifier`.
- Automation: `sequences` defined as ordered steps `{ offsetDays, channel, template, condition? }`. A `sequence_enrollments` table tracks per-client progress; a daily BullMQ job advances enrolments and dispatches due steps. Welcome sequence auto-enrols on client `onboarding` status.
- `automation_sequence_id` stamped on generated messages for attribution.

**Testing**:
- `Integration: send message → appears in thread, unread`.
- `Integration: GET thread as recipient → messages marked read`.
- `Integration: enrol client in welcome sequence, advance clock 1 day → day-1 step dispatched once (idempotent)`.

#### 5.3 — Packages, subscriptions & invoicing (Stripe)
**What**: Sell session packs and recurring memberships; generate and reconcile invoices via Stripe.

**Design**:
- `POST /packages` (catalogue). `POST /clients/:id/packages` `{ packageId, paymentMethodId }` → creates Stripe Customer (if absent) + Subscription (recurring) or one-off PaymentIntent; persists `client_packages`.
- Payment methods stored as **tokens only** (`processor_token`, `last_four`, `brand`) — PCI DSS v4.0; card data never touches our servers (Stripe Elements/SetupIntent on the client).
- Invoice state machine: `draft→sent→paid` / `failed→past_due` with dunning (`retry_count`, `next_retry_at`); a billing-cycle BullMQ job generates invoices on `next_billing_date` and decrements `sessions_remaining` on session completion.
- `POST /webhooks/stripe`: verify signature (`STRIPE_WEBHOOK_SECRET`), enqueue event for the worker (`invoice.paid`, `invoice.payment_failed`, `customer.subscription.deleted`), respond 200 fast.

**Testing**:
- `Integration (Stripe mock): create monthly package → subscription created, client_package active`.
- `Integration: webhook invoice.payment_failed → invoice status past_due, retry scheduled`.
- `Integration: webhook with bad signature → 400, no state change`.
- `Unit: session completion decrements sessions_remaining; reaching 0 → status='completed'`.
- `Security: stored payment row contains only token + last_four, never PAN`.

---

## Phase 6: Wearable Ingestion & Nutrition

### Purpose
Build the normalised wearable data layer (the cross-vendor differentiator) via Terra, plus nutrition tracking with a Photo Food Diary. This phase produces the biometric and nutrition signals the adaptive-programming and narrative AI consume in Phase 7.

### Tasks

#### 6.1 — Terra integration & wearable normalisation
**What**: Connect clients to wearables through Terra and ingest normalised time-series readings.

**Design**:
- `POST /clients/:id/wearables/connect` → Terra widget session URL; on success Terra stores the user mapping in `integrations`.
- `POST /webhooks/terra`: verify Terra signature (`TERRA_SIGNING_SECRET`), 200 fast, enqueue `terra-ingest` job. Worker maps Terra payloads → `wearable_data` rows using an Open mHealth / IEEE P1752-aligned normaliser (`packages/core/wearables/normalise.ts`): `data_type` + `unit` use Open mHealth vocabulary; original payload kept in `raw_payload`.
- Supported `data_type`s per Data Model 1 (HRV, resting_heart_rate, recovery_score, sleep_*, vo2_max, steps, etc.). Partition-aware writes.
- FHIR export endpoint `GET /clients/:id/fhir/observations` emitting FHIR R4 Observation resources incl. Exercise Vital Sign (LOINC 89574-8) — for medically-adjacent/referral contexts.

**Testing**:
- `Unit: normalise Terra HRV sample → wearable_data{data_type:'heart_rate_variability', unit:'ms', value}`.
- `Integration: terra webhook (fixture) → N wearable_data rows in correct monthly partition`.
- `Integration: terra webhook bad signature → 401, nothing enqueued`.
- `Unit: FHIR export of a steps reading → valid Observation with LOINC code`.

#### 6.2 — Nutrition plans & Photo Food Diary
**What**: Macro/calorie targets, meal-plan document sharing, and photo-based food logging.

**Design**:
- `POST /clients/:id/nutrition-plans` `{ calories_target, protein_g, carbs_g, fat_g, fibre_g, mealPlan? }` (meal plan = presigned doc upload).
- `POST /portal/food-logs` `{ logged_date, meal_type, description?, photo? }`; `ai_estimated` left false here (AI macro estimation from photo is Phase 7).
- `GET /clients/:id/nutrition/adherence?from=&to=` compares logged macros vs targets where macros present.

**Testing**:
- `Integration: create nutrition plan → 201, retrievable`.
- `Integration (client token): post food log with photo → presigned URL issued, row created`.
- `Unit: adherence calc with 1800/2000 kcal logged days → 90%`.

---

## Phase 7: AI-Native Layer

### Purpose
Deliver the differentiating AI loop: adaptive prescription suggestions, LLM check-in sentiment, automated weekly progress narratives, churn prediction, and photo macro estimation. Built on the data assembled in Phases 3–6. All AI is advisory (trainer-approved) to respect scope-of-practice boundaries.

### Tasks

#### 7.1 — AI infrastructure & guardrails
**What**: A typed LLM client with prompt caching, structured-output validation, cost/usage logging, and coaching guardrails.

**Design**:
- `packages/core/ai/client.ts` wraps the Anthropic SDK: cached system prompt (coaching policy: NASM OPT phasing, ACE scope-of-practice limits, "never give medical advice; flag, don't diagnose"), `model` from env, retries with backoff, token-usage logged to `audit_log` (`ce_type='ai.completion'`).
- All AI features request **structured JSON** validated by a Zod schema; on validation failure → one re-ask, then fail soft (feature returns "unavailable", never blocks core flows).
- Per-team feature flags in `teams.settings.ai` so self-hosters without an API key can disable AI cleanly.

**Testing**:
- `Unit (mocked SDK): completion returns malformed JSON → one retry then graceful failure`.
- `Unit: system prompt includes scope-of-practice guardrail text`.
- `Integration: AI disabled flag → AI endpoints return 200 {available:false}`.

#### 7.2 — Adaptive programming suggestions
**What**: Suggest prescription adjustments for upcoming workouts from logged performance + wearable recovery signals.

**Design**:
- `POST /programmes/:id/adapt` gathers context: last N `exercise_logs` vs prescriptions (hit reps at/under target RPE?), recent HRV/recovery/sleep trend from `wearable_data`, recent `workout_completions` mood/energy.
- LLM returns `{ adjustments: [{ workoutExerciseId, field: 'weight_kg'|'reps'|'sets', from, to, reason }], deload?: boolean }`. Persisted as **proposed** (writes `ai_adjustment_reason`, `ai_prescribed=true` only after trainer approval via `POST /programmes/:id/adapt/:suggestionId/approve`).
- Rule guard: never increase load when recovery_score below threshold or HRV trend sharply down → force deload suggestion.

**Testing**:
- `Unit: client hit all reps at RPE≤7 → suggestion increases weight; reason references performance`.
- `Unit: low HRV/recovery context → no load increase, deload=true regardless of performance`.
- `Integration: approve suggestion → workout_exercises updated, ai_prescribed=true`.
- `Integration: suggestion never auto-applies without approval`.

#### 7.3 — Check-in sentiment analysis
**What**: Classify client check-in messages for sentiment/recovery/stress signals.

**Design**:
- On inbound client `message_type='check_in'`, worker calls LLM → `{ sentiment: 'positive'|'neutral'|'negative'|'concerning', signals: string[] }`; writes `messages.ai_sentiment`. `concerning` raises a dashboard alert for the trainer.

**Testing**:
- `Unit (mocked): "exhausted, knee hurts, thinking of quitting" → 'concerning', signals include injury+attrition`.
- `Integration: concerning check-in → trainer alert created`.

#### 7.4 — Weekly progress narratives
**What**: Generate a branded weekly client summary from the week's data.

**Design**:
- Scheduled weekly BullMQ job per active client assembles facts (completed workouts, compliance, PRs, weight trend, HRV trend, nutrition adherence) and asks the LLM for a short trainer-branded narrative. Stored as a `messages` row (`message_type='ai_summary'`, `ai_generated=true`) and optionally rendered to PDF (team branding) in S3. Trainer can review/edit before send (config: auto-send vs review).

**Testing**:
- `Unit: narrative input with 2 PRs and +1.2kg trend → prompt contains those facts`.
- `Integration: weekly job → ai_summary message created per active client`.
- `Integration: review mode → message stays draft until trainer approves`.

#### 7.5 — Churn prediction & business co-pilot (MCP)
**What**: Score disengagement risk and expose a natural-language business co-pilot via MCP.

**Design**:
- Churn job computes `clients.ai_churn_score` (0–100) and `ai_churn_factors` from features: completion-rate slope, login recency, message-response latency, session no-shows, package status. A lightweight logistic model in `packages/core/ai/churn.ts` (deterministic, testable); LLM only narrates the factors. Augments the Phase 4 deterministic `atRisk` flag.
- `apps/mcp` exposes MCP resources (`client_profile`, `programme_history`, `session_logs`, `wearable_metrics`, `compliance`, `invoices`) and tools (`query_revenue`, `package_utilisation`, `client_ltv`) scoped to the authenticated team — enabling "show me John's last 4 weeks of compliance and HRV trend" and "what's my MRR this quarter?" without spreadsheets.

**Testing**:
- `Unit: declining completion + 14d no login → high churn_score; engaged client → low`.
- `Unit: churn scorer is deterministic for fixed input`.
- `Integration (MCP): query_revenue tool for team → sums paid invoices in range, team-scoped only`.
- `Security (MCP): tool call cannot read another team's data`.

---

## Phase 8: Web Frontend (Trainer Dashboard & Client Portal)

### Purpose
Expose the API through two accessible (WCAG 2.2) web surfaces sharing the generated SDK. After this phase the platform is usable end-to-end by non-technical trainers and their clients.

### Tasks

#### 8.1 — App shell, auth & typed SDK
**What**: Routing, auth flows, token handling, and the OpenAPI-generated client.

**Design**:
- `packages/sdk` generated from `/openapi.json` (openapi-typescript + typed fetch). TanStack Query for data; token refresh interceptor; role-aware route guards. Separate `/app` (trainer) and `/portal` (client) route trees.
- shadcn/ui + Tailwind; theme tokens read from `teams.branding` to support white-label.

**Testing**:
- `E2E (Playwright): login → dashboard; expired access token auto-refreshes`.
- `E2E: client logs into portal, cannot reach /app routes`.
- `a11y: axe scan of login + dashboard → no critical violations (WCAG 2.2)`.

#### 8.2 — Trainer dashboard
**What**: Client roster with at-risk surfacing, programme builder UI, calendar, messaging, billing, and the co-pilot panel.

**Design**:
- Roster sorted by churn/at-risk (Phases 4 & 7). Drag-and-drop programme builder over the Phase 3 endpoints (supersets/circuits, prescriptions, reorder). Calendar (week/month) over `/sessions`. Threaded messaging with sentiment badges. Billing views (packages, invoices, dunning). Co-pilot chat panel wired to the MCP server.

**Testing**:
- `E2E: build programme (workout + superset) → assign to client → appears on client portal`.
- `E2E: book session → reminder job enqueued (assert via API)`.
- `E2E: concerning check-in → red sentiment badge in thread`.

#### 8.3 — Client portal
**What**: Client-facing programme delivery, logging, progress, nutrition, messaging.

**Design**:
- Today's workout with set-by-set logging (Phase 4), progress charts (weight/measurements/PRs), wearable summaries, food log with photo upload, weekly AI summaries, trainer chat. Responsive/mobile-first (precursor to native apps).

**Testing**:
- `E2E: client completes today's workout logging all sets → compliance 100% on trainer dashboard`.
- `E2E: client uploads progress photo → visible to trainer`.
- `a11y: workout logging flow keyboard-navigable`.

---

## Phase 9: Hardening, Compliance & Release

### Purpose
Make the platform production- and self-host-ready: security hardening (OWASP Top 10), GDPR/HIPAA tooling, observability, backups, public OpenAPI for community connectors, and CI/CD.

### Tasks

#### 9.1 — Security & compliance hardening
**What**: OWASP Top 10 pass, data-subject tooling, encryption, audit completeness.

**Design**:
- Centralised authz tests for every route (A01); parameterised queries via Drizzle (A03); strict CSP/HSTS/security headers (`@fastify/helmet`); secrets via env only; rate limiting on all auth + webhook routes.
- GDPR: data export (full client document as JSON + FHIR) and erasure endpoints; `data_retention_until` enforced by a purge job. HIPAA-aware: encrypt `medical_intake`/`assessments` at rest (pgcrypto or app-layer envelope), access logged to `audit_log`.
- Verify `audit_log` captures CloudEvents-shaped entries for sensitive reads/writes.

**Testing**:
- `Integration: every mutating route rejects missing/invalid token (matrix test)`.
- `Integration: cross-team access attempt on each resource type → 403/404 (RLS + authz)`.
- `Integration: GDPR export → bundle contains all client data; erasure → PII gone, audit entry written`.
- `Security: helmet headers present; CSP blocks inline scripts`.

#### 9.2 — Observability, backups & CI/CD
**What**: Tracing/metrics/logs, automated backups, and a release pipeline.

**Design**:
- OpenTelemetry traces across api↔worker↔db; pino structured logs with request/trace IDs; `/metrics` (Prometheus). Nightly `pg_dump` + MinIO/S3 snapshot job with documented restore.
- GitHub Actions: lint → `tsc --noEmit` → unit → integration (Testcontainers) → build Docker images → Playwright E2E against compose → publish images. Publish versioned OpenAPI 3.1 spec as a release artefact (enables Zapier/Make/n8n connectors).
- Seed/demo dataset and a quick-start in README for self-hosters.

**Testing**:
- `Integration: trace context propagates api → worker job`.
- `Integration: backup job produces restorable dump (restore into throwaway DB, row counts match)`.
- `CI: pipeline green on a clean checkout; published openapi.json validates`.

---

## Phase Summary & Dependencies

```
Phase 1: Foundation & Data Layer        ─── required by everything
    │
Phase 2: Identity, Teams & Access       ─── requires Phase 1
    │
Phase 3: Exercise Library & Programmes  ─── requires Phase 2  ┐
Phase 4: Clients, Logging & Compliance  ─── requires Phase 2/3 ┘ (4.x logging needs 3.3)
    │
    ├── Phase 5: Scheduling/Messaging/Billing ─── requires Phase 4 (can parallel Phase 6)
    └── Phase 6: Wearables & Nutrition        ─── requires Phase 4 (can parallel Phase 5)
             │
Phase 7: AI-Native Layer                ─── requires Phases 4,5,6 (consumes their data)
    │
Phase 8: Web Frontend                   ─── requires the APIs it surfaces
             │   (8.1 after Phase 2; 8.2/8.3 incrementally as 3–7 land)
Phase 9: Hardening, Compliance & Release ─── requires all feature phases
```

Parallelism opportunities:
- Phases 5 and 6 can be built concurrently once Phase 4 is complete.
- Frontend (8.1) can start right after Phase 2; dashboard/portal panels (8.2/8.3) can be built feature-by-feature alongside Phases 3–7 rather than waiting.
- Within Phase 7, sentiment (7.3), narratives (7.4), and churn/MCP (7.5) are independent once 7.1 exists.

Scope: **large** (full-stack, multi-tenant SaaS with payments, wearables, AI, two web surfaces, and an MCP server).

---

## Definition of Done (per phase)

1. All tasks in the phase implemented.
2. All unit and integration tests pass (`pnpm test`); integration tests run against real Postgres/Redis via Testcontainers.
3. Linting and formatting pass (`pnpm lint`).
4. Type checking passes (`pnpm typecheck` / `tsc --noEmit`) with no errors.
5. `docker compose build` and `docker compose up` succeed; `/readyz` returns healthy.
6. The phase's feature works end-to-end (verified by an integration or Playwright E2E test).
7. New configuration/env vars documented in `.env.example` and README.
8. New API endpoints appear in the auto-generated `/openapi.json` (OpenAPI 3.1) with request/response schemas.
9. Database changes ship as Drizzle migrations; RLS policies added for any new team-scoped table.
10. Multi-tenant isolation verified: a cross-team access test exists and passes for any new resource.
11. For AI tasks: structured output is Zod-validated, fails soft, and is logged to `audit_log`.
```

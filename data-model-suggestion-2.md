# Data Model Suggestion 2: Hybrid Relational + JSONB

> Project: Personal Trainer Management · Created: 2025-05-25

## Philosophy

This model keeps the scheduling, billing, and exercise logging backbone relational while embedding variable-shape data in JSONB columns. Clients carry their medical intake, assessments, progress photos, wearable summaries, and nutrition as JSONB. Teams carry their trainers, exercises, packages, and integrations as JSONB.

The personal training domain has significant variation by training style: a strength coach tracks 1RM percentages, tempo, and RPE; a rehab-focused trainer tracks movement screen scores and physician clearance; a body-composition coach tracks photos, measurements, and macros. A JSONB hybrid absorbs this variation without schema migrations — training-style-specific fields are JSONB, not nullable columns.

Programme design is the most complex domain in personal training: exercises grouped into supersets and circuits, each with multi-dimensional prescriptions (sets × reps × weight × RPE × tempo × rest). The programme → workout → exercise chain stays relational because trainers need to query, reorder, copy, and version individual exercises. But the exercise prescription itself — which varies by exercise type (strength vs. cardio vs. flexibility) — is JSONB on each workout exercise row.

**Best for:** Solo trainers and small teams building an MVP, platforms targeting multiple training styles (strength, rehab, bodybuilding, endurance) where per-style field variation is high, and apps where the client profile must be exportable as a single document.

**Trade-offs:**
- (+) ~9 tables — dramatically simpler than the normalized model
- (+) Training-style-specific data (1RM charts, movement screens, postural analysis) is JSONB
- (+) Client profile is a single document — easy to export for data portability
- (+) Wearable data summaries embedded on client avoid a high-volume time-series table
- (+) Programmes and workouts remain fully relational for scheduling and compliance tracking
- (-) Cross-client wearable trend analysis requires JSONB extraction across all clients
- (-) Personal record detection requires scanning JSONB exercise log history
- (-) Assessment comparison across clients requires JSONB extraction
- (-) Nutrition aggregate queries (average daily macros) are harder in JSONB than relational

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| HL7 FHIR R4 | `clients.wearable_summary` structured for FHIR Observation export; `clients` map to FHIR Patient |
| FHIR Physical Activity IG | Exercise Vital Sign (LOINC 89574-8) computable from wearable summary and exercise logs |
| Open mHealth / IEEE P1752 | `clients.wearable_summary` data types and units align with Open mHealth schema names |
| Terra API | `clients.integrations[]` stores Terra connection state; raw data in wearable_summary |
| PCI DSS v4.0 | `clients.payment_methods[]` stores processor tokens only |
| GDPR | `clients.consent_status` and `data_retention_until` support data subject rights |
| HIPAA | `clients.medical_intake` and `clients.assessments[]` stored with appropriate access controls |
| CloudEvents | `audit_log` follows CloudEvents attribute naming |

---

## Teams

```sql
CREATE TABLE teams (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name TEXT NOT NULL,
    owner_email TEXT NOT NULL,
    timezone TEXT NOT NULL DEFAULT 'America/New_York',
    currency TEXT DEFAULT 'USD',
    -- Trainers
    trainers JSONB DEFAULT '[]',
    -- trainers[]: [{id, email, full_name, display_name, role, is_active,
    --   phone, bio, photo_path, specialities[], certifications[],
    --   certifications[]: [{body, type, number, expiry_date, verified}],
    --   hourly_rate_cents, session_rate_cents, colour_hex}]
    -- Exercise library
    exercises JSONB DEFAULT '[]',
    -- exercises[]: [{id, name, category, muscle_groups[], equipment[],
    --   difficulty, video_url, video_path, thumbnail_path, coaching_cues,
    --   common_errors, is_system, created_by_id}]
    -- Packages
    packages JSONB DEFAULT '[]',
    -- packages[]: [{id, name, package_type, price_cents, sessions_included,
    --   billing_interval, is_active}]
    -- Integrations
    integrations JSONB DEFAULT '[]',
    -- integrations[]: [{provider, status, oauth_access_token, oauth_refresh_token,
    --   token_expires_at, last_sync_at}]
    -- Settings
    settings JSONB DEFAULT '{}',
    branding JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Clients

```sql
CREATE TABLE clients (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    team_id UUID NOT NULL REFERENCES teams(id),
    trainer_id TEXT NOT NULL,             -- references teams.trainers[].id
    email TEXT,
    full_name TEXT NOT NULL,
    phone TEXT,
    status TEXT NOT NULL CHECK (status IN (
        'lead', 'onboarding', 'active', 'paused', 'churned', 'archived'
    )) DEFAULT 'lead',
    client_type TEXT CHECK (client_type IN ('in_person', 'remote', 'hybrid')) DEFAULT 'hybrid',
    -- Profile
    profile JSONB DEFAULT '{}',
    -- profile: {date_of_birth, gender, photo_path, primary_goal, secondary_goals[],
    --   height_cm, starting_weight_kg, allergies}
    -- Medical
    medical_intake JSONB DEFAULT '{}',
    -- medical_intake: {injuries[], medications[], conditions[], physician_clearance,
    --   par_q_completed, par_q_date, physician_name, physician_phone}
    -- Assessments
    assessments JSONB DEFAULT '[]',
    -- assessments[]: [{id, type, date, trainer_id, results{}, photo_paths[], notes}]
    -- results varies by type:
    --   postural: {anterior_view{}, lateral_view{}, posterior_view{}}
    --   movement_screen: {overhead_squat: 2, hurdle_step: 3, ...}
    --   fitness_test: {bench_press_1rm_kg, squat_1rm_kg, deadlift_1rm_kg, mile_time_seconds}
    -- Progress
    progress JSONB DEFAULT '[]',
    -- progress[]: [{id, date, type, weight_kg, body_fat_pct, measurements{},
    --   photo_paths[], photo_type, notes}]
    -- Wearable summary (latest values, not raw time-series)
    wearable_summary JSONB DEFAULT '{}',
    -- wearable_summary: {last_synced_at, sources[],
    --   resting_hr, hrv_avg_7d, hrv_trend, recovery_score, recovery_source,
    --   sleep_avg_hours_7d, sleep_quality_avg_7d, steps_avg_7d,
    --   vo2_max, vo2_max_source, calories_avg_7d,
    --   latest_values: [{data_type, value, unit, source, recorded_at}]}
    -- Nutrition
    nutrition JSONB DEFAULT '{}',
    -- nutrition: {active_plan: {name, calories_target, protein_g, carbs_g, fat_g, fibre_g,
    --   meal_plan_path, ai_generated, start_date, end_date},
    --   recent_food_logs: [{date, meal_type, description, calories, protein_g, carbs_g,
    --     fat_g, photo_path, ai_estimated}]}
    -- Personal records
    personal_records JSONB DEFAULT '{}',
    -- personal_records: {exercise_id: {weight_kg, reps, volume, date}, ...}
    -- Packages & billing
    active_package JSONB,
    -- active_package: {package_name, package_type, status, start_date, end_date,
    --   sessions_remaining, sessions_used, next_billing_date, cancelled_at}
    payment_methods JSONB DEFAULT '[]',
    -- payment_methods[]: [{id, method_type, processor_token, last_four, brand,
    --   is_default}]
    -- Messages
    messages JSONB DEFAULT '[]',
    -- messages[]: [{direction, channel, message_type, subject, body, media_paths[],
    --   ai_generated, ai_sentiment, read_at, sent_at}]
    -- Privacy
    consent_status TEXT CHECK (consent_status IN ('pending', 'given', 'withdrawn')) DEFAULT 'pending',
    consent_given_at TIMESTAMPTZ,
    data_retention_until DATE,
    marketing_opt_in BOOLEAN DEFAULT FALSE,
    -- AI
    ai_churn_score NUMERIC(5,2),
    ai_churn_factors JSONB,
    ai_compliance_trend TEXT,
    ai_next_session_predicted DATE,
    ai_programme_adjustment JSONB,
    -- Portal
    portal_enabled BOOLEAN DEFAULT TRUE,
    portal_last_login_at TIMESTAMPTZ,
    tags TEXT[] DEFAULT '{}',
    notes TEXT,
    referred_by_id UUID REFERENCES clients(id),
    onboarded_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_clients_team ON clients(team_id);
CREATE INDEX idx_clients_trainer ON clients(trainer_id);
CREATE INDEX idx_clients_status ON clients(team_id, status);
CREATE INDEX idx_clients_tags ON clients USING GIN (tags);
CREATE INDEX idx_clients_churn ON clients(ai_churn_score) WHERE status = 'active';
CREATE INDEX idx_clients_assessments ON clients USING GIN (assessments);
```

### Example: Latest Assessment Lookup

```sql
-- Surface the most recent movement screen for a client
SELECT full_name, a->>'type' AS assessment_type,
       a->>'date' AS assessment_date, a->'results' AS results
FROM clients,
     jsonb_array_elements(assessments) AS a
WHERE id = :client_id
  AND a->>'type' = 'movement_screen'
ORDER BY (a->>'date')::date DESC
LIMIT 1;
```

### Example: Wearable Recovery Check

```sql
-- Check if client's HRV trend supports today's planned intensity
SELECT full_name,
       wearable_summary->>'hrv_avg_7d' AS hrv_7d,
       wearable_summary->>'hrv_trend' AS hrv_trend,
       wearable_summary->>'recovery_score' AS recovery,
       wearable_summary->>'sleep_avg_hours_7d' AS sleep_hours
FROM clients
WHERE id = :client_id;
```

---

## Programmes & Workouts

```sql
CREATE TABLE programmes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    team_id UUID NOT NULL REFERENCES teams(id),
    client_id UUID REFERENCES clients(id),  -- NULL = template
    trainer_id TEXT NOT NULL,                -- references teams.trainers[].id
    name TEXT NOT NULL,
    description TEXT,
    programme_type TEXT CHECK (programme_type IN (
        'strength', 'hypertrophy', 'endurance', 'weight_loss',
        'rehab', 'sport_specific', 'general', 'custom'
    )),
    status TEXT NOT NULL CHECK (status IN (
        'draft', 'active', 'completed', 'paused', 'archived'
    )) DEFAULT 'draft',
    start_date DATE,
    end_date DATE,
    duration_weeks INT,
    -- AI
    ai_generated BOOLEAN DEFAULT FALSE,
    ai_model_version TEXT,
    ai_adaptation_enabled BOOLEAN DEFAULT FALSE,
    is_template BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_programmes_team ON programmes(team_id);
CREATE INDEX idx_programmes_client ON programmes(client_id);
CREATE INDEX idx_programmes_status ON programmes(status);

CREATE TABLE workouts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    programme_id UUID NOT NULL REFERENCES programmes(id),
    name TEXT NOT NULL,
    description TEXT,
    day_number INT,
    week_number INT,
    workout_type TEXT CHECK (workout_type IN (
        'strength', 'cardio', 'hiit', 'flexibility', 'recovery', 'mixed'
    )),
    estimated_duration_minutes INT,
    sort_order INT DEFAULT 0,
    -- Exercises with prescriptions (JSONB for flexible prescription shapes)
    exercises JSONB DEFAULT '[]',
    -- exercises[]: [{id, exercise_id, exercise_name, sort_order,
    --   group_type, group_id,
    --   prescription: {sets, reps, weight_kg, weight_pct_1rm, rpe, tempo,
    --     rest_seconds, duration_seconds, distance_metres},
    --   notes, ai_prescribed, ai_adjustment_reason}]
    ai_generated BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_workouts_programme ON workouts(programme_id);
```

### Example: Build Workout View

```sql
-- Render the full workout with exercises grouped by superset/circuit
SELECT w.name AS workout_name, w.workout_type,
       e->>'exercise_name' AS exercise, e->>'group_type' AS grouping,
       e->>'group_id' AS group_id, e->'prescription' AS prescription
FROM workouts w,
     jsonb_array_elements(w.exercises) AS e
WHERE w.id = :workout_id
ORDER BY (e->>'sort_order')::int;
```

---

## Exercise Logs & Completions

```sql
CREATE TABLE exercise_logs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    client_id UUID NOT NULL REFERENCES clients(id),
    workout_id UUID REFERENCES workouts(id),
    exercise_id TEXT NOT NULL,              -- references teams.exercises[].id
    exercise_name TEXT NOT NULL,
    logged_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    -- Actual performance (per set)
    set_number INT NOT NULL,
    reps_completed INT,
    weight_kg NUMERIC(6,1),
    rpe_actual NUMERIC(3,1),
    duration_seconds INT,
    distance_metres NUMERIC(10,1),
    -- PR detection
    is_personal_record BOOLEAN DEFAULT FALSE,
    pr_type TEXT CHECK (pr_type IN ('weight', 'reps', 'volume', 'time', 'distance')),
    notes TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_exercise_logs_client ON exercise_logs(client_id, logged_at);
CREATE INDEX idx_exercise_logs_exercise ON exercise_logs(exercise_id, client_id);
CREATE INDEX idx_exercise_logs_workout ON exercise_logs(workout_id);
CREATE INDEX idx_exercise_logs_pr ON exercise_logs(client_id, exercise_id) WHERE is_personal_record = TRUE;

CREATE TABLE workout_completions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    client_id UUID NOT NULL REFERENCES clients(id),
    workout_id UUID NOT NULL REFERENCES workouts(id),
    programme_id UUID NOT NULL REFERENCES programmes(id),
    scheduled_date DATE,
    completed_at TIMESTAMPTZ,
    status TEXT NOT NULL CHECK (status IN (
        'scheduled', 'in_progress', 'completed', 'skipped', 'partial'
    )) DEFAULT 'scheduled',
    duration_minutes INT,
    compliance_pct NUMERIC(5,2),
    trainer_feedback TEXT,
    client_notes TEXT,
    mood_rating INT CHECK (mood_rating BETWEEN 1 AND 5),
    energy_rating INT CHECK (energy_rating BETWEEN 1 AND 5),
    -- Wearable data captured during workout
    wearable_session JSONB,
    -- wearable_session: {avg_hr, max_hr, calories, hrv_start, hrv_end, source}
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_workout_completions_client ON workout_completions(client_id);
CREATE INDEX idx_workout_completions_programme ON workout_completions(programme_id);
CREATE INDEX idx_workout_completions_status ON workout_completions(status);
```

---

## Sessions

```sql
CREATE TABLE sessions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    team_id UUID NOT NULL REFERENCES teams(id),
    trainer_id TEXT NOT NULL,
    client_id UUID NOT NULL REFERENCES clients(id),
    session_type TEXT NOT NULL CHECK (session_type IN (
        'in_person', 'video', 'check_in', 'assessment', 'group'
    )),
    status TEXT NOT NULL CHECK (status IN (
        'scheduled', 'confirmed', 'in_progress', 'completed',
        'cancelled', 'no_show', 'late_cancel'
    )) DEFAULT 'scheduled',
    starts_at TIMESTAMPTZ NOT NULL,
    ends_at TIMESTAMPTZ NOT NULL,
    location TEXT,
    video_link TEXT,
    workout_id UUID REFERENCES workouts(id),
    workout_completion_id UUID REFERENCES workout_completions(id),
    -- Cancellation
    cancelled_at TIMESTAMPTZ,
    cancellation_fee_cents BIGINT DEFAULT 0,
    -- Notes
    trainer_notes TEXT,
    client_notes TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_sessions_team ON sessions(team_id, starts_at);
CREATE INDEX idx_sessions_trainer ON sessions(trainer_id, starts_at);
CREATE INDEX idx_sessions_client ON sessions(client_id, starts_at);
CREATE INDEX idx_sessions_status ON sessions(status, starts_at);
```

---

## Invoices

```sql
CREATE TABLE invoices (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    team_id UUID NOT NULL REFERENCES teams(id),
    client_id UUID NOT NULL REFERENCES clients(id),
    invoice_number TEXT NOT NULL,
    status TEXT NOT NULL CHECK (status IN (
        'draft', 'sent', 'paid', 'failed', 'refunded', 'void', 'past_due'
    )) DEFAULT 'draft',
    -- Line items
    line_items JSONB DEFAULT '[]',
    -- line_items[]: [{description, quantity, unit_price_cents, total_cents}]
    amount_cents BIGINT NOT NULL,
    tax_cents BIGINT DEFAULT 0,
    total_cents BIGINT NOT NULL,
    due_date DATE,
    paid_at TIMESTAMPTZ,
    payment_method TEXT,
    -- Dunning
    retry_count INT DEFAULT 0,
    next_retry_at TIMESTAMPTZ,
    failure_reason TEXT,
    processor_reference TEXT,
    refund_of_id UUID REFERENCES invoices(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (team_id, invoice_number)
);
CREATE INDEX idx_invoices_team ON invoices(team_id);
CREATE INDEX idx_invoices_client ON invoices(client_id);
CREATE INDEX idx_invoices_status ON invoices(team_id, status);
CREATE INDEX idx_invoices_dunning ON invoices(next_retry_at) WHERE status = 'failed';
```

---

## Audit Log

```sql
CREATE TABLE audit_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    team_id UUID NOT NULL,
    ce_source TEXT NOT NULL,
    ce_type TEXT NOT NULL,
    ce_time TIMESTAMPTZ NOT NULL DEFAULT now(),
    ce_specversion TEXT NOT NULL DEFAULT '1.0',
    actor_type TEXT NOT NULL CHECK (actor_type IN ('trainer', 'client', 'system', 'ai')),
    actor_id UUID,
    actor_ip INET,
    resource_type TEXT NOT NULL,
    resource_id UUID,
    action TEXT NOT NULL,
    detail JSONB,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (ce_time);

CREATE INDEX idx_audit_team_time ON audit_log(team_id, ce_time);
CREATE INDEX idx_audit_resource ON audit_log(resource_type, resource_id);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Teams | 1 | teams (trainers, exercises, packages, integrations embedded) |
| Clients | 1 | clients (medical, assessments, progress, wearable, nutrition, PRs, billing, messages embedded) |
| Programmes & Workouts | 2 | programmes, workouts (exercises embedded as JSONB) |
| Exercise Logging | 2 | exercise_logs, workout_completions (relational — compliance tracking anchor) |
| Sessions | 1 | sessions (relational — scheduling constraints) |
| Billing | 1 | invoices (line items embedded as JSONB) |
| Audit | 1 | audit_log (partitioned) |
| **Total** | **9** | |

---

## Key Design Decisions

1. **Clients as rich documents** — medical intake, assessments, progress entries, wearable summaries, nutrition, personal records, payment methods, and messages are JSONB arrays/objects on the client. This makes the client profile self-contained — the trainer sees everything about a client in a single read. The AI adaptive programming engine reads `wearable_summary` and `personal_records` from the same row it reads the client's goals and compliance trend.

2. **Wearable data as summary, not time-series** — instead of a high-volume `wearable_data` table, the client carries a `wearable_summary` with rolling averages and latest values. The Terra webhook handler updates this summary on each sync. This trades individual-reading queryability for dramatically lower write volume and simpler queries. Raw time-series data stays in Terra or the source platform.

3. **Exercise prescriptions as JSONB on workouts** — `workouts.exercises[]` embeds the exercise list with prescriptions because exercises are always read and written as a set when building a workout. The prescription shape varies by exercise type (strength has sets/reps/weight/RPE/tempo, cardio has duration/distance, flexibility has hold_seconds). JSONB absorbs this without nullable columns.

4. **Exercise logs stay relational** — per-set exercise logs must be queryable for personal record detection, progression analysis, and cross-exercise volume calculations. These are the highest-frequency write and the most important analytical query surface. The relational structure supports `WHERE exercise_id = ? AND client_id = ? ORDER BY logged_at` efficiently.

5. **Workout completions stay relational** — compliance percentage, mood/energy ratings, and completion status are the core metrics for programme effectiveness analysis. The trainer dashboard queries `WHERE programme_id = ? AND status = 'completed'` to see programme compliance. Wearable session data captured during the workout is embedded as JSONB.

6. **Sessions stay relational** — sessions enforce scheduling constraints (time-range conflict detection, availability checks). They anchor the billing lifecycle — session completion triggers package decrement and invoice generation. These require indexed time-range queries that JSONB can't support.

7. **Assessments embedded on clients** — assessments are always accessed in the context of a specific client. The trainer opens a client profile and scrolls through assessment history. Embedding avoids a JOIN and makes the client record self-contained. Different assessment types (postural, movement screen, fitness test) have different result shapes — JSONB absorbs this naturally.

8. **Personal records embedded on clients** — `clients.personal_records` is a JSONB map keyed by exercise ID with the best result per type (weight, reps, volume). Updated when `exercise_logs.is_personal_record = TRUE` is detected. The client profile renders the PR board from this single field without scanning the exercise log history.

9. **Nutrition embedded on clients** — the active nutrition plan and recent food logs are embedded because they're always read in the client profile context. The Photo Food Diary entries include `photo_path` and `ai_estimated` flag. Older food logs are archived to keep the JSONB manageable — the active plan and last 14 days is sufficient for coaching.

10. **Packages embedded on clients** — `clients.active_package` captures the current package state (sessions remaining, next billing date). Package definitions live in `teams.packages[]`. This avoids a junction table for a 1:1 relationship that changes infrequently — package changes are infrequent compared to session bookings and exercise logs.

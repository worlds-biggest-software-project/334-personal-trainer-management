# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Personal Trainer Management · Created: 2025-05-25

## Philosophy

This model gives every personal training domain concept its own table with explicit foreign keys. The operational lifecycle flows through: trainers/teams → clients → programmes → workouts → exercises → sessions → progress tracking → billing. Wearable data, nutrition, messaging, and assessments each have dedicated relational structures.

The personal training domain centres on programme delivery and progress tracking. A normalized model makes the relationships explicit: a programme contains workouts, workouts contain exercises with prescribed sets/reps/weight, clients log actual performance against prescriptions, and the system compares planned vs. actual to track compliance and adaptation. Wearable data from multiple sources is normalized into a single time-series table for unified querying.

**Best for:** Training teams (3-20 trainers) with shared client rosters, platforms needing cross-client analytics for programme effectiveness, and medically-adjacent deployments requiring HIPAA-compliant charting with FHIR-compatible data export.

**Trade-offs:**
- (+) Full referential integrity across the programme-workout-exercise prescription chain
- (+) Compliance tracking is standard SQL (completed exercises / prescribed exercises)
- (+) Wearable data from all sources queryable in one table with consistent schema
- (+) Cross-client programme effectiveness analysis (which programmes produce best outcomes)
- (+) Personal records computed from historical exercise logs
- (-) ~25 tables is a large schema
- (-) Exercise prescriptions with supersets, circuits, and rest periods add modelling complexity
- (-) High write volume for wearable data and exercise logging

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| HL7 FHIR R4 | `wearable_data` structured to export as FHIR Observation resources; `clients` map to FHIR Patient |
| FHIR Physical Activity IG | Exercise Vital Sign (LOINC 89574-8) computable from `wearable_data` and `exercise_logs` |
| Open mHealth / IEEE P1752 | `wearable_data.data_type` and `unit` align with Open mHealth schema names |
| Terra API | `wearable_data.source` and `raw_payload` accommodate Terra-normalized data |
| PCI DSS v4.0 | `payment_methods` stores processor tokens only |
| GDPR | `clients.consent_status` and `data_retention_until` support data subject rights |
| HIPAA | `clients.medical_intake` and `assessments` stored with appropriate access controls |
| CloudEvents | `audit_log` follows CloudEvents attribute naming |
| OAuth 2.0 | `integrations` stores OAuth tokens for wearable and third-party services |

---

## Trainers & Teams

```sql
CREATE TABLE teams (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name TEXT NOT NULL,
    owner_email TEXT NOT NULL,
    timezone TEXT NOT NULL DEFAULT 'America/New_York',
    currency TEXT DEFAULT 'USD',
    branding JSONB DEFAULT '{}',
    settings JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE trainers (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    team_id UUID NOT NULL REFERENCES teams(id),
    email TEXT NOT NULL,
    full_name TEXT NOT NULL,
    role TEXT NOT NULL CHECK (role IN ('owner', 'head_trainer', 'trainer', 'intern', 'admin')),
    is_active BOOLEAN DEFAULT TRUE,
    phone TEXT,
    bio TEXT,
    photo_path TEXT,
    specialities TEXT[] DEFAULT '{}',
    certifications JSONB DEFAULT '[]',
    -- certifications[]: [{body: 'NASM', type: 'CPT', number, expiry_date, verified}]
    hourly_rate_cents BIGINT,
    session_rate_cents BIGINT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (team_id, email)
);
CREATE INDEX idx_trainers_team ON trainers(team_id);
CREATE INDEX idx_trainers_specialities ON trainers USING GIN (specialities);
```

---

## Clients

```sql
CREATE TABLE clients (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    team_id UUID NOT NULL REFERENCES teams(id),
    trainer_id UUID NOT NULL REFERENCES trainers(id),
    email TEXT,
    full_name TEXT NOT NULL,
    phone TEXT,
    date_of_birth DATE,
    gender TEXT,
    photo_path TEXT,
    status TEXT NOT NULL CHECK (status IN (
        'lead', 'onboarding', 'active', 'paused', 'churned', 'archived'
    )) DEFAULT 'lead',
    client_type TEXT CHECK (client_type IN ('in_person', 'remote', 'hybrid')) DEFAULT 'hybrid',
    -- Goals
    primary_goal TEXT,                   -- 'weight_loss', 'muscle_gain', 'strength', 'endurance', 'rehab', 'general'
    secondary_goals TEXT[] DEFAULT '{}',
    -- Health
    medical_intake JSONB DEFAULT '{}',
    -- medical_intake: {injuries[], medications[], conditions[], physician_clearance, par_q_completed}
    allergies TEXT,
    -- Body baseline
    height_cm NUMERIC(5,1),
    starting_weight_kg NUMERIC(5,1),
    -- Privacy
    consent_status TEXT CHECK (consent_status IN ('pending', 'given', 'withdrawn')) DEFAULT 'pending',
    consent_given_at TIMESTAMPTZ,
    data_retention_until DATE,
    marketing_opt_in BOOLEAN DEFAULT FALSE,
    -- AI
    ai_churn_score NUMERIC(5,2),
    ai_churn_factors JSONB,
    ai_compliance_trend TEXT,            -- 'improving', 'stable', 'declining'
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
```

---

## Exercise Library

```sql
CREATE TABLE exercises (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    team_id UUID NOT NULL REFERENCES teams(id),
    name TEXT NOT NULL,
    category TEXT NOT NULL CHECK (category IN (
        'strength', 'cardio', 'flexibility', 'balance', 'plyometric',
        'core', 'functional', 'warmup', 'cooldown', 'custom'
    )),
    muscle_groups TEXT[] DEFAULT '{}',
    equipment TEXT[] DEFAULT '{}',
    difficulty TEXT CHECK (difficulty IN ('beginner', 'intermediate', 'advanced')),
    video_url TEXT,
    video_path TEXT,
    thumbnail_path TEXT,
    coaching_cues TEXT,
    common_errors TEXT,
    is_system BOOLEAN DEFAULT FALSE,      -- pre-loaded library vs. custom
    created_by_id UUID REFERENCES trainers(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_exercises_team ON exercises(team_id);
CREATE INDEX idx_exercises_category ON exercises(category);
CREATE INDEX idx_exercises_muscle ON exercises USING GIN (muscle_groups);
CREATE INDEX idx_exercises_equipment ON exercises USING GIN (equipment);
```

---

## Programmes & Workouts

```sql
CREATE TABLE programmes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    team_id UUID NOT NULL REFERENCES teams(id),
    client_id UUID REFERENCES clients(id),   -- NULL = template
    trainer_id UUID NOT NULL REFERENCES trainers(id),
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
CREATE INDEX idx_programmes_trainer ON programmes(trainer_id);
CREATE INDEX idx_programmes_status ON programmes(status);

CREATE TABLE workouts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    programme_id UUID NOT NULL REFERENCES programmes(id),
    name TEXT NOT NULL,
    description TEXT,
    day_number INT,                       -- day within programme cycle
    week_number INT,                      -- week within programme
    workout_type TEXT CHECK (workout_type IN (
        'strength', 'cardio', 'hiit', 'flexibility', 'recovery', 'mixed'
    )),
    estimated_duration_minutes INT,
    sort_order INT DEFAULT 0,
    ai_generated BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_workouts_programme ON workouts(programme_id);

CREATE TABLE workout_exercises (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workout_id UUID NOT NULL REFERENCES workouts(id) ON DELETE CASCADE,
    exercise_id UUID NOT NULL REFERENCES exercises(id),
    sort_order INT NOT NULL,
    group_type TEXT CHECK (group_type IN ('straight', 'superset', 'circuit', 'drop_set', 'rest_pause')),
    group_id TEXT,                        -- exercises in same group share this ID
    -- Prescription
    sets INT,
    reps TEXT,                           -- "8-12" or "AMRAP" or "30s"
    weight_kg NUMERIC(6,1),
    weight_pct_1rm NUMERIC(5,2),         -- percentage of 1RM
    rpe NUMERIC(3,1),                    -- rate of perceived exertion
    tempo TEXT,                          -- "3-1-2-0" (eccentric-pause-concentric-pause)
    rest_seconds INT,
    duration_seconds INT,                -- for timed exercises
    distance_metres NUMERIC(10,1),       -- for distance-based
    notes TEXT,
    -- AI
    ai_prescribed BOOLEAN DEFAULT FALSE,
    ai_adjustment_reason TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_workout_exercises ON workout_exercises(workout_id);
```

---

## Exercise Logging & Personal Records

```sql
CREATE TABLE exercise_logs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    client_id UUID NOT NULL REFERENCES clients(id),
    workout_exercise_id UUID REFERENCES workout_exercises(id),
    exercise_id UUID NOT NULL REFERENCES exercises(id),
    logged_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    -- Actual performance
    set_number INT NOT NULL,
    reps_completed INT,
    weight_kg NUMERIC(6,1),
    rpe_actual NUMERIC(3,1),
    duration_seconds INT,
    distance_metres NUMERIC(10,1),
    -- Comparison
    is_personal_record BOOLEAN DEFAULT FALSE,
    pr_type TEXT CHECK (pr_type IN ('weight', 'reps', 'volume', 'time', 'distance')),
    notes TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_exercise_logs_client ON exercise_logs(client_id, logged_at);
CREATE INDEX idx_exercise_logs_exercise ON exercise_logs(exercise_id, client_id);
CREATE INDEX idx_exercise_logs_workout ON exercise_logs(workout_exercise_id);
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
    compliance_pct NUMERIC(5,2),         -- exercises completed / exercises prescribed
    trainer_feedback TEXT,
    client_notes TEXT,
    mood_rating INT CHECK (mood_rating BETWEEN 1 AND 5),
    energy_rating INT CHECK (energy_rating BETWEEN 1 AND 5),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_workout_completions_client ON workout_completions(client_id);
CREATE INDEX idx_workout_completions_programme ON workout_completions(programme_id);
CREATE INDEX idx_workout_completions_status ON workout_completions(status);
```

---

## Progress & Wearable Data

```sql
CREATE TABLE progress_entries (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    client_id UUID NOT NULL REFERENCES clients(id),
    recorded_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    entry_type TEXT NOT NULL CHECK (entry_type IN (
        'weight', 'body_fat', 'measurements', 'photo', 'assessment'
    )),
    -- Weight
    weight_kg NUMERIC(5,1),
    body_fat_pct NUMERIC(5,2),
    -- Measurements (cm)
    measurements JSONB,
    -- measurements: {chest, waist, hips, bicep_l, bicep_r, thigh_l, thigh_r, calf_l, calf_r, neck}
    -- Photos
    photo_paths TEXT[] DEFAULT '{}',
    photo_type TEXT CHECK (photo_type IN ('front', 'side', 'back', 'other')),
    notes TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_progress_client ON progress_entries(client_id, recorded_at);
CREATE INDEX idx_progress_type ON progress_entries(entry_type, recorded_at);

CREATE TABLE wearable_data (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    client_id UUID NOT NULL REFERENCES clients(id),
    source TEXT NOT NULL,                 -- 'apple_health', 'fitbit', 'garmin', 'oura', 'whoop', 'terra'
    data_type TEXT NOT NULL CHECK (data_type IN (
        'heart_rate', 'heart_rate_variability', 'resting_heart_rate',
        'steps', 'calories_active', 'calories_total', 'distance',
        'sleep_duration', 'sleep_quality', 'sleep_stages',
        'vo2_max', 'recovery_score', 'stress', 'respiratory_rate',
        'body_temperature', 'blood_oxygen', 'activity_minutes'
    )),
    value NUMERIC(10,2),
    unit TEXT NOT NULL,
    recorded_at TIMESTAMPTZ NOT NULL,
    session_id UUID,                      -- links to workout if during training
    raw_payload JSONB,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_wearable_client ON wearable_data(client_id, recorded_at);
CREATE INDEX idx_wearable_type ON wearable_data(client_id, data_type, recorded_at);

CREATE TABLE assessments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    client_id UUID NOT NULL REFERENCES clients(id),
    trainer_id UUID NOT NULL REFERENCES trainers(id),
    assessment_type TEXT NOT NULL CHECK (assessment_type IN (
        'initial_intake', 'postural', 'movement_screen', 'fitness_test',
        'flexibility', 'body_composition', 'follow_up'
    )),
    assessment_date DATE NOT NULL,
    results JSONB NOT NULL,
    -- results: varies by type:
    -- postural: {anterior_view{}, lateral_view{}, posterior_view{}, notes}
    -- movement_screen: {overhead_squat: 2, hurdle_step: 3, ...}
    -- fitness_test: {bench_press_1rm_kg, squat_1rm_kg, deadlift_1rm_kg, mile_time_seconds, ...}
    photo_paths TEXT[] DEFAULT '{}',
    notes TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_assessments_client ON assessments(client_id);
```

---

## Nutrition

```sql
CREATE TABLE nutrition_plans (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    client_id UUID NOT NULL REFERENCES clients(id),
    trainer_id UUID NOT NULL REFERENCES trainers(id),
    name TEXT NOT NULL,
    status TEXT NOT NULL CHECK (status IN ('active', 'paused', 'completed')) DEFAULT 'active',
    -- Macro targets
    calories_target INT,
    protein_g INT,
    carbs_g INT,
    fat_g INT,
    fibre_g INT,
    -- Meal plan documents
    meal_plan_path TEXT,
    notes TEXT,
    ai_generated BOOLEAN DEFAULT FALSE,
    start_date DATE,
    end_date DATE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_nutrition_plans_client ON nutrition_plans(client_id);

CREATE TABLE food_logs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    client_id UUID NOT NULL REFERENCES clients(id),
    logged_date DATE NOT NULL,
    meal_type TEXT CHECK (meal_type IN ('breakfast', 'lunch', 'dinner', 'snack', 'pre_workout', 'post_workout')),
    description TEXT,
    calories INT,
    protein_g NUMERIC(5,1),
    carbs_g NUMERIC(5,1),
    fat_g NUMERIC(5,1),
    photo_path TEXT,                      -- Photo Food Diary
    ai_estimated BOOLEAN DEFAULT FALSE,
    notes TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_food_logs_client ON food_logs(client_id, logged_date);
```

---

## Sessions & Scheduling

```sql
CREATE TABLE sessions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    team_id UUID NOT NULL REFERENCES teams(id),
    trainer_id UUID NOT NULL REFERENCES trainers(id),
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
    -- Workout link
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

## Billing

```sql
CREATE TABLE packages (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    team_id UUID NOT NULL REFERENCES teams(id),
    name TEXT NOT NULL,
    package_type TEXT NOT NULL CHECK (package_type IN (
        'session_pack', 'monthly', 'quarterly', 'annual', 'drop_in', 'custom'
    )),
    price_cents BIGINT NOT NULL,
    sessions_included INT,               -- NULL = unlimited
    billing_interval TEXT CHECK (billing_interval IN ('one_time', 'weekly', 'monthly', 'quarterly', 'annual')),
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_packages_team ON packages(team_id);

CREATE TABLE client_packages (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    client_id UUID NOT NULL REFERENCES clients(id),
    package_id UUID NOT NULL REFERENCES packages(id),
    status TEXT NOT NULL CHECK (status IN (
        'active', 'past_due', 'frozen', 'cancelled', 'expired', 'completed'
    )) DEFAULT 'active',
    start_date DATE NOT NULL,
    end_date DATE,
    sessions_remaining INT,
    sessions_used INT DEFAULT 0,
    next_billing_date DATE,
    payment_method_id UUID REFERENCES payment_methods(id),
    cancelled_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_client_packages_client ON client_packages(client_id);
CREATE INDEX idx_client_packages_status ON client_packages(status);
CREATE INDEX idx_client_packages_billing ON client_packages(next_billing_date) WHERE status = 'active';

CREATE TABLE payment_methods (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    client_id UUID NOT NULL REFERENCES clients(id),
    method_type TEXT NOT NULL CHECK (method_type IN ('card', 'bank_account')),
    processor TEXT DEFAULT 'stripe',
    processor_token TEXT NOT NULL,
    last_four TEXT,
    brand TEXT,
    is_default BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_payment_methods_client ON payment_methods(client_id);

CREATE TABLE invoices (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    team_id UUID NOT NULL REFERENCES teams(id),
    client_id UUID NOT NULL REFERENCES clients(id),
    invoice_number TEXT NOT NULL,
    status TEXT NOT NULL CHECK (status IN (
        'draft', 'sent', 'paid', 'failed', 'refunded', 'void', 'past_due'
    )) DEFAULT 'draft',
    amount_cents BIGINT NOT NULL,
    tax_cents BIGINT DEFAULT 0,
    total_cents BIGINT NOT NULL,
    description TEXT,
    due_date DATE,
    paid_at TIMESTAMPTZ,
    retry_count INT DEFAULT 0,
    next_retry_at TIMESTAMPTZ,
    failure_reason TEXT,
    processor_reference TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (team_id, invoice_number)
);
CREATE INDEX idx_invoices_team ON invoices(team_id);
CREATE INDEX idx_invoices_client ON invoices(client_id);
CREATE INDEX idx_invoices_status ON invoices(team_id, status);
```

---

## Messaging & Check-ins

```sql
CREATE TABLE messages (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    team_id UUID NOT NULL REFERENCES teams(id),
    sender_type TEXT NOT NULL CHECK (sender_type IN ('trainer', 'client', 'system', 'ai')),
    sender_id UUID,
    recipient_type TEXT NOT NULL CHECK (recipient_type IN ('trainer', 'client')),
    recipient_id UUID NOT NULL,
    channel TEXT NOT NULL CHECK (channel IN ('in_app', 'email', 'sms', 'push')),
    message_type TEXT CHECK (message_type IN (
        'direct', 'check_in', 'progress_report', 'welcome', 'reengagement',
        'reminder', 'ai_summary', 'video'
    )),
    subject TEXT,
    body TEXT NOT NULL,
    media_paths TEXT[] DEFAULT '{}',
    read_at TIMESTAMPTZ,
    ai_generated BOOLEAN DEFAULT FALSE,
    ai_sentiment TEXT,                   -- 'positive', 'neutral', 'negative', 'concerning'
    automation_sequence_id UUID,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_messages_client ON messages(recipient_id, created_at) WHERE recipient_type = 'client';
CREATE INDEX idx_messages_team ON messages(team_id, created_at);
```

---

## Integrations & Audit

```sql
CREATE TABLE integrations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    team_id UUID NOT NULL REFERENCES teams(id),
    provider TEXT NOT NULL CHECK (provider IN (
        'stripe', 'terra', 'apple_health', 'fitbit', 'garmin',
        'oura', 'whoop', 'myfitnesspal', 'google_calendar',
        'zoom', 'zapier'
    )),
    status TEXT NOT NULL CHECK (status IN ('connected', 'disconnected', 'error', 'expired')) DEFAULT 'disconnected',
    oauth_access_token TEXT,
    oauth_refresh_token TEXT,
    token_expires_at TIMESTAMPTZ,
    last_sync_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (team_id, provider)
);

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
| Teams & Trainers | 2 | teams, trainers |
| Clients | 1 | clients |
| Exercise Library | 1 | exercises |
| Programmes & Workouts | 3 | programmes, workouts, workout_exercises |
| Logging | 2 | exercise_logs, workout_completions |
| Progress & Wearable | 3 | progress_entries, wearable_data, assessments |
| Nutrition | 2 | nutrition_plans, food_logs |
| Sessions | 1 | sessions |
| Billing | 4 | packages, client_packages, payment_methods, invoices |
| Messaging | 1 | messages |
| Integrations & Audit | 2 | integrations, audit_log (partitioned) |
| **Total** | **22** | |

---

## Key Design Decisions

1. **Workout exercises with group types** — `workout_exercises.group_type` and `group_id` model supersets, circuits, and drop sets. Exercises sharing a `group_id` are performed together. This supports the advanced periodisation that Trainerize and TrueCoach users expect.

2. **Prescription vs. actual** — `workout_exercises` stores the prescription (sets, reps, weight, RPE, tempo). `exercise_logs` stores what the client actually did. Comparing these enables compliance scoring and automatic progression: if a client hits all prescribed reps at the prescribed RPE, the AI can suggest a weight increase.

3. **Personal records from exercise logs** — `exercise_logs.is_personal_record` is computed by comparing the new log against previous logs for the same exercise and client. The PR flag and type enable leaderboard displays and progress celebration — key for client engagement.

4. **Wearable data as a unified time-series table** — `wearable_data` normalizes data from Apple Health, Fitbit, Garmin, Oura, WHOOP, and Terra into one table with `source`, `data_type`, `value`, and `unit`. The AI adaptive programming engine queries this table for HRV trends, recovery scores, and sleep quality to adjust workout prescriptions.

5. **Programme templates vs. client programmes** — `programmes` with `is_template = TRUE` and `client_id = NULL` are reusable templates. Assigning a programme to a client copies the template structure. This supports the common workflow of building a programme once and assigning it to multiple clients with individual adjustments.

6. **Workout completions separate from exercise logs** — `workout_completions` captures the workout-level summary (duration, compliance percentage, mood/energy ratings). `exercise_logs` captures the per-exercise detail. This two-level structure supports both the trainer's dashboard view (completion rates) and the detailed training log.

7. **Food logs with Photo Food Diary** — `food_logs.photo_path` supports the Photo Food Diary pattern identified in PT Distinction. Some trainers prefer qualitative coaching ("let me see what you ate") over quantitative macro tracking. The `ai_estimated` flag marks entries where AI estimated macros from the photo.

8. **AI sentiment on messages** — `messages.ai_sentiment` classifies check-in responses as positive, neutral, negative, or concerning. This surfaces clients who may be struggling without requiring the trainer to read every message — addressing the LLM check-in analysis opportunity.

9. **Sessions linked to workouts** — `sessions.workout_id` links a training session to the prescribed workout. `workout_completion_id` links to the actual completion record. This enables scheduling a workout and then recording what happened, closing the prescription-execution loop.

10. **Assessments with flexible JSONB results** — `assessments.results` is JSONB because assessment types vary widely (postural analysis, movement screens, fitness tests, body composition). Each type has different measured variables. The relational `assessment_type` column supports filtering and cross-client comparison.

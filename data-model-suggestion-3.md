# Data Model Suggestion 3: Event-Sourced / Audit-First

> Project: Personal Trainer Management · Created: 2025-05-25

## Philosophy

Every state change — client onboarded, programme assigned, workout prescribed, exercise logged, personal record set, session completed, payment processed, wearable data synced — is an immutable event. The `event_store` is the single source of truth; relational read models are materialised projections rebuilt from the event stream.

The personal training domain is well-suited to event sourcing because client progression is the core business signal. Adaptive programming depends on the complete sequence of prescriptions, actual performance, recovery data, and compliance patterns — not just current state. Churn prediction needs the full timeline of session attendance, workout completion gaps, and engagement decline. Programme effectiveness analysis requires comparing prescribed vs. actual across the entire training history. Event sourcing makes these analytics native.

AI features benefit enormously: the adaptive programming engine trains on the full event sequence per client (prescribed weight → actual weight → RPE → recovery score → next prescription). Churn prediction models consume booking patterns, completion gaps, and engagement signals. Automated progress narratives replay the client's event history to generate coaching summaries ("you've hit 3 PRs this month, your squat is up 12% since programme start, and your HRV trend shows improved recovery").

**Best for:** Training platforms that want AI-driven programme adaptation and retention analytics, multi-trainer teams where client handoffs require complete training history, and medically-adjacent deployments needing full audit trails for exercise prescription liability.

**Trade-offs:**
- (+) Complete training history — every prescription, performance log, and adaptation preserved
- (+) AI adaptive programming trains on raw event sequences (prescription → performance → recovery → adjustment)
- (+) Programme effectiveness is a projection of prescription vs. actual events across all clients
- (+) Read models can be rebuilt or reshaped for new analytics without migrating source data
- (+) Personal records are inherently event-based — replay to recompute after exercise corrections
- (-) Higher write amplification — every exercise log writes to event store + updates read models
- (-) Eventual consistency between event store and read models
- (-) More complex application code: command handlers, event handlers, projection rebuilders
- (-) Exercise logging during a workout requires low-latency projection updates

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| HL7 FHIR R4 | Wearable data events structured for FHIR Observation export |
| FHIR Physical Activity IG | Exercise Vital Sign (LOINC 89574-8) computable from exercise and wearable events |
| Open mHealth / IEEE P1752 | Wearable sync events use Open mHealth data type names and units |
| Terra API | `wearable.synced` events carry Terra-normalized payloads |
| PCI DSS v4.0 | Payment events store processor tokens only — no cardholder data in the event store |
| GDPR | Consent events are immutable; data retention enforced via crypto-shredding |
| HIPAA | Medical intake and assessment events auditable for exercise prescription liability |
| CloudEvents | Every event follows CloudEvents attribute naming |

---

## Event Infrastructure

```sql
CREATE TABLE event_store (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_type TEXT NOT NULL,
    stream_id UUID NOT NULL,
    event_type TEXT NOT NULL,
    event_version INT NOT NULL,
    payload JSONB NOT NULL,
    metadata JSONB DEFAULT '{}',
    -- CloudEvents
    ce_source TEXT NOT NULL,
    ce_type TEXT NOT NULL,
    ce_time TIMESTAMPTZ NOT NULL DEFAULT now(),
    ce_specversion TEXT NOT NULL DEFAULT '1.0',
    -- Actor
    actor_type TEXT NOT NULL CHECK (actor_type IN ('trainer', 'client', 'system', 'ai')),
    actor_id UUID,
    team_id UUID NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_type, stream_id, event_version)
) PARTITION BY RANGE (ce_time);

CREATE INDEX idx_events_stream ON event_store(stream_type, stream_id, event_version);
CREATE INDEX idx_events_team ON event_store(team_id, ce_time);
CREATE INDEX idx_events_type ON event_store(event_type, ce_time);
CREATE INDEX idx_events_actor ON event_store(actor_id, ce_time);

-- Stream Types:
--   'client'      — onboarded, profile_updated, consented, medical_updated, assessed, goal_changed, churned
--   'programme'   — created, assigned, started, paused, completed, archived, adapted
--   'workout'     — prescribed, exercise_added, exercise_removed, reordered
--   'training'    — workout_started, exercise_logged, set_logged, personal_record_set, workout_completed, workout_skipped
--   'session'     — scheduled, confirmed, completed, cancelled, late_cancelled, no_show
--   'wearable'    — synced, recovery_scored, sleep_recorded, activity_recorded
--   'nutrition'   — plan_created, food_logged, photo_logged, macros_estimated
--   'payment'     — invoice_created, payment_attempted, payment_succeeded, payment_failed, refunded, dunning_retried
--   'package'     — activated, session_used, renewed, frozen, cancelled, expired
--   'message'     — sent, received, read, ai_sentiment_scored
--   'ai'          — churn_scored, programme_adapted, progress_narrated, workout_generated, recovery_flagged

-- Example Events:
-- {stream_type: 'training', event_type: 'training.set_logged', payload: {client_id, workout_id, exercise_id, exercise_name, set_number, reps_completed, weight_kg, rpe_actual, duration_seconds, distance_metres}}
-- {stream_type: 'training', event_type: 'training.personal_record_set', payload: {client_id, exercise_id, exercise_name, pr_type, new_value, previous_value, previous_date}}
-- {stream_type: 'training', event_type: 'training.workout_completed', payload: {client_id, workout_id, programme_id, duration_minutes, compliance_pct, mood_rating, energy_rating, exercises_completed, exercises_prescribed}}
-- {stream_type: 'programme', event_type: 'programme.adapted', payload: {client_id, programme_id, adaptations[], reason, based_on_events[], ai_model_version}}
-- {stream_type: 'wearable', event_type: 'wearable.synced', payload: {client_id, source, data_points: [{data_type, value, unit, recorded_at}], sync_id}}
-- {stream_type: 'wearable', event_type: 'wearable.recovery_scored', payload: {client_id, hrv_avg, resting_hr, sleep_hours, sleep_quality, recovery_score, source}}
-- {stream_type: 'nutrition', event_type: 'nutrition.photo_logged', payload: {client_id, meal_type, photo_path, ai_estimated_calories, ai_estimated_protein_g, ai_estimated_carbs_g, ai_estimated_fat_g}}
-- {stream_type: 'ai', event_type: 'ai.programme_adapted', payload: {client_id, programme_id, exercise_id, old_prescription{}, new_prescription{}, reason, hrv_trend, compliance_trend, recovery_score}}
-- {stream_type: 'ai', event_type: 'ai.churn_scored', payload: {client_id, score, factors[], model_version, based_on_event_count}}
-- {stream_type: 'ai', event_type: 'ai.progress_narrated', payload: {client_id, period_start, period_end, narrative_text, prs_count, compliance_pct, weight_change_kg, key_metrics{}}}
-- {stream_type: 'ai', event_type: 'ai.recovery_flagged', payload: {client_id, hrv_drop_pct, sleep_deficit_hours, recommendation, affected_workout_id}}

CREATE TABLE stream_snapshots (
    stream_type TEXT NOT NULL,
    stream_id UUID NOT NULL,
    snapshot_version INT NOT NULL,
    state JSONB NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (stream_type, stream_id, snapshot_version)
);

CREATE TABLE projection_checkpoints (
    projection_name TEXT PRIMARY KEY,
    last_event_id UUID NOT NULL,
    last_event_time TIMESTAMPTZ NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Read Models

### Clients

```sql
CREATE TABLE rm_clients (
    id UUID NOT NULL,
    team_id UUID NOT NULL,
    trainer_id TEXT NOT NULL,
    email TEXT,
    full_name TEXT NOT NULL,
    phone TEXT,
    status TEXT NOT NULL,
    client_type TEXT,
    -- Profile snapshot
    profile JSONB DEFAULT '{}',
    -- Medical snapshot
    medical_intake JSONB DEFAULT '{}',
    -- Goals
    primary_goal TEXT,
    secondary_goals TEXT[] DEFAULT '{}',
    -- Active programme summary
    active_programme JSONB,
    -- active_programme: {programme_id, name, type, status, start_date, weeks_completed,
    --   total_workouts, completed_workouts, compliance_pct, ai_adaptation_enabled}
    -- Training summary (materialised from training events)
    total_workouts_completed INT DEFAULT 0,
    workouts_this_week INT DEFAULT 0,
    workouts_last_30_days INT DEFAULT 0,
    avg_workouts_per_week NUMERIC(4,2),
    last_workout_at TIMESTAMPTZ,
    days_since_last_workout INT,
    avg_compliance_pct NUMERIC(5,2),
    -- Session summary
    total_sessions INT DEFAULT 0,
    sessions_this_month INT DEFAULT 0,
    last_session_at TIMESTAMPTZ,
    no_show_count INT DEFAULT 0,
    late_cancel_count INT DEFAULT 0,
    -- Personal records
    personal_records JSONB DEFAULT '{}',
    -- personal_records: {exercise_id: {exercise_name, weight_kg, reps, volume, date}, ...}
    recent_prs JSONB DEFAULT '[]',
    -- recent_prs[]: [{exercise_name, pr_type, new_value, previous_value, date}]
    -- Wearable summary
    wearable_summary JSONB DEFAULT '{}',
    -- wearable_summary: {sources[], resting_hr, hrv_avg_7d, hrv_trend,
    --   recovery_score, sleep_avg_hours_7d, vo2_max, last_synced_at}
    -- Body composition (from progress events)
    current_weight_kg NUMERIC(5,1),
    weight_change_kg NUMERIC(5,1),
    body_fat_pct NUMERIC(5,2),
    -- Nutrition compliance
    nutrition_compliance JSONB DEFAULT '{}',
    -- nutrition_compliance: {avg_calories_7d, target_calories, protein_adherence_pct,
    --   food_logs_this_week, photo_logs_this_week}
    -- Financial summary
    active_package JSONB,
    lifetime_revenue_cents BIGINT DEFAULT 0,
    outstanding_balance_cents BIGINT DEFAULT 0,
    failed_payment_count INT DEFAULT 0,
    -- AI
    ai_churn_score NUMERIC(5,2),
    ai_churn_factors JSONB,
    ai_compliance_trend TEXT,
    ai_last_scored_at TIMESTAMPTZ,
    ai_next_session_predicted DATE,
    -- Engagement
    last_message_at TIMESTAMPTZ,
    last_app_login_at TIMESTAMPTZ,
    engagement_streak_days INT DEFAULT 0,
    -- Privacy
    consent_status TEXT,
    data_retention_until DATE,
    tags TEXT[] DEFAULT '{}',
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (id)
);
CREATE INDEX idx_rm_clients_team ON rm_clients(team_id);
CREATE INDEX idx_rm_clients_trainer ON rm_clients(trainer_id);
CREATE INDEX idx_rm_clients_status ON rm_clients(team_id, status);
CREATE INDEX idx_rm_clients_churn ON rm_clients(ai_churn_score) WHERE status = 'active';
CREATE INDEX idx_rm_clients_last_workout ON rm_clients(last_workout_at) WHERE status = 'active';
```

### Programme Performance

```sql
CREATE TABLE rm_programme_performance (
    team_id UUID NOT NULL,
    programme_id UUID NOT NULL,
    programme_name TEXT,
    programme_type TEXT,
    trainer_id TEXT,
    trainer_name TEXT,
    is_template BOOLEAN DEFAULT FALSE,
    -- Assignments
    total_assignments INT DEFAULT 0,
    active_assignments INT DEFAULT 0,
    completed_assignments INT DEFAULT 0,
    -- Compliance (materialised from training events)
    avg_compliance_pct NUMERIC(5,2),
    avg_completion_days INT,
    dropout_count INT DEFAULT 0,
    dropout_rate NUMERIC(5,2),
    -- Outcomes (materialised from progress events)
    avg_strength_gain_pct NUMERIC(5,2),
    avg_weight_change_kg NUMERIC(5,1),
    prs_generated INT DEFAULT 0,
    -- Per-exercise effectiveness
    exercise_stats JSONB DEFAULT '[]',
    -- exercise_stats[]: [{exercise_id, exercise_name, avg_prescribed_weight,
    --   avg_actual_weight, avg_rpe, progression_rate_pct, skip_rate}]
    -- AI insights
    ai_effectiveness_score NUMERIC(5,2),
    ai_recommended_adjustments JSONB,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (team_id, programme_id)
);
CREATE INDEX idx_rm_programme_type ON rm_programme_performance(programme_type);
```

### Schedule

```sql
CREATE TABLE rm_schedule (
    team_id UUID NOT NULL,
    trainer_id TEXT NOT NULL,
    schedule_date DATE NOT NULL,
    -- Sessions (materialised from session events)
    sessions JSONB DEFAULT '[]',
    -- sessions[]: [{id, client_id, client_name, session_type, starts_at, ends_at,
    --   status, location, video_link, workout_name, notes}]
    -- Workout assignments due (materialised from programme events)
    workouts_due JSONB DEFAULT '[]',
    -- workouts_due[]: [{client_id, client_name, programme_name, workout_name,
    --   workout_id, status, completed_at}]
    -- Day metrics
    total_sessions INT DEFAULT 0,
    completed_sessions INT DEFAULT 0,
    cancelled_sessions INT DEFAULT 0,
    no_shows INT DEFAULT 0,
    clients_with_workouts_due INT DEFAULT 0,
    workouts_completed_today INT DEFAULT 0,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (team_id, trainer_id, schedule_date)
);
```

### Revenue

```sql
CREATE TABLE rm_revenue (
    team_id UUID NOT NULL,
    period_date DATE NOT NULL,
    -- Package revenue
    package_revenue_cents BIGINT DEFAULT 0,
    new_packages INT DEFAULT 0,
    renewed_packages INT DEFAULT 0,
    cancelled_packages INT DEFAULT 0,
    active_packages INT DEFAULT 0,
    -- Session revenue
    drop_in_revenue_cents BIGINT DEFAULT 0,
    cancellation_fee_cents BIGINT DEFAULT 0,
    -- Total
    total_revenue_cents BIGINT DEFAULT 0,
    -- Payment health
    failed_payments INT DEFAULT 0,
    failed_amount_cents BIGINT DEFAULT 0,
    recovered_amount_cents BIGINT DEFAULT 0,
    outstanding_balance_cents BIGINT DEFAULT 0,
    -- Refunds
    refund_count INT DEFAULT 0,
    refund_amount_cents BIGINT DEFAULT 0,
    -- Per-trainer breakdown
    trainer_revenue JSONB DEFAULT '[]',
    -- trainer_revenue[]: [{trainer_id, trainer_name, sessions_completed, session_revenue_cents,
    --   package_revenue_cents, total_revenue_cents, clients_active}]
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (team_id, period_date)
);
```

### Team Dashboard

```sql
CREATE TABLE rm_team_dashboard (
    team_id UUID NOT NULL,
    period_date DATE NOT NULL,
    -- Clients
    total_active_clients INT DEFAULT 0,
    new_clients_this_week INT DEFAULT 0,
    churned_clients_this_week INT DEFAULT 0,
    at_risk_clients INT DEFAULT 0,
    clients_without_programme INT DEFAULT 0,
    -- Training
    workouts_completed_today INT DEFAULT 0,
    avg_daily_workouts NUMERIC(5,2),
    avg_compliance_pct NUMERIC(5,2),
    prs_this_week INT DEFAULT 0,
    -- Sessions
    sessions_today INT DEFAULT 0,
    sessions_this_week INT DEFAULT 0,
    no_show_rate_pct NUMERIC(5,2),
    -- Revenue
    revenue_mtd_cents BIGINT DEFAULT 0,
    revenue_target_cents BIGINT DEFAULT 0,
    outstanding_balance_cents BIGINT DEFAULT 0,
    failed_payments_pending INT DEFAULT 0,
    -- Nutrition
    clients_logging_food INT DEFAULT 0,
    avg_calorie_adherence_pct NUMERIC(5,2),
    -- Wearable
    clients_with_wearable INT DEFAULT 0,
    low_recovery_alerts INT DEFAULT 0,
    -- AI
    ai_churn_alerts INT DEFAULT 0,
    ai_programme_adaptations INT DEFAULT 0,
    ai_progress_narratives_sent INT DEFAULT 0,
    ai_recovery_flags INT DEFAULT 0,
    -- Trainer leaderboard
    trainer_leaderboard JSONB DEFAULT '[]',
    -- trainer_leaderboard[]: [{name, active_clients, sessions_completed,
    --   avg_compliance_pct, retention_rate_pct}]
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (team_id, period_date)
);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Infrastructure | 3 | event_store (partitioned), stream_snapshots, projection_checkpoints |
| Read Model: Clients | 1 | rm_clients (training stats, PRs, wearable, nutrition, financials, AI scores) |
| Read Model: Programmes | 1 | rm_programme_performance (compliance, outcomes, per-exercise effectiveness) |
| Read Model: Schedule | 1 | rm_schedule (daily sessions and workout assignments per trainer) |
| Read Model: Revenue | 1 | rm_revenue (daily financial aggregation with per-trainer breakdown) |
| Read Model: Dashboard | 1 | rm_team_dashboard (daily KPIs) |
| **Total** | **8** | 3 infrastructure + 5 read models |

---

## Key Design Decisions

1. **Training events as the primary progression signal** — every `training.set_logged` event captures what the client actually did against what was prescribed. The AI adaptive programming engine replays the training event sequence for a client and exercise to detect plateaus (same weight/reps for 3+ sessions), consistent RPE undershoot (prescribed RPE 8, actual RPE 6 → weight increase), and recovery-limited performance (high RPE + low HRV → deload). This is the core AI training loop.

2. **Personal records from event replay** — `training.personal_record_set` events are emitted when a new set exceeds all previous sets for that client and exercise. The `rm_clients` read model maintains `personal_records` as a JSONB map and `recent_prs` as the last N records. If an exercise log is corrected, the PR can be recomputed by replaying the training stream — something impossible with a mutable flag.

3. **Programme adaptation as events** — when the AI adjusts a workout prescription (increase weight, reduce volume, swap exercise), it writes a `programme.adapted` event with the old prescription, new prescription, reason, and the events that informed the decision (HRV trend, compliance pattern, RPE history). This creates an auditable trail of AI decisions that the trainer can review and override.

4. **Wearable data as sync events** — `wearable.synced` events carry batches of data points from Terra or direct integrations. `wearable.recovery_scored` events summarise the recovery signal (HRV, sleep, resting HR) into an actionable score. The `rm_clients` read model surfaces the latest recovery state. The AI recovery engine consumes these events to flag clients who should deload before their next session.

5. **Programme effectiveness from event projections** — the `rm_programme_performance` read model aggregates training events across all clients who used a programme template: average compliance, dropout rate, strength gains, PRs generated, and per-exercise progression rates. This enables data-driven programme design — the trainer sees which programme templates produce the best outcomes.

6. **Churn scoring from event patterns** — the AI system analyses the full event sequence per client: workout completion frequency trend, session attendance gaps, payment failures, app login frequency, message response times, and wearable sync gaps. Scores are written as `ai.churn_scored` events with contributing factors, creating an audit trail of predictions that can be compared against actual churn.

7. **Automated progress narratives from events** — `ai.progress_narrated` events are generated by replaying the client's recent event history. The narrative summarises PRs, compliance trends, body composition changes, and wearable improvements into natural language. This powers the automated check-in feature — the AI writes the first draft of the weekly progress summary.

8. **Recovery-aware scheduling from wearable events** — `ai.recovery_flagged` events fire when the intersection of `wearable.recovery_scored` events shows concerning trends (HRV drop > 15%, sleep deficit > 2 hours for 3+ days). The flag references the affected workout and recommends a deload or rest day. The `rm_schedule` read model surfaces these flags on the trainer's daily view.

9. **Nutrition events for coaching, not tracking** — `nutrition.food_logged` and `nutrition.photo_logged` events capture what the client ate. The `rm_clients` read model materialises `nutrition_compliance` with rolling averages. The AI doesn't micro-manage macros — it watches for patterns (consistently low protein, skipping meals before training) and alerts the trainer.

10. **GDPR via crypto-shredding** — when a client exercises their right to erasure, a `client.data_shredded` event replaces PII in previous events with tombstone markers. The event structure (timestamps, exercise types, performance data) is preserved for aggregate programme effectiveness analytics while individual identity is removed.

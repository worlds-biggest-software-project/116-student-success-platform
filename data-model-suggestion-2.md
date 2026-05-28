# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: Student Success Platform · Created: 2026-05-19

## Philosophy

The event-sourced model treats every change to a student's record as an immutable event appended to a log. The event store is the single source of truth; all queryable state (student profiles, risk dashboards, advisor caseloads) is derived by replaying or projecting events into materialised read models. This is the CQRS (Command Query Responsibility Segregation) pattern: writes go to the event store, reads come from purpose-built projections.

This approach is deeply aligned with FERPA's audit requirements. Instead of bolting an audit log onto a relational model, the audit trail *is* the data model. Every access, every grade change, every alert flag, every advising note — all are events with full provenance. The Department of Education's March 2025 enforcement guidance requiring timestamped access logging with minimum three-year retention is satisfied by the fundamental architecture, not an afterthought.

Event sourcing is used in banking (every transaction is an event), healthcare (HL7 FHIR event logs), and compliance-heavy SaaS platforms. For a student success platform, it enables temporal queries that existing products cannot easily answer: "What was this student's risk profile on October 15th?", "Show me all interventions that were active when the student's GPA dropped below 2.0", or "Replay the alert history for this cohort to evaluate whether the new risk model would have flagged them earlier."

**Best for:** Institutions that require bulletproof audit trails, temporal analysis of student trajectories, and the ability to replay history for model training and fairness audits.

**Trade-offs:**
- Pro: 100% reliable, immutable audit trail — FERPA compliance is built in, not bolted on
- Pro: Temporal queries are native — "what was true at time T?" is trivial
- Pro: New read models can be built retroactively by replaying events — ideal for evolving analytics
- Pro: Model fairness audits can replay risk scoring history to check for demographic bias
- Con: Higher complexity — developers must think in events rather than CRUD operations
- Con: Event schema evolution requires careful versioning (upcasters)
- Con: Read model consistency is eventually consistent, not immediate
- Con: Event store grows continuously; requires compaction or snapshotting strategy
- Con: More infrastructure — needs event store + projection engine + read database

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| IMS Caliper 1.2 | Caliper events are natively event-structured; ingested directly into the event store without transformation |
| xAPI | xAPI statements (actor-verb-object) are already events; stored as-is in the event store |
| FERPA | The event store IS the audit trail — every data access and modification is an event with full provenance |
| HIPAA | Counselling-related events carry a `restricted` classification; projections enforce access filtering |
| GDPR Art. 22 | Human review events recorded before any AI-triggered intervention is sent to a student |
| OneRoster 1.2 | Roster sync events capture create/update/delete operations from SIS integration |
| OAuth 2.0 / OIDC | Authentication events (login, logout, token refresh) stored in the event stream |

---

## Event Store

```sql
-- The single source of truth: an append-only, immutable event log
CREATE TABLE events (
    event_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id UUID NOT NULL,          -- aggregate root ID (student, advisor, course, institution)
    stream_type VARCHAR(50) NOT NULL, -- 'student', 'alert', 'appointment', 'course_section', 'campaign'
    event_type VARCHAR(100) NOT NULL, -- 'StudentEnrolled', 'AlertFlagRaised', 'RiskScoreComputed', etc.
    event_version INTEGER NOT NULL,   -- sequence number within the stream
    institution_id UUID NOT NULL,
    actor_id UUID NOT NULL,           -- who or what caused this event (user ID or system ID)
    actor_type VARCHAR(20) NOT NULL,  -- 'user', 'system', 'ai_agent', 'integration'
    payload JSONB NOT NULL,           -- event-specific data
    metadata JSONB NOT NULL DEFAULT '{}',
    -- metadata example:
    -- {
    --   "ip_address": "192.168.1.100",
    --   "user_agent": "Mozilla/5.0...",
    --   "correlation_id": "abc-123",
    --   "source_system": "banner_sis",
    --   "ferpa_classification": "education_record"
    -- }
    occurred_at TIMESTAMPTZ NOT NULL,
    recorded_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_id, event_version)
) PARTITION BY RANGE (occurred_at);

-- Monthly partitions for manageable storage
CREATE TABLE events_2026_01 PARTITION OF events
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
-- ... additional monthly partitions

CREATE INDEX idx_events_stream ON events(stream_id, event_version);
CREATE INDEX idx_events_type ON events(event_type, occurred_at);
CREATE INDEX idx_events_institution ON events(institution_id, occurred_at);
CREATE INDEX idx_events_actor ON events(actor_id, occurred_at);

-- Snapshots for performance: periodically capture aggregate state to avoid full replay
CREATE TABLE snapshots (
    stream_id UUID NOT NULL,
    stream_type VARCHAR(50) NOT NULL,
    snapshot_version INTEGER NOT NULL,
    state JSONB NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (stream_id, snapshot_version)
);
```

## Event Type Catalogue

The following event types define the domain language of the platform. Each event type has a defined payload schema.

```sql
-- Reference table documenting all event types and their payload schemas
CREATE TABLE event_type_registry (
    event_type VARCHAR(100) PRIMARY KEY,
    category VARCHAR(30) NOT NULL,  -- 'student', 'academic', 'alert', 'advising', 'risk', 'ai', 'integration', 'access'
    description TEXT NOT NULL,
    payload_schema JSONB NOT NULL,  -- JSON Schema defining the payload structure
    version INTEGER NOT NULL DEFAULT 1,
    introduced_at DATE NOT NULL,
    deprecated_at DATE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Example event types and their payloads:
--
-- StudentRegistered:
--   { "student_id": "uuid", "sis_id": "B00123456", "first_name": "Jane", "last_name": "Doe",
--     "email": "jdoe@uni.edu", "admission_type": "transfer", "programme_code": "CS-BS" }
--
-- EnrollmentCreated:
--   { "enrollment_id": "uuid", "course_section_id": "uuid", "course_code": "MATH-201",
--     "term": "Fall 2026", "credits": 3.0 }
--
-- GradeRecorded:
--   { "enrollment_id": "uuid", "grade_type": "midterm", "grade": "B+",
--     "grade_points": 3.3, "previous_grade": null }
--
-- AlertFlagRaised:
--   { "alert_id": "uuid", "flag_type": "academic", "severity": "high",
--     "course_section_id": "uuid", "title": "Missing 3 consecutive assignments",
--     "description": "..." }
--
-- AlertFlagResolved:
--   { "alert_id": "uuid", "resolution_notes": "Student met with tutor and submitted work",
--     "outcome": "successful" }
--
-- RiskScoreComputed:
--   { "model_id": "uuid", "model_version": "2.3", "overall_score": 0.78,
--     "risk_level": "high", "factors": { "gpa_trajectory": 0.72, "lms_engagement": 0.45 } }
--
-- AppointmentScheduled:
--   { "appointment_id": "uuid", "advisor_id": "uuid", "type": "advising",
--     "scheduled_start": "2026-10-15T14:00:00Z", "mode": "virtual" }
--
-- AdvisingNoteCreated:
--   { "note_id": "uuid", "note_type": "advising", "content": "Discussed course load...",
--     "is_restricted": false }
--
-- ChatbotConversationStarted:
--   { "conversation_id": "uuid", "initial_query": "What courses do I need for my minor?" }
--
-- ChatbotEscalated:
--   { "conversation_id": "uuid", "escalated_to": "uuid", "reason": "distress_detected",
--     "sentiment_score": -0.82 }
--
-- InterventionRecommended:
--   { "recommendation_id": "uuid", "intervention_type": "tutoring_referral",
--     "confidence": 0.87, "peer_success_rate": 0.73, "requires_human_review": true }
--
-- StudentRecordAccessed:
--   { "resource_type": "student_profile", "resource_id": "uuid", "access_type": "view",
--     "fields_accessed": ["gpa", "enrollment_status", "risk_score"] }
--
-- CaliperEventReceived:
--   { "caliper_profile": "AssessmentProfile", "event_type": "AssessmentEvent",
--     "action": "Submitted", "object_id": "urn:canvas:quiz:4567", "result_score": 85.0 }
--
-- TransferCreditEvaluated:
--   { "transfer_id": "uuid", "source_institution": "Community College X",
--     "source_course": "ENG 101", "equivalent_course_id": "uuid",
--     "ai_confidence": 0.92, "evaluator_decision": "approved" }
```

## Read Model Projections (Materialised Views)

Read models are built by event processors that subscribe to the event store and maintain queryable tables. These tables are *derived* — they can be rebuilt from scratch by replaying events.

```sql
-- ============================================================
-- PROJECTION: Current Student State
-- Built by: StudentProjector (subscribes to Student* events)
-- ============================================================
CREATE TABLE rm_students (
    student_id UUID PRIMARY KEY,
    institution_id UUID NOT NULL,
    sis_id VARCHAR(100),
    email VARCHAR(255) NOT NULL,
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    preferred_name VARCHAR(100),
    user_type VARCHAR(20) NOT NULL DEFAULT 'student',
    admission_type VARCHAR(30),
    enrollment_status VARCHAR(30),
    academic_standing VARCHAR(30),
    cumulative_gpa NUMERIC(4, 3),
    total_credits_earned NUMERIC(6, 2),
    current_risk_level VARCHAR(20),
    current_risk_score NUMERIC(5, 4),
    primary_advisor_id UUID,
    primary_programme_code VARCHAR(30),
    expected_graduation_term VARCHAR(50),
    last_login_at TIMESTAMPTZ,
    last_event_version INTEGER NOT NULL,
    projected_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_students_institution ON rm_students(institution_id);
CREATE INDEX idx_rm_students_risk ON rm_students(institution_id, current_risk_level);
CREATE INDEX idx_rm_students_standing ON rm_students(institution_id, academic_standing);
CREATE INDEX idx_rm_students_advisor ON rm_students(primary_advisor_id);

-- ============================================================
-- PROJECTION: Advisor Caseload Dashboard
-- Built by: AdvisorProjector
-- ============================================================
CREATE TABLE rm_advisor_caseload (
    advisor_id UUID NOT NULL,
    institution_id UUID NOT NULL,
    total_advisees INTEGER NOT NULL DEFAULT 0,
    high_risk_count INTEGER NOT NULL DEFAULT 0,
    open_alerts_count INTEGER NOT NULL DEFAULT 0,
    upcoming_appointments_count INTEGER NOT NULL DEFAULT 0,
    avg_response_time_hours NUMERIC(8, 2),
    projected_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (advisor_id)
);

-- ============================================================
-- PROJECTION: Active Alerts
-- Built by: AlertProjector (subscribes to AlertFlag* events)
-- ============================================================
CREATE TABLE rm_active_alerts (
    alert_id UUID PRIMARY KEY,
    institution_id UUID NOT NULL,
    student_id UUID NOT NULL,
    raised_by UUID NOT NULL,
    assigned_to UUID,
    flag_type VARCHAR(30) NOT NULL,
    severity VARCHAR(20) NOT NULL,
    title VARCHAR(255) NOT NULL,
    status VARCHAR(20) NOT NULL,
    course_section_id UUID,
    course_code VARCHAR(30),
    raised_at TIMESTAMPTZ NOT NULL,
    last_updated_at TIMESTAMPTZ NOT NULL,
    projected_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_alerts_assigned ON rm_active_alerts(assigned_to, status);
CREATE INDEX idx_rm_alerts_student ON rm_active_alerts(student_id);
CREATE INDEX idx_rm_alerts_institution ON rm_active_alerts(institution_id, status, severity);

-- ============================================================
-- PROJECTION: Degree Progress
-- Built by: DegreeProjector (subscribes to Enrollment*, Grade*, TransferCredit* events)
-- ============================================================
CREATE TABLE rm_degree_progress (
    student_id UUID NOT NULL,
    programme_id UUID NOT NULL,
    programme_name VARCHAR(255) NOT NULL,
    total_credits_required NUMERIC(6, 2),
    credits_completed NUMERIC(6, 2) NOT NULL DEFAULT 0,
    credits_in_progress NUMERIC(6, 2) NOT NULL DEFAULT 0,
    credits_remaining NUMERIC(6, 2) NOT NULL DEFAULT 0,
    requirements_met INTEGER NOT NULL DEFAULT 0,
    requirements_total INTEGER NOT NULL DEFAULT 0,
    completion_percentage NUMERIC(5, 2) NOT NULL DEFAULT 0,
    projected_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (student_id, programme_id)
);

-- ============================================================
-- PROJECTION: Student Timeline (denormalised for UI rendering)
-- Built by: TimelineProjector (subscribes to all student-related events)
-- ============================================================
CREATE TABLE rm_student_timeline (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    student_id UUID NOT NULL,
    institution_id UUID NOT NULL,
    event_type VARCHAR(100) NOT NULL,
    category VARCHAR(30) NOT NULL,  -- 'academic', 'alert', 'advising', 'engagement', 'financial'
    title VARCHAR(255) NOT NULL,
    description TEXT,
    actor_name VARCHAR(200),
    is_restricted BOOLEAN NOT NULL DEFAULT FALSE,
    occurred_at TIMESTAMPTZ NOT NULL,
    projected_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_timeline_student ON rm_student_timeline(student_id, occurred_at DESC);

-- ============================================================
-- PROJECTION: Risk Score History (for temporal analysis and bias audits)
-- Built by: RiskProjector (subscribes to RiskScoreComputed events)
-- ============================================================
CREATE TABLE rm_risk_history (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    student_id UUID NOT NULL,
    institution_id UUID NOT NULL,
    model_id UUID NOT NULL,
    model_version VARCHAR(20) NOT NULL,
    risk_level VARCHAR(20) NOT NULL,
    overall_score NUMERIC(5, 4) NOT NULL,
    factor_scores JSONB,
    computed_at TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_rm_risk_history_student ON rm_risk_history(student_id, computed_at);
CREATE INDEX idx_rm_risk_history_model ON rm_risk_history(model_id, computed_at);

-- ============================================================
-- PROJECTION: Engagement Metrics (aggregated from Caliper/xAPI events)
-- Built by: EngagementProjector
-- ============================================================
CREATE TABLE rm_engagement_metrics (
    student_id UUID NOT NULL,
    course_section_id UUID NOT NULL,
    institution_id UUID NOT NULL,
    week_start DATE NOT NULL,
    lms_login_count INTEGER NOT NULL DEFAULT 0,
    assignment_submissions INTEGER NOT NULL DEFAULT 0,
    discussion_posts INTEGER NOT NULL DEFAULT 0,
    content_views INTEGER NOT NULL DEFAULT 0,
    total_active_minutes INTEGER NOT NULL DEFAULT 0,
    last_activity_at TIMESTAMPTZ,
    projected_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (student_id, course_section_id, week_start)
);
```

## Command Handlers

Commands are the write side of CQRS. Each command validates business rules and emits events.

```sql
-- Command log for debugging and replay (optional — events are the source of truth)
CREATE TABLE command_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    command_type VARCHAR(100) NOT NULL,  -- 'RaiseAlertFlag', 'ScheduleAppointment', 'RecordGrade'
    actor_id UUID NOT NULL,
    institution_id UUID NOT NULL,
    payload JSONB NOT NULL,
    status VARCHAR(20) NOT NULL,  -- 'accepted', 'rejected', 'failed'
    rejection_reason TEXT,
    events_emitted UUID[],  -- array of event_ids produced by this command
    received_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    processed_at TIMESTAMPTZ
);

CREATE INDEX idx_command_log_actor ON command_log(actor_id, received_at);
```

## Reference Data (Non-Event-Sourced)

Some data is static reference data that does not benefit from event sourcing.

```sql
CREATE TABLE institutions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    short_code VARCHAR(50) NOT NULL UNIQUE,
    timezone VARCHAR(50) NOT NULL DEFAULT 'America/New_York',
    country_code CHAR(2) NOT NULL DEFAULT 'US',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE academic_terms (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    institution_id UUID NOT NULL REFERENCES institutions(id),
    name VARCHAR(100) NOT NULL,
    term_type VARCHAR(20) NOT NULL,
    start_date DATE NOT NULL,
    end_date DATE NOT NULL,
    census_date DATE,
    is_current BOOLEAN NOT NULL DEFAULT FALSE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE courses (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    institution_id UUID NOT NULL REFERENCES institutions(id),
    subject_code VARCHAR(10) NOT NULL,
    course_number VARCHAR(20) NOT NULL,
    title VARCHAR(255) NOT NULL,
    credits NUMERIC(4, 2) NOT NULL,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (institution_id, subject_code, course_number)
);

CREATE TABLE risk_models (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    institution_id UUID NOT NULL REFERENCES institutions(id),
    name VARCHAR(255) NOT NULL,
    version VARCHAR(20) NOT NULL,
    model_type VARCHAR(30) NOT NULL,
    feature_set TEXT[],
    auc_score NUMERIC(5, 4),
    fairness_audit_date DATE,
    is_active BOOLEAN NOT NULL DEFAULT FALSE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Projection Infrastructure

```sql
-- Tracks the position of each projector in the event stream
CREATE TABLE projector_checkpoints (
    projector_name VARCHAR(100) PRIMARY KEY,
    last_event_id UUID,
    last_occurred_at TIMESTAMPTZ,
    events_processed BIGINT NOT NULL DEFAULT 0,
    last_error TEXT,
    status VARCHAR(20) NOT NULL DEFAULT 'running',  -- 'running', 'paused', 'rebuilding', 'error'
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Dead letter queue for events that failed projection
CREATE TABLE projection_failures (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    projector_name VARCHAR(100) NOT NULL,
    event_id UUID NOT NULL,
    error_message TEXT NOT NULL,
    retry_count INTEGER NOT NULL DEFAULT 0,
    max_retries INTEGER NOT NULL DEFAULT 5,
    next_retry_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Example Queries

### Temporal query: "What was the student's risk level on a specific date?"

```sql
-- Replay risk events up to the target date to find the most recent risk score
SELECT payload->>'risk_level' AS risk_level,
       payload->>'overall_score' AS score,
       occurred_at
FROM events
WHERE stream_id = 'student-uuid-here'
  AND event_type = 'RiskScoreComputed'
  AND occurred_at <= '2026-10-15T23:59:59Z'
ORDER BY occurred_at DESC
LIMIT 1;
```

### Audit query: "Who accessed this student's record in the last 30 days?"

```sql
SELECT actor_id,
       event_type,
       payload->>'access_type' AS access_type,
       payload->>'fields_accessed' AS fields,
       metadata->>'ip_address' AS ip_address,
       occurred_at
FROM events
WHERE stream_id = 'student-uuid-here'
  AND event_type = 'StudentRecordAccessed'
  AND occurred_at >= now() - INTERVAL '30 days'
ORDER BY occurred_at DESC;
```

### Fairness audit: "Compare average risk scores by demographic group for a model version"

```sql
SELECT sp.ethnicity,
       AVG((e.payload->>'overall_score')::numeric) AS avg_risk_score,
       COUNT(*) AS student_count
FROM events e
JOIN rm_students sp ON sp.student_id = e.stream_id
WHERE e.event_type = 'RiskScoreComputed'
  AND e.payload->>'model_version' = '2.3'
  AND e.occurred_at BETWEEN '2026-09-01' AND '2026-12-31'
GROUP BY sp.ethnicity
ORDER BY avg_risk_score DESC;
```

### Replay: "Rebuild the student timeline projection from scratch"

```sql
-- 1. Reset the projector checkpoint
UPDATE projector_checkpoints
SET last_event_id = NULL,
    last_occurred_at = NULL,
    events_processed = 0,
    status = 'rebuilding'
WHERE projector_name = 'TimelineProjector';

-- 2. Truncate the read model
TRUNCATE rm_student_timeline;

-- 3. The projection engine replays all events from the beginning
-- (handled by application code, not SQL)
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 2 | events (partitioned), snapshots |
| Event Infrastructure | 3 | event_type_registry, command_log, projector_checkpoints + projection_failures |
| Reference Data | 4 | institutions, academic_terms, courses, risk_models |
| Read Model: Students | 1 | rm_students |
| Read Model: Alerts | 1 | rm_active_alerts |
| Read Model: Advising | 1 | rm_advisor_caseload |
| Read Model: Degree | 1 | rm_degree_progress |
| Read Model: Timeline | 1 | rm_student_timeline |
| Read Model: Risk | 1 | rm_risk_history |
| Read Model: Engagement | 1 | rm_engagement_metrics |
| **Total** | **16** | Plus partitions; read models can be added without schema migration |

---

## Key Design Decisions

1. **Events are the source of truth, not read models** — read model tables (prefixed `rm_`) are derived and disposable. If a projection is corrupted or a new reporting need arises, the projection is rebuilt by replaying the event store. This eliminates the "we didn't think to log that" problem.

2. **Single `events` table with JSONB payload** — rather than a table per event type, all events share one table. This simplifies the event store infrastructure and makes cross-type queries (timeline, audit) straightforward. The `event_type_registry` table documents the schema for each event type's payload.

3. **FERPA audit is free** — every `StudentRecordAccessed` event captures who viewed what, when, from where. There is no separate audit log to maintain or worry about gaps in coverage. The event store IS the audit log.

4. **Caliper and xAPI events stored natively** — learning events from LMS platforms are already event-structured. They flow directly into the event store as `CaliperEventReceived` or `XapiStatementReceived` events without transformation, preserving the original payload.

5. **Snapshots for performance** — for students with thousands of events over multiple years, full replay is expensive. Periodic snapshots capture the current aggregate state so replay only needs to process events since the last snapshot.

6. **Projector checkpoints enable reliable rebuilds** — each projector tracks its position in the event stream. If a projector crashes, it resumes from its last checkpoint. If a new read model is needed, a new projector starts from the beginning.

7. **Reference data is not event-sourced** — institutions, academic terms, and course catalogues change infrequently and don't benefit from event sourcing. They use standard relational tables to keep the model pragmatic.

8. **Command log separates intent from outcome** — the command_log captures what was requested; the events table captures what happened. A rejected command (e.g., "schedule appointment outside office hours") is logged with its rejection reason but produces no events.

9. **Event-driven model fairness auditing** — because every risk score computation is an event with the model version and all factor scores, fairness audits can replay historical scoring to check for demographic bias across any time period or model version without maintaining separate analytics tables.

10. **Eventually consistent read models are acceptable** — advising dashboards, risk scores, and student timelines do not require real-time consistency. A sub-second delay between event publication and projection update is acceptable for this domain, and the trade-off enables much simpler scaling of the write path.

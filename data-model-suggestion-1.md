# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Student Success Platform · Created: 2026-05-19

## Philosophy

The entity-centric normalized relational model follows the classical approach of defining a separate table for every domain concept, connected by explicit foreign key relationships. Every fact is stored exactly once, referential integrity is enforced at the database level, and complex queries are composed through JOINs. This approach mirrors how products like Ellucian CRM Advise and Salesforce Education Cloud structure their data — with dedicated objects for students, advisors, appointments, alerts, degree programmes, and course enrollments.

This model prioritises data integrity, clear entity ownership, and predictable query patterns. It is the most natural fit for teams experienced with relational databases and ORMs (Django, Rails, Prisma). The trade-off is a higher table count and more JOIN-heavy queries, but PostgreSQL handles this efficiently with proper indexing.

For a student success platform that must integrate with SIS, LMS, and financial aid systems — each with its own entity structure — a normalised model provides the clearest mapping between external data sources and internal tables. OneRoster 1.2's six core entities (organizations, users, courses, classes, enrollments, demographics) map directly to normalised tables.

**Best for:** Institutions that need strong data integrity, complex cross-entity reporting, and clear FERPA audit boundaries.

**Trade-offs:**
- Pro: Strongest referential integrity; no data duplication
- Pro: Simplest to reason about for compliance audits (each table has a clear owner)
- Pro: Well-supported by every ORM, reporting tool, and BI platform
- Con: High table count (~45-55 tables) increases schema complexity
- Con: JOIN-heavy queries can become slow without careful indexing
- Con: Schema migrations required for every new field — less flexible for institution-specific customisation
- Con: Temporal queries ("what was true on date X?") require additional versioning tables

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| OneRoster 1.2 | Six core entities (orgs, users, courses, classes, enrollments, demographics) map directly to normalised tables |
| IMS Caliper 1.2 | Learning events stored in a dedicated `learning_events` table with profile-specific columns |
| xAPI | Statements stored with actor/verb/object decomposed into foreign keys |
| FERPA | Separate `audit_log` table records every access/modification with user, timestamp, IP, and action |
| ISO 3166-1/2 | `jurisdictions` reference table uses standard country and subdivision codes |
| WCAG 2.2 | No direct schema impact; accessibility is a UI concern |
| OAuth 2.0 / OIDC | `oauth_sessions` and `identity_providers` tables for SSO integration |
| SCIM 2.0 | User provisioning maps to `users` and `user_roles` tables |

---

## Core Identity & Multi-Tenancy

```sql
CREATE TABLE institutions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    short_code VARCHAR(50) NOT NULL UNIQUE,
    timezone VARCHAR(50) NOT NULL DEFAULT 'America/New_York',
    country_code CHAR(2) NOT NULL DEFAULT 'US',  -- ISO 3166-1 alpha-2
    ferpa_agreement_date DATE,
    hipaa_baa_date DATE,
    gdpr_applicable BOOLEAN NOT NULL DEFAULT FALSE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    institution_id UUID NOT NULL REFERENCES institutions(id),
    external_sis_id VARCHAR(100),  -- Banner PIDM, Colleague ID, etc.
    email VARCHAR(255) NOT NULL,
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    preferred_name VARCHAR(100),
    phone VARCHAR(30),
    user_type VARCHAR(20) NOT NULL CHECK (user_type IN ('student', 'advisor', 'faculty', 'staff', 'admin')),
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    last_login_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (institution_id, email)
);

CREATE INDEX idx_users_institution ON users(institution_id);
CREATE INDEX idx_users_sis_id ON users(institution_id, external_sis_id);
CREATE INDEX idx_users_type ON users(institution_id, user_type);

CREATE TABLE user_roles (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id),
    role VARCHAR(50) NOT NULL,  -- 'academic_advisor', 'financial_aid_advisor', 'faculty', 'tutor', 'counsellor', 'admin'
    department_id UUID REFERENCES departments(id),
    granted_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    revoked_at TIMESTAMPTZ,
    granted_by UUID REFERENCES users(id)
);

CREATE INDEX idx_user_roles_user ON user_roles(user_id);
```

## Student Demographics & Academic Standing

```sql
CREATE TABLE student_profiles (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL UNIQUE REFERENCES users(id),
    student_id_number VARCHAR(50) NOT NULL,  -- institution-assigned student ID
    date_of_birth DATE,
    gender VARCHAR(30),
    ethnicity VARCHAR(50),
    first_generation BOOLEAN,
    pell_eligible BOOLEAN,
    residency_status VARCHAR(30),  -- 'in_state', 'out_of_state', 'international'
    admission_type VARCHAR(30),  -- 'freshman', 'transfer', 'readmit'
    admission_term_id UUID REFERENCES academic_terms(id),
    expected_graduation_term_id UUID REFERENCES academic_terms(id),
    cumulative_gpa NUMERIC(4, 3),
    total_credits_earned NUMERIC(6, 2),
    academic_standing VARCHAR(30),  -- 'good', 'probation', 'suspension', 'dean_list'
    enrollment_status VARCHAR(30),  -- 'full_time', 'part_time', 'withdrawn', 'graduated'
    financial_aid_status VARCHAR(30),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_student_profiles_standing ON student_profiles(academic_standing);
CREATE INDEX idx_student_profiles_status ON student_profiles(enrollment_status);

-- Demographics stored separately for FERPA-controlled access
CREATE TABLE student_demographics (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    student_profile_id UUID NOT NULL UNIQUE REFERENCES student_profiles(id),
    race_codes TEXT[],  -- IPEDS race/ethnicity codes
    disability_registered BOOLEAN,
    veteran_status BOOLEAN,
    housing_status VARCHAR(30),  -- 'on_campus', 'off_campus', 'commuter'
    employment_hours_per_week INTEGER,
    dependents_count INTEGER,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Academic Structure

```sql
CREATE TABLE departments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    institution_id UUID NOT NULL REFERENCES institutions(id),
    name VARCHAR(255) NOT NULL,
    code VARCHAR(20) NOT NULL,
    parent_department_id UUID REFERENCES departments(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (institution_id, code)
);

CREATE TABLE academic_terms (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    institution_id UUID NOT NULL REFERENCES institutions(id),
    name VARCHAR(100) NOT NULL,  -- 'Fall 2026', 'Spring 2027'
    term_type VARCHAR(20) NOT NULL,  -- 'fall', 'spring', 'summer', 'winter'
    start_date DATE NOT NULL,
    end_date DATE NOT NULL,
    census_date DATE,
    is_current BOOLEAN NOT NULL DEFAULT FALSE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE degree_programmes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    institution_id UUID NOT NULL REFERENCES institutions(id),
    department_id UUID NOT NULL REFERENCES departments(id),
    name VARCHAR(255) NOT NULL,
    code VARCHAR(30) NOT NULL,
    level VARCHAR(20) NOT NULL,  -- 'associate', 'bachelor', 'master', 'doctoral', 'certificate'
    total_credits_required NUMERIC(6, 2) NOT NULL,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE programme_requirements (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    degree_programme_id UUID NOT NULL REFERENCES degree_programmes(id),
    parent_requirement_id UUID REFERENCES programme_requirements(id),  -- for nested requirement groups
    name VARCHAR(255) NOT NULL,  -- 'Core Requirements', 'Major Electives', 'Gen Ed - Humanities'
    requirement_type VARCHAR(30) NOT NULL,  -- 'required_course', 'elective_group', 'credit_minimum', 'gpa_minimum'
    min_credits NUMERIC(6, 2),
    min_courses INTEGER,
    min_gpa NUMERIC(4, 3),
    sort_order INTEGER NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE requirement_courses (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    requirement_id UUID NOT NULL REFERENCES programme_requirements(id),
    course_id UUID NOT NULL REFERENCES courses(id),
    is_required BOOLEAN NOT NULL DEFAULT TRUE,  -- false = elective option
    min_grade VARCHAR(5),  -- 'C', 'B-', etc.
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE courses (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    institution_id UUID NOT NULL REFERENCES institutions(id),
    department_id UUID NOT NULL REFERENCES departments(id),
    subject_code VARCHAR(10) NOT NULL,  -- 'MATH', 'ENGL'
    course_number VARCHAR(20) NOT NULL,  -- '101', '201A'
    title VARCHAR(255) NOT NULL,
    credits NUMERIC(4, 2) NOT NULL,
    description TEXT,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (institution_id, subject_code, course_number)
);

CREATE TABLE course_prerequisites (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    course_id UUID NOT NULL REFERENCES courses(id),
    prerequisite_course_id UUID NOT NULL REFERENCES courses(id),
    min_grade VARCHAR(5),
    is_corequisite BOOLEAN NOT NULL DEFAULT FALSE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE course_sections (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    course_id UUID NOT NULL REFERENCES courses(id),
    academic_term_id UUID NOT NULL REFERENCES academic_terms(id),
    section_number VARCHAR(20) NOT NULL,
    instructor_id UUID REFERENCES users(id),
    capacity INTEGER,
    enrollment_count INTEGER NOT NULL DEFAULT 0,
    delivery_mode VARCHAR(20),  -- 'in_person', 'online', 'hybrid'
    crn VARCHAR(20),  -- Course Reference Number
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_sections_term ON course_sections(academic_term_id);
CREATE INDEX idx_sections_instructor ON course_sections(instructor_id);
```

## Enrollments & Grades

```sql
CREATE TABLE enrollments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    student_profile_id UUID NOT NULL REFERENCES student_profiles(id),
    course_section_id UUID NOT NULL REFERENCES course_sections(id),
    enrollment_status VARCHAR(20) NOT NULL DEFAULT 'enrolled',  -- 'enrolled', 'dropped', 'withdrawn', 'completed'
    midterm_grade VARCHAR(5),
    final_grade VARCHAR(5),
    grade_points NUMERIC(4, 3),
    credits_attempted NUMERIC(4, 2),
    credits_earned NUMERIC(4, 2),
    last_activity_at TIMESTAMPTZ,
    enrolled_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    dropped_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (student_profile_id, course_section_id)
);

CREATE INDEX idx_enrollments_student ON enrollments(student_profile_id);
CREATE INDEX idx_enrollments_section ON enrollments(course_section_id);
CREATE INDEX idx_enrollments_status ON enrollments(enrollment_status);

CREATE TABLE student_programme_enrollments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    student_profile_id UUID NOT NULL REFERENCES student_profiles(id),
    degree_programme_id UUID NOT NULL REFERENCES degree_programmes(id),
    enrollment_type VARCHAR(20) NOT NULL,  -- 'major', 'minor', 'concentration', 'certificate'
    declared_at DATE NOT NULL,
    completed_at DATE,
    status VARCHAR(20) NOT NULL DEFAULT 'active',  -- 'active', 'completed', 'withdrawn'
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE transfer_credits (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    student_profile_id UUID NOT NULL REFERENCES student_profiles(id),
    source_institution_name VARCHAR(255) NOT NULL,
    source_course_name VARCHAR(255) NOT NULL,
    source_course_code VARCHAR(50),
    equivalent_course_id UUID REFERENCES courses(id),
    credits_transferred NUMERIC(4, 2) NOT NULL,
    grade_received VARCHAR(5),
    evaluation_status VARCHAR(20) NOT NULL DEFAULT 'pending',  -- 'pending', 'approved', 'denied'
    evaluated_by UUID REFERENCES users(id),
    evaluated_at TIMESTAMPTZ,
    ai_confidence_score NUMERIC(4, 3),  -- AI-suggested equivalency confidence
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Early Alert & Intervention

```sql
CREATE TABLE alert_flags (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    institution_id UUID NOT NULL REFERENCES institutions(id),
    student_profile_id UUID NOT NULL REFERENCES student_profiles(id),
    raised_by UUID NOT NULL REFERENCES users(id),
    flag_type VARCHAR(30) NOT NULL,  -- 'academic', 'attendance', 'financial', 'wellbeing', 'engagement'
    severity VARCHAR(20) NOT NULL DEFAULT 'medium',  -- 'low', 'medium', 'high', 'critical'
    course_section_id UUID REFERENCES course_sections(id),
    title VARCHAR(255) NOT NULL,
    description TEXT,
    status VARCHAR(20) NOT NULL DEFAULT 'open',  -- 'open', 'in_progress', 'resolved', 'dismissed'
    assigned_to UUID REFERENCES users(id),
    resolved_at TIMESTAMPTZ,
    resolved_by UUID REFERENCES users(id),
    resolution_notes TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_alerts_student ON alert_flags(student_profile_id);
CREATE INDEX idx_alerts_status ON alert_flags(institution_id, status);
CREATE INDEX idx_alerts_assigned ON alert_flags(assigned_to, status);
CREATE INDEX idx_alerts_type ON alert_flags(institution_id, flag_type, status);

CREATE TABLE interventions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    alert_flag_id UUID REFERENCES alert_flags(id),
    student_profile_id UUID NOT NULL REFERENCES student_profiles(id),
    intervention_type VARCHAR(30) NOT NULL,  -- 'tutoring_referral', 'financial_aid_review', 'schedule_change', 'counselling_referral', 'peer_mentoring'
    initiated_by UUID NOT NULL REFERENCES users(id),
    description TEXT,
    outcome VARCHAR(30),  -- 'successful', 'partially_successful', 'no_change', 'declined_by_student'
    status VARCHAR(20) NOT NULL DEFAULT 'planned',  -- 'planned', 'in_progress', 'completed', 'cancelled'
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_interventions_student ON interventions(student_profile_id);
CREATE INDEX idx_interventions_alert ON interventions(alert_flag_id);

CREATE TABLE success_plans (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    student_profile_id UUID NOT NULL REFERENCES student_profiles(id),
    created_by UUID NOT NULL REFERENCES users(id),
    title VARCHAR(255) NOT NULL,
    description TEXT,
    status VARCHAR(20) NOT NULL DEFAULT 'active',  -- 'active', 'completed', 'archived'
    target_date DATE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE success_plan_tasks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    success_plan_id UUID NOT NULL REFERENCES success_plans(id),
    title VARCHAR(255) NOT NULL,
    description TEXT,
    due_date DATE,
    status VARCHAR(20) NOT NULL DEFAULT 'pending',  -- 'pending', 'in_progress', 'completed', 'overdue'
    completed_at TIMESTAMPTZ,
    sort_order INTEGER NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Risk Scoring & Predictive Analytics

```sql
CREATE TABLE risk_scores (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    student_profile_id UUID NOT NULL REFERENCES student_profiles(id),
    academic_term_id UUID NOT NULL REFERENCES academic_terms(id),
    model_id UUID NOT NULL REFERENCES risk_models(id),
    overall_score NUMERIC(5, 4) NOT NULL,  -- 0.0000 to 1.0000
    risk_level VARCHAR(20) NOT NULL,  -- 'low', 'moderate', 'high', 'critical'
    factor_scores JSONB,
    -- Example factor_scores:
    -- {
    --   "gpa_trajectory": 0.72,
    --   "lms_engagement": 0.45,
    --   "attendance": 0.88,
    --   "financial_stress": 0.30
    -- }
    computed_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_risk_scores_student_term ON risk_scores(student_profile_id, academic_term_id);
CREATE INDEX idx_risk_scores_level ON risk_scores(risk_level, academic_term_id);

CREATE TABLE risk_models (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    institution_id UUID NOT NULL REFERENCES institutions(id),
    name VARCHAR(255) NOT NULL,
    version VARCHAR(20) NOT NULL,
    model_type VARCHAR(30) NOT NULL,  -- 'logistic_regression', 'gradient_boost', 'neural_network'
    feature_set TEXT[],
    training_cohort_size INTEGER,
    auc_score NUMERIC(5, 4),
    fairness_audit_date DATE,
    is_active BOOLEAN NOT NULL DEFAULT FALSE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE course_risk_indicators (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    course_id UUID NOT NULL REFERENCES courses(id),
    academic_term_id UUID NOT NULL REFERENCES academic_terms(id),
    dfw_rate NUMERIC(5, 4),  -- Drop/Fail/Withdraw rate
    average_grade_points NUMERIC(4, 3),
    enrollment_count INTEGER,
    computed_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Advising & Appointments

```sql
CREATE TABLE advisor_assignments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    student_profile_id UUID NOT NULL REFERENCES student_profiles(id),
    advisor_id UUID NOT NULL REFERENCES users(id),
    assignment_type VARCHAR(30) NOT NULL,  -- 'primary', 'secondary', 'financial_aid', 'career'
    assigned_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    ended_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_advisor_assignments_advisor ON advisor_assignments(advisor_id) WHERE ended_at IS NULL;
CREATE INDEX idx_advisor_assignments_student ON advisor_assignments(student_profile_id);

CREATE TABLE appointments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    institution_id UUID NOT NULL REFERENCES institutions(id),
    student_profile_id UUID NOT NULL REFERENCES student_profiles(id),
    advisor_id UUID NOT NULL REFERENCES users(id),
    appointment_type VARCHAR(30) NOT NULL,  -- 'advising', 'tutoring', 'financial_aid', 'career', 'counselling'
    scheduled_start TIMESTAMPTZ NOT NULL,
    scheduled_end TIMESTAMPTZ NOT NULL,
    actual_start TIMESTAMPTZ,
    actual_end TIMESTAMPTZ,
    location VARCHAR(255),
    mode VARCHAR(20),  -- 'in_person', 'virtual', 'phone'
    status VARCHAR(20) NOT NULL DEFAULT 'scheduled',  -- 'scheduled', 'confirmed', 'completed', 'cancelled', 'no_show'
    cancellation_reason TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_appointments_advisor_date ON appointments(advisor_id, scheduled_start);
CREATE INDEX idx_appointments_student ON appointments(student_profile_id);

CREATE TABLE advising_notes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    student_profile_id UUID NOT NULL REFERENCES student_profiles(id),
    author_id UUID NOT NULL REFERENCES users(id),
    appointment_id UUID REFERENCES appointments(id),
    note_type VARCHAR(30) NOT NULL,  -- 'advising', 'financial_aid', 'career', 'counselling', 'general'
    content TEXT NOT NULL,
    is_restricted BOOLEAN NOT NULL DEFAULT FALSE,  -- HIPAA-restricted counselling notes
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_advising_notes_student ON advising_notes(student_profile_id);
```

## AI Advising & Conversations

```sql
CREATE TABLE chatbot_conversations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    student_profile_id UUID NOT NULL REFERENCES student_profiles(id),
    started_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    ended_at TIMESTAMPTZ,
    escalated_to UUID REFERENCES users(id),
    escalation_reason TEXT,
    sentiment_score NUMERIC(4, 3),  -- -1.000 to 1.000
    topic_tags TEXT[],
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE chatbot_messages (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    conversation_id UUID NOT NULL REFERENCES chatbot_conversations(id),
    role VARCHAR(10) NOT NULL,  -- 'student', 'assistant', 'system'
    content TEXT NOT NULL,
    confidence_score NUMERIC(4, 3),
    sentiment VARCHAR(20),  -- 'positive', 'neutral', 'negative', 'distress'
    sent_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_chatbot_messages_conversation ON chatbot_messages(conversation_id, sent_at);

CREATE TABLE intervention_recommendations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    student_profile_id UUID NOT NULL REFERENCES student_profiles(id),
    risk_score_id UUID REFERENCES risk_scores(id),
    recommended_intervention_type VARCHAR(30) NOT NULL,
    confidence_score NUMERIC(4, 3) NOT NULL,
    reasoning TEXT,
    peer_outcome_data JSONB,
    -- Example: {"similar_students": 245, "success_rate": 0.73, "avg_gpa_improvement": 0.4}
    was_accepted BOOLEAN,
    accepted_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Outreach & Campaigns

```sql
CREATE TABLE campaigns (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    institution_id UUID NOT NULL REFERENCES institutions(id),
    name VARCHAR(255) NOT NULL,
    description TEXT,
    target_criteria JSONB NOT NULL,
    -- Example: {"risk_level": ["high", "critical"], "enrollment_status": "full_time", "term": "Fall 2026"}
    channel VARCHAR(20) NOT NULL,  -- 'email', 'sms', 'in_app', 'multi'
    status VARCHAR(20) NOT NULL DEFAULT 'draft',  -- 'draft', 'scheduled', 'active', 'completed', 'cancelled'
    scheduled_at TIMESTAMPTZ,
    created_by UUID NOT NULL REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE campaign_messages (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    campaign_id UUID NOT NULL REFERENCES campaigns(id),
    student_profile_id UUID NOT NULL REFERENCES student_profiles(id),
    channel VARCHAR(20) NOT NULL,
    content TEXT NOT NULL,
    sent_at TIMESTAMPTZ,
    delivered_at TIMESTAMPTZ,
    opened_at TIMESTAMPTZ,
    responded_at TIMESTAMPTZ,
    response_text TEXT,
    status VARCHAR(20) NOT NULL DEFAULT 'queued',  -- 'queued', 'sent', 'delivered', 'opened', 'responded', 'bounced'
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_campaign_messages_campaign ON campaign_messages(campaign_id);
CREATE INDEX idx_campaign_messages_student ON campaign_messages(student_profile_id);
```

## Learning Events (Caliper / xAPI)

```sql
CREATE TABLE learning_events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    institution_id UUID NOT NULL REFERENCES institutions(id),
    student_profile_id UUID NOT NULL REFERENCES student_profiles(id),
    source_standard VARCHAR(10) NOT NULL,  -- 'caliper', 'xapi'
    event_type VARCHAR(50) NOT NULL,  -- Caliper: 'NavigationEvent', 'AssessmentEvent'; xAPI verb
    actor_id VARCHAR(255) NOT NULL,
    verb VARCHAR(100) NOT NULL,
    object_type VARCHAR(100) NOT NULL,
    object_id VARCHAR(255) NOT NULL,
    course_section_id UUID REFERENCES course_sections(id),
    result_score NUMERIC(8, 4),
    result_success BOOLEAN,
    result_completion BOOLEAN,
    duration_seconds INTEGER,
    raw_payload JSONB NOT NULL,  -- full Caliper event or xAPI statement
    occurred_at TIMESTAMPTZ NOT NULL,
    received_at TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (occurred_at);

-- Partition by month for query performance
CREATE TABLE learning_events_2026_01 PARTITION OF learning_events
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
-- ... additional monthly partitions

CREATE INDEX idx_learning_events_student_time ON learning_events(student_profile_id, occurred_at);
CREATE INDEX idx_learning_events_section ON learning_events(course_section_id, occurred_at);
CREATE INDEX idx_learning_events_type ON learning_events(event_type, occurred_at);
```

## FERPA Audit Log

```sql
CREATE TABLE audit_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    institution_id UUID NOT NULL REFERENCES institutions(id),
    user_id UUID NOT NULL REFERENCES users(id),
    action VARCHAR(30) NOT NULL,  -- 'view', 'create', 'update', 'delete', 'export', 'share'
    resource_type VARCHAR(50) NOT NULL,  -- 'student_profile', 'advising_note', 'alert_flag', 'risk_score'
    resource_id UUID NOT NULL,
    student_profile_id UUID,  -- the student whose record was accessed
    ip_address INET,
    user_agent TEXT,
    old_values JSONB,  -- previous field values for updates
    new_values JSONB,  -- new field values for updates
    occurred_at TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (occurred_at);

-- Retained for minimum 3 years per FERPA enforcement guidance (March 2025)
CREATE TABLE audit_log_2026_q1 PARTITION OF audit_log
    FOR VALUES FROM ('2026-01-01') TO ('2026-04-01');
-- ... additional quarterly partitions

CREATE INDEX idx_audit_log_user ON audit_log(user_id, occurred_at);
CREATE INDEX idx_audit_log_student ON audit_log(student_profile_id, occurred_at);
CREATE INDEX idx_audit_log_resource ON audit_log(resource_type, resource_id);
```

## Integration & SSO

```sql
CREATE TABLE identity_providers (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    institution_id UUID NOT NULL REFERENCES institutions(id),
    name VARCHAR(255) NOT NULL,
    protocol VARCHAR(20) NOT NULL,  -- 'saml', 'oidc'
    metadata_url TEXT,
    client_id VARCHAR(255),
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE data_integrations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    institution_id UUID NOT NULL REFERENCES institutions(id),
    integration_type VARCHAR(30) NOT NULL,  -- 'sis', 'lms', 'financial_aid', 'housing'
    source_system VARCHAR(50) NOT NULL,  -- 'banner', 'colleague', 'canvas', 'blackboard'
    protocol VARCHAR(20) NOT NULL,  -- 'oneroster', 'caliper', 'xapi', 'eduapi', 'custom_api', 'sftp'
    endpoint_url TEXT,
    last_sync_at TIMESTAMPTZ,
    sync_status VARCHAR(20),  -- 'healthy', 'error', 'stale'
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Core Identity & Multi-Tenancy | 3 | institutions, users, user_roles |
| Student Demographics | 2 | student_profiles, student_demographics |
| Academic Structure | 7 | departments, terms, programmes, requirements, courses, prerequisites, sections |
| Enrollments & Grades | 3 | enrollments, programme_enrollments, transfer_credits |
| Early Alert & Intervention | 4 | alert_flags, interventions, success_plans, success_plan_tasks |
| Risk Scoring | 3 | risk_scores, risk_models, course_risk_indicators |
| Advising & Appointments | 3 | advisor_assignments, appointments, advising_notes |
| AI Advising | 3 | chatbot_conversations, chatbot_messages, intervention_recommendations |
| Outreach | 2 | campaigns, campaign_messages |
| Learning Events | 1 | learning_events (partitioned) |
| Audit & Compliance | 1 | audit_log (partitioned) |
| Integration & SSO | 2 | identity_providers, data_integrations |
| **Total** | **34** | Plus requirement_courses junction table |

---

## Key Design Decisions

1. **UUID primary keys throughout** — supports distributed ID generation across multiple deployment environments without coordination; aligns with modern SaaS patterns used by Salesforce Education Cloud and Stellic.

2. **Multi-tenant via `institution_id` foreign key** — all core tables include an institution_id, enabling row-level security (RLS) in PostgreSQL. Simpler than schema-per-tenant and supports cross-institution analytics if permitted.

3. **Separate `student_profiles` from `users`** — advisors and faculty are users too; the student_profile table holds student-specific data (GPA, enrollment status, demographics) while the users table holds authentication and identity data for all user types.

4. **FERPA-driven audit log with immutable partitioned table** — the audit_log captures every access to student records with user, timestamp, IP, and before/after values. Partitioned quarterly and retained for 3+ years per 2025 FERPA enforcement guidance.

5. **Demographics separated from student_profiles** — sensitive demographic data (race, disability, veteran status) is in a separate table with more restrictive access controls, supporting FERPA's principle of minimum necessary disclosure.

6. **Learning events partitioned by time** — Caliper and xAPI events are high-volume; monthly partitioning enables efficient range queries and data retention management.

7. **Risk scores stored per term, not updated in place** — each risk computation creates a new row, preserving the full history of risk assessments for a student. This supports temporal analysis and model fairness audits.

8. **HIPAA-restricted advising notes** — the `is_restricted` flag on advising_notes allows counselling referral notes to be stored in the same table but filtered from standard advisor queries, supporting institutions that integrate counselling and advising.

9. **Transfer credit AI confidence score** — the transfer_credits table includes an AI-generated confidence score for equivalency matching, supporting the human-in-the-loop review workflow required by GDPR Article 22.

10. **Standards-aligned integration tables** — the data_integrations table explicitly tracks which protocol (OneRoster, Caliper, xAPI, Edu-API) each integration uses, making it clear how data flows into the platform from each SIS/LMS.

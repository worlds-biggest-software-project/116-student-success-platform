# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Student Success Platform · Created: 2026-05-19

## Philosophy

The hybrid relational + JSONB model keeps a normalised relational backbone for core entities (students, courses, enrollments, alerts) while using PostgreSQL JSONB columns for data that varies by institution, jurisdiction, or integration source. This is the pragmatic middle ground: the stability and query performance of relational tables for the 80% of data that is universal, combined with the flexibility of schema-on-write for the 20% that differs.

This approach recognises a fundamental reality of the higher education market: every institution is different. A community college in Texas has different demographic fields, compliance requirements, and SIS data structures than a research university in Germany. EAB Navigate360 and Salesforce Education Cloud both handle this through configuration layers and custom fields. The JSONB hybrid model achieves the same flexibility at the database level, without requiring an application-level metadata system or institution-specific schema migrations.

PostgreSQL's JSONB support is mature and performant. GIN indexes on JSONB columns enable efficient queries against flexible fields. Containment operators (`@>`) and path operators (`->`, `->>`) allow SQL queries to filter and project JSONB data alongside relational columns. This means a single query can JOIN a student's relational enrollment data with their institution-specific JSONB attributes without leaving SQL.

**Best for:** Multi-institution SaaS deployments where each institution has unique fields, compliance requirements, and SIS integrations — and where rapid feature iteration matters more than schema purity.

**Trade-offs:**
- Pro: Fastest path to MVP — new institution-specific fields require no schema migration
- Pro: Handles multi-jurisdictional compliance (FERPA, GDPR, state-specific) with jurisdiction-aware JSONB
- Pro: Reduces table count compared to fully normalised model
- Pro: PostgreSQL JSONB performance is excellent with GIN indexes
- Con: JSONB fields lack database-enforced constraints (no FK, no NOT NULL at field level)
- Con: Schema documentation burden shifts from DDL to application code and API contracts
- Con: JSONB fields can become dumping grounds without disciplined governance
- Con: Reporting tools and BI platforms may struggle with JSONB fields compared to flat columns

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| OneRoster 1.2 | Core entities (orgs, users, courses, enrollments) are relational; OneRoster extensions map to JSONB |
| IMS Caliper 1.2 | Full Caliper event payload stored as JSONB; extracted fields indexed for querying |
| xAPI | Full xAPI statement stored as JSONB with actor/verb/object extracted as relational columns |
| FERPA | Audit log with relational core + JSONB for institution-specific compliance metadata |
| GDPR | JSONB `compliance_config` on institutions table defines data retention and consent rules per jurisdiction |
| ISO 3166-1/2 | Jurisdiction reference table with relational codes; jurisdiction-specific rules stored as JSONB |
| Edu-API 1.0 | Flexible JSONB accommodates evolving Edu-API field mappings during adoption transition |

---

## Core Identity & Configuration

```sql
CREATE TABLE institutions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    short_code VARCHAR(50) NOT NULL UNIQUE,
    timezone VARCHAR(50) NOT NULL DEFAULT 'America/New_York',
    country_code CHAR(2) NOT NULL DEFAULT 'US',
    compliance_config JSONB NOT NULL DEFAULT '{}',
    -- Example compliance_config:
    -- {
    --   "ferpa": { "enabled": true, "audit_retention_years": 5 },
    --   "gdpr": { "enabled": true, "dpia_completed": "2026-03-15", "art22_human_review": true },
    --   "hipaa": { "enabled": true, "baa_signed": "2026-01-10" },
    --   "state_specific": { "california_ccpa": true, "texas_sb820": false }
    -- }
    integration_config JSONB NOT NULL DEFAULT '{}',
    -- Example integration_config:
    -- {
    --   "sis": { "type": "banner", "version": "9.x", "api_base": "https://sis.uni.edu/api" },
    --   "lms": { "type": "canvas", "caliper_endpoint": "https://..." },
    --   "identity": { "protocol": "saml", "provider": "azure_ad", "metadata_url": "https://..." }
    -- }
    custom_fields_schema JSONB NOT NULL DEFAULT '{}',
    -- Defines institution-specific custom fields and their validation rules:
    -- {
    --   "student_profile": {
    --     "tribal_affiliation": { "type": "string", "label": "Tribal Affiliation", "required": false },
    --     "high_school_gpa": { "type": "number", "label": "HS GPA", "min": 0, "max": 4.0 }
    --   },
    --   "alert_flag": {
    --     "referral_source": { "type": "enum", "options": ["faculty", "RA", "self", "peer"] }
    --   }
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    institution_id UUID NOT NULL REFERENCES institutions(id),
    external_ids JSONB NOT NULL DEFAULT '{}',
    -- Flexible external ID mapping across multiple systems:
    -- {
    --   "banner_pidm": "12345678",
    --   "canvas_user_id": "987654",
    --   "sis_id": "B00123456",
    --   "scim_id": "ext-user-abc"
    -- }
    email VARCHAR(255) NOT NULL,
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    preferred_name VARCHAR(100),
    user_type VARCHAR(20) NOT NULL CHECK (user_type IN ('student', 'advisor', 'faculty', 'staff', 'admin')),
    roles JSONB NOT NULL DEFAULT '[]',
    -- Example roles:
    -- [
    --   { "role": "academic_advisor", "department": "Computer Science", "since": "2024-08-15" },
    --   { "role": "faculty", "department": "Computer Science", "since": "2020-01-10" }
    -- ]
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    profile_data JSONB NOT NULL DEFAULT '{}',
    -- User-type-specific and institution-specific fields:
    -- For advisors: { "max_caseload": 150, "specialisations": ["pre-med", "transfer"] }
    -- For faculty: { "office_hours": "MWF 2-4pm", "office_location": "Smith 301" }
    last_login_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (institution_id, email)
);

CREATE INDEX idx_users_institution ON users(institution_id);
CREATE INDEX idx_users_type ON users(institution_id, user_type);
CREATE INDEX idx_users_external_ids ON users USING GIN (external_ids);
CREATE INDEX idx_users_roles ON users USING GIN (roles);
```

## Student Profiles

```sql
CREATE TABLE student_profiles (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL UNIQUE REFERENCES users(id),
    institution_id UUID NOT NULL REFERENCES institutions(id),
    student_id_number VARCHAR(50) NOT NULL,

    -- Universal relational fields (every institution has these)
    cumulative_gpa NUMERIC(4, 3),
    total_credits_earned NUMERIC(6, 2),
    academic_standing VARCHAR(30),
    enrollment_status VARCHAR(30),
    admission_type VARCHAR(30),

    -- Demographics — core relational for IPEDS reporting
    date_of_birth DATE,
    gender VARCHAR(30),
    ethnicity VARCHAR(50),
    first_generation BOOLEAN,
    pell_eligible BOOLEAN,

    -- Current risk assessment (denormalised for dashboard performance)
    current_risk_level VARCHAR(20),
    current_risk_score NUMERIC(5, 4),
    risk_updated_at TIMESTAMPTZ,

    -- Institution-specific and jurisdiction-specific extended attributes
    extended_attributes JSONB NOT NULL DEFAULT '{}',
    -- Example for a US community college:
    -- {
    --   "residency_status": "in_state",
    --   "high_school_gpa": 3.2,
    --   "placement_test_scores": { "math": 85, "english": 72, "reading": 90 },
    --   "developmental_ed_required": true,
    --   "financial_aid": {
    --     "pell_amount": 6895,
    --     "work_study_eligible": true,
    --     "satisfactory_academic_progress": "good"
    --   },
    --   "housing_status": "commuter",
    --   "employment_hours_per_week": 25,
    --   "dependents": 1,
    --   "veteran_status": false
    -- }
    --
    -- Example for a German university (GDPR context):
    -- {
    --   "matriculation_number": "DE-2026-45678",
    --   "visa_status": "student_visa",
    --   "bafog_recipient": true,
    --   "gdpr_consent_date": "2026-09-01",
    --   "gdpr_consent_scope": ["academic_analytics", "advising_notes"],
    --   "semester_ticket": true
    -- }

    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_student_profiles_institution ON student_profiles(institution_id);
CREATE INDEX idx_student_profiles_risk ON student_profiles(institution_id, current_risk_level);
CREATE INDEX idx_student_profiles_standing ON student_profiles(institution_id, academic_standing);
CREATE INDEX idx_student_profiles_extended ON student_profiles USING GIN (extended_attributes);
```

## Academic Structure

```sql
CREATE TABLE academic_terms (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    institution_id UUID NOT NULL REFERENCES institutions(id),
    name VARCHAR(100) NOT NULL,
    term_type VARCHAR(20) NOT NULL,
    start_date DATE NOT NULL,
    end_date DATE NOT NULL,
    census_date DATE,
    is_current BOOLEAN NOT NULL DEFAULT FALSE,
    term_config JSONB NOT NULL DEFAULT '{}',
    -- Example: { "add_drop_deadline": "2026-09-15", "withdrawal_deadline": "2026-11-01",
    --            "grade_submission_deadline": "2026-12-20" }
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
    course_metadata JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {
    --   "prerequisites": [
    --     { "course_code": "MATH-101", "min_grade": "C" },
    --     { "course_code": "MATH-102", "min_grade": "C", "corequisite": true }
    --   ],
    --   "gen_ed_categories": ["quantitative_reasoning"],
    --   "delivery_modes": ["in_person", "online"],
    --   "historical_dfw_rate": 0.18,
    --   "tags": ["stem", "gateway"]
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (institution_id, subject_code, course_number)
);

CREATE INDEX idx_courses_metadata ON courses USING GIN (course_metadata);

CREATE TABLE course_sections (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    course_id UUID NOT NULL REFERENCES courses(id),
    academic_term_id UUID NOT NULL REFERENCES academic_terms(id),
    section_number VARCHAR(20) NOT NULL,
    instructor_id UUID REFERENCES users(id),
    capacity INTEGER,
    enrollment_count INTEGER NOT NULL DEFAULT 0,
    delivery_mode VARCHAR(20),
    crn VARCHAR(20),
    section_details JSONB NOT NULL DEFAULT '{}',
    -- Example: { "meeting_times": [{"days": "MWF", "start": "10:00", "end": "10:50", "room": "Hall 201"}],
    --            "ta_ids": ["uuid1", "uuid2"], "waitlist_count": 5 }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE degree_programmes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    institution_id UUID NOT NULL REFERENCES institutions(id),
    name VARCHAR(255) NOT NULL,
    code VARCHAR(30) NOT NULL,
    level VARCHAR(20) NOT NULL,
    total_credits_required NUMERIC(6, 2) NOT NULL,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    requirements JSONB NOT NULL DEFAULT '[]',
    -- Entire requirement tree stored as nested JSONB:
    -- [
    --   {
    --     "name": "Core Requirements",
    --     "type": "required_courses",
    --     "min_credits": 30,
    --     "courses": [
    --       { "course_code": "CS-101", "required": true, "min_grade": "C" },
    --       { "course_code": "CS-201", "required": true, "min_grade": "C" }
    --     ]
    --   },
    --   {
    --     "name": "Major Electives",
    --     "type": "elective_group",
    --     "min_credits": 15,
    --     "min_courses": 5,
    --     "courses": [
    --       { "course_code": "CS-310" },
    --       { "course_code": "CS-320" },
    --       { "course_code": "CS-330" }
    --     ]
    --   },
    --   {
    --     "name": "Gen Ed - Humanities",
    --     "type": "elective_group",
    --     "min_credits": 6,
    --     "course_filter": { "gen_ed_category": "humanities" }
    --   }
    -- ]
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Enrollments & Grades

```sql
CREATE TABLE enrollments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    student_profile_id UUID NOT NULL REFERENCES student_profiles(id),
    course_section_id UUID NOT NULL REFERENCES course_sections(id),
    enrollment_status VARCHAR(20) NOT NULL DEFAULT 'enrolled',
    midterm_grade VARCHAR(5),
    final_grade VARCHAR(5),
    grade_points NUMERIC(4, 3),
    credits_attempted NUMERIC(4, 2),
    credits_earned NUMERIC(4, 2),
    last_activity_at TIMESTAMPTZ,
    enrollment_details JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {
    --   "registration_date": "2026-08-20",
    --   "registration_method": "self_service",
    --   "advisor_approved": true,
    --   "repeat_indicator": false,
    --   "grade_mode": "standard",
    --   "attendance_percentage": 0.87
    -- }
    enrolled_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    dropped_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (student_profile_id, course_section_id)
);

CREATE INDEX idx_enrollments_student ON enrollments(student_profile_id);
CREATE INDEX idx_enrollments_section ON enrollments(course_section_id);

CREATE TABLE student_programme_enrollments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    student_profile_id UUID NOT NULL REFERENCES student_profiles(id),
    degree_programme_id UUID NOT NULL REFERENCES degree_programmes(id),
    enrollment_type VARCHAR(20) NOT NULL,
    declared_at DATE NOT NULL,
    completed_at DATE,
    status VARCHAR(20) NOT NULL DEFAULT 'active',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE transfer_credits (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    student_profile_id UUID NOT NULL REFERENCES student_profiles(id),
    source_institution_name VARCHAR(255) NOT NULL,
    equivalent_course_id UUID REFERENCES courses(id),
    credits_transferred NUMERIC(4, 2) NOT NULL,
    evaluation_status VARCHAR(20) NOT NULL DEFAULT 'pending',
    evaluated_by UUID REFERENCES users(id),
    ai_confidence_score NUMERIC(4, 3),
    transfer_details JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {
    --   "source_course_code": "ENG 101",
    --   "source_course_title": "English Composition I",
    --   "source_credits": 3.0,
    --   "source_grade": "B+",
    --   "transcript_document_id": "doc-uuid",
    --   "ai_match_reasoning": "Course description and learning outcomes match 92% with ENGL-101",
    --   "evaluator_notes": "Approved — syllabus reviewed"
    -- }
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
    flag_type VARCHAR(30) NOT NULL,
    severity VARCHAR(20) NOT NULL DEFAULT 'medium',
    course_section_id UUID REFERENCES course_sections(id),
    title VARCHAR(255) NOT NULL,
    description TEXT,
    status VARCHAR(20) NOT NULL DEFAULT 'open',
    assigned_to UUID REFERENCES users(id),
    resolved_at TIMESTAMPTZ,
    resolved_by UUID REFERENCES users(id),
    resolution_notes TEXT,
    alert_metadata JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {
    --   "trigger_source": "faculty_manual",
    --   "auto_detected": false,
    --   "related_assignments": ["Assignment 3", "Midterm Exam"],
    --   "consecutive_absences": 3,
    --   "care_network_notified": ["advisor", "tutor"],
    --   "institution_custom": {
    --     "referral_source": "RA"
    --   }
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_alerts_student ON alert_flags(student_profile_id);
CREATE INDEX idx_alerts_status ON alert_flags(institution_id, status);
CREATE INDEX idx_alerts_assigned ON alert_flags(assigned_to, status);
CREATE INDEX idx_alerts_metadata ON alert_flags USING GIN (alert_metadata);

CREATE TABLE interventions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    alert_flag_id UUID REFERENCES alert_flags(id),
    student_profile_id UUID NOT NULL REFERENCES student_profiles(id),
    intervention_type VARCHAR(30) NOT NULL,
    initiated_by UUID NOT NULL REFERENCES users(id),
    status VARCHAR(20) NOT NULL DEFAULT 'planned',
    outcome VARCHAR(30),
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    intervention_data JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {
    --   "description": "Referred to Math tutoring center",
    --   "tutoring_sessions_attended": 4,
    --   "gpa_before": 1.8,
    --   "gpa_after": 2.3,
    --   "student_feedback": "Helpful — would recommend",
    --   "peer_outcome_comparison": { "similar_students": 180, "success_rate": 0.71 }
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE success_plans (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    student_profile_id UUID NOT NULL REFERENCES student_profiles(id),
    created_by UUID NOT NULL REFERENCES users(id),
    title VARCHAR(255) NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'active',
    target_date DATE,
    tasks JSONB NOT NULL DEFAULT '[]',
    -- Tasks stored as JSONB array rather than separate table:
    -- [
    --   { "id": "task-uuid-1", "title": "Meet with financial aid", "due_date": "2026-10-20",
    --     "status": "completed", "completed_at": "2026-10-18" },
    --   { "id": "task-uuid-2", "title": "Schedule tutoring session", "due_date": "2026-10-25",
    --     "status": "pending" },
    --   { "id": "task-uuid-3", "title": "Submit missing assignments", "due_date": "2026-10-30",
    --     "status": "overdue" }
    -- ]
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Risk Scoring

```sql
CREATE TABLE risk_scores (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    student_profile_id UUID NOT NULL REFERENCES student_profiles(id),
    academic_term_id UUID NOT NULL REFERENCES academic_terms(id),
    model_version VARCHAR(20) NOT NULL,
    overall_score NUMERIC(5, 4) NOT NULL,
    risk_level VARCHAR(20) NOT NULL,
    factors JSONB NOT NULL,
    -- Example:
    -- {
    --   "gpa_trajectory": { "score": 0.72, "weight": 0.25, "detail": "GPA dropped 0.4 from last term" },
    --   "lms_engagement": { "score": 0.45, "weight": 0.20, "detail": "Login frequency down 60%" },
    --   "attendance": { "score": 0.88, "weight": 0.15, "detail": "Missed 2 of last 10 classes" },
    --   "financial_stress": { "score": 0.30, "weight": 0.15, "detail": "Financial hold on account" },
    --   "assignment_completion": { "score": 0.55, "weight": 0.15, "detail": "3 late submissions" },
    --   "social_engagement": { "score": 0.70, "weight": 0.10, "detail": "Active in 2 clubs" }
    -- }
    recommended_interventions JSONB,
    -- Example:
    -- [
    --   { "type": "tutoring_referral", "confidence": 0.87, "peer_success_rate": 0.73 },
    --   { "type": "financial_aid_review", "confidence": 0.65, "peer_success_rate": 0.58 }
    -- ]
    computed_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_risk_scores_student ON risk_scores(student_profile_id, computed_at DESC);
CREATE INDEX idx_risk_scores_level ON risk_scores(risk_level, academic_term_id);
```

## Advising & Appointments

```sql
CREATE TABLE advisor_assignments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    student_profile_id UUID NOT NULL REFERENCES student_profiles(id),
    advisor_id UUID NOT NULL REFERENCES users(id),
    assignment_type VARCHAR(30) NOT NULL,
    assigned_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    ended_at TIMESTAMPTZ
);

CREATE TABLE appointments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    institution_id UUID NOT NULL REFERENCES institutions(id),
    student_profile_id UUID NOT NULL REFERENCES student_profiles(id),
    advisor_id UUID NOT NULL REFERENCES users(id),
    appointment_type VARCHAR(30) NOT NULL,
    scheduled_start TIMESTAMPTZ NOT NULL,
    scheduled_end TIMESTAMPTZ NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'scheduled',
    mode VARCHAR(20),
    appointment_data JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {
    --   "location": "Advising Center Room 201",
    --   "virtual_link": "https://zoom.us/j/123456",
    --   "actual_start": "2026-10-15T14:02:00Z",
    --   "actual_end": "2026-10-15T14:28:00Z",
    --   "cancellation_reason": null,
    --   "check_in_method": "kiosk"
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_appointments_advisor ON appointments(advisor_id, scheduled_start);
CREATE INDEX idx_appointments_student ON appointments(student_profile_id);

CREATE TABLE advising_notes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    student_profile_id UUID NOT NULL REFERENCES student_profiles(id),
    author_id UUID NOT NULL REFERENCES users(id),
    appointment_id UUID REFERENCES appointments(id),
    note_type VARCHAR(30) NOT NULL,
    content TEXT NOT NULL,
    is_restricted BOOLEAN NOT NULL DEFAULT FALSE,
    note_metadata JSONB NOT NULL DEFAULT '{}',
    -- Example: { "tags": ["schedule_change", "gpa_concern"], "follow_up_date": "2026-11-01",
    --            "shared_with": ["financial_aid_advisor"] }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_notes_student ON advising_notes(student_profile_id);
```

## AI Advising & Chatbot

```sql
CREATE TABLE chatbot_conversations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    student_profile_id UUID NOT NULL REFERENCES student_profiles(id),
    institution_id UUID NOT NULL REFERENCES institutions(id),
    started_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    ended_at TIMESTAMPTZ,
    escalated_to UUID REFERENCES users(id),
    conversation_data JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {
    --   "topic_tags": ["degree_planning", "transfer_credits"],
    --   "sentiment_scores": [-0.1, 0.2, -0.5, -0.82],
    --   "escalation_reason": "distress_detected",
    --   "messages_count": 12,
    --   "avg_confidence": 0.78,
    --   "gdpr_human_review_required": false
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE chatbot_messages (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    conversation_id UUID NOT NULL REFERENCES chatbot_conversations(id),
    role VARCHAR(10) NOT NULL,
    content TEXT NOT NULL,
    message_metadata JSONB NOT NULL DEFAULT '{}',
    -- Example: { "confidence": 0.92, "sentiment": "neutral", "sources_cited": ["degree_requirements_cs_bs"],
    --            "model_version": "gpt-4o-2026-05" }
    sent_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_chatbot_messages_conversation ON chatbot_messages(conversation_id, sent_at);
```

## Learning Events & Outreach

```sql
CREATE TABLE learning_events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    institution_id UUID NOT NULL REFERENCES institutions(id),
    student_profile_id UUID NOT NULL REFERENCES student_profiles(id),
    source_standard VARCHAR(10) NOT NULL,
    event_type VARCHAR(50) NOT NULL,
    course_section_id UUID REFERENCES course_sections(id),
    occurred_at TIMESTAMPTZ NOT NULL,
    event_data JSONB NOT NULL,
    -- Full Caliper event or xAPI statement payload preserved as JSONB:
    -- Caliper example:
    -- {
    --   "profile": "AssessmentProfile",
    --   "action": "Submitted",
    --   "object": { "id": "urn:canvas:quiz:4567", "type": "Assessment" },
    --   "result": { "score": 85.0, "maxScore": 100.0 },
    --   "duration": "PT45M"
    -- }
    received_at TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (occurred_at);

CREATE INDEX idx_learning_events_student ON learning_events(student_profile_id, occurred_at);
CREATE INDEX idx_learning_events_section ON learning_events(course_section_id, occurred_at);
CREATE INDEX idx_learning_events_type ON learning_events(event_type, occurred_at);

CREATE TABLE campaigns (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    institution_id UUID NOT NULL REFERENCES institutions(id),
    name VARCHAR(255) NOT NULL,
    channel VARCHAR(20) NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'draft',
    created_by UUID NOT NULL REFERENCES users(id),
    campaign_config JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {
    --   "target_criteria": { "risk_level": ["high", "critical"], "term": "Fall 2026" },
    --   "message_template": "Hi {{first_name}}, we noticed...",
    --   "schedule": { "send_at": "2026-10-20T09:00:00Z", "timezone": "America/New_York" },
    --   "results": { "sent": 245, "delivered": 238, "opened": 156, "responded": 42 }
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE campaign_messages (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    campaign_id UUID NOT NULL REFERENCES campaigns(id),
    student_profile_id UUID NOT NULL REFERENCES student_profiles(id),
    channel VARCHAR(20) NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'queued',
    message_data JSONB NOT NULL DEFAULT '{}',
    -- Example: { "content": "Hi Jane, ...", "sent_at": "...", "delivered_at": "...",
    --            "opened_at": "...", "response_text": "Thanks, I'll schedule..." }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Audit & Compliance

```sql
CREATE TABLE audit_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    institution_id UUID NOT NULL,
    user_id UUID NOT NULL,
    action VARCHAR(30) NOT NULL,
    resource_type VARCHAR(50) NOT NULL,
    resource_id UUID NOT NULL,
    student_profile_id UUID,
    ip_address INET,
    audit_details JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {
    --   "user_agent": "Mozilla/5.0...",
    --   "old_values": { "academic_standing": "good" },
    --   "new_values": { "academic_standing": "probation" },
    --   "fields_accessed": ["gpa", "enrollment_status"],
    --   "ferpa_classification": "education_record",
    --   "gdpr_lawful_basis": "legitimate_interest"
    -- }
    occurred_at TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (occurred_at);

CREATE INDEX idx_audit_user ON audit_log(user_id, occurred_at);
CREATE INDEX idx_audit_student ON audit_log(student_profile_id, occurred_at);
CREATE INDEX idx_audit_resource ON audit_log(resource_type, resource_id);
```

---

## Example Queries

### Query institution-specific extended attributes

```sql
-- Find all students with food insecurity indicators at an institution
SELECT u.first_name, u.last_name, sp.cumulative_gpa, sp.current_risk_level
FROM student_profiles sp
JOIN users u ON u.id = sp.user_id
WHERE sp.institution_id = 'inst-uuid'
  AND sp.extended_attributes @> '{"housing_status": "unstable"}'
  AND sp.current_risk_level IN ('high', 'critical');
```

### Query JSONB degree requirements for what-if analysis

```sql
-- Find all courses that satisfy a specific requirement group
SELECT c.subject_code || '-' || c.course_number AS course_code, c.title, c.credits
FROM degree_programmes dp,
     jsonb_array_elements(dp.requirements) AS req,
     jsonb_array_elements(req->'courses') AS course_ref
JOIN courses c ON c.institution_id = dp.institution_id
             AND c.subject_code || '-' || c.course_number = course_ref->>'course_code'
WHERE dp.id = 'programme-uuid'
  AND req->>'name' = 'Major Electives';
```

### Cross-system external ID lookup

```sql
-- Find a student by their Canvas user ID
SELECT sp.*, u.first_name, u.last_name
FROM users u
JOIN student_profiles sp ON sp.user_id = u.id
WHERE u.external_ids @> '{"canvas_user_id": "987654"}'
  AND u.institution_id = 'inst-uuid';
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Core Identity | 2 | institutions, users (roles as JSONB) |
| Student Profiles | 1 | student_profiles (demographics + extended attributes in JSONB) |
| Academic Structure | 4 | academic_terms, courses, course_sections, degree_programmes (requirements as JSONB) |
| Enrollments & Grades | 3 | enrollments, programme_enrollments, transfer_credits |
| Early Alert & Intervention | 3 | alert_flags, interventions, success_plans (tasks as JSONB) |
| Risk Scoring | 1 | risk_scores (factors + recommendations as JSONB) |
| Advising & Appointments | 3 | advisor_assignments, appointments, advising_notes |
| AI Advising | 2 | chatbot_conversations, chatbot_messages |
| Learning Events | 1 | learning_events (partitioned, full payload as JSONB) |
| Outreach | 2 | campaigns, campaign_messages |
| Audit | 1 | audit_log (partitioned) |
| **Total** | **23** | ~30% fewer tables than normalised model |

---

## Key Design Decisions

1. **JSONB for institution-specific fields, not for universal fields** — GPA, enrollment status, and academic standing are relational columns because every institution has them and they are queried constantly. Housing status, tribal affiliation, and placement test scores are JSONB because they vary by institution.

2. **`custom_fields_schema` on institutions table** — each institution defines its custom field schema as JSONB. The application layer validates incoming data against this schema, providing field-level validation without database constraints. This replaces the need for an EAV (Entity-Attribute-Value) pattern.

3. **`external_ids` as JSONB on users** — a student might have a Banner PIDM, Canvas user ID, SCIM external ID, and SSO subject claim. Rather than a separate `external_identifiers` junction table, these are stored as a JSONB object with GIN indexing for efficient lookup by any external system ID.

4. **Degree requirements as nested JSONB** — the requirement tree (core courses, elective groups, gen-ed categories) is stored as a single JSONB document on `degree_programmes`. This eliminates 2-3 tables from the normalised model (programme_requirements, requirement_courses, requirement_groups) and makes the requirement structure self-contained for what-if analysis queries.

5. **Success plan tasks as JSONB array** — a success plan typically has 3-10 tasks. Storing them as a JSONB array within the success_plans table eliminates a join and keeps the plan as a single atomic document. Task status updates modify the JSONB in place.

6. **Compliance configuration as JSONB per institution** — FERPA, GDPR, HIPAA, and state-specific compliance flags are stored as a JSONB document on the institutions table. This allows the application to make jurisdiction-aware decisions (e.g., require human review before AI-triggered outreach for GDPR institutions) without separate compliance tables.

7. **Full Caliper/xAPI payloads preserved as JSONB** — learning events store the entire original payload, ensuring no data loss during ingestion. Extracted relational columns (event_type, occurred_at, course_section_id) enable efficient indexing, while the full JSONB payload supports ad-hoc analytics.

8. **GIN indexes on all JSONB columns** — every table with JSONB columns has a GIN index to support containment queries (`@>`). This makes JSONB queries performant for dashboard-style lookups, though complex analytical queries across JSONB fields may still benefit from materialised views.

9. **Risk factor details and intervention recommendations as JSONB** — each risk score includes detailed factor breakdowns and recommended interventions as JSONB. This keeps the risk_scores table to a single row per computation while preserving rich detail for the advisor-facing explanation UI.

10. **~30% fewer tables than the normalised model** — by consolidating variable fields into JSONB and eliminating junction tables for small embedded collections (tasks, roles, requirements), the schema is significantly simpler to deploy, migrate, and maintain across multiple institution tenants.

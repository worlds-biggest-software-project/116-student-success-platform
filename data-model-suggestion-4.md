# Data Model Suggestion 4: Graph-Relational (Care Network Model)

> Project: Student Success Platform · Created: 2026-05-19

## Philosophy

The graph-relational model combines relational tables for operational CRUD (students, courses, enrollments, grades) with a property graph layer for modelling the complex, multi-directional relationships at the heart of a student success platform. The care-network concept — where faculty, advisors, tutors, financial aid staff, counsellors, and peer mentors all contribute to a student's success — is fundamentally a graph problem. So are prerequisite chains, referral networks, and influence patterns.

Anthology Starfish pioneered the "care network" model in student success: any staff member can see shared notes and flags for a student, and the student can see their full care team. This pattern is poorly served by traditional junction tables, where querying "who is connected to this student, through what relationship, with what permissions?" requires multiple JOINs across multiple tables. In a property graph, this is a single traversal query.

This model uses PostgreSQL with Apache AGE (a graph extension) or a dedicated graph layer (Neo4j, Amazon Neptune) alongside relational tables. The relational tables handle transactional operations (recording grades, scheduling appointments), while the graph handles relationship queries (care network traversal, prerequisite chain analysis, intervention pathway discovery, peer mentoring matching). The graph also enables AI-powered features like "find students with similar risk profiles who were successfully retained" — a nearest-neighbour query that is natural in graph space.

**Best for:** Institutions that prioritise the care-network model, need complex relationship queries (who is connected to whom, through what), and want AI-driven peer matching and intervention pathway analysis.

**Trade-offs:**
- Pro: Care network queries (traversal, path finding) are orders of magnitude faster than relational JOINs
- Pro: Natural fit for prerequisite chain analysis and degree pathway exploration
- Pro: Peer mentoring matching and "similar student" queries are native graph operations
- Pro: Intervention pathway analysis ("what path of interventions worked for similar students?")
- Con: Additional infrastructure — graph database alongside PostgreSQL
- Con: Smaller talent pool — graph query languages (Cypher, openCypher) are less widely known than SQL
- Con: Dual data stores require synchronisation strategy (eventual consistency risk)
- Con: Graph databases are less mature for ACID transactions than PostgreSQL
- Con: Reporting and BI tools have weaker graph support than relational support

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| OneRoster 1.2 | Core entities stored relationally; enrollment relationships modelled as graph edges |
| IMS Caliper 1.2 | Learning events stored relationally; engagement patterns create/update graph edges between students and resources |
| xAPI | Actor-verb-object statements naturally map to graph triples (subject-predicate-object) |
| FERPA | Audit log is relational; graph edges carry access-permission metadata for care network visibility |
| SCIM 2.0 | User provisioning updates both relational user table and graph node |
| GDPR Art. 22 | Human review edges in the graph track advisor approval before AI-triggered actions |

---

## Relational Layer (PostgreSQL)

The relational layer handles all transactional CRUD operations and serves as the system of record for data that does not benefit from graph traversal.

```sql
-- ============================================================
-- INSTITUTIONS & USERS
-- ============================================================
CREATE TABLE institutions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    short_code VARCHAR(50) NOT NULL UNIQUE,
    timezone VARCHAR(50) NOT NULL DEFAULT 'America/New_York',
    country_code CHAR(2) NOT NULL DEFAULT 'US',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    institution_id UUID NOT NULL REFERENCES institutions(id),
    external_sis_id VARCHAR(100),
    email VARCHAR(255) NOT NULL,
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    preferred_name VARCHAR(100),
    user_type VARCHAR(20) NOT NULL CHECK (user_type IN ('student', 'advisor', 'faculty', 'staff', 'admin')),
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    last_login_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (institution_id, email)
);

CREATE INDEX idx_users_institution ON users(institution_id);
CREATE INDEX idx_users_type ON users(institution_id, user_type);

-- ============================================================
-- STUDENT PROFILES
-- ============================================================
CREATE TABLE student_profiles (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL UNIQUE REFERENCES users(id),
    institution_id UUID NOT NULL REFERENCES institutions(id),
    student_id_number VARCHAR(50) NOT NULL,
    cumulative_gpa NUMERIC(4, 3),
    total_credits_earned NUMERIC(6, 2),
    academic_standing VARCHAR(30),
    enrollment_status VARCHAR(30),
    admission_type VARCHAR(30),
    date_of_birth DATE,
    gender VARCHAR(30),
    ethnicity VARCHAR(50),
    first_generation BOOLEAN,
    pell_eligible BOOLEAN,
    current_risk_level VARCHAR(20),
    current_risk_score NUMERIC(5, 4),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_sp_institution ON student_profiles(institution_id);
CREATE INDEX idx_sp_risk ON student_profiles(institution_id, current_risk_level);

-- ============================================================
-- ACADEMIC STRUCTURE
-- ============================================================
CREATE TABLE academic_terms (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    institution_id UUID NOT NULL REFERENCES institutions(id),
    name VARCHAR(100) NOT NULL,
    term_type VARCHAR(20) NOT NULL,
    start_date DATE NOT NULL,
    end_date DATE NOT NULL,
    is_current BOOLEAN NOT NULL DEFAULT FALSE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE departments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    institution_id UUID NOT NULL REFERENCES institutions(id),
    name VARCHAR(255) NOT NULL,
    code VARCHAR(20) NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (institution_id, code)
);

CREATE TABLE courses (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    institution_id UUID NOT NULL REFERENCES institutions(id),
    department_id UUID NOT NULL REFERENCES departments(id),
    subject_code VARCHAR(10) NOT NULL,
    course_number VARCHAR(20) NOT NULL,
    title VARCHAR(255) NOT NULL,
    credits NUMERIC(4, 2) NOT NULL,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (institution_id, subject_code, course_number)
);

CREATE TABLE course_sections (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    course_id UUID NOT NULL REFERENCES courses(id),
    academic_term_id UUID NOT NULL REFERENCES academic_terms(id),
    section_number VARCHAR(20) NOT NULL,
    instructor_id UUID REFERENCES users(id),
    capacity INTEGER,
    enrollment_count INTEGER NOT NULL DEFAULT 0,
    delivery_mode VARCHAR(20),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE degree_programmes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    institution_id UUID NOT NULL REFERENCES institutions(id),
    department_id UUID NOT NULL REFERENCES departments(id),
    name VARCHAR(255) NOT NULL,
    code VARCHAR(30) NOT NULL,
    level VARCHAR(20) NOT NULL,
    total_credits_required NUMERIC(6, 2) NOT NULL,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- ENROLLMENTS & GRADES
-- ============================================================
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
    enrolled_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    dropped_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (student_profile_id, course_section_id)
);

CREATE INDEX idx_enrollments_student ON enrollments(student_profile_id);
CREATE INDEX idx_enrollments_section ON enrollments(course_section_id);

-- ============================================================
-- ALERT FLAGS & INTERVENTIONS
-- ============================================================
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
    resolution_notes TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_alerts_student ON alert_flags(student_profile_id);
CREATE INDEX idx_alerts_status ON alert_flags(institution_id, status);

CREATE TABLE interventions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    alert_flag_id UUID REFERENCES alert_flags(id),
    student_profile_id UUID NOT NULL REFERENCES student_profiles(id),
    intervention_type VARCHAR(30) NOT NULL,
    initiated_by UUID NOT NULL REFERENCES users(id),
    description TEXT,
    outcome VARCHAR(30),
    status VARCHAR(20) NOT NULL DEFAULT 'planned',
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- APPOINTMENTS & NOTES
-- ============================================================
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
    location VARCHAR(255),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE advising_notes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    student_profile_id UUID NOT NULL REFERENCES student_profiles(id),
    author_id UUID NOT NULL REFERENCES users(id),
    appointment_id UUID REFERENCES appointments(id),
    note_type VARCHAR(30) NOT NULL,
    content TEXT NOT NULL,
    is_restricted BOOLEAN NOT NULL DEFAULT FALSE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- RISK SCORING
-- ============================================================
CREATE TABLE risk_scores (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    student_profile_id UUID NOT NULL REFERENCES student_profiles(id),
    academic_term_id UUID NOT NULL REFERENCES academic_terms(id),
    model_version VARCHAR(20) NOT NULL,
    overall_score NUMERIC(5, 4) NOT NULL,
    risk_level VARCHAR(20) NOT NULL,
    factor_scores JSONB,
    computed_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_risk_student ON risk_scores(student_profile_id, computed_at DESC);

-- ============================================================
-- CHATBOT
-- ============================================================
CREATE TABLE chatbot_conversations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    student_profile_id UUID NOT NULL REFERENCES student_profiles(id),
    started_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    ended_at TIMESTAMPTZ,
    escalated_to UUID REFERENCES users(id),
    sentiment_score NUMERIC(4, 3),
    topic_tags TEXT[],
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE chatbot_messages (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    conversation_id UUID NOT NULL REFERENCES chatbot_conversations(id),
    role VARCHAR(10) NOT NULL,
    content TEXT NOT NULL,
    confidence_score NUMERIC(4, 3),
    sent_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- LEARNING EVENTS
-- ============================================================
CREATE TABLE learning_events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    institution_id UUID NOT NULL,
    student_profile_id UUID NOT NULL,
    source_standard VARCHAR(10) NOT NULL,
    event_type VARCHAR(50) NOT NULL,
    course_section_id UUID REFERENCES course_sections(id),
    raw_payload JSONB NOT NULL,
    occurred_at TIMESTAMPTZ NOT NULL,
    received_at TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (occurred_at);

CREATE INDEX idx_le_student ON learning_events(student_profile_id, occurred_at);

-- ============================================================
-- AUDIT LOG
-- ============================================================
CREATE TABLE audit_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    institution_id UUID NOT NULL,
    user_id UUID NOT NULL,
    action VARCHAR(30) NOT NULL,
    resource_type VARCHAR(50) NOT NULL,
    resource_id UUID NOT NULL,
    student_profile_id UUID,
    ip_address INET,
    old_values JSONB,
    new_values JSONB,
    occurred_at TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (occurred_at);

CREATE INDEX idx_audit_user ON audit_log(user_id, occurred_at);
CREATE INDEX idx_audit_student ON audit_log(student_profile_id, occurred_at);
```

---

## Graph Layer (Neo4j / Apache AGE)

The graph layer models relationships between people, courses, interventions, and outcomes. All entities in the graph reference their relational counterpart by UUID.

### Node Types

```cypher
// ============================================================
// PERSON NODES — students, advisors, faculty, tutors, counsellors
// ============================================================

// Student node (synced from relational student_profiles)
CREATE (:Student {
    id: 'student-uuid',
    institution_id: 'inst-uuid',
    name: 'Jane Doe',
    risk_level: 'high',
    gpa: 2.1,
    enrollment_status: 'full_time',
    admission_type: 'transfer',
    first_generation: true,
    pell_eligible: true
})

// Staff node (synced from relational users)
CREATE (:Staff {
    id: 'advisor-uuid',
    institution_id: 'inst-uuid',
    name: 'Dr. Sarah Johnson',
    user_type: 'advisor',
    roles: ['academic_advisor', 'transfer_specialist'],
    department: 'Computer Science',
    current_caseload: 87
})

// ============================================================
// ACADEMIC NODES
// ============================================================

CREATE (:Course {
    id: 'course-uuid',
    code: 'MATH-201',
    title: 'Calculus II',
    credits: 4.0,
    department: 'Mathematics',
    dfw_rate: 0.22
})

CREATE (:Section {
    id: 'section-uuid',
    course_id: 'course-uuid',
    term: 'Fall 2026',
    instructor_id: 'faculty-uuid',
    delivery_mode: 'in_person'
})

CREATE (:Programme {
    id: 'programme-uuid',
    code: 'CS-BS',
    name: 'B.S. Computer Science',
    total_credits: 120
})

// ============================================================
// SUPPORT & INTERVENTION NODES
// ============================================================

CREATE (:AlertFlag {
    id: 'alert-uuid',
    flag_type: 'academic',
    severity: 'high',
    status: 'open',
    title: 'Missing 3 assignments',
    created_at: datetime('2026-10-10')
})

CREATE (:Intervention {
    id: 'intervention-uuid',
    type: 'tutoring_referral',
    status: 'completed',
    outcome: 'successful',
    gpa_improvement: 0.4
})

CREATE (:Resource {
    id: 'resource-uuid',
    type: 'tutoring_center',
    name: 'Math Tutoring Center',
    department: 'Academic Support',
    location: 'Library 2nd Floor'
})
```

### Edge Types (Relationships)

```cypher
// ============================================================
// CARE NETWORK RELATIONSHIPS
// These edges define who is connected to a student and in what capacity
// ============================================================

// Advisor-student care relationship
CREATE (advisor:Staff)-[:ADVISES {
    type: 'primary',
    since: date('2026-08-15'),
    assignment_id: 'assignment-uuid',
    can_view: ['profile', 'grades', 'alerts', 'notes'],
    can_edit: ['alerts', 'notes', 'success_plans']
}]->(student:Student)

// Faculty-student teaching relationship
CREATE (faculty:Staff)-[:TEACHES {
    section_id: 'section-uuid',
    term: 'Fall 2026',
    can_view: ['profile', 'grades_own_course', 'alerts_own_course'],
    can_edit: ['alerts']
}]->(student:Student)

// Tutor-student support relationship
CREATE (tutor:Staff)-[:TUTORS {
    subject: 'Mathematics',
    since: date('2026-09-20'),
    sessions_completed: 4,
    can_view: ['profile', 'grades'],
    can_edit: []
}]->(student:Student)

// Counsellor-student restricted relationship
CREATE (counsellor:Staff)-[:COUNSELS {
    since: date('2026-10-12'),
    is_restricted: true,
    hipaa_applicable: true,
    can_view: ['profile'],
    can_edit: ['restricted_notes']
}]->(student:Student)

// Peer mentor relationship
CREATE (mentor:Student)-[:MENTORS {
    programme: 'peer_mentoring_2026',
    matched_by: 'ai_algorithm',
    match_score: 0.89,
    since: date('2026-09-01'),
    meetings_completed: 3
}]->(mentee:Student)

// ============================================================
// ACADEMIC RELATIONSHIPS
// ============================================================

// Student enrolled in section
CREATE (student:Student)-[:ENROLLED_IN {
    enrollment_id: 'enrollment-uuid',
    status: 'enrolled',
    midterm_grade: 'C-',
    current_grade: 'C',
    last_activity: datetime('2026-10-14')
}]->(section:Section)

// Section is instance of course
CREATE (section:Section)-[:INSTANCE_OF]->(course:Course)

// Course prerequisites (directed graph for chain analysis)
CREATE (calc2:Course)-[:REQUIRES {
    min_grade: 'C',
    is_corequisite: false
}]->(calc1:Course)

// Student pursuing programme
CREATE (student:Student)-[:PURSUING {
    type: 'major',
    declared_date: date('2026-01-15'),
    credits_completed: 45,
    credits_remaining: 75,
    completion_pct: 37.5
}]->(programme:Programme)

// Course satisfies programme requirement
CREATE (course:Course)-[:SATISFIES {
    requirement_group: 'Core Requirements',
    required: true,
    min_grade: 'C'
}]->(programme:Programme)

// ============================================================
// ALERT & INTERVENTION RELATIONSHIPS
// ============================================================

// Alert raised for student
CREATE (alert:AlertFlag)-[:FLAGGED]->(student:Student)

// Alert raised by staff member
CREATE (faculty:Staff)-[:RAISED]->(alert:AlertFlag)

// Alert assigned to staff member
CREATE (alert:AlertFlag)-[:ASSIGNED_TO]->(advisor:Staff)

// Intervention triggered by alert
CREATE (alert:AlertFlag)-[:TRIGGERED]->(intervention:Intervention)

// Intervention applied to student
CREATE (intervention:Intervention)-[:APPLIED_TO]->(student:Student)

// Intervention involved resource
CREATE (intervention:Intervention)-[:USED]->(resource:Resource)

// Staff initiated intervention
CREATE (advisor:Staff)-[:INITIATED]->(intervention:Intervention)

// ============================================================
// REFERRAL CHAINS
// Staff member referred student to another staff member
// ============================================================
CREATE (advisor:Staff)-[:REFERRED {
    student_id: 'student-uuid',
    reason: 'financial_stress',
    date: date('2026-10-15'),
    alert_id: 'alert-uuid'
}]->(financial_aid:Staff)
```

### Graph Sync Table (Relational)

```sql
-- Tracks synchronisation between relational and graph layers
CREATE TABLE graph_sync_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    entity_type VARCHAR(50) NOT NULL,  -- 'student', 'staff', 'course', 'enrollment', 'alert'
    entity_id UUID NOT NULL,
    operation VARCHAR(20) NOT NULL,  -- 'create_node', 'update_node', 'create_edge', 'update_edge', 'delete_edge'
    graph_node_id VARCHAR(100),  -- Neo4j internal ID or AGE vertex ID
    sync_status VARCHAR(20) NOT NULL DEFAULT 'pending',  -- 'pending', 'synced', 'failed'
    error_message TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    synced_at TIMESTAMPTZ
);

CREATE INDEX idx_graph_sync_pending ON graph_sync_log(sync_status) WHERE sync_status = 'pending';
```

---

## Example Graph Queries

### Care network: "Who is connected to this student and in what capacity?"

```cypher
// Full care network for a student
MATCH (person)-[r]->(s:Student {id: 'student-uuid'})
WHERE type(r) IN ['ADVISES', 'TEACHES', 'TUTORS', 'COUNSELS', 'MENTORS']
RETURN person.name AS name,
       person.user_type AS role,
       type(r) AS relationship,
       r.since AS since,
       r.can_view AS permissions
ORDER BY r.since
```

### Prerequisite chain: "What is the full prerequisite tree for this course?"

```cypher
// Recursive prerequisite chain traversal
MATCH path = (target:Course {code: 'CS-401'})-[:REQUIRES*]->(prereq:Course)
RETURN [node IN nodes(path) | node.code] AS prerequisite_chain,
       length(path) AS depth
ORDER BY depth DESC
```

### Peer matching: "Find similar students who were successfully retained"

```cypher
// Find students with similar risk profiles who improved after intervention
MATCH (target:Student {id: 'student-uuid'})
MATCH (peer:Student)-[:PURSUING]->(prog:Programme)
WHERE peer.institution_id = target.institution_id
  AND peer.id <> target.id
  AND peer.first_generation = target.first_generation
  AND peer.pell_eligible = target.pell_eligible
  AND abs(peer.gpa - target.gpa) < 0.3
  AND peer.enrollment_status = 'full_time'

// Find peers who had interventions with successful outcomes
MATCH (intervention:Intervention)-[:APPLIED_TO]->(peer)
WHERE intervention.outcome = 'successful'

RETURN peer.name,
       peer.gpa AS gpa_now,
       intervention.type AS intervention_type,
       intervention.gpa_improvement,
       count(intervention) AS successful_interventions
ORDER BY intervention.gpa_improvement DESC
LIMIT 10
```

### Intervention pathway analysis: "What sequence of interventions worked for similar at-risk students?"

```cypher
// Find successful intervention pathways for students with similar risk profiles
MATCH (s:Student)
WHERE s.institution_id = 'inst-uuid'
  AND s.risk_level IN ['high', 'critical']

MATCH path = (alert:AlertFlag)-[:TRIGGERED]->(i1:Intervention)-[:APPLIED_TO]->(s)
WHERE i1.outcome = 'successful'

// Optionally find multi-step intervention chains
OPTIONAL MATCH (i1)-[:FOLLOWED_BY]->(i2:Intervention)-[:APPLIED_TO]->(s)
WHERE i2.outcome = 'successful'

RETURN i1.type AS first_intervention,
       i2.type AS follow_up_intervention,
       count(DISTINCT s) AS student_count,
       avg(i1.gpa_improvement) AS avg_gpa_improvement
ORDER BY student_count DESC
LIMIT 10
```

### Referral network: "Which advisors refer most frequently to which support services?"

```cypher
MATCH (advisor:Staff)-[:REFERRED]->(target:Staff)
WHERE advisor.institution_id = 'inst-uuid'
RETURN advisor.name AS referrer,
       target.name AS referred_to,
       target.user_type AS target_role,
       count(*) AS referral_count
ORDER BY referral_count DESC
```

### Degree pathway: "What courses can this student take next that satisfy remaining requirements?"

```cypher
// Courses the student has completed
MATCH (s:Student {id: 'student-uuid'})-[e:ENROLLED_IN]->(sec:Section)-[:INSTANCE_OF]->(completed:Course)
WHERE e.status = 'completed' AND e.final_grade >= 'C'
WITH s, collect(completed) AS completed_courses

// Courses that satisfy programme requirements and whose prerequisites are met
MATCH (s)-[:PURSUING]->(prog:Programme)<-[:SATISFIES]-(candidate:Course)
WHERE NOT candidate IN completed_courses

// Check prerequisites are met
OPTIONAL MATCH (candidate)-[:REQUIRES]->(prereq:Course)
WITH s, candidate, completed_courses,
     collect(prereq) AS prereqs

WHERE ALL(p IN prereqs WHERE p IN completed_courses)

RETURN candidate.code AS course_code,
       candidate.title AS title,
       candidate.credits AS credits,
       candidate.dfw_rate AS dfw_rate
ORDER BY candidate.dfw_rate ASC
```

---

## Table Count Summary

| Category | Tables (Relational) | Graph Nodes | Graph Edges | Notes |
|----------|-------------------|-------------|-------------|-------|
| Core Identity | 2 | 2 types | — | institutions, users / Student, Staff nodes |
| Student Profiles | 1 | — | — | Synced to Student graph nodes |
| Academic Structure | 5 | 3 types | 3 types | departments, terms, courses, sections, programmes |
| Enrollments & Grades | 1 | — | 1 type | ENROLLED_IN edge |
| Early Alert & Intervention | 2 | 2 types | 5 types | Rich graph edge types for alert/intervention workflow |
| Advising & Appointments | 2 | — | 4 types | Care network edges (ADVISES, TEACHES, TUTORS, COUNSELS) |
| Risk Scoring | 1 | — | — | Relational; risk_level synced to Student node property |
| AI Advising | 2 | — | — | Relational only |
| Learning Events | 1 | — | — | Partitioned relational |
| Outreach | 0 | — | — | Could be graph-based or added relationally |
| Audit | 1 | — | — | Relational; graph_sync_log for sync tracking |
| Graph Sync | 1 | — | — | Relational tracking of graph synchronisation |
| **Relational Total** | **19** | **7 types** | **13 types** | Plus graph layer |

---

## Key Design Decisions

1. **Graph for relationships, relational for records** — grades, appointments, chatbot messages, and audit logs are transactional data that belongs in PostgreSQL. The care network, prerequisite chains, and intervention pathways are relationship-heavy data that belongs in a graph. Each layer does what it does best.

2. **Care network edges carry permission metadata** — each edge (ADVISES, TEACHES, TUTORS, COUNSELS) includes `can_view` and `can_edit` arrays. This enables FERPA-compliant access control: a tutor can view a student's grades but not their financial aid records. The graph query "what can this staff member see for this student?" is a single edge traversal.

3. **HIPAA segregation via edge properties** — counselling relationships carry `is_restricted: true` and `hipaa_applicable: true`. Queries that populate the care team UI filter out restricted edges unless the requesting user has the counsellor role. This is cleaner than maintaining separate restricted-access tables.

4. **Prerequisite chains as directed graph** — course prerequisites form a DAG (directed acyclic graph). In a relational model, querying "what is the full prerequisite chain for CS-401?" requires recursive CTEs. In the graph, it is a simple variable-length path match: `(target)-[:REQUIRES*]->(prereq)`.

5. **Peer mentoring matching as graph nearest-neighbour** — matching at-risk students with successful near-peers is a similarity query. The graph enables queries like "find students with similar attributes who were retained after tutoring interventions" without pre-computing a similarity matrix.

6. **Graph sync via change-data-capture** — when a relational record changes (new enrollment, alert status update, grade recorded), a row is inserted into `graph_sync_log`. A background worker processes pending sync records and updates the graph. This provides eventual consistency with a clear audit trail of what has been synced.

7. **Relational audit log, not graph** — the audit log is append-only, time-series data that benefits from partitioning and range queries. It stays in PostgreSQL. The graph does not need to track every record access — it focuses on relationships that power the care network and analytics features.

8. **Intervention pathway analysis is a unique graph capability** — "what sequence of interventions worked for students like this one?" is a path-finding query that is impractical in relational SQL. The graph can discover multi-step intervention chains (tutoring then schedule change then financial aid review) that correlated with successful retention.

9. **Degree pathway exploration uses graph traversal** — "what courses can this student take next?" requires checking completed courses, prerequisite satisfaction, and programme requirement coverage simultaneously. In the graph, this is a pattern match; in SQL, it is a complex multi-table JOIN with recursive CTEs.

10. **Apache AGE as a pragmatic alternative to Neo4j** — for teams that want to avoid operating a separate database, Apache AGE adds openCypher support directly to PostgreSQL. This reduces infrastructure complexity at the cost of some graph query performance compared to a dedicated graph database. The schema is designed to work with either approach.

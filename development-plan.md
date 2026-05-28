# Student Success Platform — Phased Development Plan

> Project: Student Success Platform (Candidate #116)
> Created: 2026-05-25
> Based on: research.md, features.md, standards.md, README.md, data-model-suggestion-1 through 4

---

## Table of Contents

1. [Technology Decisions](#technology-decisions)
2. [Project Structure](#project-structure)
3. [Phase Dependency Graph](#phase-dependency-graph)
4. [Phase 1 — Foundation: Core Platform, Auth, and Multi-Tenancy](#phase-1--foundation-core-platform-auth-and-multi-tenancy)
5. [Phase 2 — SIS Integration and Academic Data](#phase-2--sis-integration-and-academic-data)
6. [Phase 3 — Early Alert and Advising Workflow](#phase-3--early-alert-and-advising-workflow)
7. [Phase 4 — Student Portal and Appointment Scheduling](#phase-4--student-portal-and-appointment-scheduling)
8. [Phase 5 — LMS Integration and Learning Event Ingestion](#phase-5--lms-integration-and-learning-event-ingestion)
9. [Phase 6 — Risk Scoring and Predictive Analytics](#phase-6--risk-scoring-and-predictive-analytics)
10. [Phase 7 — Degree Audit and Planning](#phase-7--degree-audit-and-planning)
11. [Phase 8 — AI Advising Chatbot](#phase-8--ai-advising-chatbot)
12. [Phase 9 — Outreach Campaigns and Multi-Channel Messaging](#phase-9--outreach-campaigns-and-multi-channel-messaging)
13. [Phase 10 — Transfer Credit Intelligence](#phase-10--transfer-credit-intelligence)
14. [Phase 11 — Causal Intervention Recommendations and Fairness Auditing](#phase-11--causal-intervention-recommendations-and-fairness-auditing)
15. [Phase 12 — MCP Server, Co-Curricular Data, and Wellbeing Signals](#phase-12--mcp-server-co-curricular-data-and-wellbeing-signals)
16. [Cross-Phase Definitions of Done](#cross-phase-definitions-of-done)

---

## Technology Decisions

### Data Model Selection: Hybrid Relational + JSONB (Suggestion 3) with Selective Event Sourcing from Suggestion 2

**Rationale:** After evaluating all four data model suggestions:

- **Suggestion 1 (Normalized Relational)** provides the strongest referential integrity but at 45-55 tables the schema is rigid and requires migrations for every institution-specific field. Given the higher education market's diversity (US community colleges vs. EU research universities, Banner vs. Colleague vs. PeopleSoft), the inflexibility is a real cost.

- **Suggestion 2 (Event-Sourced CQRS)** provides the best audit trail and temporal query capability, which is deeply aligned with FERPA's March 2025 enforcement guidance. However, the full event-sourcing model adds substantial developer complexity and infrastructure overhead for an MVP. The team would need expertise in event stores, projection engines, and eventual consistency from day one.

- **Suggestion 3 (Hybrid Relational + JSONB)** offers the best balance: 23 tables instead of 45+, relational columns for universal fields (GPA, enrollment status, academic standing), JSONB for institution-specific attributes (placement tests, tribal affiliation, BAfoG status). This gets to MVP fastest while remaining flexible enough for multi-institution SaaS.

- **Suggestion 4 (Graph-Relational)** excels at care-network traversal and prerequisite analysis, but requires dual-database infrastructure from the start. The care-network model can be implemented adequately in relational SQL for initial phases and migrated to a graph layer later if traversal performance becomes a bottleneck.

**Hybrid approach:** Use Suggestion 3 as the primary data model, but adopt Suggestion 2's audit-first philosophy by implementing the `audit_log` as an append-only, partitioned, event-style table. This provides FERPA compliance without the full CQRS machinery. In Phase 11, consider adding selective event sourcing for risk-score history and intervention pathway analysis.

### Technology Stack

| Layer | Technology | Rationale |
|-------|-----------|-----------|
| **Backend API** | Node.js 22 LTS + TypeScript 5.x | Strong async I/O for real-time event ingestion (Caliper, xAPI). TypeScript provides type safety across a large domain model. Largest pool of available developers. |
| **API Framework** | Fastify 5.x | Higher throughput than Express for event ingestion endpoints. Schema-based validation via JSON Schema aligns with OpenAPI spec generation. Plugin architecture matches the platform's modular feature set. |
| **Database** | PostgreSQL 17 | JSONB with GIN indexes for institution-specific fields. Partitioning for learning_events and audit_log. Row-level security for multi-tenancy. Mature, battle-tested, and supported by every cloud provider. |
| **ORM / Query Builder** | Prisma 6.x | Type-safe database access with excellent migration tooling. JSONB support via `Json` type. Prisma Client generates TypeScript types from the schema. |
| **Cache / Session** | Redis 7.x (Valkey) | Session storage for SSO tokens. Cache layer for risk scores and degree audit results. Pub/sub for real-time event distribution to connected clients. |
| **Frontend** | Next.js 15 + React 19 | Server components for SEO-irrelevant advisor dashboards. App Router for layout nesting (institution > role > feature). Server actions for form submissions (alert flags, advising notes). |
| **UI Component Library** | shadcn/ui + Radix UI + Tailwind CSS 4 | Accessible by default (Radix primitives meet WCAG 2.2 AA). Composable, unstyled primitives that can be themed per institution. No vendor lock-in — components are copied into the project. |
| **Authentication** | NextAuth.js 5 (Auth.js) + custom SAML/OIDC provider adapter | Supports both SAML 2.0 (Shibboleth, Azure AD) and OIDC (Okta, Google Workspace) — the two dominant SSO protocols in higher education. SCIM 2.0 adapter for automated provisioning. |
| **AI / LLM** | Anthropic Claude API (Claude Sonnet for chatbot, Claude Haiku for classification) | Conversational advising chatbot and sentiment analysis. Structured tool use for degree audit queries. Prompt caching for institution knowledge base. Cost-effective at scale with batched classification. |
| **Search** | PostgreSQL full-text search (initial); Meilisearch (Phase 7+) | Full-text search over advising notes, course catalogue, and knowledge base. PostgreSQL tsvector is adequate for MVP; Meilisearch adds typo tolerance and faceted search for degree planning. |
| **File Storage** | S3-compatible object storage (AWS S3, MinIO for self-hosted) | Transfer credit transcript documents, export reports, and institution knowledge base PDFs. |
| **Background Jobs** | BullMQ (Redis-backed) | Risk score computation, Caliper/xAPI event processing, campaign message delivery, transfer credit AI matching. Reliable retry semantics and dead-letter queues. |
| **Email / SMS** | Resend (email), Twilio (SMS) | Campaign outreach and appointment reminders. Resend for transactional email with webhook tracking. Twilio for SMS with delivery status callbacks. |
| **Monitoring** | OpenTelemetry + Grafana stack (Prometheus, Loki, Tempo) | Distributed tracing across API, background workers, and LMS integrations. FERPA audit log monitoring. SLA dashboards for institutional clients. |
| **CI/CD** | GitHub Actions | Automated testing, linting, database migrations, and deployment. Preview deployments per PR for QA. |
| **Deployment** | Docker + Kubernetes (cloud-managed) or Docker Compose (self-hosted) | Multi-tenant cloud deployment via Kubernetes with namespace-per-institution isolation. Docker Compose for smaller institutions that self-host. |
| **API Documentation** | OpenAPI 3.1 (auto-generated from Fastify schemas) | Institutional IT teams need reliable API docs for SIS/LMS integration. OpenAPI enables SDK auto-generation. |

### Standards Compliance Strategy

| Standard | Implementation Approach |
|----------|----------------------|
| **FERPA** | Append-only partitioned `audit_log` table logging every data access (user, timestamp, IP, resource, action). Row-level security in PostgreSQL for multi-tenant isolation. `is_restricted` flag on advising notes for HIPAA-sensitive data. 3+ year audit retention per March 2025 guidance. |
| **HIPAA** | Separate `is_restricted` flag on advising notes and chatbot conversations. Counselling-related data filtered from standard advisor queries. Business Associate Agreement template provided to institutional clients. |
| **GDPR** | `compliance_config` JSONB on institutions table with GDPR flags. Human-review workflow gate before any AI-triggered outreach for GDPR-flagged institutions. Data retention policies configurable per institution. Right-to-erasure endpoint for student data. |
| **WCAG 2.2 AA** | Radix UI primitives for accessible components. Automated axe-core testing in CI. Manual audit checklist per WCAG 2.2 success criteria. Focus management and keyboard navigation testing. |
| **IMS Caliper 1.2** | Dedicated ingestion endpoint accepting Caliper Sensor API payloads. Events stored in partitioned `learning_events` table with full JSONB payload. Support for all six v1.2 metric profiles. |
| **xAPI** | Dedicated ingestion endpoint accepting xAPI statements. Actor/verb/object decomposed into relational columns; full statement preserved as JSONB. LRS-compatible conformance. |
| **OneRoster 1.2** | SIS integration adapter consuming OneRoster REST API and CSV batch formats. Maps six core entities to platform tables. |
| **LTI 1.3** | LTI launch endpoint for embedding student success dashboards inside LMS. OIDC + signed JWT authentication. Assignment and Grade Services (AGS) v2.0 for grade write-back. |
| **OAuth 2.0 / OIDC** | API authentication via OAuth 2.0 Bearer tokens. Institutional SSO via OIDC or SAML 2.0 provider adapters. SCIM 2.0 for automated user provisioning. |
| **OpenAPI 3.1** | All REST endpoints documented in OpenAPI 3.1. Auto-generated from Fastify route schemas. Published at `/api/docs`. |

---

## Project Structure

```
student-success-platform/
├── apps/
│   ├── api/                          # Fastify API server
│   │   ├── src/
│   │   │   ├── modules/
│   │   │   │   ├── auth/             # SSO, SAML, OIDC, SCIM
│   │   │   │   ├── institutions/     # Multi-tenant config
│   │   │   │   ├── users/            # User management, roles
│   │   │   │   ├── students/         # Student profiles, demographics
│   │   │   │   ├── academic/         # Terms, courses, sections, programmes
│   │   │   │   ├── enrollments/      # Enrollments, grades, transfers
│   │   │   │   ├── alerts/           # Early alert flags, resolution
│   │   │   │   ├── advising/         # Appointments, notes, success plans
│   │   │   │   ├── risk/             # Risk scoring, models, computation
│   │   │   │   ├── degree-audit/     # Degree audit, what-if analysis
│   │   │   │   ├── chatbot/          # AI advising conversations
│   │   │   │   ├── campaigns/        # Outreach campaigns, messages
│   │   │   │   ├── transfers/        # Transfer credit intelligence
│   │   │   │   ├── integrations/     # SIS, LMS, Caliper, xAPI adapters
│   │   │   │   └── audit/            # FERPA audit log, compliance
│   │   │   ├── plugins/             # Fastify plugins (auth, RLS, rate-limit)
│   │   │   ├── lib/                 # Shared utilities
│   │   │   └── server.ts
│   │   ├── prisma/
│   │   │   ├── schema.prisma
│   │   │   └── migrations/
│   │   └── package.json
│   │
│   ├── web/                          # Next.js frontend
│   │   ├── src/
│   │   │   ├── app/
│   │   │   │   ├── (auth)/           # Login, SSO callback
│   │   │   │   ├── (advisor)/        # Advisor dashboards
│   │   │   │   ├── (faculty)/        # Faculty alert creation
│   │   │   │   ├── (student)/        # Student portal
│   │   │   │   ├── (admin)/          # Institution admin
│   │   │   │   └── api/              # Next.js API routes (BFF)
│   │   │   ├── components/
│   │   │   │   ├── ui/               # shadcn/ui components
│   │   │   │   ├── alerts/
│   │   │   │   ├── advising/
│   │   │   │   ├── degree-audit/
│   │   │   │   ├── chatbot/
│   │   │   │   └── dashboards/
│   │   │   └── lib/
│   │   └── package.json
│   │
│   └── workers/                      # Background job processors
│       ├── src/
│       │   ├── risk-computation/
│       │   ├── event-ingestion/
│       │   ├── campaign-delivery/
│       │   └── transfer-matching/
│       └── package.json
│
├── packages/
│   ├── shared-types/                 # TypeScript types shared across apps
│   ├── caliper-adapter/              # Caliper 1.2 event parser
│   ├── xapi-adapter/                 # xAPI statement parser
│   ├── oneroster-adapter/            # OneRoster 1.2 client
│   ├── lti-provider/                 # LTI 1.3 launch provider
│   └── risk-engine/                  # Risk scoring algorithms
│
├── infrastructure/
│   ├── docker/
│   │   ├── docker-compose.yml        # Local dev + self-hosted
│   │   ├── docker-compose.prod.yml
│   │   └── Dockerfile.*
│   ├── kubernetes/                   # Cloud deployment manifests
│   └── terraform/                    # Infrastructure as code
│
├── docs/
│   ├── api/                          # OpenAPI specs
│   ├── architecture/                 # ADRs, diagrams
│   ├── compliance/                   # FERPA, HIPAA, GDPR guides
│   └── integration/                  # SIS/LMS integration guides
│
├── turbo.json                        # Turborepo config
├── pnpm-workspace.yaml
└── package.json
```

**Monorepo rationale:** Turborepo with pnpm workspaces enables shared TypeScript types across the API, frontend, and worker processes. The `packages/` directory contains reusable adapters for education standards (Caliper, xAPI, OneRoster, LTI) that are versioned and tested independently. This structure supports the modular deployment model (cloud-managed Kubernetes for large institutions, Docker Compose for self-hosted community colleges).

---

## Phase Dependency Graph

```
Phase 1 ─── Foundation (Auth, Multi-Tenancy, Core Schema)
   │
   ├──────► Phase 2 ─── SIS Integration + Academic Data
   │           │
   │           ├──────► Phase 3 ─── Early Alert + Advising Workflow
   │           │           │
   │           │           ├──────► Phase 4 ─── Student Portal + Appointments
   │           │           │           │
   │           │           │           └──────► Phase 9 ─── Outreach Campaigns
   │           │           │
   │           │           └──────► Phase 6 ─── Risk Scoring (also needs Phase 5)
   │           │                       │
   │           │                       ├──────► Phase 8 ─── AI Advising Chatbot (also needs Phase 7)
   │           │                       │
   │           │                       └──────► Phase 11 ── Causal Interventions + Fairness
   │           │
   │           ├──────► Phase 5 ─── LMS Integration + Learning Events
   │           │
   │           ├──────► Phase 7 ─── Degree Audit + Planning
   │           │           │
   │           │           └──────► Phase 10 ── Transfer Credit Intelligence
   │           │
   │           └──────► Phase 12 ── MCP Server + Co-Curricular + Wellbeing
   │
   └──────► (all phases depend on Phase 1)
```

**Critical path:** Phase 1 > Phase 2 > Phase 3 > Phase 6 > Phase 11

**Parallelizable:** After Phase 2 completes, Phases 3, 5, and 7 can proceed in parallel. After Phase 3, Phases 4 and 6 can proceed in parallel (Phase 6 also requires Phase 5).

---

## Phase 1 — Foundation: Core Platform, Auth, and Multi-Tenancy

**Duration estimate:** 6-8 weeks
**Dependencies:** None (starting phase)

### Task 1.1 — Project Scaffolding and Monorepo Setup

**What:** Initialize the Turborepo monorepo with pnpm workspaces. Create the `apps/api`, `apps/web`, `apps/workers`, and `packages/shared-types` workspaces. Configure TypeScript, ESLint, Prettier, and Husky pre-commit hooks. Set up CI pipeline with GitHub Actions.

**Design:**

```typescript
// turbo.json
{
  "$schema": "https://turbo.build/schema.json",
  "globalDependencies": ["**/.env.*local"],
  "pipeline": {
    "build": { "dependsOn": ["^build"], "outputs": ["dist/**", ".next/**"] },
    "dev": { "cache": false, "persistent": true },
    "lint": {},
    "test": { "dependsOn": ["^build"] },
    "test:e2e": { "dependsOn": ["build"] },
    "db:migrate": { "cache": false },
    "db:seed": { "cache": false, "dependsOn": ["db:migrate"] }
  }
}

// pnpm-workspace.yaml
packages:
  - 'apps/*'
  - 'packages/*'
```

```typescript
// packages/shared-types/src/index.ts
export type InstitutionId = string & { readonly __brand: 'InstitutionId' };
export type UserId = string & { readonly __brand: 'UserId' };
export type StudentProfileId = string & { readonly __brand: 'StudentProfileId' };

export type UserType = 'student' | 'advisor' | 'faculty' | 'staff' | 'admin';

export type RiskLevel = 'low' | 'moderate' | 'high' | 'critical';

export type AlertStatus = 'open' | 'in_progress' | 'resolved' | 'dismissed';
export type AlertSeverity = 'low' | 'medium' | 'high' | 'critical';
export type AlertType = 'academic' | 'attendance' | 'financial' | 'wellbeing' | 'engagement';

export type EnrollmentStatus = 'enrolled' | 'dropped' | 'withdrawn' | 'completed';
export type AcademicStanding = 'good' | 'probation' | 'suspension' | 'dean_list';
```

**Testing:**
- `pnpm build` completes without errors across all workspaces
- `pnpm lint` passes with zero warnings
- `pnpm test` runs (initially empty test suites, but harness works)
- GitHub Actions CI pipeline triggers on push and PR, runs build + lint + test
- TypeScript strict mode enabled; no `any` types in shared-types

### Task 1.2 — PostgreSQL Schema and Prisma Setup

**What:** Implement the core database schema following the Hybrid Relational + JSONB data model (Suggestion 3). Create the `institutions`, `users`, `student_profiles`, `academic_terms`, and `audit_log` tables. Configure Prisma with migration support and seed scripts.

**Design:**

```prisma
// apps/api/prisma/schema.prisma
generator client {
  provider        = "prisma-client-js"
  previewFeatures = ["multiSchema", "postgresqlExtensions"]
}

datasource db {
  provider   = "postgresql"
  url        = env("DATABASE_URL")
  extensions = [pgcrypto]
}

model Institution {
  id               String   @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  name             String   @db.VarChar(255)
  shortCode        String   @unique @map("short_code") @db.VarChar(50)
  timezone         String   @default("America/New_York") @db.VarChar(50)
  countryCode      String   @default("US") @map("country_code") @db.Char(2)
  complianceConfig Json     @default("{}") @map("compliance_config")
  integrationConfig Json    @default("{}") @map("integration_config")
  customFieldsSchema Json   @default("{}") @map("custom_fields_schema")
  createdAt        DateTime @default(now()) @map("created_at") @db.Timestamptz()
  updatedAt        DateTime @default(now()) @updatedAt @map("updated_at") @db.Timestamptz()

  users         User[]
  academicTerms AcademicTerm[]

  @@map("institutions")
}

model User {
  id            String   @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  institutionId String   @map("institution_id") @db.Uuid
  externalIds   Json     @default("{}") @map("external_ids")
  email         String   @db.VarChar(255)
  firstName     String   @map("first_name") @db.VarChar(100)
  lastName      String   @map("last_name") @db.VarChar(100)
  preferredName String?  @map("preferred_name") @db.VarChar(100)
  userType      String   @map("user_type") @db.VarChar(20)
  roles         Json     @default("[]")
  isActive      Boolean  @default(true) @map("is_active")
  profileData   Json     @default("{}") @map("profile_data")
  lastLoginAt   DateTime? @map("last_login_at") @db.Timestamptz()
  createdAt     DateTime @default(now()) @map("created_at") @db.Timestamptz()
  updatedAt     DateTime @default(now()) @updatedAt @map("updated_at") @db.Timestamptz()

  institution    Institution     @relation(fields: [institutionId], references: [id])
  studentProfile StudentProfile?

  @@unique([institutionId, email])
  @@index([institutionId])
  @@index([institutionId, userType])
  @@map("users")
}

model StudentProfile {
  id                 String   @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  userId             String   @unique @map("user_id") @db.Uuid
  institutionId      String   @map("institution_id") @db.Uuid
  studentIdNumber    String   @map("student_id_number") @db.VarChar(50)
  cumulativeGpa      Decimal? @map("cumulative_gpa") @db.Decimal(4, 3)
  totalCreditsEarned Decimal? @map("total_credits_earned") @db.Decimal(6, 2)
  academicStanding   String?  @map("academic_standing") @db.VarChar(30)
  enrollmentStatus   String?  @map("enrollment_status") @db.VarChar(30)
  admissionType      String?  @map("admission_type") @db.VarChar(30)
  dateOfBirth        DateTime? @map("date_of_birth") @db.Date
  gender             String?  @db.VarChar(30)
  ethnicity          String?  @db.VarChar(50)
  firstGeneration    Boolean? @map("first_generation")
  pellEligible       Boolean? @map("pell_eligible")
  currentRiskLevel   String?  @map("current_risk_level") @db.VarChar(20)
  currentRiskScore   Decimal? @map("current_risk_score") @db.Decimal(5, 4)
  riskUpdatedAt      DateTime? @map("risk_updated_at") @db.Timestamptz()
  extendedAttributes Json     @default("{}") @map("extended_attributes")
  createdAt          DateTime @default(now()) @map("created_at") @db.Timestamptz()
  updatedAt          DateTime @default(now()) @updatedAt @map("updated_at") @db.Timestamptz()

  user User @relation(fields: [userId], references: [id])

  @@index([institutionId])
  @@index([institutionId, currentRiskLevel])
  @@index([institutionId, academicStanding])
  @@map("student_profiles")
}
```

```sql
-- Manual migration for partitioned audit_log (Prisma does not support partitioning natively)
-- apps/api/prisma/migrations/manual/001_audit_log_partitioned.sql

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
    occurred_at TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (occurred_at);

CREATE TABLE audit_log_2026_q3 PARTITION OF audit_log
    FOR VALUES FROM ('2026-07-01') TO ('2026-10-01');
CREATE TABLE audit_log_2026_q4 PARTITION OF audit_log
    FOR VALUES FROM ('2026-10-01') TO ('2027-01-01');

CREATE INDEX idx_audit_user ON audit_log(user_id, occurred_at);
CREATE INDEX idx_audit_student ON audit_log(student_profile_id, occurred_at);
CREATE INDEX idx_audit_resource ON audit_log(resource_type, resource_id);
```

**Testing:**
- `prisma migrate dev` creates tables successfully in a fresh PostgreSQL 17 database
- `prisma db seed` populates a demo institution with 3 users (admin, advisor, student)
- Verify all indexes exist via `\di` in psql
- Verify audit_log partitions exist via `\d+ audit_log`
- Round-trip test: create institution via Prisma, read back, verify all JSONB fields deserialize correctly
- Test unique constraint: duplicate `(institution_id, email)` insert fails with expected error

### Task 1.3 — Fastify API Server with Health Check and OpenAPI

**What:** Set up the Fastify API server with auto-generated OpenAPI 3.1 documentation, request validation via JSON Schema, CORS configuration, and health check endpoint. Configure the Prisma client as a Fastify plugin.

**Design:**

```typescript
// apps/api/src/server.ts
import Fastify from 'fastify';
import fastifySwagger from '@fastify/swagger';
import fastifySwaggerUi from '@fastify/swagger-ui';
import fastifyCors from '@fastify/cors';
import { prismaPlugin } from './plugins/prisma.js';
import { auditPlugin } from './plugins/audit.js';
import { healthRoutes } from './modules/health/routes.js';
import { institutionRoutes } from './modules/institutions/routes.js';

const app = Fastify({
  logger: {
    level: process.env.LOG_LEVEL || 'info',
    transport: process.env.NODE_ENV === 'development'
      ? { target: 'pino-pretty' }
      : undefined,
  },
  genReqId: () => crypto.randomUUID(),
});

await app.register(fastifyCors, {
  origin: process.env.CORS_ORIGINS?.split(',') || ['http://localhost:3000'],
  credentials: true,
});

await app.register(fastifySwagger, {
  openapi: {
    openapi: '3.1.0',
    info: {
      title: 'Student Success Platform API',
      version: '0.1.0',
      description: 'API for the Student Success Platform',
    },
    components: {
      securitySchemes: {
        bearerAuth: { type: 'http', scheme: 'bearer', bearerFormat: 'JWT' },
      },
    },
  },
});

await app.register(fastifySwaggerUi, { routePrefix: '/api/docs' });
await app.register(prismaPlugin);
await app.register(auditPlugin);
await app.register(healthRoutes, { prefix: '/api/health' });
await app.register(institutionRoutes, { prefix: '/api/v1/institutions' });

await app.listen({ port: parseInt(process.env.PORT || '4000'), host: '0.0.0.0' });
```

```typescript
// apps/api/src/plugins/audit.ts
import { FastifyPluginAsync } from 'fastify';
import fp from 'fastify-plugin';

const auditPlugin: FastifyPluginAsync = async (fastify) => {
  fastify.decorate('audit', {
    async log(params: {
      institutionId: string;
      userId: string;
      action: string;
      resourceType: string;
      resourceId: string;
      studentProfileId?: string;
      ipAddress?: string;
      details?: Record<string, unknown>;
    }) {
      await fastify.prisma.$executeRaw`
        INSERT INTO audit_log (institution_id, user_id, action, resource_type, resource_id,
                               student_profile_id, ip_address, audit_details)
        VALUES (${params.institutionId}::uuid, ${params.userId}::uuid, ${params.action},
                ${params.resourceType}, ${params.resourceId}::uuid,
                ${params.studentProfileId ? params.studentProfileId : null}::uuid,
                ${params.ipAddress}::inet,
                ${JSON.stringify(params.details || {})}::jsonb)
      `;
    },
  });
};

export default fp(auditPlugin);
export { auditPlugin };
```

**Testing:**
- `GET /api/health` returns `{ status: "ok", database: "connected", version: "0.1.0" }` with 200
- `GET /api/health` returns 503 when database is unreachable
- `GET /api/docs` serves Swagger UI with OpenAPI 3.1 spec
- `GET /api/docs/json` returns valid OpenAPI JSON parseable by swagger-parser
- CORS headers present for configured origins; absent for unconfigured origins
- Every request gets a unique `x-request-id` header
- Audit log plugin writes to the partitioned audit_log table successfully

### Task 1.4 — Authentication: SAML 2.0 and OIDC SSO

**What:** Implement institutional SSO via SAML 2.0 (for Shibboleth/Azure AD institutions) and OpenID Connect (for Okta/Google Workspace institutions). Store identity provider configuration per institution. Implement JWT session tokens with role-based claims. Add SCIM 2.0 endpoint for automated user provisioning.

**Design:**

```typescript
// apps/api/src/modules/auth/saml-strategy.ts
import { Strategy as SamlStrategy } from 'passport-saml';

export function createSamlStrategy(idpConfig: {
  entryPoint: string;
  issuer: string;
  cert: string;
  callbackUrl: string;
}) {
  return new SamlStrategy(
    {
      entryPoint: idpConfig.entryPoint,
      issuer: idpConfig.issuer,
      cert: idpConfig.cert,
      callbackUrl: idpConfig.callbackUrl,
      wantAuthnResponseSigned: true,
      wantAssertionsSigned: true,
    },
    (profile, done) => {
      // Map SAML attributes to platform user
      const userData = {
        email: profile.email || profile.nameID,
        firstName: profile.firstName || profile['urn:oid:2.5.4.42'],
        lastName: profile.lastName || profile['urn:oid:2.5.4.4'],
        externalId: profile.nameID,
      };
      done(null, userData);
    },
  );
}

// apps/api/src/modules/auth/jwt.ts
import { SignJWT, jwtVerify } from 'jose';

export interface JwtPayload {
  sub: string;           // user ID
  institutionId: string;
  userType: string;
  roles: string[];
  iat: number;
  exp: number;
}

export async function createSessionToken(user: {
  id: string;
  institutionId: string;
  userType: string;
  roles: Array<{ role: string }>;
}): Promise<string> {
  const secret = new TextEncoder().encode(process.env.JWT_SECRET);
  return new SignJWT({
    sub: user.id,
    institutionId: user.institutionId,
    userType: user.userType,
    roles: user.roles.map(r => r.role),
  })
    .setProtectedHeader({ alg: 'HS256' })
    .setIssuedAt()
    .setExpirationTime('8h')
    .sign(secret);
}

export async function verifySessionToken(token: string): Promise<JwtPayload> {
  const secret = new TextEncoder().encode(process.env.JWT_SECRET);
  const { payload } = await jwtVerify(token, secret);
  return payload as unknown as JwtPayload;
}
```

```typescript
// apps/api/src/modules/auth/rbac.ts
export const PERMISSIONS = {
  student: {
    canView: ['own_profile', 'own_grades', 'own_alerts', 'own_appointments', 'own_degree_progress'],
    canEdit: ['own_profile_limited'],
  },
  advisor: {
    canView: ['assigned_students', 'student_profiles', 'student_grades', 'alerts', 'advising_notes'],
    canEdit: ['alerts', 'advising_notes', 'success_plans', 'appointments'],
  },
  faculty: {
    canView: ['enrolled_students', 'own_course_grades'],
    canEdit: ['alerts_own_course'],
  },
  admin: {
    canView: ['all'],
    canEdit: ['all'],
  },
} as const;

export function requireRole(...allowedRoles: string[]) {
  return async (request: FastifyRequest, reply: FastifyReply) => {
    const { userType, roles } = request.user;
    if (!allowedRoles.includes(userType) && !roles.some(r => allowedRoles.includes(r))) {
      return reply.status(403).send({
        type: 'https://api.studentsuccess.dev/errors/forbidden',
        title: 'Forbidden',
        status: 403,
        detail: `Role ${userType} is not authorized for this action`,
      });
    }
  };
}
```

**Testing:**
- SAML SSO flow: initiate login > redirect to mock IdP > callback with assertion > JWT issued
- OIDC SSO flow: initiate login > redirect to mock OIDC provider > callback with code > token exchange > JWT issued
- JWT token contains correct `sub`, `institutionId`, `userType`, and `roles` claims
- Expired JWT returns 401 with RFC 7807 error body
- Invalid JWT signature returns 401
- RBAC: advisor can access `GET /api/v1/students/:id` for assigned student
- RBAC: student cannot access `GET /api/v1/students/:otherId` (403)
- RBAC: faculty can create alert for own course section only
- SCIM `POST /api/v1/scim/Users` creates user with correct institution mapping
- SCIM `DELETE /api/v1/scim/Users/:id` deactivates (not deletes) user
- Audit log entry created for every login event

### Task 1.5 — Row-Level Security and Multi-Tenant Isolation

**What:** Implement PostgreSQL row-level security (RLS) policies to enforce data isolation between institutions at the database level. Configure the Prisma client to set the session-level `app.current_institution_id` variable on every query.

**Design:**

```sql
-- apps/api/prisma/migrations/manual/002_row_level_security.sql

ALTER TABLE users ENABLE ROW LEVEL SECURITY;
ALTER TABLE student_profiles ENABLE ROW LEVEL SECURITY;
ALTER TABLE audit_log ENABLE ROW LEVEL SECURITY;

CREATE POLICY users_institution_isolation ON users
    USING (institution_id = current_setting('app.current_institution_id')::uuid);

CREATE POLICY student_profiles_institution_isolation ON student_profiles
    USING (institution_id = current_setting('app.current_institution_id')::uuid);

CREATE POLICY audit_log_institution_isolation ON audit_log
    USING (institution_id = current_setting('app.current_institution_id')::uuid);
```

```typescript
// apps/api/src/plugins/prisma.ts
import { PrismaClient } from '@prisma/client';
import { FastifyPluginAsync } from 'fastify';
import fp from 'fastify-plugin';

const prismaPlugin: FastifyPluginAsync = async (fastify) => {
  const prisma = new PrismaClient();

  fastify.decorate('prisma', prisma);

  // Set institution context for RLS on each request
  fastify.addHook('preHandler', async (request) => {
    if (request.user?.institutionId) {
      await prisma.$executeRawUnsafe(
        `SET LOCAL app.current_institution_id = '${request.user.institutionId}'`
      );
    }
  });

  fastify.addHook('onClose', async () => {
    await prisma.$disconnect();
  });
};

export default fp(prismaPlugin);
export { prismaPlugin };
```

**Testing:**
- Institution A user cannot read Institution B's users even with a direct SQL query through Prisma
- Admin user from Institution A sees only Institution A's student profiles
- Audit log entries from Institution A are invisible to Institution B admin
- RLS policies do not interfere with database migrations (superuser bypasses RLS)
- Performance test: RLS-enabled queries add <1ms overhead vs. non-RLS queries on 10k row dataset

### Task 1.6 — Docker Compose Development Environment

**What:** Create Docker Compose configuration for local development including PostgreSQL 17, Redis 7, and the API server with hot reload. Include a seed script that populates a demo institution.

**Design:**

```yaml
# infrastructure/docker/docker-compose.yml
version: '3.9'

services:
  postgres:
    image: postgres:17-alpine
    environment:
      POSTGRES_DB: student_success
      POSTGRES_USER: ssp
      POSTGRES_PASSWORD: localdev
    ports:
      - '5432:5432'
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ['CMD-SHELL', 'pg_isready -U ssp -d student_success']
      interval: 5s
      timeout: 3s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - '6379:6379'
    healthcheck:
      test: ['CMD', 'redis-cli', 'ping']
      interval: 5s
      timeout: 3s
      retries: 5

  api:
    build:
      context: ../..
      dockerfile: infrastructure/docker/Dockerfile.api
      target: development
    ports:
      - '4000:4000'
    environment:
      DATABASE_URL: postgresql://ssp:localdev@postgres:5432/student_success
      REDIS_URL: redis://redis:6379
      JWT_SECRET: local-dev-secret-do-not-use-in-production
      NODE_ENV: development
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    volumes:
      - ../../apps/api/src:/app/apps/api/src

volumes:
  pgdata:
```

**Testing:**
- `docker compose up` starts all services without errors
- API health check returns 200 within 30 seconds of startup
- Hot reload: changing a route file triggers automatic server restart
- `docker compose down -v` cleanly removes all containers and volumes
- Seed data accessible: `GET /api/v1/institutions` returns the demo institution

### Definition of Done — Phase 1

- [ ] Monorepo builds and lints cleanly across all workspaces
- [ ] PostgreSQL schema created with all Phase 1 tables and indexes
- [ ] Fastify API server running with OpenAPI 3.1 docs at `/api/docs`
- [ ] SAML 2.0 and OIDC SSO flows complete end-to-end with mock providers
- [ ] JWT-based session with role claims issued on login
- [ ] RBAC middleware enforces role-based access control
- [ ] Row-level security isolates institution data at the database level
- [ ] FERPA audit log captures every data access event
- [ ] Docker Compose local dev environment starts in under 60 seconds
- [ ] CI pipeline passes build, lint, and test on every push
- [ ] 90%+ unit test coverage on auth and RBAC modules

---

## Phase 2 — SIS Integration and Academic Data

**Duration estimate:** 5-7 weeks
**Dependencies:** Phase 1

### Task 2.1 — Academic Structure Tables and CRUD API

**What:** Add Prisma models and API endpoints for `academic_terms`, `departments`, `courses`, `course_sections`, and `degree_programmes`. These are the reference data tables that SIS integration will populate.

**Design:**

```typescript
// apps/api/src/modules/academic/routes.ts
import { FastifyPluginAsync } from 'fastify';

const academicRoutes: FastifyPluginAsync = async (fastify) => {
  // GET /api/v1/academic/terms
  fastify.get('/terms', {
    schema: {
      querystring: {
        type: 'object',
        properties: {
          is_current: { type: 'boolean' },
        },
      },
      response: {
        200: {
          type: 'array',
          items: {
            type: 'object',
            properties: {
              id: { type: 'string', format: 'uuid' },
              name: { type: 'string' },
              termType: { type: 'string', enum: ['fall', 'spring', 'summer', 'winter'] },
              startDate: { type: 'string', format: 'date' },
              endDate: { type: 'string', format: 'date' },
              isCurrent: { type: 'boolean' },
            },
          },
        },
      },
    },
    handler: async (request) => {
      const { institutionId } = request.user;
      return fastify.prisma.academicTerm.findMany({
        where: { institutionId, ...(request.query.is_current !== undefined && { isCurrent: request.query.is_current }) },
        orderBy: { startDate: 'desc' },
      });
    },
  });

  // GET /api/v1/academic/courses
  fastify.get('/courses', {
    schema: {
      querystring: {
        type: 'object',
        properties: {
          subject_code: { type: 'string' },
          search: { type: 'string' },
          page: { type: 'integer', minimum: 1, default: 1 },
          limit: { type: 'integer', minimum: 1, maximum: 100, default: 50 },
        },
      },
    },
    handler: async (request) => {
      const { institutionId } = request.user;
      const { subject_code, search, page, limit } = request.query;
      const skip = (page - 1) * limit;

      const where = {
        institutionId,
        ...(subject_code && { subjectCode: subject_code }),
        ...(search && {
          OR: [
            { title: { contains: search, mode: 'insensitive' } },
            { subjectCode: { contains: search, mode: 'insensitive' } },
          ],
        }),
      };

      const [courses, total] = await Promise.all([
        fastify.prisma.course.findMany({ where, skip, take: limit, orderBy: { subjectCode: 'asc' } }),
        fastify.prisma.course.count({ where }),
      ]);

      return { data: courses, total, page, limit };
    },
  });
};
```

**Testing:**
- CRUD operations for all academic entities return correct HTTP status codes (200, 201, 400, 404)
- Pagination returns correct `total`, `page`, and `limit` metadata
- Search by course title substring returns matching results (case-insensitive)
- Institution isolation: courses from Institution A not visible to Institution B user
- Audit log entry created for each write operation
- OpenAPI spec updated automatically with new endpoints

### Task 2.2 — OneRoster 1.2 Integration Adapter

**What:** Build a reusable adapter in `packages/oneroster-adapter` that consumes OneRoster 1.2 REST API and CSV batch formats. Map the six core OneRoster entities (organizations, users, courses, classes, enrollments, demographics) to platform tables.

**Design:**

```typescript
// packages/oneroster-adapter/src/client.ts
export interface OneRosterConfig {
  baseUrl: string;
  clientId: string;
  clientSecret: string;
  version: '1.1' | '1.2';
}

export class OneRosterClient {
  constructor(private config: OneRosterConfig) {}

  async getUsers(params?: { filter?: string; limit?: number; offset?: number }) {
    return this.get<OneRosterUser[]>('/ims/oneroster/v1p2/users', params);
  }

  async getEnrollments(params?: { filter?: string; limit?: number; offset?: number }) {
    return this.get<OneRosterEnrollment[]>('/ims/oneroster/v1p2/enrollments', params);
  }

  async getCourses(params?: { filter?: string }) {
    return this.get<OneRosterCourse[]>('/ims/oneroster/v1p2/courses', params);
  }

  async getClasses(params?: { filter?: string }) {
    return this.get<OneRosterClass[]>('/ims/oneroster/v1p2/classes', params);
  }

  private async get<T>(path: string, params?: Record<string, unknown>): Promise<T> {
    const token = await this.authenticate();
    const url = new URL(path, this.config.baseUrl);
    if (params) {
      Object.entries(params).forEach(([key, value]) => {
        if (value !== undefined) url.searchParams.set(key, String(value));
      });
    }
    const response = await fetch(url, {
      headers: { Authorization: `Bearer ${token}`, Accept: 'application/json' },
    });
    if (!response.ok) throw new OneRosterError(response.status, await response.text());
    return response.json();
  }

  private async authenticate(): Promise<string> {
    // OAuth 2.0 client credentials flow per OneRoster spec
    const response = await fetch(`${this.config.baseUrl}/oauth/token`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
      body: new URLSearchParams({
        grant_type: 'client_credentials',
        client_id: this.config.clientId,
        client_secret: this.config.clientSecret,
      }),
    });
    const data = await response.json();
    return data.access_token;
  }
}

// packages/oneroster-adapter/src/mapper.ts
export function mapOneRosterUserToStudent(orUser: OneRosterUser): Partial<StudentProfile> {
  return {
    studentIdNumber: orUser.sourcedId,
    firstName: orUser.givenName,
    lastName: orUser.familyName,
    email: orUser.email,
    userType: orUser.role === 'student' ? 'student' : mapRole(orUser.role),
    extendedAttributes: {
      oneroster_sourced_id: orUser.sourcedId,
      oneroster_status: orUser.status,
      grades: orUser.grades,
    },
  };
}
```

**Testing:**
- OneRoster client authenticates via OAuth 2.0 client credentials flow against a mock server
- `getUsers` returns paginated user list; filter by role works correctly
- `getEnrollments` returns enrollment records linked to classes
- CSV batch import: upload a OneRoster CSV ZIP, verify all 6 entity types parsed and mapped
- Mapper correctly transforms OneRoster user to platform `StudentProfile` shape
- Error handling: 401 from SIS triggers re-authentication; 500 triggers retry with backoff
- Idempotency: re-importing the same OneRoster data does not create duplicates (upsert by `sourcedId`)

### Task 2.3 — SIS Sync Scheduler and Data Integration Management

**What:** Build the integration management UI and background sync scheduler. Institutions configure their SIS connection (Banner, Colleague, PeopleSoft) via the admin panel. The scheduler runs periodic imports (configurable: hourly, daily, or on-demand) using the OneRoster adapter.

**Design:**

```typescript
// apps/api/src/modules/integrations/sync-worker.ts
import { Worker, Job } from 'bullmq';
import { OneRosterClient } from '@ssp/oneroster-adapter';

const sisSyncWorker = new Worker('sis-sync', async (job: Job) => {
  const { institutionId, integrationId } = job.data;

  const integration = await prisma.dataIntegration.findUnique({ where: { id: integrationId } });
  const client = new OneRosterClient({
    baseUrl: integration.endpointUrl,
    clientId: integration.credentials.clientId,
    clientSecret: integration.credentials.clientSecret,
    version: '1.2',
  });

  // Sync in dependency order: orgs > users > courses > classes > enrollments
  const stats = { users: 0, courses: 0, sections: 0, enrollments: 0 };

  const users = await client.getUsers();
  for (const user of users) {
    await upsertUser(institutionId, user);
    stats.users++;
  }

  const courses = await client.getCourses();
  for (const course of courses) {
    await upsertCourse(institutionId, course);
    stats.courses++;
  }

  // ... classes, enrollments

  await prisma.dataIntegration.update({
    where: { id: integrationId },
    data: { lastSyncAt: new Date(), syncStatus: 'healthy' },
  });

  return stats;
}, { connection: redisConnection });
```

**Testing:**
- Admin can create a SIS integration with connection details via `POST /api/v1/admin/integrations`
- Scheduled sync runs at configured interval and populates academic data tables
- Sync stats (users imported, courses synced, errors) returned and stored
- Failed sync updates integration status to `error` with error message
- Partial sync failure (e.g., one course fails) does not abort entire sync — errors logged per entity
- Stale data detection: integration marked `stale` if no sync in 2x the configured interval
- Admin dashboard shows integration health status (healthy/error/stale) with last sync timestamp

### Task 2.4 — Student Profile Import and Enrichment

**What:** Import student profiles from SIS data, mapping OneRoster user records to the `student_profiles` table. Enrich with institution-specific extended attributes. Build the advisor-facing student list and profile detail views.

**Design:**

```typescript
// apps/web/src/app/(advisor)/students/page.tsx
import { StudentListTable } from '@/components/students/student-list-table';

export default async function StudentsPage({
  searchParams,
}: {
  searchParams: { risk?: string; standing?: string; search?: string; page?: string };
}) {
  const students = await fetchStudents({
    riskLevel: searchParams.risk,
    academicStanding: searchParams.standing,
    search: searchParams.search,
    page: parseInt(searchParams.page || '1'),
  });

  return (
    <div className="space-y-6">
      <h1 className="text-2xl font-semibold">My Students</h1>
      <StudentListTable
        students={students.data}
        total={students.total}
        filters={{
          riskLevels: ['low', 'moderate', 'high', 'critical'],
          standings: ['good', 'probation', 'suspension', 'dean_list'],
        }}
      />
    </div>
  );
}
```

**Testing:**
- Student list displays with pagination (50 per page default)
- Filter by risk level returns only matching students
- Filter by academic standing returns only matching students
- Search by name matches first name, last name, and preferred name
- Student profile detail page shows all relational fields and extended attributes
- FERPA audit: viewing a student profile creates an audit_log entry with `action: 'view'`
- Performance: student list query returns in <200ms for 10,000 students with filters applied
- Accessibility: student list table is navigable by keyboard; screen reader announces column headers

### Definition of Done — Phase 2

- [ ] All academic structure tables created with full CRUD API endpoints
- [ ] OneRoster 1.2 adapter handles REST API and CSV batch import
- [ ] SIS integration configurable per institution via admin panel
- [ ] Scheduled sync runs reliably with health monitoring
- [ ] Student profiles imported from SIS with institution-specific extended attributes
- [ ] Advisor-facing student list with filtering, search, and pagination
- [ ] All SIS data access logged in FERPA audit trail
- [ ] Integration health dashboard shows sync status per institution
- [ ] 85%+ test coverage on OneRoster adapter and sync worker

---

## Phase 3 — Early Alert and Advising Workflow

**Duration estimate:** 5-6 weeks
**Dependencies:** Phase 2

### Task 3.1 — Alert Flag CRUD and Assignment

**What:** Implement the `alert_flags` table API with create, read, update, and resolve operations. Faculty can raise flags for students in their course sections. Flags are auto-assigned to the student's primary advisor. Support bulk flag creation for a class section.

**Design:**

```typescript
// apps/api/src/modules/alerts/routes.ts
fastify.post('/', {
  schema: {
    body: {
      type: 'object',
      required: ['studentProfileId', 'flagType', 'severity', 'title'],
      properties: {
        studentProfileId: { type: 'string', format: 'uuid' },
        flagType: { type: 'string', enum: ['academic', 'attendance', 'financial', 'wellbeing', 'engagement'] },
        severity: { type: 'string', enum: ['low', 'medium', 'high', 'critical'] },
        courseSectionId: { type: 'string', format: 'uuid' },
        title: { type: 'string', maxLength: 255 },
        description: { type: 'string' },
      },
    },
  },
  preHandler: [requireRole('faculty', 'advisor', 'staff', 'admin')],
  handler: async (request, reply) => {
    const { institutionId, sub: userId } = request.user;
    const data = request.body;

    // Auto-assign to primary advisor
    const primaryAssignment = await fastify.prisma.advisorAssignment.findFirst({
      where: { studentProfileId: data.studentProfileId, assignmentType: 'primary', endedAt: null },
    });

    const alert = await fastify.prisma.alertFlag.create({
      data: {
        institutionId,
        studentProfileId: data.studentProfileId,
        raisedBy: userId,
        flagType: data.flagType,
        severity: data.severity,
        courseSectionId: data.courseSectionId,
        title: data.title,
        description: data.description,
        assignedTo: primaryAssignment?.advisorId,
        alertMetadata: {},
      },
    });

    await fastify.audit.log({
      institutionId,
      userId,
      action: 'create',
      resourceType: 'alert_flag',
      resourceId: alert.id,
      studentProfileId: data.studentProfileId,
      ipAddress: request.ip,
    });

    // Notify assigned advisor (Phase 9 will add multi-channel; for now, in-app only)
    await fastify.prisma.notification.create({
      data: {
        userId: primaryAssignment?.advisorId,
        type: 'alert_assigned',
        title: `New ${data.severity} alert: ${data.title}`,
        resourceType: 'alert_flag',
        resourceId: alert.id,
      },
    });

    return reply.status(201).send(alert);
  },
});
```

**Testing:**
- Faculty can create alert for student in their course section
- Faculty cannot create alert for student not in their course section (403)
- Alert auto-assigned to student's primary advisor
- Bulk alert creation for a section: `POST /api/v1/alerts/bulk` with array of student IDs
- Alert status transitions: open > in_progress > resolved (valid); open > completed (invalid, 400)
- Resolving an alert requires `resolution_notes` (400 if missing)
- Alert list filterable by status, severity, flag_type, and assigned_to
- Audit log captures alert creation, status changes, and resolution
- FERPA: only users with care-network relationship to the student can view the alert

### Task 3.2 — Advisor Inbox and Caseload Dashboard

**What:** Build the advisor-facing inbox showing open alerts assigned to them, sorted by severity and age. Include a caseload dashboard with summary metrics: total advisees, high-risk count, open alerts, and average response time.

**Design:**

```typescript
// apps/web/src/app/(advisor)/inbox/page.tsx
export default async function AdvisorInboxPage() {
  const [alerts, caseloadStats] = await Promise.all([
    fetchAlerts({ assignedTo: 'me', status: ['open', 'in_progress'], sort: 'severity_desc' }),
    fetchCaseloadStats(),
  ]);

  return (
    <div className="grid grid-cols-12 gap-6">
      {/* Summary Cards */}
      <div className="col-span-12 grid grid-cols-4 gap-4">
        <StatCard label="Total Advisees" value={caseloadStats.totalAdvisees} />
        <StatCard label="High/Critical Risk" value={caseloadStats.highRiskCount}
                  variant={caseloadStats.highRiskCount > 10 ? 'warning' : 'default'} />
        <StatCard label="Open Alerts" value={caseloadStats.openAlerts} />
        <StatCard label="Avg Response Time" value={`${caseloadStats.avgResponseHours}h`} />
      </div>

      {/* Alert Inbox */}
      <div className="col-span-8">
        <AlertInboxList alerts={alerts.data} />
      </div>

      {/* Quick Actions Sidebar */}
      <div className="col-span-4">
        <UpcomingAppointments limit={5} />
        <RecentNotes limit={5} />
      </div>
    </div>
  );
}
```

**Testing:**
- Advisor sees only alerts assigned to them
- Alerts sorted by severity (critical first) then by creation date (oldest first)
- Clicking an alert navigates to the student profile with the alert detail expanded
- Caseload stats are accurate: total advisees matches `advisor_assignments` count
- Average response time calculated correctly from alert creation to first status change
- Dashboard loads in <500ms for advisors with 200 advisees and 50 open alerts
- Responsive layout: usable on tablet (768px) and desktop (1280px+)
- Screen reader: stat cards announce label and value; alert list items announce severity and title

### Task 3.3 — Advising Notes and Care Network Visibility

**What:** Implement advisor note-taking with FERPA-compliant access control. Notes are linked to student profiles and optionally to appointments. Implement the care-network model: faculty, advisors, tutors, and financial aid staff with active relationships to a student can view shared notes (excluding HIPAA-restricted notes visible only to counsellors).

**Design:**

```typescript
// apps/api/src/modules/advising/care-network.ts
export async function getStudentCareNetwork(studentProfileId: string, requestingUserId: string) {
  // Find all active relationships to this student
  const assignments = await prisma.advisorAssignment.findMany({
    where: { studentProfileId, endedAt: null },
    include: { advisor: { select: { id: true, firstName: true, lastName: true, userType: true, roles: true } } },
  });

  const enrollments = await prisma.enrollment.findMany({
    where: { studentProfileId, enrollmentStatus: 'enrolled' },
    include: {
      courseSection: {
        include: { instructor: { select: { id: true, firstName: true, lastName: true, userType: true } } },
      },
    },
  });

  // Build care team with role-specific visibility
  const careTeam = [
    ...assignments.map(a => ({
      userId: a.advisor.id,
      name: `${a.advisor.firstName} ${a.advisor.lastName}`,
      role: a.assignmentType,
      canViewNotes: !['counselling'].includes(a.assignmentType) || a.advisor.id === requestingUserId,
      canViewRestrictedNotes: a.assignmentType === 'counselling' && a.advisor.id === requestingUserId,
    })),
    ...enrollments.map(e => ({
      userId: e.courseSection.instructor?.id,
      name: `${e.courseSection.instructor?.firstName} ${e.courseSection.instructor?.lastName}`,
      role: 'faculty',
      canViewNotes: true,
      canViewRestrictedNotes: false,
    })),
  ];

  return careTeam;
}
```

**Testing:**
- Advisor can create a note linked to a student profile
- Note with `is_restricted: true` visible only to the creating counsellor and other counsellors
- Faculty member in student's care network can view non-restricted notes
- User not in student's care network cannot view notes (403)
- Care network endpoint returns all active relationships (advisors, faculty, tutors)
- Note creation audited in FERPA audit log
- Note search: full-text search over note content returns matching results

### Task 3.4 — Intervention Tracking and Success Plans

**What:** Build the intervention and success plan workflow. When an alert is raised, advisors can create an intervention (tutoring referral, financial aid review, schedule change). Success plans are personalised checklists assigned to at-risk students.

**Design:**

```typescript
// apps/api/src/modules/advising/interventions.ts
fastify.post('/', {
  schema: {
    body: {
      type: 'object',
      required: ['studentProfileId', 'interventionType'],
      properties: {
        alertFlagId: { type: 'string', format: 'uuid' },
        studentProfileId: { type: 'string', format: 'uuid' },
        interventionType: {
          type: 'string',
          enum: ['tutoring_referral', 'financial_aid_review', 'schedule_change',
                 'counselling_referral', 'peer_mentoring', 'other'],
        },
        description: { type: 'string' },
      },
    },
  },
  handler: async (request, reply) => {
    const intervention = await fastify.prisma.intervention.create({
      data: {
        ...request.body,
        initiatedBy: request.user.sub,
        status: 'planned',
        interventionData: {},
      },
    });
    return reply.status(201).send(intervention);
  },
});
```

**Testing:**
- Intervention created and linked to alert flag
- Intervention status progression: planned > in_progress > completed
- Outcome recorded on completion (successful, partially_successful, no_change, declined_by_student)
- Success plan created with JSONB task array
- Individual tasks within a plan can be marked complete/overdue
- Overdue tasks detected by background job and flagged in advisor dashboard
- Student profile timeline shows interventions and success plan milestones in chronological order

### Definition of Done — Phase 3

- [ ] Alert flags: create, assign, triage, resolve with full audit trail
- [ ] Advisor inbox with severity-sorted alert list and caseload statistics
- [ ] Care-network visibility model: shared notes with HIPAA-restricted filtering
- [ ] Intervention tracking with outcome recording
- [ ] Success plans with task management
- [ ] All endpoints documented in OpenAPI spec
- [ ] FERPA audit log covers all alert and note access
- [ ] E2E tests covering the full alert lifecycle (raise > assign > triage > intervene > resolve)

---

## Phase 4 — Student Portal and Appointment Scheduling

**Duration estimate:** 4-5 weeks
**Dependencies:** Phase 3

### Task 4.1 — Student-Facing Portal

**What:** Build the student-facing dashboard showing degree progress summary, upcoming appointments, to-do checklist (from success plans), and recent alert notifications. Students see their care team and how to contact each member.

**Design:**

```typescript
// apps/web/src/app/(student)/dashboard/page.tsx
export default async function StudentDashboardPage() {
  const [profile, appointments, tasks, careTeam] = await Promise.all([
    fetchMyProfile(),
    fetchMyAppointments({ upcoming: true, limit: 5 }),
    fetchMyTasks({ status: ['pending', 'overdue'] }),
    fetchMyCareTeam(),
  ]);

  return (
    <div className="space-y-8">
      {/* Degree Progress Summary */}
      <DegreeProgressCard
        creditsEarned={profile.totalCreditsEarned}
        creditsRequired={profile.programme?.totalCreditsRequired}
        gpa={profile.cumulativeGpa}
        standing={profile.academicStanding}
      />

      <div className="grid grid-cols-1 md:grid-cols-2 gap-6">
        {/* Upcoming Appointments */}
        <AppointmentsList appointments={appointments} />

        {/* To-Do Checklist */}
        <TaskChecklist tasks={tasks} onComplete={markTaskComplete} />
      </div>

      {/* My Care Team */}
      <CareTeamCard members={careTeam} />
    </div>
  );
}
```

**Testing:**
- Student sees only their own profile data (no access to other students)
- Degree progress card shows correct credit counts and GPA
- Upcoming appointments sorted by date (nearest first)
- To-do checklist shows tasks from active success plans
- Marking a task complete updates the success plan's JSONB task array
- Care team card shows all active advisor, faculty, and tutor relationships
- Contact buttons for each care team member (email link, appointment booking)
- Mobile responsive: usable on 375px screens
- WCAG 2.2 AA: all interactive elements have accessible labels; colour is not the only means of conveying information

### Task 4.2 — Appointment Scheduling System

**What:** Build appointment scheduling with advisor availability management, student self-service booking, and calendar sync. Support in-person, virtual (Zoom link), and phone appointment modes.

**Design:**

```typescript
// apps/api/src/modules/advising/appointments.ts

// Advisor sets weekly availability
fastify.put('/availability', {
  schema: {
    body: {
      type: 'object',
      properties: {
        weeklySlots: {
          type: 'array',
          items: {
            type: 'object',
            properties: {
              dayOfWeek: { type: 'integer', minimum: 0, maximum: 6 },
              startTime: { type: 'string', pattern: '^\\d{2}:\\d{2}$' },
              endTime: { type: 'string', pattern: '^\\d{2}:\\d{2}$' },
              mode: { type: 'string', enum: ['in_person', 'virtual', 'phone'] },
              location: { type: 'string' },
            },
          },
        },
        slotDurationMinutes: { type: 'integer', enum: [15, 30, 45, 60], default: 30 },
      },
    },
  },
  handler: async (request) => {
    // Store availability in advisor's profile_data JSONB
    await fastify.prisma.user.update({
      where: { id: request.user.sub },
      data: {
        profileData: {
          ...request.user.profileData,
          availability: request.body,
        },
      },
    });
  },
});

// Student books available slot
fastify.post('/', {
  schema: {
    body: {
      type: 'object',
      required: ['advisorId', 'scheduledStart', 'scheduledEnd', 'appointmentType'],
      properties: {
        advisorId: { type: 'string', format: 'uuid' },
        scheduledStart: { type: 'string', format: 'date-time' },
        scheduledEnd: { type: 'string', format: 'date-time' },
        appointmentType: { type: 'string', enum: ['advising', 'tutoring', 'financial_aid', 'career', 'counselling'] },
        mode: { type: 'string', enum: ['in_person', 'virtual', 'phone'] },
        reason: { type: 'string' },
      },
    },
  },
  handler: async (request, reply) => {
    // Check slot availability (no double-booking)
    const conflict = await fastify.prisma.appointment.findFirst({
      where: {
        advisorId: request.body.advisorId,
        status: { in: ['scheduled', 'confirmed'] },
        scheduledStart: { lt: new Date(request.body.scheduledEnd) },
        scheduledEnd: { gt: new Date(request.body.scheduledStart) },
      },
    });

    if (conflict) {
      return reply.status(409).send({
        type: 'https://api.studentsuccess.dev/errors/slot-unavailable',
        title: 'Slot Unavailable',
        status: 409,
        detail: 'This time slot is already booked',
      });
    }

    const appointment = await fastify.prisma.appointment.create({
      data: { ...request.body, institutionId: request.user.institutionId, studentProfileId: request.user.studentProfileId },
    });

    return reply.status(201).send(appointment);
  },
});
```

**Testing:**
- Advisor can set weekly availability with configurable slot durations
- Student sees available slots for their assigned advisor
- Booking an available slot creates an appointment record
- Double-booking the same slot returns 409 Conflict
- Appointment cancellation by student or advisor updates status and records reason
- No-show tracking: advisor marks appointment as `no_show`
- Appointment reminder: background job queues email/SMS reminder 24h before
- Calendar view (week/day) for advisors shows all appointments with student names
- Student appointment list shows upcoming and past appointments

### Task 4.3 — In-App Notification System

**What:** Build a notification system for in-app alerts (new alert assigned, appointment reminder, task overdue). Notifications appear in a bell-icon dropdown in the header. Real-time delivery via Server-Sent Events (SSE) or WebSocket.

**Design:**

```typescript
// apps/api/src/modules/notifications/sse.ts
fastify.get('/stream', {
  handler: async (request, reply) => {
    const userId = request.user.sub;

    reply.raw.writeHead(200, {
      'Content-Type': 'text/event-stream',
      'Cache-Control': 'no-cache',
      Connection: 'keep-alive',
    });

    // Subscribe to Redis pub/sub for this user
    const subscriber = redisClient.duplicate();
    await subscriber.subscribe(`notifications:${userId}`);

    subscriber.on('message', (channel, message) => {
      reply.raw.write(`data: ${message}\n\n`);
    });

    request.raw.on('close', () => {
      subscriber.unsubscribe();
      subscriber.disconnect();
    });
  },
});
```

**Testing:**
- New alert assignment triggers notification to assigned advisor
- Notification appears in real-time via SSE without page refresh
- Notification bell shows unread count badge
- Clicking notification navigates to the relevant resource (alert, appointment, etc.)
- Mark-as-read updates notification state
- Notification preferences: user can disable specific notification types
- SSE connection reconnects automatically on network interruption

### Definition of Done — Phase 4

- [ ] Student portal with degree progress, appointments, tasks, and care team
- [ ] Appointment scheduling with availability, booking, and conflict detection
- [ ] In-app notification system with real-time delivery via SSE
- [ ] Student can self-book appointments with assigned advisors
- [ ] Appointment reminders scheduled 24h in advance
- [ ] Mobile-responsive student portal meeting WCAG 2.2 AA
- [ ] All interactions audited for FERPA compliance

---

## Phase 5 — LMS Integration and Learning Event Ingestion

**Duration estimate:** 4-5 weeks
**Dependencies:** Phase 2

### Task 5.1 — Caliper 1.2 Event Ingestion Endpoint

**What:** Build the `packages/caliper-adapter` and a dedicated ingestion endpoint that accepts IMS Caliper 1.2 Sensor API payloads. Events are validated, mapped to platform entities (student, course section), and stored in the partitioned `learning_events` table with full JSONB payload preserved.

**Design:**

```typescript
// packages/caliper-adapter/src/parser.ts
import { CaliperEnvelope, CaliperEvent } from './types.js';

export function parseCaliperEnvelope(payload: unknown): CaliperEnvelope {
  // Validate top-level envelope structure
  const envelope = payload as CaliperEnvelope;
  if (!envelope.sensor || !envelope.sendTime || !Array.isArray(envelope.data)) {
    throw new CaliperValidationError('Invalid Caliper envelope structure');
  }
  return envelope;
}

export function extractEventFields(event: CaliperEvent) {
  return {
    eventType: event.type,          // e.g., 'NavigationEvent', 'AssessmentEvent'
    actorId: event.actor?.id,       // e.g., 'urn:canvas:user:12345'
    verb: event.action,             // e.g., 'Navigated', 'Submitted', 'Graded'
    objectType: event.object?.type, // e.g., 'Page', 'Assessment', 'AssignableDigitalResource'
    objectId: event.object?.id,
    resultScore: event.generated?.score?.scoreGiven,
    resultMaxScore: event.generated?.score?.maxScore,
    duration: event.duration,       // ISO 8601 duration
    occurredAt: event.eventTime,
  };
}

// apps/api/src/modules/integrations/caliper-ingestion.ts
fastify.post('/caliper', {
  schema: {
    body: { type: 'object' },  // Validated by adapter, not JSON Schema
  },
  handler: async (request, reply) => {
    const envelope = parseCaliperEnvelope(request.body);
    const events = envelope.data;
    let ingested = 0;

    for (const event of events) {
      const fields = extractEventFields(event);
      const studentProfile = await resolveStudentByLmsId(fields.actorId, request.user.institutionId);
      const courseSection = await resolveCourseSectionByLmsId(fields.objectId, request.user.institutionId);

      await prisma.$executeRaw`
        INSERT INTO learning_events (institution_id, student_profile_id, source_standard,
                                     event_type, course_section_id, occurred_at, event_data)
        VALUES (${request.user.institutionId}::uuid, ${studentProfile?.id}::uuid, 'caliper',
                ${fields.eventType}, ${courseSection?.id}::uuid,
                ${new Date(fields.occurredAt)}::timestamptz, ${JSON.stringify(event)}::jsonb)
      `;
      ingested++;
    }

    return reply.status(202).send({ ingested, total: events.length });
  },
});
```

**Testing:**
- Valid Caliper 1.2 envelope accepted with 202 status
- All six v1.2 metric profiles parsed: Session, Navigation, Assessment, Assignable, Forum, Media
- Invalid envelope structure returns 400 with descriptive error
- Events with unresolvable actor (unknown student) logged with null `student_profile_id` for later resolution
- Duplicate event detection: same event ID re-ingested does not create duplicate row
- High-volume ingestion: 1,000 events in a single envelope processed in <5 seconds
- Partitioned table: events land in correct monthly partition
- Event data JSONB preserves full original Caliper payload without modification

### Task 5.2 — xAPI Statement Ingestion Endpoint

**What:** Build the `packages/xapi-adapter` and ingestion endpoint accepting xAPI v1.0.3 statements. Support both single statement and batch statement submission. Store statements in the same `learning_events` table with `source_standard = 'xapi'`.

**Design:**

```typescript
// packages/xapi-adapter/src/parser.ts
export interface XapiStatement {
  id?: string;
  actor: { mbox?: string; account?: { name: string; homePage: string } };
  verb: { id: string; display?: Record<string, string> };
  object: { id: string; objectType?: string; definition?: { name?: Record<string, string> } };
  result?: { score?: { raw?: number; max?: number; scaled?: number }; success?: boolean; completion?: boolean; duration?: string };
  timestamp?: string;
}

export function mapXapiToLearningEvent(statement: XapiStatement, institutionId: string) {
  return {
    sourceStandard: 'xapi',
    eventType: extractVerbShortName(statement.verb.id),  // e.g., 'completed', 'answered', 'attended'
    actorId: statement.actor.mbox || statement.actor.account?.name,
    verb: statement.verb.id,
    objectId: statement.object.id,
    resultScore: statement.result?.score?.raw,
    occurredAt: statement.timestamp || new Date().toISOString(),
    rawPayload: statement,
  };
}
```

**Testing:**
- Single xAPI statement accepted with 200 status
- Batch of 100 xAPI statements accepted with 200 and count returned
- Actor resolution via mbox email or account name
- Verb IDs from ADL verb registry correctly mapped to short names
- Result scores (raw, max, scaled) correctly extracted
- Invalid statement structure returns 400
- Statements without timestamp default to server receive time

### Task 5.3 — LTI 1.3 Launch Provider

**What:** Build the `packages/lti-provider` implementing LTI 1.3 launch so that the student success dashboard can be embedded inside an LMS (Canvas, Blackboard, Moodle, Brightspace) as a tool. Support OIDC login initiation and signed JWT validation.

**Design:**

```typescript
// packages/lti-provider/src/launch.ts
import { importJWK, jwtVerify } from 'jose';

export async function validateLtiLaunch(idToken: string, platformConfig: {
  issuer: string;
  jwksUri: string;
  clientId: string;
}) {
  // Fetch platform JWKS
  const jwksResponse = await fetch(platformConfig.jwksUri);
  const jwks = await jwksResponse.json();

  // Verify the id_token JWT
  const { payload } = await jwtVerify(idToken, async (header) => {
    const key = jwks.keys.find((k: any) => k.kid === header.kid);
    return importJWK(key);
  }, {
    issuer: platformConfig.issuer,
    audience: platformConfig.clientId,
  });

  // Extract LTI claims
  return {
    deploymentId: payload['https://purl.imsglobal.org/spec/lti/claim/deployment_id'],
    targetLinkUri: payload['https://purl.imsglobal.org/spec/lti/claim/target_link_uri'],
    roles: payload['https://purl.imsglobal.org/spec/lti/claim/roles'],
    context: payload['https://purl.imsglobal.org/spec/lti/claim/context'],
    resourceLink: payload['https://purl.imsglobal.org/spec/lti/claim/resource_link'],
    launchPresentation: payload['https://purl.imsglobal.org/spec/lti/claim/launch_presentation'],
    lis: payload['https://purl.imsglobal.org/spec/lti/claim/lis'],
    sub: payload.sub,
    email: payload.email,
    name: payload.name,
  };
}
```

**Testing:**
- LTI 1.3 OIDC login initiation redirects to platform authorization endpoint
- Valid id_token with correct signature and claims returns launch context
- Invalid signature returns 401
- Expired id_token returns 401
- Role mapping: LTI `Instructor` role maps to platform `faculty`; `Learner` maps to `student`
- Launch context includes course and section identifiers for scoping the dashboard
- Embedded dashboard renders correctly inside Canvas and Blackboard iframes (tested with mock LMS)

### Definition of Done — Phase 5

- [ ] Caliper 1.2 ingestion endpoint accepting all v1.2 metric profiles
- [ ] xAPI statement ingestion endpoint (single and batch)
- [ ] Learning events stored in partitioned table with full payload
- [ ] LTI 1.3 launch provider enabling embedded dashboard in LMS
- [ ] Event deduplication prevents duplicate storage
- [ ] Performance: 1,000 events/second ingestion throughput
- [ ] Integration tests with mock Caliper sensor and xAPI LRS client

---

## Phase 6 — Risk Scoring and Predictive Analytics

**Duration estimate:** 6-8 weeks
**Dependencies:** Phase 3, Phase 5

### Task 6.1 — Risk Engine Package and Rule-Based Scoring

**What:** Build the `packages/risk-engine` with a configurable, rule-based risk scoring system as the initial implementation. Risk factors include GPA trajectory, LMS engagement metrics, attendance patterns, and alert history. Institutions can configure factor weights and thresholds.

**Design:**

```typescript
// packages/risk-engine/src/rule-engine.ts
export interface RiskFactor {
  name: string;
  weight: number;
  compute: (data: StudentRiskData) => number;  // Returns 0.0 - 1.0 (higher = more risk)
  explain: (data: StudentRiskData, score: number) => string;
}

export const DEFAULT_RISK_FACTORS: RiskFactor[] = [
  {
    name: 'gpa_trajectory',
    weight: 0.25,
    compute: (data) => {
      if (!data.previousTermGpa || !data.currentGpa) return 0.5;
      const delta = data.currentGpa - data.previousTermGpa;
      if (delta >= 0) return Math.max(0, 0.3 - delta * 0.3);  // Improving = low risk
      return Math.min(1, 0.5 + Math.abs(delta) * 0.5);         // Declining = high risk
    },
    explain: (data, score) => {
      const delta = data.currentGpa - (data.previousTermGpa || data.currentGpa);
      return delta >= 0
        ? `GPA improved by ${delta.toFixed(2)} from last term`
        : `GPA declined by ${Math.abs(delta).toFixed(2)} from last term`;
    },
  },
  {
    name: 'lms_engagement',
    weight: 0.20,
    compute: (data) => {
      if (!data.engagementMetrics) return 0.5;
      const { loginFrequency, assignmentSubmissionRate, contentViewRate } = data.engagementMetrics;
      // Composite engagement score
      const engagement = (loginFrequency * 0.3 + assignmentSubmissionRate * 0.5 + contentViewRate * 0.2);
      return 1 - engagement;  // Invert: low engagement = high risk
    },
    explain: (data, score) =>
      `LMS engagement is ${score > 0.6 ? 'significantly below' : score > 0.4 ? 'below' : 'at or above'} peer average`,
  },
  {
    name: 'alert_history',
    weight: 0.15,
    compute: (data) => {
      const activeAlerts = data.openAlertCount || 0;
      const recentAlerts = data.alertsLast30Days || 0;
      return Math.min(1, (activeAlerts * 0.3 + recentAlerts * 0.15));
    },
    explain: (data, score) =>
      `${data.openAlertCount} open alerts; ${data.alertsLast30Days} alerts in last 30 days`,
  },
  {
    name: 'financial_stress',
    weight: 0.15,
    compute: (data) => {
      let score = 0;
      if (data.financialHold) score += 0.5;
      if (data.pellEligible) score += 0.1;
      if (data.extendedAttributes?.employment_hours_per_week > 30) score += 0.2;
      return Math.min(1, score);
    },
    explain: (data, score) => {
      const factors: string[] = [];
      if (data.financialHold) factors.push('financial hold on account');
      if (data.extendedAttributes?.employment_hours_per_week > 30) factors.push('working 30+ hours/week');
      return factors.length > 0 ? factors.join('; ') : 'No financial stress indicators';
    },
  },
  {
    name: 'assignment_completion',
    weight: 0.15,
    compute: (data) => {
      if (!data.engagementMetrics?.assignmentSubmissionRate) return 0.5;
      return 1 - data.engagementMetrics.assignmentSubmissionRate;
    },
    explain: (data, score) =>
      `Assignment completion rate: ${((1 - score) * 100).toFixed(0)}%`,
  },
  {
    name: 'social_engagement',
    weight: 0.10,
    compute: (data) => {
      const activities = data.coCurricularActivities || 0;
      if (activities >= 2) return 0.1;
      if (activities === 1) return 0.3;
      return 0.6;
    },
    explain: (data, score) =>
      `Active in ${data.coCurricularActivities || 0} co-curricular activities`,
  },
];

export function computeRiskScore(
  data: StudentRiskData,
  factors: RiskFactor[] = DEFAULT_RISK_FACTORS,
): RiskScoreResult {
  const factorScores: Record<string, { score: number; weight: number; explanation: string }> = {};
  let weightedSum = 0;
  let totalWeight = 0;

  for (const factor of factors) {
    const score = factor.compute(data);
    const explanation = factor.explain(data, score);
    factorScores[factor.name] = { score, weight: factor.weight, explanation };
    weightedSum += score * factor.weight;
    totalWeight += factor.weight;
  }

  const overallScore = totalWeight > 0 ? weightedSum / totalWeight : 0.5;
  const riskLevel = overallScore >= 0.75 ? 'critical'
    : overallScore >= 0.55 ? 'high'
    : overallScore >= 0.35 ? 'moderate'
    : 'low';

  return { overallScore, riskLevel, factorScores };
}
```

**Testing:**
- Student with declining GPA, low LMS engagement, and financial hold scores `high` or `critical`
- Student with good GPA, active engagement, and no alerts scores `low`
- Each factor's `explain()` returns human-readable text
- Custom factor weights per institution override defaults
- Risk score computation is deterministic: same inputs produce same output
- Edge cases: missing data (no LMS events, no financial info) defaults to moderate risk, not zero
- Performance: batch computation for 10,000 students completes in <60 seconds

### Task 6.2 — Risk Score Batch Computation Worker

**What:** Build the BullMQ worker that computes risk scores for all students at an institution on a scheduled basis (nightly, or triggered by SIS sync). Each computation creates a new `risk_scores` row (append-only for historical tracking) and updates the denormalized `current_risk_level` on `student_profiles`.

**Design:**

```typescript
// apps/workers/src/risk-computation/worker.ts
const riskComputationWorker = new Worker('risk-computation', async (job: Job) => {
  const { institutionId, termId, modelVersion } = job.data;

  const students = await prisma.studentProfile.findMany({
    where: { institutionId, enrollmentStatus: { in: ['full_time', 'part_time'] } },
    include: { user: true },
  });

  let computed = 0;
  for (const student of students) {
    const riskData = await assembleStudentRiskData(student.id, termId);
    const result = computeRiskScore(riskData);

    await prisma.$transaction([
      prisma.riskScore.create({
        data: {
          studentProfileId: student.id,
          academicTermId: termId,
          modelVersion,
          overallScore: result.overallScore,
          riskLevel: result.riskLevel,
          factors: result.factorScores,
          recommendedInterventions: result.recommendations || [],
        },
      }),
      prisma.studentProfile.update({
        where: { id: student.id },
        data: {
          currentRiskLevel: result.riskLevel,
          currentRiskScore: result.overallScore,
          riskUpdatedAt: new Date(),
        },
      }),
    ]);

    // Auto-generate alert if risk level changed to high/critical
    if (['high', 'critical'].includes(result.riskLevel) && student.currentRiskLevel !== result.riskLevel) {
      await createSystemAlert(student, result);
    }

    computed++;
  }

  return { computed, institutionId };
}, { connection: redisConnection });
```

**Testing:**
- Nightly batch creates one `risk_scores` row per active student
- `student_profiles.current_risk_level` updated to match latest computation
- Risk level change from `moderate` to `high` triggers auto-generated alert
- Risk level change from `high` to `moderate` does not trigger alert (improvement)
- Historical risk scores preserved: querying by student + term returns all past scores
- Transaction: if profile update fails, risk score insert is rolled back
- Job retry: failed job retries 3 times with exponential backoff

### Task 6.3 — Risk Dashboard and Reporting

**What:** Build the advisor and institutional research dashboards for risk analytics. Advisors see risk distribution across their caseload. Institutional researchers see aggregate risk trends by programme, cohort, and demographic.

**Design:**

```typescript
// apps/web/src/app/(advisor)/risk/page.tsx
export default async function RiskDashboardPage() {
  const [distribution, trends, topRisk] = await Promise.all([
    fetchRiskDistribution(),     // { low: 120, moderate: 80, high: 35, critical: 8 }
    fetchRiskTrends({ weeks: 8 }), // weekly risk level counts
    fetchHighRiskStudents({ limit: 20 }),
  ]);

  return (
    <div className="space-y-6">
      <h1 className="text-2xl font-semibold">Risk Overview</h1>

      <div className="grid grid-cols-4 gap-4">
        <RiskDistributionChart data={distribution} />
      </div>

      <RiskTrendChart data={trends} />

      <HighRiskStudentList students={topRisk} />
    </div>
  );
}
```

**Testing:**
- Risk distribution chart shows correct counts per risk level for advisor's caseload
- Institutional researcher sees institution-wide risk distribution
- Trend chart shows risk level counts over 8-week period
- High-risk student list sorted by risk score (highest first)
- Clicking student navigates to profile with risk factor breakdown
- Risk factor breakdown shows each factor's score, weight, and explanation
- Export to CSV: risk report downloadable for institutional research teams

### Definition of Done — Phase 6

- [ ] Rule-based risk engine with 6 configurable factors
- [ ] Nightly batch risk computation for all active students
- [ ] Risk scores stored historically (append-only, per-term)
- [ ] Auto-generated alerts when risk level escalates
- [ ] Advisor risk dashboard with distribution and trends
- [ ] Risk factor explanations visible on student profile
- [ ] Institution-level risk reporting for institutional research
- [ ] Risk engine unit tests cover all factor computations and edge cases

---

## Phase 7 — Degree Audit and Planning

**Duration estimate:** 6-7 weeks
**Dependencies:** Phase 2

### Task 7.1 — Degree Requirements Engine

**What:** Build the degree audit engine that evaluates a student's completed and in-progress courses against their programme's JSONB requirement tree. Compute completion percentage, credits remaining, and unsatisfied requirements.

**Design:**

```typescript
// packages/risk-engine/src/degree-audit.ts  (or a dedicated packages/degree-audit)
export interface RequirementNode {
  name: string;
  type: 'required_courses' | 'elective_group' | 'credit_minimum' | 'gpa_minimum';
  minCredits?: number;
  minCourses?: number;
  minGpa?: number;
  courses?: Array<{ courseCode: string; required?: boolean; minGrade?: string }>;
  courseFilter?: Record<string, string>;
  children?: RequirementNode[];
}

export interface AuditResult {
  programmeId: string;
  programmeName: string;
  totalCreditsRequired: number;
  creditsCompleted: number;
  creditsInProgress: number;
  creditsRemaining: number;
  completionPercentage: number;
  requirements: RequirementAuditResult[];
}

export interface RequirementAuditResult {
  name: string;
  type: string;
  status: 'satisfied' | 'in_progress' | 'not_started';
  creditsEarned: number;
  creditsRequired: number;
  coursesCompleted: Array<{ code: string; grade: string; credits: number }>;
  coursesRemaining: Array<{ code: string; credits: number }>;
}

export function auditDegreeProgress(
  requirements: RequirementNode[],
  completedCourses: Array<{ courseCode: string; grade: string; credits: number }>,
  inProgressCourses: Array<{ courseCode: string; credits: number }>,
): RequirementAuditResult[] {
  return requirements.map(req => {
    if (req.type === 'required_courses') {
      const completed = (req.courses || [])
        .filter(c => c.required)
        .filter(c => completedCourses.some(
          cc => cc.courseCode === c.courseCode && (!c.minGrade || cc.grade >= c.minGrade)
        ));
      const remaining = (req.courses || [])
        .filter(c => c.required)
        .filter(c => !completedCourses.some(cc => cc.courseCode === c.courseCode));

      return {
        name: req.name,
        type: req.type,
        status: remaining.length === 0 ? 'satisfied' : 'in_progress',
        creditsEarned: completed.reduce((sum, c) =>
          sum + (completedCourses.find(cc => cc.courseCode === c.courseCode)?.credits || 0), 0),
        creditsRequired: req.minCredits || 0,
        coursesCompleted: completed.map(c => {
          const cc = completedCourses.find(x => x.courseCode === c.courseCode)!;
          return { code: c.courseCode, grade: cc.grade, credits: cc.credits };
        }),
        coursesRemaining: remaining.map(c => ({ code: c.courseCode, credits: 3 })),
      };
    }
    // Handle elective_group, credit_minimum, gpa_minimum...
    return { name: req.name, type: req.type, status: 'not_started', creditsEarned: 0, creditsRequired: req.minCredits || 0, coursesCompleted: [], coursesRemaining: [] };
  });
}
```

**Testing:**
- Student with all required courses completed: completion percentage = 100%, all requirements `satisfied`
- Student with partial completion: correct counts of completed, in-progress, and remaining courses
- Grade minimum enforcement: course completed with `D` when `C` required shows as not satisfied
- Elective group: 3 of 5 required electives completed shows 60% group completion
- GPA minimum requirement: GPA below threshold shows requirement as `not_satisfied`
- What-if: adding a hypothetical course recalculates completion percentage
- Performance: audit computation for 40-requirement programme completes in <100ms

### Task 7.2 — What-If Analysis API

**What:** Build the what-if analysis endpoint allowing students and advisors to explore the impact of changing major, adding a minor, or taking specific courses. Returns a new `AuditResult` without modifying any data.

**Design:**

```typescript
// apps/api/src/modules/degree-audit/what-if.ts
fastify.post('/what-if', {
  schema: {
    body: {
      type: 'object',
      properties: {
        studentProfileId: { type: 'string', format: 'uuid' },
        hypotheticalProgrammeId: { type: 'string', format: 'uuid' },
        addCourses: { type: 'array', items: { type: 'string' } },  // course codes to add
        removeCourses: { type: 'array', items: { type: 'string' } },  // course codes to remove
      },
    },
  },
  handler: async (request) => {
    const { studentProfileId, hypotheticalProgrammeId, addCourses, removeCourses } = request.body;

    // Get actual completed courses
    let courses = await getCompletedCourses(studentProfileId);

    // Apply hypothetical changes
    if (addCourses) {
      const additionalCourses = await prisma.course.findMany({
        where: { subjectCode: { in: addCourses.map(c => c.split('-')[0]) } },
      });
      courses = [...courses, ...additionalCourses.map(c => ({
        courseCode: `${c.subjectCode}-${c.courseNumber}`,
        grade: 'IP',  // In progress
        credits: Number(c.credits),
      }))];
    }
    if (removeCourses) {
      courses = courses.filter(c => !removeCourses.includes(c.courseCode));
    }

    // Get target programme requirements
    const programme = await prisma.degreeProgramme.findUnique({
      where: { id: hypotheticalProgrammeId || (await getCurrentProgramme(studentProfileId)).id },
    });

    const auditResult = auditDegreeProgress(programme.requirements, courses, []);
    return { ...auditResult, isHypothetical: true };
  },
});
```

**Testing:**
- Changing major recalculates completion against new programme requirements
- Adding a course improves completion percentage for the relevant requirement group
- Removing a course decreases completion percentage
- What-if does not modify actual enrollment or programme data (read-only)
- Multiple what-if scenarios can be compared side-by-side
- What-if for adding a minor shows additional requirements layered on top of major

### Task 7.3 — Student-Facing Degree Planning UI

**What:** Build the drag-and-drop multi-term course planning interface. Students see their completed courses, current enrollments, and can plan future terms by dragging available courses into term slots. The UI shows real-time requirement satisfaction as courses are added.

**Design:**

```typescript
// apps/web/src/components/degree-audit/course-planner.tsx
'use client';

import { DndContext, DragEndEvent } from '@dnd-kit/core';

export function CoursePlanner({
  auditResult,
  availableCourses,
  plannedTerms,
}: {
  auditResult: AuditResult;
  availableCourses: Course[];
  plannedTerms: PlannedTerm[];
}) {
  const [terms, setTerms] = useState(plannedTerms);

  function handleDragEnd(event: DragEndEvent) {
    const { active, over } = event;
    if (!over) return;

    const courseCode = active.id as string;
    const targetTermId = over.id as string;

    setTerms(prev => prev.map(term =>
      term.id === targetTermId
        ? { ...term, courses: [...term.courses, courseCode] }
        : term
    ));

    // Trigger real-time what-if recalculation
    recalculateAudit(terms);
  }

  return (
    <DndContext onDragEnd={handleDragEnd}>
      <div className="grid grid-cols-1 lg:grid-cols-3 gap-6">
        {/* Requirements sidebar */}
        <RequirementsSidebar requirements={auditResult.requirements} />

        {/* Term planning grid */}
        <div className="lg:col-span-2 space-y-4">
          {terms.map(term => (
            <TermSlot key={term.id} term={term} />
          ))}
        </div>
      </div>
    </DndContext>
  );
}
```

**Testing:**
- Drag-and-drop: course dragged to a term slot updates the plan
- Requirement sidebar updates in real-time as courses are added/removed
- Prerequisites enforced: cannot plan a course before its prerequisite is completed or planned
- Credit limit warning: term exceeding 18 credits shows warning
- Save plan: student can save their multi-term plan for advisor review
- Advisor can view and comment on student's saved plan
- Mobile: touch-based drag-and-drop works on tablets
- Keyboard: courses can be added to terms via keyboard selection (accessibility fallback)

### Definition of Done — Phase 7

- [ ] Degree audit engine evaluates all requirement types (required, elective, credit min, GPA min)
- [ ] What-if analysis API supports major change, minor addition, and course exploration
- [ ] Student-facing drag-and-drop course planner with real-time audit updates
- [ ] Prerequisite chain enforcement in course planning
- [ ] Advisor can view and comment on student degree plans
- [ ] Audit results cached in Redis for <200ms response time
- [ ] Unit tests cover all requirement evaluation logic including edge cases

---

## Phase 8 — AI Advising Chatbot

**Duration estimate:** 5-7 weeks
**Dependencies:** Phase 6, Phase 7

### Task 8.1 — Chatbot Backend with Claude API Integration

**What:** Build the conversational AI advising chatbot using the Anthropic Claude API. The chatbot answers student questions about degree requirements, course registration, campus resources, and financial aid. It uses structured tool use to query the degree audit engine and student profile data in real-time.

**Design:**

```typescript
// apps/api/src/modules/chatbot/engine.ts
import Anthropic from '@anthropic-ai/sdk';

const anthropic = new Anthropic();

const SYSTEM_PROMPT = `You are an academic advising assistant for a higher education institution.
You help students with questions about degree requirements, course registration, campus resources,
and general academic guidance. You have access to tools that let you look up the student's
degree progress, course catalog, and appointment availability.

Rules:
- Always ground answers in actual data from the tools. Never guess at degree requirements.
- If you detect signs of emotional distress, recommend the student contact the counselling center
  and offer to connect them with their advisor.
- Never share one student's information with another student.
- If you are unsure about an answer, say so and recommend the student meet with their advisor.
- Keep responses concise and actionable.`;

const TOOLS = [
  {
    name: 'get_degree_progress',
    description: 'Get the student\'s current degree progress including completed courses, remaining requirements, and completion percentage',
    input_schema: {
      type: 'object',
      properties: { studentProfileId: { type: 'string' } },
      required: ['studentProfileId'],
    },
  },
  {
    name: 'search_courses',
    description: 'Search the course catalog by subject, keyword, or requirement group',
    input_schema: {
      type: 'object',
      properties: {
        query: { type: 'string' },
        subjectCode: { type: 'string' },
        genEdCategory: { type: 'string' },
      },
    },
  },
  {
    name: 'get_available_appointments',
    description: 'Get available appointment slots with the student\'s advisor',
    input_schema: {
      type: 'object',
      properties: { advisorId: { type: 'string' }, weekOffset: { type: 'integer' } },
    },
  },
  {
    name: 'escalate_to_advisor',
    description: 'Escalate the conversation to a human advisor when the question is too complex or the student needs personal attention',
    input_schema: {
      type: 'object',
      properties: { reason: { type: 'string' }, urgency: { type: 'string', enum: ['normal', 'urgent'] } },
      required: ['reason'],
    },
  },
];

export async function processMessage(
  conversationId: string,
  studentProfileId: string,
  userMessage: string,
  conversationHistory: Array<{ role: string; content: string }>,
) {
  // Add the new message
  const messages = [
    ...conversationHistory,
    { role: 'user' as const, content: userMessage },
  ];

  const response = await anthropic.messages.create({
    model: 'claude-sonnet-4-20250514',
    max_tokens: 1024,
    system: SYSTEM_PROMPT,
    tools: TOOLS,
    messages,
  });

  // Handle tool use
  if (response.stop_reason === 'tool_use') {
    const toolUse = response.content.find(block => block.type === 'tool_use');
    const toolResult = await executeToolCall(toolUse.name, toolUse.input, studentProfileId);

    // Continue conversation with tool result
    const followUp = await anthropic.messages.create({
      model: 'claude-sonnet-4-20250514',
      max_tokens: 1024,
      system: SYSTEM_PROMPT,
      tools: TOOLS,
      messages: [
        ...messages,
        { role: 'assistant', content: response.content },
        { role: 'user', content: [{ type: 'tool_result', tool_use_id: toolUse.id, content: JSON.stringify(toolResult) }] },
      ],
    });

    return followUp;
  }

  return response;
}
```

**Testing:**
- "What courses do I need to graduate?" triggers `get_degree_progress` tool and returns a grounded answer
- "What humanities courses are available?" triggers `search_courses` with gen-ed filter
- "Can I meet with my advisor this week?" triggers `get_available_appointments` and lists slots
- Sentiment detection: message containing "I can't handle this anymore" triggers escalation to advisor
- Escalation creates an alert flag with `flag_type: 'wellbeing'` and notifies the advisor
- Conversation history maintained: follow-up questions reference previous context
- Confidence threshold: when Claude indicates uncertainty, response includes "I recommend meeting with your advisor"
- FERPA: chatbot only accesses the requesting student's data; cannot be tricked into revealing other students' info

### Task 8.2 — Chatbot UI and Conversation Management

**What:** Build the student-facing chat interface with real-time message streaming. Implement conversation logging, sentiment tagging, and the advisor-facing escalation queue.

**Design:**

```typescript
// apps/web/src/components/chatbot/chat-interface.tsx
'use client';

import { useChat } from 'ai/react';

export function ChatInterface({ studentProfileId }: { studentProfileId: string }) {
  const { messages, input, handleInputChange, handleSubmit, isLoading } = useChat({
    api: '/api/v1/chatbot/message',
    body: { studentProfileId },
  });

  return (
    <div className="flex flex-col h-[600px] border rounded-lg">
      {/* Message list */}
      <div className="flex-1 overflow-y-auto p-4 space-y-4" role="log" aria-label="Chat messages">
        {messages.map((msg) => (
          <ChatBubble key={msg.id} role={msg.role} content={msg.content} />
        ))}
        {isLoading && <TypingIndicator />}
      </div>

      {/* Input area */}
      <form onSubmit={handleSubmit} className="border-t p-4">
        <div className="flex gap-2">
          <input
            value={input}
            onChange={handleInputChange}
            placeholder="Ask about your degree, courses, or campus resources..."
            className="flex-1 rounded-md border px-3 py-2"
            aria-label="Type your message"
          />
          <button type="submit" disabled={isLoading} className="rounded-md bg-primary px-4 py-2 text-white">
            Send
          </button>
        </div>
      </form>
    </div>
  );
}
```

**Testing:**
- Messages stream token-by-token (SSE streaming from Claude API)
- Conversation persisted to `chatbot_conversations` and `chatbot_messages` tables
- Sentiment score computed and stored for each conversation
- Escalation: advisor sees escalated conversation in their queue with full history
- Advisor can take over the conversation and respond directly
- Chat available 24/7; response time <3 seconds for non-tool-use messages
- Mobile: chat interface usable on 375px screens
- Accessibility: chat log announced by screen reader; keyboard-navigable

### Task 8.3 — Institution Knowledge Base Management

**What:** Build the admin interface for managing the chatbot's institution-specific knowledge base (FAQ documents, policy PDFs, course catalog descriptions). Knowledge base content is used for retrieval-augmented generation (RAG) grounding.

**Design:**

```typescript
// apps/api/src/modules/chatbot/knowledge-base.ts
export async function indexKnowledgeBaseDocument(institutionId: string, document: {
  title: string;
  content: string;
  category: string;
  sourceUrl?: string;
}) {
  // Chunk the document into ~500-token segments
  const chunks = chunkDocument(document.content, 500);

  // Generate embeddings and store
  for (const chunk of chunks) {
    const embedding = await generateEmbedding(chunk.text);
    await prisma.knowledgeBaseChunk.create({
      data: {
        institutionId,
        documentTitle: document.title,
        category: document.category,
        content: chunk.text,
        embedding,  // pgvector column
        metadata: { sourceUrl: document.sourceUrl, chunkIndex: chunk.index },
      },
    });
  }
}

export async function searchKnowledgeBase(institutionId: string, query: string, limit: number = 5) {
  const queryEmbedding = await generateEmbedding(query);
  // Cosine similarity search via pgvector
  return prisma.$queryRaw`
    SELECT content, document_title, category,
           1 - (embedding <=> ${queryEmbedding}::vector) AS similarity
    FROM knowledge_base_chunks
    WHERE institution_id = ${institutionId}::uuid
    ORDER BY embedding <=> ${queryEmbedding}::vector
    LIMIT ${limit}
  `;
}
```

**Testing:**
- Admin can upload PDF and text documents to the knowledge base
- Documents chunked and indexed with embeddings
- Search returns semantically relevant chunks for a given query
- Chatbot responses grounded in knowledge base content when relevant
- Stale document detection: admin warned when documents older than 6 months
- Knowledge base search results include source document title for citation

### Definition of Done — Phase 8

- [ ] AI chatbot answers student questions grounded in real data via tool use
- [ ] Streaming message delivery with <3 second time-to-first-token
- [ ] Sentiment analysis detects distress and escalates to counsellor
- [ ] Advisor escalation queue with full conversation history
- [ ] Institution knowledge base with RAG-based grounding
- [ ] FERPA: chatbot accesses only requesting student's data
- [ ] GDPR: human review gate for GDPR-flagged institutions before AI-triggered actions
- [ ] Conversation logging with sentiment scores for advisor review

---

## Phase 9 — Outreach Campaigns and Multi-Channel Messaging

**Duration estimate:** 4-5 weeks
**Dependencies:** Phase 4

### Task 9.1 — Campaign Builder and Targeting

**What:** Build the campaign management system for advisors and student affairs staff to create targeted outreach campaigns. Campaigns target student cohorts based on risk level, academic standing, enrollment status, and custom JSONB attributes.

**Design:**

```typescript
// apps/api/src/modules/campaigns/targeting.ts
export async function resolveTargetStudents(
  institutionId: string,
  criteria: {
    riskLevel?: string[];
    academicStanding?: string[];
    enrollmentStatus?: string[];
    programmes?: string[];
    customFilters?: Record<string, unknown>;
  },
): Promise<string[]> {
  const where: any = { institutionId };

  if (criteria.riskLevel) where.currentRiskLevel = { in: criteria.riskLevel };
  if (criteria.academicStanding) where.academicStanding = { in: criteria.academicStanding };
  if (criteria.enrollmentStatus) where.enrollmentStatus = { in: criteria.enrollmentStatus };
  if (criteria.customFilters) {
    where.extendedAttributes = { path: [], ...criteria.customFilters };
  }

  const students = await prisma.studentProfile.findMany({
    where,
    select: { id: true },
  });

  return students.map(s => s.id);
}
```

**Testing:**
- Campaign created with target criteria; preview shows count of matching students
- Campaign scheduled for future delivery
- Targeting by risk level returns correct student subset
- Combined filters (risk + standing + programme) intersect correctly
- Custom JSONB attribute filter (e.g., `employment_hours_per_week > 30`) works
- Campaign cannot target students who have opted out of communications

### Task 9.2 — Multi-Channel Message Delivery

**What:** Implement email (via Resend), SMS (via Twilio), and in-app message delivery. Track delivery status (sent, delivered, opened, responded) via webhooks.

**Design:**

```typescript
// apps/workers/src/campaign-delivery/worker.ts
const campaignDeliveryWorker = new Worker('campaign-delivery', async (job: Job) => {
  const { campaignId, studentProfileId, channel, content } = job.data;

  const student = await prisma.studentProfile.findUnique({
    where: { id: studentProfileId },
    include: { user: true },
  });

  let deliveryResult: DeliveryResult;

  switch (channel) {
    case 'email':
      deliveryResult = await resend.emails.send({
        from: 'advising@studentsuccess.institution.edu',
        to: student.user.email,
        subject: content.subject,
        html: content.html,
        tags: [{ name: 'campaign_id', value: campaignId }],
      });
      break;
    case 'sms':
      deliveryResult = await twilio.messages.create({
        to: student.user.phone,
        from: process.env.TWILIO_PHONE_NUMBER,
        body: content.smsBody,
        statusCallback: `${process.env.API_URL}/api/v1/webhooks/twilio/status`,
      });
      break;
    case 'in_app':
      await prisma.notification.create({
        data: {
          userId: student.userId,
          type: 'campaign_message',
          title: content.subject,
          body: content.text,
          resourceType: 'campaign',
          resourceId: campaignId,
        },
      });
      deliveryResult = { status: 'delivered' };
      break;
  }

  await prisma.campaignMessage.update({
    where: { id: job.data.messageId },
    data: { status: 'sent', sentAt: new Date() },
  });
}, { connection: redisConnection });
```

**Testing:**
- Email delivery via Resend: message sent, delivery confirmed via webhook
- SMS delivery via Twilio: message sent, delivery status tracked
- In-app notification created and visible in student portal
- Bounce handling: bounced email updates campaign message status to `bounced`
- Response tracking: student reply to campaign email recorded
- Rate limiting: messages sent at configurable rate to avoid provider throttling
- Unsubscribe: student opt-out honored for all future campaigns

### Task 9.3 — Campaign Analytics Dashboard

**What:** Build the campaign results dashboard showing delivery metrics (sent, delivered, opened, responded) and conversion analysis (did outreach correlate with student behavior change?).

**Testing:**
- Campaign dashboard shows funnel: sent > delivered > opened > responded
- Open rate and response rate calculated correctly
- Drill-down: clicking a metric shows the list of students in that stage
- Export campaign results to CSV

### Definition of Done — Phase 9

- [ ] Campaign builder with cohort targeting by risk, standing, and custom attributes
- [ ] Email, SMS, and in-app message delivery with status tracking
- [ ] Delivery webhook processing for open and response tracking
- [ ] Campaign analytics dashboard with funnel metrics
- [ ] Student opt-out honored across all channels
- [ ] Rate limiting prevents provider throttling

---

## Phase 10 — Transfer Credit Intelligence

**Duration estimate:** 4-5 weeks
**Dependencies:** Phase 7

### Task 10.1 — Transfer Credit Evaluation Workflow

**What:** Build the transfer credit evaluation workflow where incoming transfer students upload transcripts, the system suggests course equivalencies using AI, and an advisor reviews and approves/denies each equivalency.

**Design:**

```typescript
// apps/api/src/modules/transfers/ai-matching.ts
export async function suggestEquivalencies(
  institutionId: string,
  sourceInstitution: string,
  sourceCourses: Array<{ code: string; title: string; description?: string; credits: number }>,
): Promise<Array<{
  sourceCourse: { code: string; title: string };
  suggestedEquivalent: { courseId: string; courseCode: string; title: string } | null;
  confidence: number;
  reasoning: string;
}>> {
  // Fetch institution course catalog
  const catalog = await prisma.course.findMany({
    where: { institutionId, isActive: true },
    select: { id: true, subjectCode: true, courseNumber: true, title: true, credits: true, courseMetadata: true },
  });

  const suggestions = [];
  for (const sourceCourse of sourceCourses) {
    // Use Claude to match based on title, description, and credit alignment
    const response = await anthropic.messages.create({
      model: 'claude-haiku-4-20250514',
      max_tokens: 512,
      system: 'You are a transfer credit evaluation assistant. Match source courses to equivalent courses in the target catalog based on title similarity, credit alignment, and subject area.',
      messages: [{
        role: 'user',
        content: `Source course: ${sourceCourse.code} - ${sourceCourse.title} (${sourceCourse.credits} credits)
${sourceCourse.description ? `Description: ${sourceCourse.description}` : ''}

Target catalog courses (JSON):
${JSON.stringify(catalog.slice(0, 100).map(c => ({
  id: c.id,
  code: `${c.subjectCode}-${c.courseNumber}`,
  title: c.title,
  credits: Number(c.credits),
})))}

Return JSON: { "matchId": "course-uuid or null", "matchCode": "CODE or null", "confidence": 0.0-1.0, "reasoning": "..." }`,
      }],
    });

    const match = JSON.parse(response.content[0].text);
    suggestions.push({
      sourceCourse: { code: sourceCourse.code, title: sourceCourse.title },
      suggestedEquivalent: match.matchId ? { courseId: match.matchId, courseCode: match.matchCode, title: catalog.find(c => c.id === match.matchId)?.title } : null,
      confidence: match.confidence,
      reasoning: match.reasoning,
    });
  }

  return suggestions;
}
```

**Testing:**
- AI suggests correct equivalency for common course (e.g., "ENG 101" matches "ENGL-101")
- Confidence score reflects match quality: exact title match > 0.9, partial match 0.5-0.8
- No match found returns null with reasoning
- Advisor review UI shows source course, suggested equivalent, confidence, and reasoning
- Advisor can approve, deny, or override the AI suggestion
- GDPR: human review required before AI suggestion is finalized (Art. 22 compliance)
- Approved transfer credits reflected in degree audit immediately

### Task 10.2 — Transcript Upload and OCR

**What:** Build transcript document upload with OCR-based extraction of course information from PDF transcripts.

**Testing:**
- PDF transcript uploaded to S3-compatible storage
- OCR extracts course codes, titles, grades, and credits from standard transcript formats
- Extracted data presented for student/advisor review before AI matching
- Unsupported transcript format flagged for manual entry

### Definition of Done — Phase 10

- [ ] AI-powered transfer credit equivalency suggestions with confidence scores
- [ ] Advisor review and approval workflow
- [ ] Transcript PDF upload with OCR extraction
- [ ] Approved credits flow into degree audit automatically
- [ ] GDPR Art. 22 human review gate enforced

---

## Phase 11 — Causal Intervention Recommendations and Fairness Auditing

**Duration estimate:** 6-8 weeks
**Dependencies:** Phase 6

### Task 11.1 — Intervention Recommendation Engine

**What:** Build the causal intervention recommendation engine that goes beyond risk scores to suggest which specific intervention type (tutoring, financial aid, schedule change) has the highest probability of reversing a given student's trajectory. Uses historical intervention outcome data from similar students.

**Design:**

```typescript
// packages/risk-engine/src/intervention-recommender.ts
export interface InterventionRecommendation {
  interventionType: string;
  confidence: number;
  peerOutcomeData: {
    similarStudents: number;
    successRate: number;
    avgGpaImprovement: number;
  };
  reasoning: string;
}

export async function recommendInterventions(
  studentProfileId: string,
  riskFactors: Record<string, { score: number; weight: number }>,
): Promise<InterventionRecommendation[]> {
  const student = await prisma.studentProfile.findUnique({ where: { id: studentProfileId } });

  // Find similar students based on risk profile
  const similarStudents = await prisma.$queryRaw`
    SELECT sp.id, sp.first_generation, sp.pell_eligible, sp.cumulative_gpa,
           i.intervention_type, i.outcome,
           CASE WHEN i.outcome = 'successful' THEN 1 ELSE 0 END AS success
    FROM student_profiles sp
    JOIN interventions i ON i.student_profile_id = sp.id
    WHERE sp.institution_id = ${student.institutionId}::uuid
      AND sp.id != ${studentProfileId}::uuid
      AND ABS(sp.cumulative_gpa - ${student.cumulativeGpa}) < 0.5
      AND sp.first_generation = ${student.firstGeneration}
      AND i.status = 'completed'
  `;

  // Group by intervention type and compute success rates
  const interventionStats = groupBy(similarStudents, 'intervention_type');
  const recommendations: InterventionRecommendation[] = [];

  for (const [type, students] of Object.entries(interventionStats)) {
    const successRate = students.filter(s => s.success).length / students.length;
    const avgGpaImprovement = students
      .filter(s => s.success)
      .reduce((sum, s) => sum + (s.gpa_after - s.gpa_before), 0) / Math.max(1, students.filter(s => s.success).length);

    recommendations.push({
      interventionType: type,
      confidence: successRate,
      peerOutcomeData: {
        similarStudents: students.length,
        successRate,
        avgGpaImprovement,
      },
      reasoning: `For ${students.length} similar students, ${type.replace('_', ' ')} had a ${(successRate * 100).toFixed(0)}% success rate`,
    });
  }

  return recommendations.sort((a, b) => b.confidence - a.confidence);
}
```

**Testing:**
- Recommendation engine suggests interventions sorted by success rate
- Peer outcome data is accurate: student count, success rate, GPA improvement
- Reasoning is human-readable and actionable
- Recommendation based on minimum 10 similar students (below threshold = low confidence warning)
- No recommendations returned if no similar students with completed interventions exist
- GDPR: recommendations include `requires_human_review: true` flag

### Task 11.2 — Algorithmic Fairness Auditing

**What:** Build the fairness audit module that analyzes risk model outputs for demographic bias. Compare average risk scores and alert rates across demographic groups (race/ethnicity, gender, first-generation, Pell eligibility). Generate fairness reports with statistical significance tests.

**Design:**

```typescript
// packages/risk-engine/src/fairness-audit.ts
export interface FairnessReport {
  modelVersion: string;
  auditDate: string;
  termId: string;
  overallMetrics: { totalStudents: number; avgRiskScore: number };
  demographicBreakdown: Array<{
    dimension: string;        // 'ethnicity', 'gender', 'first_generation'
    group: string;            // 'Hispanic', 'Female', 'true'
    studentCount: number;
    avgRiskScore: number;
    highRiskRate: number;     // % flagged as high/critical
    alertRate: number;        // alerts per student
    interventionRate: number; // interventions per student
    disparateImpactRatio: number;  // group rate / reference group rate (< 0.8 = concern)
  }>;
  recommendations: string[];
}

export async function runFairnessAudit(
  institutionId: string,
  termId: string,
  modelVersion: string,
): Promise<FairnessReport> {
  const data = await prisma.$queryRaw`
    SELECT sp.ethnicity, sp.gender, sp.first_generation, sp.pell_eligible,
           rs.overall_score, rs.risk_level,
           (SELECT COUNT(*) FROM alert_flags af WHERE af.student_profile_id = sp.id
            AND af.created_at >= t.start_date AND af.created_at <= t.end_date) AS alert_count,
           (SELECT COUNT(*) FROM interventions i WHERE i.student_profile_id = sp.id
            AND i.created_at >= t.start_date AND i.created_at <= t.end_date) AS intervention_count
    FROM student_profiles sp
    JOIN risk_scores rs ON rs.student_profile_id = sp.id AND rs.academic_term_id = ${termId}::uuid
    JOIN academic_terms t ON t.id = ${termId}::uuid
    WHERE sp.institution_id = ${institutionId}::uuid
      AND rs.model_version = ${modelVersion}
  `;

  // Compute disparate impact ratios per demographic group
  // Reference group = largest group in each dimension
  // ... statistical analysis ...

  return report;
}
```

**Testing:**
- Fairness report generated for a model version and term
- Disparate impact ratio calculated: < 0.8 flagged as potential concern
- Report includes actionable recommendations when bias detected
- Statistical significance: small groups (n < 30) flagged as insufficient sample
- Historical comparison: audit reports can be compared across terms to track bias trends
- Report exportable as PDF for institutional compliance documentation

### Definition of Done — Phase 11

- [ ] Causal intervention recommendations based on peer outcomes
- [ ] Recommendations sorted by success rate with human-readable reasoning
- [ ] Fairness audit module comparing risk scores across demographic groups
- [ ] Disparate impact ratio calculation with threshold warnings
- [ ] Fairness reports exportable for institutional compliance
- [ ] GDPR human-review gate on all AI-generated recommendations

---

## Phase 12 — MCP Server, Co-Curricular Data, and Wellbeing Signals

**Duration estimate:** 5-7 weeks
**Dependencies:** Phase 2

### Task 12.1 — Model Context Protocol (MCP) Server

**What:** Expose platform data (student profiles, degree audit, advising notes, alert flags) as MCP tools callable by external AI agents. This positions the platform as the natural integration point for any institution deploying AI advising agents, leveraging the early-mover opportunity identified in the standards research (no incumbent has published an MCP server as of May 2026).

**Design:**

```typescript
// apps/api/src/modules/mcp/server.ts
import { McpServer } from '@modelcontextprotocol/sdk/server/mcp.js';
import { StdioServerTransport } from '@modelcontextprotocol/sdk/server/stdio.js';

const server = new McpServer({
  name: 'student-success-platform',
  version: '1.0.0',
});

server.tool('get_student_profile', {
  description: 'Retrieve a student profile including academic standing, GPA, risk level, and enrollment status',
  inputSchema: {
    type: 'object',
    properties: {
      studentId: { type: 'string', description: 'Student ID number or email' },
    },
    required: ['studentId'],
  },
}, async ({ studentId }) => {
  const profile = await findStudentByIdOrEmail(studentId);
  if (!profile) return { content: [{ type: 'text', text: 'Student not found' }] };
  return {
    content: [{
      type: 'text',
      text: JSON.stringify({
        name: `${profile.firstName} ${profile.lastName}`,
        gpa: profile.cumulativeGpa,
        standing: profile.academicStanding,
        riskLevel: profile.currentRiskLevel,
        enrollmentStatus: profile.enrollmentStatus,
        creditsEarned: profile.totalCreditsEarned,
      }),
    }],
  };
});

server.tool('audit_degree_progress', {
  description: 'Run a degree audit for a student showing completed requirements, remaining courses, and completion percentage',
  inputSchema: {
    type: 'object',
    properties: { studentId: { type: 'string' } },
    required: ['studentId'],
  },
}, async ({ studentId }) => {
  const auditResult = await computeDegreeAudit(studentId);
  return { content: [{ type: 'text', text: JSON.stringify(auditResult) }] };
});

server.tool('get_open_alerts', {
  description: 'Get all open alert flags for a student',
  inputSchema: {
    type: 'object',
    properties: { studentId: { type: 'string' } },
    required: ['studentId'],
  },
}, async ({ studentId }) => {
  const alerts = await getStudentAlerts(studentId, { status: 'open' });
  return { content: [{ type: 'text', text: JSON.stringify(alerts) }] };
});

server.tool('schedule_appointment', {
  description: 'Schedule an advising appointment for a student',
  inputSchema: {
    type: 'object',
    properties: {
      studentId: { type: 'string' },
      advisorId: { type: 'string' },
      dateTime: { type: 'string', format: 'date-time' },
      mode: { type: 'string', enum: ['in_person', 'virtual', 'phone'] },
    },
    required: ['studentId', 'advisorId', 'dateTime'],
  },
}, async (params) => {
  const appointment = await createAppointment(params);
  return { content: [{ type: 'text', text: `Appointment scheduled: ${appointment.id}` }] };
});
```

**Testing:**
- MCP server responds to tool discovery request listing all available tools
- `get_student_profile` returns correct profile data for valid student ID
- `audit_degree_progress` returns computed degree audit results
- `get_open_alerts` returns only open alerts for the specified student
- `schedule_appointment` creates an appointment and returns confirmation
- FERPA: MCP server enforces authentication; unauthorized requests rejected
- MCP server logs all tool invocations to FERPA audit trail

### Task 12.2 — Co-Curricular Engagement Data Integration

**What:** Integrate co-curricular data sources (club memberships, library visits, tutoring attendance, campus event participation) as additional risk signal inputs. These data points are highly predictive of retention but rarely integrated by incumbent platforms.

**Testing:**
- Co-curricular data ingested from campus systems (library, student orgs, recreation)
- Engagement metrics feed into risk scoring as additional factors
- Student profile shows co-curricular engagement summary
- Data sources configurable per institution (not all have same systems)

### Task 12.3 — Wellbeing Signal Detection

**What:** Implement NLP-based wellbeing signal detection from student-written communications (chatbot conversations, help-desk tickets). Detect early indicators of mental health stress and route sensitive cases to counselling staff with privacy safeguards.

**Design:**

```typescript
// apps/workers/src/wellbeing-detection/classifier.ts
export async function classifyWellbeingSignal(text: string): Promise<{
  hasSignal: boolean;
  severity: 'none' | 'mild' | 'moderate' | 'severe';
  category?: 'stress' | 'anxiety' | 'depression' | 'crisis';
  confidence: number;
}> {
  const response = await anthropic.messages.create({
    model: 'claude-haiku-4-20250514',
    max_tokens: 256,
    system: `You are a wellbeing signal classifier for student communications.
Classify the text for indicators of mental health distress.
Return JSON: { "hasSignal": bool, "severity": "none|mild|moderate|severe", "category": "stress|anxiety|depression|crisis|null", "confidence": 0.0-1.0 }
Be conservative: only flag when indicators are clear. False positives cause unnecessary alarm.
Never diagnose; only identify signals for referral to professional counsellors.`,
    messages: [{ role: 'user', content: text }],
  });

  return JSON.parse(response.content[0].text);
}
```

**Testing:**
- Clear distress language classified with high confidence
- Neutral academic questions classified as `none`
- False positive rate < 5% on test dataset of 500 messages
- Severe signals create immediate alert routed to counselling staff
- HIPAA: wellbeing alerts marked `is_restricted` and visible only to counsellors
- Student consent required before wellbeing monitoring is activated (opt-in per institution)
- Classification model does not store or retain the message content beyond the classification result

### Definition of Done — Phase 12

- [ ] MCP server exposing 5+ tools for external AI agent integration
- [ ] Co-curricular data ingestion from configurable campus systems
- [ ] Wellbeing signal detection with conservative classification thresholds
- [ ] HIPAA-compliant routing of wellbeing alerts to counselling staff
- [ ] Consent-gated wellbeing monitoring (opt-in)
- [ ] MCP server documented in OpenAPI and MCP specification format

---

## Cross-Phase Definitions of Done

These criteria apply to every phase and must be satisfied before a phase is considered complete.

### Code Quality
- [ ] All code passes ESLint with zero warnings
- [ ] TypeScript strict mode: no `any` types in production code
- [ ] Unit test coverage >= 80% for business logic modules
- [ ] Integration tests cover all API endpoints (happy path + error cases)
- [ ] No known security vulnerabilities in dependency audit (`pnpm audit`)

### Compliance
- [ ] FERPA audit log captures every student record access with user, timestamp, IP, resource, and action
- [ ] HIPAA-restricted data (counselling notes, wellbeing signals) filtered from standard queries
- [ ] GDPR human-review gate activated for GDPR-flagged institutions on all AI-triggered actions
- [ ] WCAG 2.2 AA compliance verified via automated axe-core tests and manual checklist
- [ ] Data retention policies configurable per institution per compliance jurisdiction

### Documentation
- [ ] OpenAPI 3.1 spec updated for all new endpoints
- [ ] Architecture decision records (ADRs) written for significant technical choices
- [ ] Integration guide updated for any new SIS/LMS/external system connectors
- [ ] Inline code documentation for complex business logic (degree audit, risk scoring)

### Performance
- [ ] API response time < 200ms at p95 for read endpoints
- [ ] API response time < 500ms at p95 for write endpoints
- [ ] Batch operations (risk computation, SIS sync) process 10,000 students in < 5 minutes
- [ ] Learning event ingestion throughput >= 1,000 events/second

### Deployment
- [ ] Docker Compose local dev environment starts in < 60 seconds
- [ ] Database migrations run cleanly on fresh database and on existing production schema
- [ ] Environment variables documented; no secrets in code or version control
- [ ] Health check endpoints return appropriate status for all dependent services

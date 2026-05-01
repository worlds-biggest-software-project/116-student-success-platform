# Student Success Platform — Feature & Functionality Survey

> Candidate #116 · Researched: 2026-05-01

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| EAB Navigate360 | Predictive analytics + advising CRM + early alert | Proprietary SaaS | eab.com/navigate |
| Anthology Starfish | Care-network advising + shared visibility | Proprietary SaaS | anthology.com/starfish |
| Civitas Learning | Data-science analytics platform | Proprietary SaaS | civitaslearning.com |
| Ellucian CRM Advise | Advising workflow + case management (Banner/Colleague integrated) | Proprietary SaaS | ellucian.com |
| Salesforce Education Cloud | CRM-based student journey management | Proprietary SaaS | salesforce.com/education |
| Stellic | Degree planning and audit with what-if analysis | Proprietary SaaS | stellic.com |
| Advisor.AI | AI-native advising assistant + automated intervention | Proprietary SaaS | advisor.ai |
| Qualtrics StudentVoice | Survey and feedback platform for disengagement signals | Proprietary SaaS | qualtrics.com |
| Campus Labs (Anthology) | Co-curricular engagement + early alert | Proprietary SaaS | anthology.com |
| Brightspace Insights (D2L) | LMS-native analytics and at-risk dashboards | Proprietary SaaS | d2l.com |

## Feature Analysis by Solution

### EAB Navigate360

**Core features**
- Predictive risk scoring for individual students based on demographic, academic, and engagement data
- Advising CRM with appointment scheduling, note-taking, and case management
- Early alert system: faculty raise flags; advisors receive and action them
- Mobile app for students (course registration, appointment booking, to-do lists)
- Campaign management: advisors send targeted outreach to at-risk cohorts
- Reporting dashboards for advising productivity and retention outcomes

**Differentiating features**
- EAB's proprietary national data set (200M+ student records from member institutions) used to benchmark and calibrate risk models
- Research-backed intervention playbooks tied to specific risk signals
- Strategic advisory services included in contract (not purely a software vendor)

**UX patterns**
- Advisor-facing inbox for flag management with triage workflow
- Student-facing mobile app with personalised task lists and appointment calendar
- Configurable nudge messaging by cohort and risk level

**Integration points**
- Banner, Colleague, PeopleSoft, Workday Student SIS
- Canvas, Blackboard, Moodle LMS via Caliper or custom API
- Microsoft 365 / Google Workspace for calendar sync
- Financial aid systems via data extract

**Known gaps**
- High cost; typically limited to large research universities and four-year institutions
- Risk models trained on historic data may embed demographic bias (students of colour flagged at higher rates)
- Limited support for co-curricular engagement tracking outside academic advising

**Licence / IP notes**
- Proprietary; institutional data contributed to EAB's benchmarking consortium; data governance must be negotiated in contract to limit EAB's use of anonymised student data

---

### Anthology Starfish

**Core features**
- Care-network model: any staff member (faculty, advisor, tutor, financial aid) can see shared notes and flags
- Appointment scheduling across multiple support offices
- Success plans: personalised checklists assigned to at-risk students
- Early alert flags with tracking through resolution
- Kiosk check-in for tracking physical visit to support centres

**Differentiating features**
- Broadest care-network model: the most roles can see and contribute to a student's record
- Deep integration with Hobsons Naviance (high school pipeline) following Anthology merger
- Kiosk-based check-in captures engagement with physical campus support services

**UX patterns**
- Advisor overview dashboard showing caseload, upcoming appointments, and open flags
- Student portal showing their "care team" and how to reach each member
- Reporting on flag resolution time and intervention outcomes

**Integration points**
- Banner, Colleague, PeopleSoft SIS; Anthology's own Banner platform most deeply integrated
- Canvas, Blackboard LMS
- Microsoft Teams, Outlook for communication and calendar
- Tutoring system integrations (Tutor.com, Smarthinking)

**Known gaps**
- Less sophisticated predictive analytics than EAB Navigate; more workflow-focused than data-science-focused
- User interface has been criticised as dated compared to newer entrants
- AI capabilities are limited; no conversational advising or generative features

**Licence / IP notes**
- Proprietary; Anthology Inc. consolidated from multiple acquisitions; data governance follows Anthology privacy policy

---

### Civitas Learning

**Core features**
- Data science platform ingesting SIS, LMS, financial aid, and engagement data to build institution-specific predictive models
- Risk indicators at student, course, and programme level
- Advisor-facing intervention recommendations tied to model outputs
- Course demand forecasting and scheduling optimisation module
- Completion coaching product for online learners

**Differentiating features**
- Institution-specific model training rather than consortium benchmarking; models are built on each client's own data
- Course-level risk identifies high-DFW (Drop, Fail, Withdraw) courses as a risk factor independent of the individual student
- Advanced data science team embedded with client institutions for model tuning

**UX patterns**
- Data science dashboard for institutional research teams (not primarily student-facing)
- Advisor-facing list views with ranked risk scores and suggested actions
- Embedded coaching messaging for online student engagement

**Integration points**
- REST API data ingestion from SIS, LMS, financial aid
- Caliper and xAPI for LMS event streams
- Salesforce integration for advising case management

**Known gaps**
- Higher total cost of ownership due to embedded data science services
- Less focus on appointment scheduling and workflow; primarily an analytics layer
- Requires significant IT integration effort to operationalise data pipelines

**Licence / IP notes**
- Proprietary; raised $60M Series C; institution owns its own model artefacts; data sharing limited to the specific institution

---

### Ellucian CRM Advise

**Core features**
- Advising workflow management with task assignment, note-taking, and appointment scheduling
- Early alert integrated with Banner and Colleague SIS grade and enrollment data
- Case management for financial aid, academic appeals, and support referrals
- Configurable communication templates for automated outreach
- Reporting on advising activity and intervention completion rates

**Differentiating features**
- Deepest native integration with Banner and Colleague ERP; eliminates the need for API middleware
- Sold as part of the broader Ellucian suite, reducing procurement complexity for Ellucian shops
- Institution can configure workflow rules directly from Banner data without custom coding

**UX patterns**
- Advisor console embedded in the Ellucian Experience portal
- Student self-service portal for appointment booking and to-do list management
- Mobile-responsive design for advisor use on tablets

**Integration points**
- Banner, Colleague natively; other SIS via Ellucian Ethos integration platform
- Canvas, Blackboard via LTI and Ethos
- Degree audit integration with Ellucian Degree Works

**Known gaps**
- Predictive analytics capabilities are weaker than EAB or Civitas
- AI features are nascent; no conversational advising or NLP
- Strong lock-in to the Ellucian ecosystem; limited third-party integrations outside Ethos

**Licence / IP notes**
- Proprietary SaaS; Ellucian data governance applies; FERPA BAA standard for US institutions

---

### Salesforce Education Cloud

**Core features**
- CRM pipeline management for prospective, current, and alumni student relationships
- Advising case management using standard Salesforce Service Cloud capabilities
- Personalised communication journeys via Salesforce Marketing Cloud
- Einstein AI for predictive lead scoring adapted to student success use cases
- AppExchange ecosystem of 200+ education-specific add-ons

**Differentiating features**
- Most flexible and extensible platform; can model complex cross-institutional workflows
- Native integration with Salesforce Marketing Cloud for multi-channel communication
- Largest partner ecosystem of any platform in this space

**UX patterns**
- Highly configurable; institution must build their own UI layouts (Lightning App Builder)
- Student 360 view aggregating all touchpoints: enrollment, grades, advising notes, financial aid
- Mobile app via Salesforce mobile (configurable)

**Integration points**
- MuleSoft integration platform for SIS, LMS, financial aid connections
- Native Salesforce APIs; REST/SOAP and bulk data APIs
- AppExchange integrations with Banner, PeopleSoft, Workday via ISV partners

**Known gaps**
- Very high implementation cost and timeline (18–36 months for full deployment)
- Education Cloud is an overlay on the general Salesforce platform; not purpose-built for advising workflows
- Limited out-of-the-box predictive analytics for student success; requires Tableau or Einstein customisation

**Licence / IP notes**
- Proprietary; Salesforce standard data processing addendum covers FERPA; pricing is per-user per-month on top of base Salesforce licences

---

### Stellic

**Core features**
- Real-time degree audit against programme requirements
- What-if analysis: students can explore impact of changing major, adding a minor, or transferring credits
- Multi-term course planning with scheduling optimisation
- Transfer credit equivalency mapping
- Four-year graduation path visualisation

**Differentiating features**
- Most intuitive student-facing degree planning UI in the market
- Real-time audit (not nightly batch) ensures students always see current completion status
- AI-assisted transfer credit equivalency reduces manual advisor work for transfer students

**UX patterns**
- Student-facing drag-and-drop course planning board
- Advisor view shows planned vs. completed requirements with gap highlighting
- Mobile-first design with strong student adoption rates reported by clients

**Integration points**
- Banner, Colleague, PeopleSoft SIS for real-time degree audit
- LMS integration for course catalog and scheduling data
- API for data export to institutional data warehouse

**Known gaps**
- Focused narrowly on degree planning; does not cover advising CRM, early alert, or predictive analytics
- No AI conversational interface for student questions about degree requirements
- Limited reporting on advising productivity compared to full-suite platforms

**Licence / IP notes**
- Proprietary SaaS; founded 2018; degree audit logic owned by institution via configuration

---

### Advisor.AI

**Core features**
- AI-powered advising chatbot for 24/7 student self-service
- Automated intervention prompts triggered by risk signals from SIS/LMS data
- Natural language Q&A on degree requirements, financial aid, and campus resources
- Escalation to human advisor when conversation exceeds bot confidence threshold
- Conversation logging and sentiment tagging for advisor review

**Differentiating features**
- Natively AI-first; purpose-built for conversational advising rather than a chatbot bolt-on
- Continuous availability reduces student anxiety at off-hours (evenings, weekends)
- Sentiment analysis on student messages to detect distress and route to counselling

**UX patterns**
- Student-facing chat interface embedded in student portal or LMS
- Advisor dashboard showing bot conversation history and escalation queue
- Institution-configurable knowledge base (FAQ, policy documents) for bot grounding

**Integration points**
- SIS and LMS data feeds for real-time degree and grade information
- CRM integration (EAB Navigate, Starfish) for escalation hand-off
- SSO via institution's identity provider

**Known gaps**
- Accuracy dependent on quality of knowledge base maintenance; stale data causes hallucination risk
- No degree audit or appointment scheduling natively; relies on integrations
- Regulatory compliance (FERPA) for AI-generated advising responses requires careful governance

**Licence / IP notes**
- Proprietary SaaS; custom contracts; data processing agreement required for FERPA compliance

---

### Qualtrics StudentVoice

**Core features**
- Institutional survey design and deployment for student experience measurement
- Pulse surveys triggered by enrollment events, grade milestones, or advisor interactions
- Longitudinal tracking of student sentiment and engagement across a cohort
- Integration of survey responses into student risk profiles
- Dashboard and reporting for institutional research and student affairs leadership

**Differentiating features**
- Most sophisticated survey methodology of any platform in this space (borrowed from enterprise CX)
- iQ AI analysis: automatically identifies themes and sentiment from open-text responses
- Experience management framework connecting student sentiment to retention outcomes

**UX patterns**
- Survey designer with skip-logic and branching
- Institutional dashboard with drill-down by programme, campus, and demographic cohort
- Alert workflows when individual student scores fall below configurable thresholds

**Integration points**
- Banner, Colleague SIS for survey targeting and result tagging
- Canvas, Blackboard LMS
- Salesforce CRM for student contact record enrichment
- API integration with EAB Navigate or Anthology for flag creation from survey responses

**Known gaps**
- Primarily a survey tool; not a full student success CRM or early alert system
- Student response rates to institutional surveys are chronically low (typically 20–40%)
- AI text analysis may miss context-specific signals relevant to a specific institution

**Licence / IP notes**
- Proprietary; Qualtrics (SAP-owned until 2023 Silver Lake spin-out) standard data processing agreement

---

### Brightspace Insights (D2L)

**Core features**
- LMS-native analytics dashboard available to instructors and administrators
- At-risk student identification based on course activity, grades, and submission patterns
- Early alert flag creation directly within the LMS without leaving the instructor workflow
- Programme and course-level analytics on engagement and completion
- Integration with D2L Awards and competency tracking

**Differentiating features**
- Zero additional integration effort for D2L Brightspace institutions — analytics live inside the LMS
- Instructor-level visibility enables earlier flags at the course level before advisors are involved
- Predictive models are trained on D2L's multi-institutional data set (similar to EAB's consortium approach)

**UX patterns**
- Instructor-facing course insights panel embedded in course home page
- Administrator-facing institutional dashboard with enrolment and retention trend lines
- Configurable alert thresholds per course or programme

**Integration points**
- Native to D2L Brightspace LMS; no separate implementation required
- Caliper events feed to external analytics platforms via D2L Data Hub
- Integration with Banner and Colleague SIS for roster and grade sync

**Known gaps**
- Only valuable for institutions using D2L Brightspace; no cross-LMS portability
- Analytics are primarily engagement-based; does not incorporate financial aid, housing, or health data
- No advising CRM or appointment scheduling; must be paired with a separate platform

**Licence / IP notes**
- Proprietary; included in D2L Brightspace institutional subscription at certain tiers; D2L standard DPA

---

## Cross-Cutting Feature Themes

### Table-Stakes Features
- Early alert flag creation and resolution tracking
- Appointment scheduling and advisor note-taking
- Role-based visibility (student, advisor, faculty, counsellor, financial aid)
- SIS integration for real-time grade and enrollment data
- FERPA-compliant data handling and audit logs
- Basic reporting on advising activity and retention outcomes

### Differentiating Features
- Consortium-benchmarked or institution-specific predictive risk models (EAB, Civitas)
- Real-time degree audit with what-if analysis (Stellic)
- Conversational AI advising available 24/7 (Advisor.AI)
- Care-network model giving cross-functional staff shared visibility (Starfish)
- Device-level student engagement signal from LMS activity (Brightspace Insights, Civitas)
- Multi-channel campaign outreach with response tracking (EAB Navigate, Salesforce)

### Underserved Areas / Opportunities
- Causal explanation of risk: existing platforms provide risk scores but rarely explain which intervention type is most likely to work for a specific student given comparable peer outcomes
- Transfer credit intelligence at scale: AI parsing of transfer equivalency tables from thousands of institutions to auto-map incoming credits
- Wellbeing signal detection from unstructured student communications without violating privacy
- Integrated co-curricular and financial data: most platforms are LMS-and-SIS-only; food insecurity, housing instability, and employment status are highly predictive but rarely integrated
- Student-side agency: most platforms are advisor-facing; student-owned goal setting and self-monitoring is underdeveloped

### AI-Augmentation Candidates
- Proactive AI advisor initiating personalised outreach based on continuous monitoring of engagement signals
- LLM-powered 24/7 degree planning assistant with live audit data integration
- Causal intervention recommendation: "for students with this risk profile, peer tutoring has improved retention by 14% at comparable institutions"
- Wellbeing signal detection via NLP on student-written communications with appropriate consent and safeguards
- Automated transfer credit equivalency matching from unstructured transcript data

## Legal & IP Summary

- **FERPA** governs all US institutional deployments: student education records (grades, advising notes, flags) may not be shared without consent except under legitimate educational interest; all platforms in this space require a FERPA-compliant data sharing agreement.
- **HIPAA / Joint HHS-ED guidance**: counselling referral data and health records require separate handling; platforms that route mental-health flags must implement HIPAA-compliant data segregation.
- **EAB Navigate**: consortium data use (student records contributed to EAB's national dataset) must be explicitly scoped in the contract; institutions should ensure student anonymisation standards before data leaves the institution.
- **Salesforce Education Cloud**: most flexible platform but requires a separately negotiated FERPA addendum and significant custom data governance work via MuleSoft.
- **GDPR**: for EU institutions, all student-level risk scoring and automated decision-making involving predictive models may qualify as "automated processing with significant effects" under GDPR Art. 22, requiring human review before intervention.
- **Algorithmic bias**: predictive risk models that produce higher flag rates for students of colour, first-generation students, or students with disabilities expose institutions to disparate impact liability; model fairness audits should be contractually required from vendors.

## Recommended Feature Scope

**Must-have (MVP)**:
- Early alert flag creation, assignment, and resolution tracking across faculty, advisor, and support staff roles
- Appointment scheduling and advisor note-taking with FERPA-compliant access control
- SIS integration for real-time grade, enrollment, and academic standing data
- Student-facing portal with degree progress, upcoming appointments, and to-do checklist
- Basic risk scoring using LMS engagement and grade trajectory data with configurable thresholds
- Reporting dashboard for advising caseload, flag volume, and resolution rates

**Should-have (v1.1)**:
- Predictive risk model trained on institution-specific historical data (not consortium only)
- Real-time degree audit with what-if major/minor exploration for students and advisors
- AI advising chatbot for 24/7 Q&A on degree requirements, course registration, and campus resources
- Multi-channel outreach campaigns with response tracking (email, SMS, in-app)
- Transfer credit equivalency mapping with AI-assisted matching for incoming transfer students

**Nice-to-have (backlog)**:
- Causal intervention recommendation engine explaining which support type has highest success probability for a given risk profile
- Wellbeing signal detection from student-written text with privacy safeguards and counsellor escalation
- Co-curricular engagement integration (club involvement, library visits, tutoring attendance) as additional risk signal inputs
- Peer mentoring matching algorithm pairing at-risk students with successful near-peers
- Longitudinal cohort outcome tracking to measure platform ROI on retention and graduation rates

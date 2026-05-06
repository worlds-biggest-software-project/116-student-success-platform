# Student Success Platform

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An open, AI-native student success platform that unifies early warning, advising workflows, degree planning, and proactive intervention for higher education institutions.

The Student Success Platform is a retention and advising system for colleges and universities, built around early-warning signals from SIS and LMS data, advisor case management, and conversational AI guidance for students. It is aimed at provosts, advising directors, institutional researchers, and CIOs who today rely on expensive proprietary suites to track at-risk students and coordinate interventions.

---

## Why Student Success Platform?

- Incumbent suites (EAB Navigate360, Anthology Starfish, Civitas Learning, Ellucian CRM Advise) are priced at $80k–$1M+ per year, putting them out of reach of community colleges and smaller institutions despite serving the students who need retention support most.
- Salesforce Education Cloud is flexible but typically requires 18–36 month implementations and significant MuleSoft integration work before producing value.
- Most existing platforms surface a risk score without explaining *why* a student is at risk or which intervention type is most likely to reverse the trajectory.
- Anthology Starfish and Ellucian CRM Advise have been criticised for dated interfaces and nascent AI capabilities; conversational advising and generative features are absent or bolt-on.
- Predictive risk models trained on historic consortium data have raised algorithmic-bias concerns, particularly higher flag rates for students of colour and first-generation students, with limited vendor-side fairness auditing.

---

## Key Features

### Early Alert and Advising Workflow

- Flag creation, assignment, and resolution tracking across faculty, advisor, and support staff roles
- Appointment scheduling and advisor note-taking with FERPA-compliant access control
- Care-network style shared visibility so faculty, advisors, tutors, and financial aid staff can see and contribute to a student's record
- Caseload, flag-volume, and resolution-rate dashboards for advising leadership

### Degree Planning and Audit

- Real-time degree audit against programme requirements (not nightly batch)
- What-if analysis for changing major, adding a minor, or evaluating transfer credits
- Multi-term course planning with a student-facing drag-and-drop course planning experience
- Transfer credit equivalency mapping with AI-assisted matching for incoming transfer students

### Predictive Analytics and Risk Scoring

- Risk scoring using LMS engagement, grade trajectory, and attendance signals with configurable thresholds
- Institution-specific predictive models trained on the institution's own historical data
- Course-level risk signals identifying high-DFW (Drop, Fail, Withdraw) courses as independent risk factors
- Reporting at student, course, and programme level for institutional research teams

### Conversational AI Advising

- 24/7 AI advising chatbot for student self-service questions on degree requirements, course registration, and campus resources
- Escalation to a human advisor when the conversation exceeds a confidence threshold
- Conversation logging and sentiment tagging for advisor review
- Institution-configurable knowledge base (FAQ, policy documents) for grounding bot responses

### Outreach, Engagement, and Student Portal

- Multi-channel outreach campaigns (email, SMS, in-app) with response tracking
- Student-facing portal showing degree progress, upcoming appointments, and a personalised to-do checklist
- Pulse-survey integration for capturing engagement and sentiment signals
- Configurable nudge messaging by cohort and risk level

---

## AI-Native Advantage

A proactive AI advisor continuously monitors LMS engagement, grade trajectories, and attendance patterns and initiates personalised outreach at the optimal moment, removing the human bottleneck in early intervention. An LLM-powered advising chatbot answers natural-language questions like "what courses should I take next semester?" using live degree-audit data, far more actionable than static degree-plan PDFs. AI parses transfer credit policies and transcripts to auto-map equivalencies, a task that today consumes hundreds of advisor-hours per institution per year. Finally, causal intervention recommendation goes beyond risk scores to suggest which specific intervention (tutoring, financial aid, schedule change) has the highest probability of reversing a given student's trajectory based on comparable peer outcomes.

---

## Tech Stack & Deployment

The platform is designed to integrate with the major SIS systems (Banner, Colleague, PeopleSoft, Workday Student) and LMS platforms (Canvas, Blackboard, Moodle, Brightspace), consuming standards-based event streams via IMS Caliper Analytics v1.2 and xAPI to a Learning Record Store. Roster and grade exchange follows IMS LIS. Student-facing portals must meet WCAG 2.2 accessibility requirements, and all deployments require FERPA-compliant data handling and audit logs, with HIPAA-aware segregation for counselling and mental-health data and GDPR Article 22 human-review safeguards for EU institutions. Deployment is expected to support both self-hosted and managed cloud modes so that smaller institutions can adopt without enterprise-suite procurement.

---

## Market Context

The student success software market is estimated at $1.5–3 billion in 2025 within the broader $28.6 billion global LMS market, growing at 12–15% CAGR driven by retention mandates and completion-rate accountability. Enterprise contracts at large research universities run $300k–$1M+ per year, with mid-tier institutions paying $80k–$300k and degree-audit-only modules at $30k–$100k. Primary buyers are provosts and VPs of Academic Affairs, directors of advising and student affairs, CIOs, institutional researchers, and financial aid directors.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.

# Standards & API Reference

> Project: Student Success Platform · Generated: 2026-05-06

## Industry Standards & Specifications

### ISO Standards

- **ISO/IEC 40500:2012 (WCAG 2.2 reaffirmed as ISO/IEC 40500:2025)**
  URL: https://www.w3.org/TR/WCAG22/
  The Web Content Accessibility Guidelines 2.2 are formally adopted as an ISO standard. All student-facing portals and advisor dashboards must meet WCAG 2.2 Level AA to comply with the ADA (US deadline: April 24, 2026) and the European Accessibility Act (in force: June 28, 2025). Content conforming to WCAG 2.2 is backward-compatible with WCAG 2.1 and 2.0.

- **ISO/IEC 27001 (Information Security Management)**
  URL: https://www.iso.org/standard/27001
  Provides the framework for securing student records, advising notes, and risk-model outputs. Particularly relevant for cloud-hosted SaaS deployments that must demonstrate information security controls to institutional procurement officers.

### W3C & IETF Standards

- **WCAG 2.2 — Web Content Accessibility Guidelines**
  URL: https://www.w3.org/TR/WCAG22/
  The principal accessibility standard for student portals and advisor dashboards. Organized around four principles: perceivable, operable, understandable, and robust. Introduces nine new success criteria over WCAG 2.1, including focus appearance and accessible authentication. Compliance is required for any US higher education institution receiving federal funding (ADA, Section 508).

- **OAuth 2.0 (RFC 6749) and OpenID Connect 1.0**
  URLs: https://datatracker.ietf.org/doc/html/rfc6749 · https://openid.net/connect/
  OAuth 2.0 is the de-facto authorization framework for delegating access to student records across institutional systems. OpenID Connect (built on OAuth 2.0) adds an identity layer used for SSO across SIS, LMS, advising portals, and student mobile apps. All major edtech vendors (EAB, Anthology, Salesforce) require OAuth 2.0 token-based authentication for API access.

- **SAML 2.0 (Security Assertion Markup Language)**
  URL: https://www.oasis-open.org/standards/#samlv2.0
  Widely used in higher education for federated SSO between identity providers (Microsoft Entra ID, Okta, Shibboleth) and service providers including student success platforms. Complements OpenID Connect in institutions that have not yet migrated to OIDC.

- **SCIM 2.0 — System for Cross-domain Identity Management (RFC 7642–7644)**
  URL: https://datatracker.ietf.org/doc/html/rfc7643
  Standard for automating user provisioning and deprovisioning across edtech applications. Relevant for onboarding and offboarding student, advisor, and faculty accounts in a student success platform without manual SIS exports.

- **RFC 7807 — Problem Details for HTTP APIs**
  URL: https://datatracker.ietf.org/doc/html/rfc7807
  Best-practice standard for consistent error response bodies in REST APIs. Recommended for any student success platform REST API to enable reliable error handling by integration clients.

### Data Model & API Specifications

- **1EdTech Caliper Analytics v1.2**
  URL: https://www.imsglobal.org/spec/caliper/v1p2 · GitHub: https://github.com/1EdTech/caliper-spec
  The primary standard for capturing fine-grained learning activity events (course access, submission, assessment attempt) and transmitting them from LMS and digital tools to analytics stores and student success platforms. Uses a Sensor API with metric profiles. Version 1.2 adds six new metric profiles. Caliper events feed predictive risk models that identify at-risk students. Implementation Guide: https://www.imsglobal.org/spec/caliper/v1p2/impl

- **xAPI (Experience API) v1.0.3 — ADL**
  URL: https://github.com/adlnet/xAPI-Spec · Overview: https://xapi.com/overview/
  A complementary learning event standard to Caliper. xAPI uses an "[actor] [verb] [object]" statement structure stored in a Learning Record Store (LRS). Supports tracking activities outside the LMS (tutoring, co-curricular, peer mentoring). LRS conformance requirements: https://adl.gitbooks.io/xapi-lrs-conformance-requirements/content/

- **1EdTech LTI (Learning Tools Interoperability) v1.3**
  URL: https://www.imsglobal.org/spec/lti/v1p3
  Core specification for launching external tools (including student success dashboards) from within an LMS with authenticated context. LTI v1.3 uses OpenID Connect + signed JWTs replacing OAuth 1.0a. Assignment and Grade Services (AGS) v2.0 (https://www.imsglobal.org/spec/lti-ags/v2p0) enables grade write-back to LMS grade books from advising tools.

- **1EdTech OneRoster v1.2**
  URL: https://www.imsglobal.org/spec/oneroster/v1p2
  Standard for exchanging roster, enrollment, and grade data between SIS and LMS/analytics platforms. Supports both REST API and CSV batch exchange. Defines six core entities: organizations, users, courses, classes, enrollments, and demographics. Critical integration point for pulling current enrollment and academic standing into a student success platform.

- **1EdTech Edu-API v1.0 (Candidate Final)**
  URL: https://www.imsglobal.org/spec/eduapi/v1p0 · Overview: https://www.1edtech.org/standards/edu-api
  A newer standard that supersedes LIS for higher education administrative data exchange between SIS and the broader teaching and learning ecosystem. Covers personal/biographic data, course catalog, enrollments, and final outcomes. Oracle PeopleSoft Campus Solutions achieved the first Edu-API certification in October 2025. Intended to reduce integration cost and replace custom SIS extracts.

- **OpenAPI Specification v3.1 / v3.2**
  URL: https://spec.openapis.org/oas/v3.1.0.html · https://spec.openapis.org/oas/v3.2.0.html
  The lingua franca for documenting REST APIs. All student success platform REST endpoints should be described in OpenAPI to enable auto-generation of SDKs, client libraries, and interactive documentation. LTI AGS v2.0 already publishes an OpenAPI spec (https://www.imsglobal.org/spec/lti-ags/v2p0/openapi).

### Security & Compliance Standards

- **FERPA — Family Educational Rights and Privacy Act (US)**
  URL: https://studentprivacy.ed.gov/ferpa · Guidance hub: https://studentprivacy.ed.gov/
  The foundational US federal law governing access to and disclosure of student education records. Any platform handling grades, advising notes, early-alert flags, or risk scores must operate under a signed FERPA-compliant data sharing agreement with each institution. The Department of Education issued stricter enforcement guidance in March 2025, requiring state agencies to certify compliance and mandating audit logging of all data access (timestamp, user, records viewed, modifications, retained minimum three years). Currently lacks clear cybersecurity requirements for edtech vendors — legislative modernisation is under discussion.

- **HIPAA / Joint HHS-ED Guidance (US)**
  URL: https://www.hhs.gov/hipaa/index.html
  Applies when the platform handles student health or mental-health records routed from counselling or wellness referrals. Platforms enabling wellbeing signal detection or counselling escalation must implement HIPAA-compliant data segregation and a signed Business Associate Agreement (BAA). The 2025 HIPAA Security Rule update introduces stricter cybersecurity requirements including MFA, ePHI inventory, and third-party vendor oversight.

- **GDPR — General Data Protection Regulation (EU)**
  URL: https://gdpr.eu/article-22-automated-individual-decision-making/
  Applies to all EU institutions and any SaaS vendor processing EU student data. Article 22 restricts solely automated decision-making that produces significant effects on individuals — student risk scoring and automated intervention triggers may fall within scope, requiring meaningful human review before action. Data Protection Impact Assessments (DPIAs) are required for AI-driven profiling. From February 2025 the EU AI Act began banning high-risk AI systems and imposing transparency obligations on others.

- **NIST SP 800-171 Rev. 3 (Protecting Controlled Unclassified Information)**
  URL: https://csrc.nist.gov/publications/detail/sp/800/171/rev-3/final
  Increasingly required for US higher education institutions handling Federal Tax Information (FTI) shared as part of FAFSA processing, DoD-funded research, and NIH genomic data. Provides 110 security controls across 14 families. EDUCAUSE QuickPoll (March 2025) confirmed widespread compliance efforts across US research universities. Can be mapped to HIPAA security rule requirements with 18 additional controls.

### MCP Server Specifications

- **Model Context Protocol (MCP)**
  URL: https://modelcontextprotocol.io/
  Anthropic's open protocol for connecting AI agents to external data sources and tools via standardised server interfaces. Relevant if the platform's AI advising agent needs real-time access to degree audit data, SIS records, or early-alert flags without bespoke API integrations. MCP servers could expose SIS data, degree audit queries, and advising note retrieval as tools callable by an LLM-powered advising assistant.

---

## Similar Products — Developer Documentation & APIs

### EAB Navigate360

- **Description:** Higher education's leading advising CRM and early alert platform, used by 850+ institutions. Combines predictive risk scoring, appointment scheduling, campaign management, and a student mobile app.
- **API Documentation:** https://help.navigate360.com/ (institution login required); public API information at https://www.eabsystems.com/api.html
- **SDKs/Libraries:** No public SDK; institutions build connectors to the Navigate REST API; time-intensive per EAB documentation.
- **Developer Guide:** Navigate360 Knowledge Base (https://help.navigate360.com/); partner resources at https://eab.com/partner-hub/navigate360-resource-hub/
- **Standards:** REST/JSON; SIS integration via Banner, Colleague, PeopleSoft, Workday Student; LMS via Caliper or custom feed.
- **Authentication:** Institutional SSO (SAML 2.0 / OIDC); API token-based for data feeds.

### Anthology (Blackboard + Starfish)

- **Description:** Anthology's developer portal covers Blackboard Learn and Student APIs. Starfish-specific integration is accessed through the broader Anthology developer ecosystem post-merger.
- **API Documentation:** https://docs.anthology.com/ · Blackboard REST APIs: https://docs.anthology.com/docs/blackboard/rest-apis/start-here · Student REST API: https://docs.anthology.com/docs/student/getting-started/first-steps
- **SDKs/Libraries:** Anthology developer portal at https://developer.blackboard.com/ provides SDK resources; GitHub: https://blackboard.github.io/
- **Developer Guide:** https://docs.anthology.com/docs/blackboard/rest-apis/getting-started/framework
- **Standards:** REST/JSON; LTI 1.3 for tool launch; Caliper for learning events; Banner and Colleague SIS integration.
- **Authentication:** OAuth 2.0 (client credentials and authorization code flows); developer portal key/secret registration.

### Ellucian Ethos Integration Platform

- **Description:** Ellucian's integration middleware connecting Banner and Colleague ERP to LMS, advising, and analytics tools. Acts as the integration backbone for Ellucian CRM Advise.
- **API Documentation:** https://resources.elluciancloud.com/ethos-data-model (login required); SDK docs: https://ellucian-developer.github.io/integration-sdk-doc/
- **SDKs/Libraries:** Java SDK: https://github.com/ellucian-developer/integration-sdk-objects-java-doc · C# SDK: https://github.com/ellucian-developer/integration-sdk-csharp · Ethos GitHub community: https://github.com/ellucianEthos
- **Developer Guide:** Terminalfour integration guide: https://docs.terminalfour.com/articles/integrating-with-ellucian-ethos/; Tray.ai connector: https://tray.ai/documentation/connectors/service/ellucian-ethos
- **Standards:** REST/JSON over the Ethos API; event streaming; Edu-API alignment in progress; LTI for tool connections via the Ellucian Experience portal.
- **Authentication:** OAuth 2.0; API key for data extract connections.

### Salesforce Education Cloud

- **Description:** CRM-based student journey management and advising pipeline platform adapted from Salesforce Service and Marketing Clouds. Einstein AI provides predictive scoring. AppExchange hosts 200+ education ISV add-ons.
- **API Documentation:** https://developer.salesforce.com/docs/atlas.en-us.edu_cloud_dev_guide.meta/edu_cloud_dev_guide/edu_cloud_intro.htm · REST Reference: https://developer.salesforce.com/docs/atlas.en-us.edu_cloud_dev_guide.meta/edu_cloud_dev_guide/edu_cloud_apis_rest_references.htm
- **SDKs/Libraries:** Salesforce Developer Center: https://developer.salesforce.com/developer-centers/education-cloud; Salesforce DX CLI, Apex, and SOQL for custom development.
- **Developer Guide:** Education Cloud Developer Guide v66.0 (Spring '26): https://resources.docs.salesforce.com/latest/latest/en-us/sfdc/pdf/edu_cloud_dev_guide.pdf
- **Standards:** REST/JSON (Salesforce REST API and Bulk API); SOQL query language; OpenAPI-compatible Swagger definitions; MuleSoft for SIS/LMS integration.
- **Authentication:** OAuth 2.0 (username-password, JWT Bearer, web server flows); per-user session tokens.

### Stellic

- **Description:** Degree planning, real-time degree audit, and what-if analysis platform. Used by CMU, Cornell, MIT, and Wisconsin System. Provides the most student-friendly degree planning UX in the market.
- **API Documentation:** https://docs.stellic.com/ (developer portal with RESTful API overview and authentication guide)
- **SDKs/Libraries:** No public SDK listed; REST API access negotiated per institution.
- **Developer Guide:** https://docs.stellic.com/ — covers resource overview, RESTful access patterns, authentication, and error codes; includes Export APIs for prospective student and appointment data.
- **Standards:** REST/JSON; real-time SIS integration with Banner, Colleague, PeopleSoft via batch feeds and event-driven API calls; LMS integration for course catalog.
- **Authentication:** API key / token-based; institution SSO via SAML 2.0 or OIDC for student and advisor login.

### D2L Brightspace (Data Hub & Data Streams)

- **Description:** LMS with native analytics (Brightspace Insights) and at-risk dashboards. Data Hub exports batch data sets; Data Streams provides real-time xAPI-based event streaming for external analytics platforms.
- **API Documentation:** Brightspace API Reference (April 2026): https://docs.valence.desire2learn.com/reference.html · Data Streams (August 2025): https://docs.datastreams.desire2learn.com/
- **SDKs/Libraries:** GitHub client example: https://github.com/Brightspace/data-hub-client-example; Brightspace community knowledge base: https://community.d2l.com/brightspace/kb/articles/1130-how-to-get-started-with-data-hubs-apis-brightspace-data-sets
- **Developer Guide:** https://community.d2l.com/brightspace/kb/articles/1102-getting-started-with-brightspace-api-automation
- **Standards:** REST/JSON (Valence API); xAPI for Data Streams event output (https://docs.datastreams.desire2learn.com/xapi/index.html); Caliper for external event consumers; LTI 1.3 for tool launch.
- **Authentication:** OAuth 2.0 (authorization code flow); per-application key/secret registered in Brightspace admin.

### Civitas Learning

- **Description:** Data science platform converting SIS, LMS, and financial aid data into institution-specific predictive risk models and advisor intervention recommendations. Standard Data Specifications documented at their terms portal.
- **API Documentation:** https://www.civitaslearning.com/termsandconditions/standard-data-specifications/ (standard data specs); documentation hub available to contracted institutions.
- **SDKs/Libraries:** REST API data ingestion; no public SDK.
- **Developer Guide:** Available to institutional clients; ingestion via REST API or secure SFTP batch extract.
- **Standards:** REST/JSON ingestion; Caliper and xAPI for LMS event streams; Salesforce CRM integration for case management hand-off.
- **Authentication:** Institution-specific API credentials; data sharing agreement required before access is provisioned.

---

## Notes

- **Edu-API adoption is still early (2025–2026):** Oracle PeopleSoft achieved the first certification in October 2025. Institutions still predominantly rely on OneRoster, LIS, and custom SIS extracts. An AI-native platform should support both Edu-API and legacy patterns during the transition period.
- **Caliper vs. xAPI:** Both standards are in active use. Caliper is more widely certified among LMS vendors in higher education; xAPI is more common in corporate and military training contexts but gaining ground for co-curricular tracking. Supporting both LRS outputs broadens the institution addressable market.
- **GDPR Art. 22 and EU AI Act tensions:** Risk scoring that triggers automated intervention messages to students may require a human-in-the-loop workflow to remain compliant, particularly for EU institutions. Platform architecture should make it straightforward for institutions to insert advisor review before any AI-generated outreach is sent.
- **MCP is emerging (not yet adopted by edtech vendors):** No incumbent student success vendor has published an MCP server as of May 2026. This represents an early-mover opportunity: exposing SIS, degree audit, and advising note retrieval as MCP tools would make the platform the natural integration point for any institution deploying AI advising agents.

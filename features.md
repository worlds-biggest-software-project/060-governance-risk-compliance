# Governance, Risk & Compliance — Feature & Functionality Survey

> Candidate #60 · Researched: 2026-05-02

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| MetricStream | Commercial SaaS | Proprietary / custom enterprise | https://www.metricstream.com |
| ServiceNow GRC | Commercial SaaS | Proprietary / Now Platform licence | https://www.servicenow.com/products/governance-risk-compliance.html |
| RSA Archer | Commercial on-prem/SaaS | Proprietary / custom | https://www.archerirm.com |
| IBM OpenPages | Commercial SaaS | Proprietary / IBM Cloud | https://www.ibm.com/products/openpages |
| OneTrust GRC | Commercial SaaS | Proprietary / modular | https://www.onetrust.com/products/grc-platform |
| LogicGate Risk Cloud | Commercial SaaS | Proprietary / subscription | https://www.logicgate.com |
| Hyperproof | Commercial SaaS | Proprietary / subscription | https://hyperproof.io |
| Eramba | Open Source / freemium | GPL v3 (Community) / proprietary (Enterprise) | https://www.eramba.org |
| SimpleRisk | Open Source / freemium | MPL 2.0 (core) / proprietary add-ons | https://www.simplerisk.com |
| CISO Assistant | Open Source | Apache 2.0 | https://github.com/intuitem/ciso-assistant-community |

## Feature Analysis by Solution

### MetricStream

**Core features**
- Enterprise risk register with hierarchical risk taxonomy and inherent/residual scoring
- Policy lifecycle management with approval workflows and attestation tracking
- Internal audit planning, fieldwork, and finding management
- Third-party/vendor risk management module with due diligence questionnaires
- Regulatory change management with control impact mapping
- Incident management and operational loss capture
- ESG risk and reporting module

**Differentiating features**
- "Connected GRC" unified data model linking risk, control, audit, and compliance objects across domains
- AI-powered risk scoring suggestions and anomaly detection on control testing results
- Pre-built regulatory content packs for Basel, DORA, SOX, GDPR, and ISO 27001
- Real-time risk heat maps and board-level dashboards
- Autonomous and automated control testing with alert-based reporting on control failures

**UX patterns**
- Role-based dashboards tailored to CRO, CCO, CISO, and Internal Audit Director personas
- Complex multi-tab configuration UI criticized as dated by Gartner peer reviewers
- Guided task workflows for periodic risk assessments and control certifications

**Integration points**
- REST APIs for ERP, ITSM (ServiceNow), and identity systems
- Pre-built connectors for SAP, Oracle, Workday, and Jira
- SIEM and vulnerability scanner integrations for cyber risk data

**Known gaps**
- UI and implementation complexity remain the most-cited complaints in peer reviews
- Very expensive; SMBs and mid-market companies are effectively priced out
- Limited natural-language or generative AI capabilities as of early 2026

**Licence / IP notes**
- Fully proprietary; no open-source components in the core platform
- No patent concerns identified for the OSS project

---

### ServiceNow GRC

**Core features**
- Risk register with risk relationships mapped to business services and IT assets
- Policy and compliance management with attestation and exception handling
- Vendor risk management with automated questionnaire workflows
- Operational resilience and business continuity modules
- Audit management as part of the Now Platform

**Differentiating features**
- Native integration with CMDB, ITSM, and ITOM data — strongest IT/cyber GRC in market
- Single platform for IT operations and GRC, eliminating data silos for tech-heavy organisations
- Now Assist AI co-pilot for generating risk summaries, drafting remediation plans, and Q&A

**UX patterns**
- Portal-driven interface consistent with broader ServiceNow Now Platform UX
- Configurable playbooks and guided setups for common frameworks (SOC 2, ISO 27001)
- Mobile-responsive risk and compliance dashboards

**Integration points**
- Deep native integration with all ServiceNow modules (ITSM, HR, SecOps)
- REST/SOAP APIs and MID Server connectors for on-premise systems
- Integration Hub for 700+ third-party connectors

**Known gaps**
- Weak for non-IT risk domains (ERM, ORM, financial risk) without heavy customisation
- Requires existing ServiceNow platform investment; prohibitively expensive as a standalone GRC tool
- No meaningful OSS alternative at this integration depth

**Licence / IP notes**
- Fully proprietary; no open-source components in core GRC modules

---

### RSA Archer

**Core features**
- Risk register with quantitative risk scoring and risk appetite tracking
- Business continuity and disaster recovery management
- Third-party risk management with continuous monitoring integrations
- Regulatory and corporate compliance management
- Policy library with acknowledgement workflows
- Audit management with workpaper and finding workflows

**Differentiating features**
- Long-standing presence in BFSI and US federal government creates deep compliance content
- Flexible application builder allows non-standard risk domains to be configured without code

**UX patterns**
- Interface widely criticised as dated; complex multi-level navigation
- High implementation consulting burden — typical go-live 6–18 months

**Integration points**
- APIs for SIEM, vulnerability management, threat intelligence feeds
- Integration with ServiceNow, Splunk, and major ERP vendors via connectors

**Known gaps**
- Product innovation cycle widely considered slow
- AI/ML capabilities underdeveloped compared to newer entrants
- On-premise architecture increasingly at odds with cloud-first buyer preferences

**Licence / IP notes**
- Fully proprietary (RSA Security LLC)

---

### IBM OpenPages

**Core features**
- Financial risk, regulatory compliance (Basel IV, DORA, IFRS), and ESG risk management
- Operational risk management with loss event capture and scenario analysis
- Control assessment library with automated control testing workflows
- Policy management and regulatory change tracking
- Model risk management module

**Differentiating features**
- AI-assisted risk insights via IBM watsonx; natural-language queries against risk data
- Strongest platform for financial services regulatory requirements (BCBS, DORA, IFRS 9)
- Pre-built content packs for 50+ global regulations updated by IBM Research

**UX patterns**
- Modern web UI deployed on IBM Cloud or on-premise
- Configurable dashboards per role (CRO, Head of Compliance, Audit Committee)

**Integration points**
- IBM Cloud Pak for Data integration for analytics workloads
- REST APIs and pre-built connectors for SAP, Oracle, and major HRIS platforms

**Known gaps**
- IBM ecosystem dependency limits appeal for non-IBM shops
- Pricing architecture is opaque; implementation costs frequently exceed licence costs
- Smaller community and partner ecosystem compared to ServiceNow or MetricStream

**Licence / IP notes**
- Fully proprietary; IBM Cloud-hosted

---

### OneTrust GRC

**Core features**
- Risk register with configurable risk scoring and risk treatment plans
- Policy lifecycle management with multi-language policy distribution
- Third-party risk management with automated vendor assessments (900+ pre-built questionnaires)
- Privacy and data risk management (DPIAs, RoPA, data mapping)
- Compliance framework management for GDPR, CCPA, ISO 27001, SOC 2, DORA, EU AI Act

**Differentiating features**
- Leading convergence of privacy management and GRC — unique for privacy-first compliance programmes
- AI-powered third-party risk scoring aggregating data from external threat intelligence feeds
- Largest library of pre-built assessment templates and regulatory content in the market

**UX patterns**
- Modern, polished UI; one of the best in class for end-user experience among enterprise GRC tools
- Self-service vendor assessment portals with guided questionnaire completion
- Drag-and-drop workflow builder for compliance processes

**Integration points**
- 200+ native integrations with cloud infrastructure, SaaS applications, and identity providers
- Continuous evidence collection via automated Hypersyncs-style connectors

**Known gaps**
- GRC depth (ERM, ORM, quantitative risk) narrower than MetricStream or IBM OpenPages
- Modular pricing structure leads to significant cost increases as modules are added
- Not designed as an internal audit management platform

**Licence / IP notes**
- Fully proprietary; no open-source components disclosed

---

### LogicGate Risk Cloud

**Core features**
- Configurable risk register with drag-and-drop workflow builder
- Compliance management with multi-framework control mapping
- Vendor risk management and third-party assessment automation
- Audit management with finding and remediation tracking
- Incident management and reporting

**Differentiating features**
- No-code workflow builder is the strongest differentiator — risk and compliance teams can configure new use cases without IT support
- Rapid implementation (weeks vs. months for enterprise competitors) drives mid-market adoption
- Pre-built solutions ("Risk Cloud Solutions") cover ERM, third-party risk, and compliance without custom build

**UX patterns**
- Clean, modern SaaS interface consistent with contemporary B2B design standards
- Kanban-style workflow views for risk and compliance task management
- In-product analytics with configurable dashboards

**Integration points**
- REST API and Zapier integration for 500+ applications
- Native integrations with Jira, Slack, ServiceNow, and major cloud providers

**Known gaps**
- Less depth for complex enterprise ERM use cases with large risk hierarchies
- Smaller pre-built regulatory content library than OneTrust or MetricStream
- AI features are nascent compared to AuditBoard or Hyperproof

**Licence / IP notes**
- Fully proprietary SaaS

---

### Hyperproof

**Core features**
- Continuous compliance operations with automated evidence collection via Hypersyncs
- Multi-framework control mapping (200+ frameworks including SOC 2, ISO 27001, NIST CSF 2.0, CMMC, EU AI Act, DORA)
- Risk register with risk-to-control linking
- Audit management and external auditor collaboration portal
- Policy management with version control and attestation

**Differentiating features**
- Hypersyncs automates evidence collection from cloud applications (AWS, GCP, GitHub, Okta, etc.) — reducing manual evidence gathering by up to 80%
- Best-in-class multi-framework overlap mapping: one control satisfies multiple framework requirements simultaneously, displayed visually
- AI-powered compliance gap analysis comparing current control state to framework requirements

**UX patterns**
- Modern, task-oriented UX designed for compliance operations managers
- Program view showing overall compliance posture across all active frameworks simultaneously
- Collaboration tools allowing external auditors to request and receive evidence without platform access

**Integration points**
- 70+ native integrations with cloud infrastructure, SaaS tools, identity providers, and ticketing systems
- Webhook and API support for custom integrations

**Known gaps**
- Narrowly focused on IT/security compliance; limited operational risk or ERM depth
- Not a full internal audit management platform (limited fieldwork and workpaper support)
- Less suited for non-technical compliance domains (financial risk, third-party risk at enterprise scale)

**Licence / IP notes**
- Fully proprietary SaaS

---

### Eramba

**Core features**
- Risk management with risk register, risk treatment plans, and risk appetite tracking
- Compliance management with framework packages (ISO 27001, GDPR, PCI DSS, HIPAA)
- Policy lifecycle management with acknowledgement tracking
- Internal audit management with finding and remediation tracking
- Incident management module
- Third-party risk assessment module
- Business continuity planning

**Differentiating features**
- Only mature open-source GRC platform with no user or data limits in the Community Edition
- Started by CISOs in 2007; pragmatic, battle-tested design based on real GRC operations
- Community Edition is completely free (GPL v3) and self-hostable; Enterprise tier adds support and advanced modules at affordable rates

**UX patterns**
- Functional but visually dated interface compared to commercial competitors
- Dashboard-driven interface with customisable risk and compliance views
- Task-based workflow for risk assessments, control reviews, and policy attestations

**Integration points**
- REST API available (primarily in Enterprise Edition)
- LDAP/Active Directory integration for authentication
- Limited native integrations with external systems compared to commercial tools

**Known gaps**
- UI significantly less polished than modern commercial tools (LogicGate, Hyperproof)
- No AI or ML capabilities
- Limited continuous control monitoring or automated evidence collection
- Community Edition support is community-forum-only; no SLA

**Licence / IP notes**
- Community Edition: GPL v3 — OSS-compatible; derivative works must be GPL
- Enterprise Edition: proprietary licence on top of GPL core
- No patent concerns identified

---

### SimpleRisk

**Core features**
- Risk register as the primary module with risk scoring, treatment planning, and review workflows
- Governance module covering policies and procedures
- Audit management with audit planning and finding tracking
- Vendor assessment and management module
- Compliance framework mapping (supplemental to risk register)

**Differentiating features**
- Fastest deployment of any GRC tool — runnable on a LAMP stack in minutes
- Simple, uncluttered risk register UI that non-GRC professionals can use on day one
- Open core with Community Edition available under Mozilla Public Licence 2.0

**UX patterns**
- Minimal, spreadsheet-like interface prioritising speed over visual sophistication
- Risk register as the entry point and primary interaction surface
- Email-based workflow notifications for risk review assignments

**Integration points**
- REST API in paid tiers
- Active Directory and SAML authentication
- Jira integration for risk-to-issue tracking

**Known gaps**
- Many practical features (API, custom reports, authentication, notifications) locked behind paid bundles
- Narrower framework coverage than CISO Assistant or Eramba
- No AI features; limited automation

**Licence / IP notes**
- Core: Mozilla Public Licence 2.0 — permissive for embedding; file-level copyleft only
- Premium bundles: proprietary

---

### CISO Assistant

**Core features**
- Multi-framework compliance management with 150+ built-in frameworks
- Automatic cross-framework control mapping (one control satisfies multiple frameworks)
- Risk register with threat and vulnerability catalogues
- Security assessment and gap analysis workflows
- Policy and procedure management
- Audit management with evidence tracking

**Differentiating features**
- Widest open-source compliance framework library: ISO 27001, SOC 2, NIST CSF 2.0, NIST 800-53, PCI DSS, HIPAA, NIS2, DORA, GDPR, EU AI Act, CIS Controls, CMMC, Essential Eight, and 130+ more
- Cross-framework overlap view is the strongest in the OSS category; visualises which controls satisfy multiple frameworks simultaneously
- Most actively developed OSS GRC project as of 2026 (3,600+ GitHub stars, 80+ contributors); Docker-based deployment is easiest in class

**UX patterns**
- Modern web application UI — significantly more polished than Eramba or SimpleRisk
- Framework-centric navigation: users work from framework requirements inward to controls
- Evidence collection and assessment workflow designed for IT/security compliance teams

**Integration points**
- REST API for programmatic access
- SSO via SAML and OIDC
- Webhook support for notifications
- SaaS hosted option available for teams that cannot self-host

**Known gaps**
- Primarily focused on IT/security compliance; limited operational risk or ERM depth
- No AI-powered features natively (LLM integration on roadmap)
- Newer and less battle-tested at scale compared to Eramba
- 409A-equivalent quantitative risk modelling absent

**Licence / IP notes**
- Apache 2.0 licence — most permissive of OSS GRC tools; compatible with commercial and proprietary use
- No patent concerns identified

---

## Cross-Cutting Feature Themes

### Table-Stakes Features
- Risk register with inherent and residual risk scoring, risk treatment plans, and review workflows
- Control library with control-to-risk and control-to-framework linkage
- Policy lifecycle management with version control and user acknowledgement tracking
- Compliance framework mapping with gap analysis against selected standards
- Internal audit management covering planning, fieldwork, finding management, and CAP tracking
- Third-party/vendor risk assessment with questionnaire workflows
- Incident and issue management with root cause and remediation tracking
- Role-based access control with at minimum CRO, CCO, CISO, and Auditor personas
- Audit trail and evidence repository for regulatory examination readiness
- Dashboard and reporting for board-level risk reporting

### Differentiating Features
- Continuous automated control monitoring rather than periodic point-in-time assessment
- Automated evidence collection from cloud systems (Hyperproof Hypersyncs model)
- Multi-framework overlap mapping showing one control satisfying multiple requirements
- Regulatory change tracking with automated impact assessment on controls
- AI-powered risk scoring and anomaly detection on control test results
- Quantitative risk analysis (Monte Carlo, risk appetite frameworks)
- Integrated ESG risk and sustainability reporting
- External auditor collaboration portal reducing friction in annual audit engagements

### Underserved Areas / Opportunities
- AI-native regulatory change monitoring that continuously ingests regulatory sources and maps changes to controls — no OSS tool does this
- Continuous control testing replacing periodic attestation cycles — no OSS tool approaches this
- Policy-to-control gap analysis using LLMs to identify misaligned, duplicate, or missing policy/control relationships
- Integrated operational risk loss data capture and modelling for financial services (Basel IV ORM)
- Affordable GRC platform for companies between $5M–$200M revenue that are priced out of MetricStream/ServiceNow but need more than SimpleRisk
- EU AI Act compliance management (CISO Assistant has framework breadth but limited workflow depth)

### AI-Augmentation Candidates
- Regulatory change monitoring: LLMs continuously ingesting official regulatory publications, classifying applicability by industry/geography, and mapping impacts to the control register
- Risk narrative drafting: generating initial risk descriptions, likelihood/impact rationale, and treatment plan text from structured register data
- Control test automation: AI agents querying source systems (ERP, HRIS, access logs), sampling populations, and asserting compliance — replacing manual evidence collection
- Policy-to-control alignment: LLM analysis of policy library and control library to identify gaps, conflicts, and duplicates
- Third-party risk scoring: AI aggregating external threat intelligence, news, and questionnaire responses into dynamic vendor risk scores
- Audit finding drafting: generating IIA-standard finding narratives, root cause analyses, and recommendations from structured audit data

---

## Legal & IP Summary

All three mature open-source GRC tools use compatible licences: Eramba Community Edition under GPL v3, SimpleRisk core under Mozilla Public Licence 2.0, and CISO Assistant under Apache 2.0. Apache 2.0 is the most permissive and best suited for a commercial open-source strategy with a hosted SaaS tier. GPL v3 would impose copyleft obligations on derivative works. No patent-encumbered techniques were identified in any of the solutions analysed; GRC workflow logic (risk scoring, control mapping, audit lifecycle) is well-established and non-novel from a patent perspective. Commercial tools (MetricStream, ServiceNow, IBM OpenPages) are fully proprietary and should not be used as implementation references. Hyperproof's term "Hypersyncs" appears to be a brand name rather than a patented mechanism; automated evidence collection via API polling is a generic technique with no known patent claims.

---

## Recommended Feature Scope

**Must-have (MVP)**:
- Risk register with configurable risk scoring, treatment plans, and periodic review workflows
- Control library with many-to-many linking of risks, controls, and compliance frameworks
- Built-in framework packs for ISO 27001, SOC 2, NIST CSF 2.0, GDPR, and EU AI Act (minimum)
- Policy lifecycle management with version control, approval workflows, and user attestation
- Internal audit management covering annual plan, audit engagements, findings, and corrective actions
- Role-based access with auditor, risk owner, compliance manager, and admin personas
- REST API and SSO (SAML/OIDC) from day one

**Should-have (v1.1)**:
- Automated evidence collection via API connectors to common cloud platforms (AWS, GCP, GitHub, Okta, Jira)
- AI-powered regulatory change monitoring with impact mapping to controls
- Multi-framework overlap view showing cross-framework control satisfaction
- Third-party risk management with questionnaire builder and vendor portal
- Natural-language risk assessment assistance (LLM-powered risk description and scoring suggestions)

**Nice-to-have (backlog)**:
- Quantitative risk modelling (Monte Carlo simulation on risk register)
- Integrated ESG/sustainability risk module aligned to ESRS and TCFD
- External auditor collaboration portal with evidence request workflow
- Continuous automated control testing agents for IT general controls
- Board-level risk reporting pack with one-click export to PDF/PowerPoint

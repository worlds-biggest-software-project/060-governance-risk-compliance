# Standards & API Reference

> Project: Governance, Risk & Compliance (GRC) · Generated: 2026-05-06

## Industry Standards & Specifications

### ISO Standards

**ISO 31000:2018 — Risk Management Guidelines**
- URL: https://www.iso.org/standard/65694.html
- The primary international standard for risk management principles, framework, and process. Applicable to any organisation regardless of size or sector. Defines the core vocabulary, risk process lifecycle (establish context → assess → treat → monitor), and governance integration principles that underpin every commercial and open-source GRC risk register design. Not certifiable but widely referenced as the canonical risk framework alongside COSO ERM.

**ISO/IEC 27001:2022 — Information Security Management Systems**
- URL: https://www.iso.org/standard/27001
- The most widely certified information security standard globally, specifying requirements for establishing, implementing, maintaining, and improving an ISMS. Annex A provides a reference control set (93 controls) that forms the compliance target for the majority of mid-market GRC programmes. Every GRC platform targeting technology or SaaS companies ships ISO 27001 as a first-class framework.

**ISO/IEC 27701:2025 — Privacy Information Management Systems (PIMS)**
- URL: https://www.iso.org/standard/27701
- Updated in October 2025, this is the international standard for privacy information management. Now a standalone certifiable standard (previously required ISO 27001 co-certification), it maps directly to GDPR, CCPA, and other privacy laws via official annexes. Includes 31 controls for PII controllers and 18 for PII processors, plus 29 shared information security controls. Directly relevant to the policy management and data protection modules of any GRC platform.

**ISO/IEC 42001:2023 — Artificial Intelligence Management Systems (AIMS)**
- URL: https://www.iso.org/standard/42001
- The first international standard for AI management systems, specifying requirements for AI governance structures, risk assessment, data governance, technical documentation, and human oversight. Directly complements EU AI Act compliance obligations (August 2026 enforcement deadline for high-risk systems). Organisations certified to ISO 27001 can achieve ISO 42001 compliance up to 40% faster. A GRC platform serving technology buyers needs ISO 42001 as a first-class framework.

---

### W3C & IETF Standards

**RFC 7519 — JSON Web Token (JWT)**
- URL: https://datatracker.ietf.org/doc/html/rfc7519
- IETF standard for compact, self-contained means of representing claims between parties. Used as the token format for GRC API authentication (bearer tokens) in all modern platforms including Hyperproof, LogicGate, and CISO Assistant.

**RFC 6749 — OAuth 2.0 Authorization Framework**
- URL: https://datatracker.ietf.org/doc/html/rfc6749
- The de-facto standard for delegated API authorisation. All modern GRC platforms (Hyperproof, LogicGate, OneTrust, IBM OpenPages) use OAuth 2.0 with bearer tokens for their REST APIs. The authorization code grant type is used for user-facing integrations; client credentials grant type for server-to-server API access.

**RFC 7636 — PKCE for OAuth 2.0 (Proof Key for Code Exchange)**
- URL: https://datatracker.ietf.org/doc/html/rfc7636
- Extension to OAuth 2.0 that prevents authorisation code interception attacks. Required for any GRC API integration involving a public client (browser, mobile, CLI). Relevant to GRC API design for single-page evidence upload portals and auditor collaboration flows.

**SAML 2.0 — Security Assertion Markup Language**
- URL: https://www.oasis-open.org/standards/#samlv2.0
- OASIS standard for federated enterprise SSO. Remains the dominant SSO protocol in Fortune 500 and regulated industries (banking, healthcare, government) that use Active Directory Federation Services or Okta as their identity provider. Every enterprise GRC platform must support SAML 2.0 as the primary workforce authentication mechanism. CISO Assistant, Eramba Enterprise, and SimpleRisk Enterprise all support SAML.

**OpenID Connect 1.0 (OIDC)**
- URL: https://openid.net/developers/how-connect-works/
- Identity layer built on top of OAuth 2.0, standardising user identity claims via ID tokens (JWTs). Preferred for modern cloud-native GRC integrations (API-first, mobile, SPA). GRC platforms should support OIDC alongside SAML 2.0 to serve both legacy enterprise and modern cloud buyer segments.

---

### Data Model & API Specifications

**OpenAPI Specification 3.1.1 / 3.2.0**
- URL: https://spec.openapis.org/oas/v3.1.1.html | https://spec.openapis.org/oas/v3.2.0.html
- The industry-standard machine-readable format for describing REST APIs. Version 3.2.0 (September 2025) adds streaming media type support and OAuth 2.0 Device Authorization Flow. All commercially mature GRC platforms (IBM OpenPages, OneTrust, MetricStream) publish OpenAPI specifications for their REST APIs. An open-source GRC platform should ship an OpenAPI spec from day one to enable SDK generation and ecosystem tooling.

**NIST OSCAL — Open Security Controls Assessment Language**
- URL: https://pages.nist.gov/OSCAL/ | https://csrc.nist.gov/projects/open-security-controls-assessment-language
- NIST-developed XML, JSON, and YAML schemas for representing security controls, system security plans, assessment plans, assessment results, and plan of action and milestones (POA&Ms) in machine-readable formats. OSCAL dramatically reduces audit durations from months to minutes by enabling automated control assessment. The Center for Internet Security provides CIS Controls in OSCAL format, and Google Cloud published the first complete OSCAL package for a cloud provider (2024). An AI-native GRC platform that can import and export OSCAL-format control data would interoperate with the US federal compliance ecosystem (FedRAMP, FISMA) and reduce framework import effort. GitHub: https://github.com/usnistgov/OSCAL

**JSON Schema (draft-07 / 2020-12)**
- URL: https://json-schema.org/
- Standard for describing the structure and validation of JSON documents. Used extensively in GRC API request/response validation and OSCAL schema definitions. Relevant to data model design for risk register objects, control assessment records, and audit evidence payloads.

---

### Security & Authentication Standards

**OWASP Top 10 (2021)**
- URL: https://owasp.org/www-project-top-ten/
- The canonical application security risk reference used globally for software risk assessments. A GRC platform must support OWASP Top 10 as a framework (most do) and must itself be secure against OWASP risks (SQL injection, broken access control, injection in AI prompts). Particularly relevant for GRC platforms incorporating LLM features (prompt injection is an emergent OWASP concern for AI).

**NIST SP 800-53 Rev. 5 — Security and Privacy Controls for Information Systems**
- URL: https://csrc.nist.gov/publications/detail/sp/800-53/rev-5/final
- The comprehensive US federal catalog of 1,196 security and privacy controls across 20 families. Required for FISMA compliance and FedRAMP authorisation. GRC platforms targeting US federal or regulated enterprise buyers must support 800-53 as a framework. NIST publishes an official crosswalk between 800-53, CSF 2.0, and ISO 27001 via the Cybersecurity and Privacy Reference Tool (CPRT): https://csrc.nist.gov/projects/cprt

**PCI DSS 4.0.1 — Payment Card Industry Data Security Standard**
- URL: https://www.pcisecuritystandards.org/
- Full compliance became mandatory 1 April 2025, adding 51 new controls including mandatory API inventory management, automated web application security, and stronger supply chain security requirements. A GRC platform must carry PCI DSS 4.0.1 as a built-in framework and must support continuous API inventory and assessment workflows for organizations subject to the standard. PDF reference: https://www.middlebury.edu/sites/default/files/2025-01/PCI-DSS-v4_0_1.pdf

---

### Regulatory Frameworks

**EU AI Act (Regulation 2024/1689)**
- URL: https://eur-lex.europa.eu/legal-content/EN/TXT/?uri=CELEX:32024R1689
- The EU's binding AI regulation. Full enforcement of high-risk AI system obligations begins 2 August 2026. Requires providers of high-risk AI systems to maintain a documented risk management system, technical documentation, automatic logging, and human oversight mechanisms. GRC platforms must support EU AI Act as a compliance framework, and AI-native GRC platforms are themselves subject to its requirements. ISO 42001 is the natural certification framework for demonstrating compliance. Fines up to €35M or 7% of global turnover.

**DORA — Digital Operational Resilience Act (EU Regulation 2022/2554)**
- URL: https://finance.ec.europa.eu/regulation-and-supervision/financial-services-legislation/implementing-and-delegated-acts/digital-operational-resilience-regulation_en
- Fully applicable across EU financial entities from 17 January 2025. Mandates ICT risk management frameworks, incident classification and reporting, digital operational resilience testing (TLPT), and third-party ICT risk management (including critical third-party providers). Detailed RTS/ITS published by EBA, EIOPA, and ESMA in the EU Official Journal between June 2024 and April 2025. A GRC platform targeting BFSI buyers must carry DORA as a first-class framework with incident reporting workflow templates aligned to the ITS reporting templates. All RTS/ITS overview: https://www.regulation-dora.eu/rts

**COSO ERM 2017 — Enterprise Risk Management: Integrating with Strategy and Performance**
- URL: https://www.coso.org/erm-framework
- The primary conceptual model for enterprise risk management, published by the Committee of Sponsoring Organizations of the Treadway Commission. Defines five components (Governance and Culture; Strategy and Objective Setting; Performance; Review and Revision; Information, Communication and Reporting) and 20 underlying principles. Provides the structural vocabulary for risk appetite frameworks, risk taxonomy design, and board-level risk reporting in commercial GRC platforms. Not a standard with a machine-readable schema; used as a design reference for risk data models.

**IIA Global Internal Audit Standards 2024 (IPPF)**
- URL: https://www.theiia.org/en/standards/2024-standards/global-internal-audit-standards/
- The mandatory professional standards for internal audit functions globally, organized into five domains: Purpose of Internal Auditing; Ethics and Professionalism; Governing the Internal Audit Function; Managing the Internal Audit Function; and Performing Internal Audit Services. Supersedes earlier versions. GRC platforms with audit management modules must align audit planning, fieldwork, evidence, finding, and corrective action workflows to IPPF terminology and lifecycle stages. The IIA's Three Lines Model (2020) defines the governance structure: https://www.theiia.org/en/content/position-papers/2020/the-iias-three-lines-model-an-update-of-the-three-lines-of-defense/

**GDPR (EU Regulation 2016/679) and ISO/IEC 27701:2025**
- URL: https://eur-lex.europa.eu/legal-content/EN/TXT/?uri=CELEX:32016R0679
- GDPR requires documented Data Protection Impact Assessments (DPIAs), Records of Processing Activities (RoPA), and breach response procedures — all core GRC functions. ISO 27701:2025 Annex D provides the official mapping between privacy controls and GDPR articles, making ISO 27701 the preferred certification framework for GDPR programmes. GRC platforms must support GDPR-aligned workflows including DPIA templates, RoPA management, and automated breach notification timelines.

**NIST AI RMF 1.0 (AI 100-1) and Generative AI Profile (AI 600-1)**
- URL: https://www.nist.gov/itl/ai-risk-management-framework | https://nvlpubs.nist.gov/nistpubs/ai/NIST.AI.600-1.pdf
- The NIST AI Risk Management Framework organises AI risk through four functions: Govern, Map, Measure, Manage. The Generative AI Profile (July 2024) extends the framework to address GenAI-specific risks. Increasingly referenced by US federal regulators as the baseline for AI governance. A GRC platform serving technology buyers must support both AI RMF 1.0 and the GenAI Profile as frameworks, particularly as buyers add AI governance to existing GRC programmes.

**BCBS 239 — Principles for Effective Risk Data Aggregation and Risk Reporting**
- URL: https://www.bis.org/publ/bcbs239.htm
- Basel Committee on Banking Supervision's 14 principles for improving risk data management and reporting in banks, introduced post-2008 financial crisis. Directly drives demand for structured risk data models, automated data lineage, and board-level risk reporting in BFSI GRC implementations. Updated principles were published in April 2025. GRC platforms targeting banking buyers should align their risk data architecture and reporting capabilities to BCBS 239 principles.

---

## Similar Products — Developer Documentation & APIs

### MetricStream

- **Description:** Enterprise GRC platform with "Connected GRC" unified data model covering enterprise risk, operational risk, compliance, audit, and third-party risk. Leading market position in BFSI and healthcare.
- **API Documentation:** https://www.metricstream.com/api-developer-portal.html
- **Developer Portal:** https://www.metricstream.com/developer-portal.html
- **Standards:** OpenAPI-compliant REST APIs for inbound and outbound integrations; Content Integration Service for bulk data pipelines; in-built scheduler for bulk requests.
- **Authentication:** Token-based (OAuth 2.0); enterprise SSO via SAML 2.0.
- **SDK/Libraries:** No public SDK; REST API + pre-configured connectors for SAP, Oracle, Workday, Jira.
- **Notes:** API access requires enterprise contract; no public sandbox.

---

### ServiceNow GRC

- **Description:** GRC module on the Now Platform, integrating IT risk, vendor risk, compliance, and operational resilience. Strongest IT/cyber GRC positioning due to CMDB and ITSM integration.
- **API Documentation:** https://www.servicenow.com/docs/bundle/yokohama-governance-risk-compliance/page/product/grc-common/reference/grc-integrations.html
- **REST API Reference:** https://servicenow.com/docs/bundle/zurich-api-reference/page/build/applications/concept/api-rest.html
- **Standards:** REST/JSON; scripted REST API for Policy and Compliance Integrator; Integration Hub with 700+ spokes.
- **Authentication:** OAuth 2.0 (recommended); Basic Auth (legacy); API Key.
- **SDK/Libraries:** Flow Designer, Spoke Generator, MID Server for on-premise. Pre-built spokes for Jira, SAP, Microsoft 365, Okta.
- **Notes:** Full API coverage of risk registers, controls, policies, assessments, audit engagements, and vendor risk workflows; access governed by Now Platform licence level.

---

### OneTrust

- **Description:** Privacy, GRC, and ESG platform; leading privacy + GRC convergence with 200+ native integrations. Modular cloud offering across Trust Intelligence Platform, Privacy & Data Governance Cloud, GRC & Security Assurance Cloud, and ESG Cloud.
- **API Documentation:** https://developer.onetrust.com/onetrust/reference/onetrust-api-reference
- **Developer Portal:** https://developer.onetrust.com/
- **Quick Start Guide:** https://developer.onetrust.com/onetrust/reference/quick-start-guide
- **Standards:** REST/JSON; OpenAPI specification published via developer portal; APIs organised by Cloud module.
- **Authentication:** OAuth 2.0 bearer token; access token generation documented in developer portal.
- **SDK/Libraries:** REST API only; no official SDK packages; Zapier and webhook support for no-code integrations.
- **Notes:** Winner of DevPortal Awards 2024; one of the most accessible GRC developer portals publicly available.

---

### LogicGate Risk Cloud

- **Description:** Modern cloud-native GRC platform with no-code workflow builder; strong mid-market focus for ERM, third-party risk, and compliance management.
- **API Documentation:** https://docs.logicgate.com/v2/index.html
- **Developer Portal:** https://www.logicgate.com/developer/
- **Getting Started:** https://www.logicgate.com/developer/risk-cloud-api-getting-started/
- **Standards:** RESTful JSON API (OpenAPI-based); v2 API is the current recommended version (v2026.3.1 as of May 2026); Postman Workspace published for exploration.
- **Authentication:** OAuth 2.0 bearer token; rate limit 10 requests/second.
- **SDK/Libraries:** No official SDK; REST API + Zapier integration for 500+ applications; Jira, Slack, ServiceNow native integrations.
- **Notes:** API versioned by release date (v2025.x, v2026.x); release notes and changelog published at https://www.logicgate.com/release-notes/

---

### Hyperproof

- **Description:** Continuous compliance operations platform with automated evidence collection via Hypersyncs; strong multi-framework overlap mapping. Targets mid-market compliance operations teams.
- **API Documentation:** https://developer.hyperproof.app/hyperproof-api
- **Developer Portal:** https://developer.hyperproof.app/
- **Hypersync SDK:** https://developer.hyperproof.app/hypersync-sdk
- **Standards:** REST/JSON; OpenAPI-documented endpoints; Hypersync SDK for building custom evidence collection integrations.
- **Authentication:** OAuth 2.0 with authorization code grant type (user-facing) and client credentials grant type (server-to-server).
- **SDK/Libraries:** Hypersync SDK (TypeScript/JavaScript) for building custom data connectors; 70+ native platform integrations.
- **Notes:** Controls API (GET/POST), Programs API, and Proof upload API are the primary endpoints for integration. Rate limit applies; contact support for higher limits.

---

### IBM OpenPages

- **Description:** Enterprise GRC platform on IBM Cloud; strongest for financial services regulatory requirements (Basel IV, DORA, IFRS). AI-assisted risk insights via IBM watsonx.
- **API Documentation (Cloud):** https://cloud.ibm.com/apidocs/openpages
- **IBM Docs (REST API V2):** https://www.ibm.com/docs/en/openpages/9.0.0?topic=guide-rest-api-v2
- **Developer Guide:** https://www.ibm.com/docs/en/openpages/9.0.0?topic=developer-guide
- **OpenAPI Spec (v9.1):** https://github.com/vperrinfr/OpenPages_MCP/blob/main/IBM%20OpenPages%20REST%20API%20V2-9.1.json
- **Standards:** REST API V2 (recommended for new implementations); OpenAPI specification published; data-centric API organised around resource URIs.
- **Authentication:** IBM Cloud IAM tokens; API Key for on-premise deployments.
- **SDK/Libraries:** Java API available alongside REST API; IBM Cloud Pak for Data connectors for analytics.
- **Notes:** REST API V2 is a significant improvement over V1; full coverage of risks, controls, assessments, policy objects, and regulatory change records. On-premise OpenAPI spec differs from cloud variant.

---

### RSA Archer (Archer IRM)

- **Description:** Legacy enterprise GRC platform with long-standing BFSI and US federal government presence. Broad risk and compliance modules including business continuity, third-party risk, and regulatory compliance.
- **API Documentation:** https://community.rsa.com/t5/archer-platform-documentation/using-the-rest-api/ta-p/528882
- **Community Reference (6.2):** https://community.rsa.com/yfcdo34327/attachments/yfcdo34327/archer-platform-documentation/1306/1/RSA%20Archer%206.2%20Patch%201%20REST%20API%20Reference%20Guide.pdf
- **Standards:** REST API (JSON) and legacy SOAP API both supported; token-based authentication.
- **Authentication:** Token-based (username/password exchange for bearer token); not OAuth 2.0 compliant.
- **SDK/Libraries:** Community Python library `rsa-archer` on PyPI; Palo Alto Cortex XSOAR integration (https://xsoar.pan.dev/docs/reference/integrations/rsa-archer-v2).
- **Notes:** API documentation is largely community-maintained; access to detailed technical docs typically requires an Archer licence and community login. Older authentication model (pre-OAuth) is a known integration pain point.

---

### AuditBoard

- **Description:** Connected risk and audit management platform for mid-enterprise; focuses on SOX, operational audit, and risk management with AI-powered insights. Publicly traded via SPAC (2023).
- **Developer Portal:** https://developer.hyperproof.app/ (Hyperproof; AuditBoard portal requires account)
- **Postman Collection:** https://www.postman.com/sahobbs/workspace/auditboard-api/collection/29276823-a5cae1d0-bb35-4191-a405-a9b9daf9c92d
- **Standards:** REST API with JSON payloads; bearer token authentication.
- **Authentication:** Two-step process: API Key + Username/Password exchange for bearer token. Contact account representative to provision API Service account.
- **SDK/Libraries:** No public SDK; REST API + Microsoft Teams native integration; HRIS integrations (ADP, Workday, TriNet) in beta.
- **Notes:** Rate limit 20 requests/second; higher limits available on request. API covers clients, engagements, documents, requests, users, and reports. Full developer portal behind authentication wall.

---

### CISO Assistant (Open Source)

- **Description:** Open-source GRC platform (Apache 2.0) with 130+ built-in compliance frameworks and automatic cross-framework control mapping. Most actively developed OSS GRC project as of 2026.
- **API Documentation:** https://intuitem.gitbook.io/ciso-assistant/integration/api-usage
- **GitHub:** https://github.com/intuitem/ciso-assistant-community
- **Standards:** REST API with Django REST Framework and Swagger/OpenAPI documentation; Docker-based deployment.
- **Authentication:** Token-based API authentication; SSO via SAML and OIDC.
- **SDK/Libraries:** No SDK; REST API accessible via Swagger UI at `/api/schema/swagger-ui/` on self-hosted instance.
- **Notes:** Swagger documentation is self-hosted alongside the application; API enables programmatic access to frameworks, risks, controls, assessments, and evidence. Actively maintained; LLM integration on roadmap.

---

### Eramba (Open Source / Enterprise)

- **Description:** Mature open-source GRC platform (GPL v3 Community / proprietary Enterprise) started in 2007; covers risk, compliance, policy, audit, and incident management.
- **API Documentation:** https://www.eramba.org/learning/courses/29/episodes/271 (login required for learning portal)
- **Standards:** REST API available primarily in Enterprise Edition; LDAP/Active Directory integration.
- **Authentication:** Session-based for UI; REST API token authentication for Enterprise.
- **SDK/Libraries:** No public SDK; community forum for integration guidance.
- **Notes:** Community Edition REST API is limited; full API access requires Enterprise licence. API documentation is not publicly available without an Eramba account — a significant barrier to ecosystem development compared to CISO Assistant.

---

## Notes

**Emerging: MCP (Model Context Protocol) for GRC**

The Model Context Protocol (MCP) is gaining adoption as the standard interface for connecting AI agents to external data sources and tools. One proof-of-concept MCP server for IBM OpenPages exists at https://github.com/vperrinfr/OpenPages_MCP, exposing OpenPages risk and control data to LLM-based agents. An AI-native GRC platform that ships an MCP server from day one would enable Claude, GPT-4, and other LLM agents to query and update risk registers, generate risk narratives, and perform compliance gap analysis — a significant differentiator over existing OSS tools.

**OSCAL Adoption Gap**

Despite NIST publishing OSCAL in 2020 and actively maintaining the schemas, adoption among commercial GRC platforms remains low. None of the commercial platforms surveyed (MetricStream, ServiceNow, Archer, OneTrust) natively import or export OSCAL-format data. An open-source GRC platform with native OSCAL support would be uniquely positioned for US federal and FedRAMP-adjacent markets where OSCAL adoption is mandated.

**Authentication Modernisation**

RSA Archer's pre-OAuth token model and AuditBoard's credential-exchange authentication represent legacy patterns that increase integration friction. All new GRC platforms should implement full OAuth 2.0 + OIDC from day one, with SAML 2.0 for enterprise SSO and client credentials grant for server-to-server API access.

**Framework Data Model Convergence**

The OSCAL Catalog and Profile layers, NIST CPRT crosswalks, and CIS Controls OSCAL repository together provide a machine-readable ecosystem of 25+ framework mappings that a GRC platform can ingest. Building on OSCAL rather than maintaining proprietary framework definitions reduces ongoing content maintenance burden and enables automatic cross-framework control mapping — the most valued differentiating feature in the market.

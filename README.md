# Governance, Risk & Compliance (GRC)

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, open-source GRC platform unifying risk register, policy management, audit management, and regulatory tracking for organisations priced out of enterprise suites.

Modern GRC programmes still rely on six-figure enterprise platforms (MetricStream, RSA Archer, IBM OpenPages, ServiceNow GRC) or open-source tools (Eramba, SimpleRisk, CISO Assistant) that lack AI capability and continuous control monitoring. This project targets risk, compliance, and audit teams at mid-market organisations who need enterprise-grade depth — risk register, policy lifecycle, audit management, regulatory change tracking — without enterprise-grade pricing or implementation complexity.

---

## Why Governance, Risk & Compliance?

- **Enterprise GRC pricing locks out the mid-market.** MetricStream, RSA Archer, and IBM OpenPages routinely run $150K–$500K+/yr with 6–18 month implementations, leaving organisations between $5M–$200M revenue without a viable path.
- **Existing OSS tools lag on AI and automation.** Eramba and SimpleRisk are mature but have minimal AI capability, no continuous control monitoring, and limited automated evidence collection. CISO Assistant has the widest framework library but is narrowly focused on IT security compliance.
- **Regulatory change tracking is the most painful manual workflow in GRC.** Tracking GDPR amendments, SEC rules, DORA implementing standards, and EU AI Act timelines — and mapping their impact to internal controls — is currently handled either via expensive intelligence subscriptions or manual monitoring.
- **Point-in-time control assessment is structurally outdated.** Most GRC platforms still run periodic (annual/quarterly) control attestations. AI enables continuous control testing — a transformation no OSS tool has achieved.
- **Policy-to-control alignment is error-prone.** Policies routinely reference controls that do not exist, and controls operate without governing policy. Manual reconciliation is the default; LLM-assisted gap analysis is not.

---

## Key Features

### Risk Management

- Risk register with configurable inherent and residual risk scoring
- Risk treatment plans with periodic review workflows
- Many-to-many linking between risks, controls, and compliance frameworks
- Risk appetite tracking and quantitative risk modelling (Monte Carlo on the backlog)

### Compliance & Framework Management

- Built-in framework packs for ISO 27001, SOC 2, NIST CSF 2.0, GDPR, and EU AI Act at minimum
- Multi-framework overlap mapping showing one control satisfying multiple framework requirements
- Cross-framework control library with gap analysis against selected standards
- Coverage roadmap including DORA, NIS2, PCI DSS 4.0, HIPAA, NIST 800-53, CMMC

### Policy & Audit Management

- Policy lifecycle management with version control, approval workflows, and user attestation
- Internal audit management covering annual planning, fieldwork, findings, and corrective action plans (CAPs)
- Audit trail and evidence repository for regulatory examination readiness
- External auditor collaboration portal with evidence request workflow (backlog)

### Third-Party & Incident Management

- Third-party risk management with questionnaire builder and vendor portal
- Incident and issue management with root cause and remediation tracking
- Vendor assessment workflows and continuous monitoring hooks

### Platform & Access

- Role-based access for auditor, risk owner, compliance manager, CRO, CCO, CISO, and admin personas
- REST API and SSO (SAML / OIDC) from day one
- Webhook support for notifications and downstream automation
- Self-hostable with Docker-based deployment

---

## AI-Native Advantage

The platform's primary differentiation is AI applied to the workflows that hurt GRC teams most. An LLM layer continuously ingests official regulatory publications, classifies applicability by industry and geography, and maps changes onto the control register — replacing expensive regulatory intelligence subscriptions and manual monitoring. AI agents query source systems (cloud platforms, identity providers, ticketing) to sample evidence and assert compliance continuously, shifting GRC from periodic attestation to continuous assurance. Additional AI assistance covers risk narrative drafting, likelihood/impact scoring suggestions from industry benchmarks, policy-to-control gap analysis, and IIA-standard audit finding generation. PwC's published case study found GenAI tools identified regulatory changes with 90% accuracy — a benchmark this project aims to meet in open source.

---

## Tech Stack & Deployment

The project targets self-hosted deployment via Docker (matching the easiest-in-class CISO Assistant model) with an optional managed SaaS tier. Automated evidence collection is delivered through API connectors to common cloud platforms (AWS, GCP, GitHub, Okta, Jira) — a generic technique with no known patent claims, and the model proven by Hyperproof's Hypersyncs. Standards alignment includes COSO ERM (2017), ISO 31000:2018, ISO 27001:2022, NIST CSF 2.0, SOC 2, PCI DSS 4.0, EU AI Act, DORA, Basel III/IV, and the IIA Three Lines Model. Authentication via SAML and OIDC; programmatic access via REST API.

---

## Market Context

GRC software market sizing varies by scope: Mordor Intelligence estimates $23.32B in 2026 growing to $39.01B by 2031 at 10.84% CAGR, while Grand View's broader enterprise GRC definition reaches $82.93B in 2026. North America holds ~39.5% of revenue; APAC is the fastest-growing region at 15.1% CAGR. Pricing tiers range from $150K–$500K+/yr for MetricStream / RSA Archer / IBM OpenPages, $50K–$300K+/yr for ServiceNow GRC, $12K–$100K/yr for OneTrust / LogicGate / Hyperproof, down to $0 licence for Eramba and SimpleRisk. Primary buyers are the Chief Risk Officer, Chief Compliance Officer, Chief Information Security Officer, Internal Audit Director, and Board / Audit Committee.

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

Licence to be determined. Among reference OSS GRC tools, Eramba Community Edition uses GPL v3, SimpleRisk core uses Mozilla Public Licence 2.0, and CISO Assistant uses Apache 2.0; Apache 2.0 is the most permissive and best suited for a commercial open-source strategy with a hosted SaaS tier.

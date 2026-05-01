# Governance, Risk & Compliance (GRC)

> Candidate #60 · Researched: 2026-05-01

## Existing Products and Software Packages

| Tool | Type | Description | Pricing | Strengths / Weaknesses |
|------|------|-------------|---------|------------------------|
| **MetricStream** | Commercial | "Connected GRC" enterprise platform covering enterprise risk, operational risk, compliance, audit management, and third-party risk with a unified data model. | Enterprise custom; typically $150K–$500K+/yr | + Unified risk data model across domains; + Strong audit management; + Broad regulatory framework coverage. − Very expensive; − Complex implementation; − UI criticized as dated |
| **ServiceNow GRC** | Commercial | GRC module on the Now Platform, integrating with IT risk, vendor risk, and operational resilience. Strong for IT-centric GRC programs. | Custom (part of Now Platform); estimated $50K–$300K+/yr for GRC modules | + Deep ITSM/IT risk integration; + Strong workflow automation; + Single platform for IT + GRC. − Expensive; − Less strong for non-IT risk domains (ORM, ERM); − Requires ServiceNow platform investment |
| **RSA Archer** | Commercial | Legacy enterprise GRC platform with broad risk and compliance modules. Long-standing market presence in BFSI and government. | Custom; typically $100K–$400K+/yr | + Broad risk domain coverage; + Strong in regulated industries. − Outdated interface (widely criticized); − Complex workflows; − Slow product innovation cycle |
| **IBM OpenPages** | Commercial | Enterprise GRC platform on IBM Cloud for financial risk, regulatory compliance, ESG, and operational risk management. | Custom enterprise pricing | + Strong financial services regulatory coverage (Basel, DORA, IFRS); + AI-assisted risk insights via Watson. − IBM ecosystem dependency; − Expensive; − Complex to implement |
| **OneTrust GRC** | Commercial | Unified privacy, security, and compliance management platform with risk register, policy management, and third-party risk. Strong privacy/data focus. | Custom; modular pricing; estimated $30K–$200K+/yr | + Leading privacy + GRC convergence; + Modern UI; + Strong third-party risk module. − GRC breadth narrower than MetricStream/Archer; − Pricing increases substantially as modules are added |
| **LogicGate Risk Cloud** | Commercial | Modern cloud-native GRC platform with configurable risk register, audit, policy, and compliance modules. Strong mid-market focus. | From ~$30K/yr; Enterprise custom | + Modern UX; + Flexible no-code configuration; + Fast implementation. − Less deep than enterprise competitors for complex ERM; − Smaller ecosystem |
| **Hyperproof** | Commercial | Compliance operations platform focused on continuous compliance automation, evidence collection, and multi-framework mapping (SOC 2, ISO 27001, NIST). | From $12K/mo (Hyperproof Pro); Enterprise custom | + Strong compliance framework overlapping; + Continuous control monitoring. − Narrower risk management depth; − IT/security compliance focus rather than enterprise-wide GRC |
| **Eramba** | Open Source / Commercial | Mature open-source GRC platform with risk assessment, policy management, internal audit, incident management, and compliance packages (ISO 27001, GDPR, PCI DSS). | Community: free; Enterprise: affordable (contact for pricing) | + True open source; + Mature and battle-tested; + No user/data limits. − Community edition support limited; − UI less polished than commercial tools; − Limited AI capabilities |
| **SimpleRisk** | Open Source | Risk-focused GRC tool with risk register, treatment planning, audit management, and governance modules. Easy to deploy. | Free (Community); Enterprise from ~$500/mo | + Simple and fast deployment; + Good risk register UX; + Open source core. − Narrower than Eramba; − Less framework breadth; − Limited audit management depth |
| **CISO Assistant** | Open Source | Open-source security and compliance assistant with multi-framework support (150+ frameworks including SOC 2, NIST CSF, ISO 27001, AI Act). | Free (self-hosted); SaaS option | + Widest open-source framework library; + Active development; + AI Act compliance support. − Security/IT compliance focus; − Limited enterprise risk management depth; − Newer (less proven at scale) |

## Relevant Industry Standards or Protocols

- **COSO ERM (2017)** — Committee of Sponsoring Organizations Enterprise Risk Management framework; the primary conceptual model for enterprise risk registers and risk appetite frameworks.
- **ISO 31000:2018 (Risk Management)** — International standard for risk management principles and guidelines; widely referenced in GRC platform risk assessment design.
- **ISO 27001:2022 (Information Security Management)** — Most widely certified information security standard; central compliance target for GRC platforms serving technology companies.
- **NIST Cybersecurity Framework (CSF) 2.0** — US NIST framework for cybersecurity risk management; heavily referenced in GRC tools for IT risk programs.
- **SOC 2 (AICPA Trust Services Criteria)** — US audit standard for SaaS and cloud service providers; a dominant compliance target in GRC platforms targeting technology companies.
- **PCI DSS 4.0** — Payment card industry standard; required for organizations processing card payments; major compliance workstream in GRC platforms.
- **EU AI Act (2024)** — First binding AI regulation globally; requires risk management systems, technical documentation, and conformity assessments for high-risk AI systems. Full enforcement of high-risk provisions begins August 2026.
- **DORA (Digital Operational Resilience Act, EU 2025)** — EU regulation for financial sector digital resilience; ICT risk management, incident reporting, and third-party risk requirements directly served by GRC platforms.
- **Basel III/IV** — International banking capital adequacy and operational risk framework; drives demand for operational risk GRC in BFSI.
- **GDPR / CCPA / global privacy regulations** — Data protection regulations requiring documented risk assessments (DPIAs), breach response procedures, and policy management — all core GRC functions.
- **Three Lines Model (IIA 2020)** — Internal audit governance model defining roles across management, risk functions, and internal audit; influences GRC platform audit management design.

## Available Research Materials

1. Business Research Insights (2026). *Governance, Risk Management and Compliance (GRC) Software Market Size & Growth 2026–2035.* https://www.businessresearchinsights.com/market-reports/governance-risk-management-and-compliance-grc-software-market-106704 — Industry report; market at $1.61B (narrower software-only definition) growing to $2.61B by 2035 at CAGR 5.5%.

2. Mordor Intelligence (2026). *GRC Software Market Size, Share & 2031 Growth Trends Report.* https://www.mordorintelligence.com/industry-reports/governance-risk-and-compliance-software-market — Industry report; market at $23.32B in 2026, growing to $39.01B by 2031 at CAGR 10.84%.

3. Grand View Research (2026). *Enterprise Governance, Risk & Compliance Market, 2033.* https://www.grandviewresearch.com/industry-analysis/enterprise-governance-risk-compliance-egrc-market — Industry report; EGRC market estimated at $82.93B in 2026 (broadest definition including services and adjacent software).

4. MetricStream (2025). *AI in GRC: Trends, Opportunities and Challenges for 2025.* https://www.metricstream.com/blog/ai-in-grc-trends-opportunities-challenges-2025.html — Industry white paper; covers GenAI use cases in risk assessment, control testing, and audit.

5. Hyperproof (2025). *The Year of Global AI and Cybersecurity Regulations: 7 GRC Predictions for 2025.* https://hyperproof.io/resource/7-grc-predictions-for-2025/ — Industry analysis; covers DORA, EU AI Act, and SEC cybersecurity disclosure impacts on GRC demand.

6. InfoSecFlow (2025). *Open-Source GRC Tools Compared — CISO Assistant vs Eramba vs Commercial Platforms.* https://infosecflow.com/blog/open-source-grc-comparison/ — Practitioner comparison; directly relevant to OSS GRC landscape.

7. Censinet (2025). *AI-Powered GRC: How Leading Organizations Are Automating Compliance in the Age of Increasing Regulation.* https://censinet.com/perspectives/ai-powered-grc-how-leading-organizations-are-automating-compliance-in-the-age-of-increasing-regulation — Industry article; Gartner prediction cited that 50%+ of enterprises will use AI for continuous compliance checks by 2025.

8. PwC (2025). Cited in GRC strategy research: GenAI tools identified regulatory changes with 90% accuracy in a PwC case study. Referenced in: https://community.trustcloud.ai/article/artificial-intelligence-the-role-in-enhancing-grc-strategies-in-2024/ — Industry case study; useful quantitative benchmark for AI regulatory tracking claims.

## Market Research

**Market Size:** GRC market sizing varies widely by research firm scope. A reasonable mid-range estimate for GRC software (excluding services) is $23.32B in 2026 (Mordor Intelligence) growing to $39.01B by 2031 at CAGR 10.84%. The broader enterprise GRC market including related services reaches $82.93B+ (Grand View). North America holds ~39.5% of revenue; APAC growing fastest at 15.1% CAGR.

**Pricing Landscape:**

| Segment | Representative Tools | Typical Cost |
|---------|---------------------|-------------|
| Enterprise GRC suites | MetricStream, RSA Archer, IBM OpenPages | $150K–$500K+/yr |
| Platform-native GRC | ServiceNow GRC, Salesforce Shield | $50K–$300K+/yr |
| Mid-market / modern cloud | OneTrust, LogicGate, Hyperproof | $12K–$100K/yr |
| SMB compliance tools | Sprinto, Drata, Secureframe | $10K–$50K/yr |
| Open source (self-hosted) | Eramba, SimpleRisk, CISO Assistant | $0 license + infra |

**Key Buyer Personas:**
- Chief Risk Officer (CRO) — enterprise risk aggregation, board-level risk reporting, risk appetite framework
- Chief Compliance Officer (CCO) — regulatory change tracking, policy lifecycle management, audit management
- Chief Information Security Officer (CISO) — IT/cyber risk, third-party risk, SOC 2 / ISO 27001 compliance
- Internal Audit Director — audit planning, fieldwork management, findings tracking, CAP management
- Board / Audit Committee — governance reporting, risk dashboard, regulatory exposure visibility

**Notable Acquisitions & Funding:**
- OneTrust raised $300M+ at $5.3B valuation (2021); expanded aggressively from privacy into GRC and third-party risk
- LogicGate raised $113M Series C (2022) at ~$1B valuation; mid-market GRC consolidation play
- Diligent acquired Galvanize (formerly ACL/HighBond) in 2021 for ~$1B; combining board governance with GRC/audit
- IBM OpenPages remains within IBM GRC portfolio; limited acquisition activity recently
- AuditBoard went public via SPAC (2023); focuses on connected risk and audit management for mid-enterprise
- Sprinto raised $30M Series B (2023); compliance automation for SaaS companies targeting SOC 2 / ISO 27001
- Hyperproof raised $40M+ Series B (2023); continuous compliance for mid-market

## AI-Native Opportunity

- **Automated regulatory change monitoring and impact assessment:** The single most painful GRC task is tracking regulatory changes (GDPR amendments, new SEC rules, DORA implementing standards, EU AI Act timelines) and mapping their impact to internal controls. Current tools either rely on expensive regulatory intelligence subscriptions or manual monitoring. An AI-native system that continuously ingests regulatory sources, identifies changes relevant to the organization's industry and geography, and automatically maps impacts to the risk register and control library would address the most acute pain point in the GRC market.
- **Continuous control testing replacing periodic audit cycles:** Most GRC platforms perform point-in-time control assessments (annual, quarterly). AI can enable continuous automated control testing — querying systems, sampling evidence, running assertions — and surface failing controls in real time before they become audit findings. This shifts GRC from periodic attestation to continuous assurance, a transformation that open-source tools have not yet achieved.
- **Intelligent risk assessment and scoring:** Building a risk register today requires manual expert workshops and subjective scoring. An AI-native GRC tool that assists risk owners with structured risk identification (by analyzing process descriptions, contracts, and incident data), suggests likelihood and impact scores based on industry benchmarks and historical data, and flags inconsistencies in the register would dramatically reduce the effort and improve risk register quality.
- **Policy-to-control gap analysis:** Organizations often have policies and controls that are misaligned — policies reference controls that don't exist, or controls exist with no governing policy. An AI layer that reads the policy library and control library, identifies gaps, duplicates, and conflicts, and proposes remediation actions would automate a currently manual and error-prone process.
- **Open-source gap:** Eramba and SimpleRisk provide solid GRC foundations but have minimal AI capability, limited regulatory change tracking, and no continuous control monitoring. CISO Assistant has excellent framework breadth but is narrowly focused on IT security compliance. An AI-native OSS GRC platform combining risk register, policy management, audit management, regulatory intelligence, and continuous control monitoring — built on a modern stack with LLM-powered features — would be genuinely differentiated and fill a real market gap, particularly for mid-market organizations priced out of MetricStream or ServiceNow GRC.

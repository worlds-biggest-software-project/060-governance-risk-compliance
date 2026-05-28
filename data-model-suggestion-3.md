# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Governance, Risk & Compliance (GRC) · Created: 2026-05-12

## Philosophy

This model uses a pragmatic hybrid approach: core structural fields that are queried, filtered, and aggregated frequently are stored as typed relational columns, while variable, jurisdiction-specific, framework-specific, and custom fields are stored in PostgreSQL JSONB columns. The JSONB columns are indexed using GIN indexes, enabling efficient containment queries without the rigidity of adding columns for every variation.

This approach is directly inspired by how modern mid-market GRC platforms like LogicGate Risk Cloud and Hyperproof handle the fundamental tension in GRC data: the core domain model (risks have likelihood/impact, controls have type/effectiveness, policies have status/version) is universal, but the details vary enormously. A financial services firm tracking Basel IV operational risk needs different risk fields than a healthcare company tracking HIPAA PHI exposure. A company operating under EU AI Act needs AI-specific risk metadata that a company focused on PCI DSS does not. In a purely normalised model, these variations require either (a) hundreds of nullable columns that are mostly empty, or (b) an EAV (Entity-Attribute-Value) anti-pattern. JSONB provides a cleaner third path: structured extensibility with type safety and indexing at the database level.

This is also the fastest path to an MVP. The reduced table count (~25 tables vs. ~37+ for fully normalised) means fewer migrations, fewer JOINs, and faster iteration. Framework-specific metadata, jurisdiction-specific risk fields, and custom assessment criteria can evolve through JSONB schema changes without ALTER TABLE operations. This is the approach most likely to reach production quickly while retaining the ability to normalise hot paths later as query patterns become clear.

**Best for:** Rapid MVP development, multi-jurisdiction deployments where risk/control metadata varies by region, teams that need to ship quickly and iterate on the schema, and organisations that span multiple compliance domains with divergent field requirements.

**Trade-offs:**
- (+) Dramatically fewer tables — simpler schema, faster migrations, easier to understand
- (+) Framework-specific and jurisdiction-specific fields evolve without ALTER TABLE
- (+) JSONB GIN indexes provide efficient containment and existence queries
- (+) Custom fields per tenant are natural — just different JSONB keys
- (+) Faster time to MVP — less schema design upfront
- (+) Schema-on-read for analytics and reporting flexibility
- (-) JSONB fields lack database-level referential integrity — application must validate
- (-) Complex JSONB queries can be slower than typed column queries for large datasets
- (-) JSONB schema documentation must be maintained separately (not self-documenting like columns)
- (-) ORM support for JSONB varies — Django JSONField is good; some ORMs are weaker
- (-) Reporting tools may struggle with nested JSONB structures
- (-) Risk of "JSONB everything" — discipline required to keep core fields relational

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO 31000:2018 | Core risk fields (likelihood, impact, treatment_type) are relational columns; risk_source, consequence detail in JSONB |
| COSO ERM 2017 | Risk taxonomy as relational hierarchy; COSO component-specific fields in risk `metadata` JSONB |
| ISO/IEC 27001:2022 | Framework requirements stored relationally; Annex A control-specific guidance in `extended_fields` JSONB |
| NIST OSCAL | OSCAL identifiers as relational fields; OSCAL-specific properties and links in JSONB |
| ISO/IEC 42001:2023 | AI risk-specific fields (model_type, training_data_source, bias_assessment) in risk `metadata` JSONB |
| DORA | ICT incident classification fields in incident `metadata` JSONB, following DORA Article 19 taxonomy |
| EU AI Act | AI system risk classification and conformity assessment fields in JSONB |
| PCI DSS 4.0 | PCI-specific control testing requirements in control `extended_fields` JSONB |
| ISO 3166 | Jurisdiction as relational CHAR(2) column on core entities |

---

## Multi-Tenancy & Identity

```sql
-- ============================================================
-- TENANT & IDENTITY (relational — queried constantly)
-- ============================================================

CREATE TABLE organization (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    industry        VARCHAR(100),
    jurisdiction    CHAR(2),                -- ISO 3166-1 alpha-2
    settings        JSONB NOT NULL DEFAULT '{}',
    -- settings example: {
    --   "risk_matrix_type": "5x5",
    --   "fiscal_year_start_month": 1,
    --   "default_review_frequency": "quarterly",
    --   "enabled_modules": ["risk", "compliance", "audit", "vendor", "incident"],
    --   "custom_risk_fields": [
    --     {"key": "velocity", "label": "Risk Velocity", "type": "select",
    --      "options": ["rapid", "moderate", "slow"]},
    --     {"key": "data_classification", "label": "Data Classification", "type": "select",
    --      "options": ["public", "internal", "confidential", "restricted"]}
    --   ]
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organization(id),
    email           VARCHAR(320) NOT NULL,
    display_name    VARCHAR(255) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    roles           JSONB NOT NULL DEFAULT '[]',
    -- roles example: [
    --   {"role": "risk_owner", "scope": {"type": "business_unit", "id": "uuid"}},
    --   {"role": "compliance_manager", "scope": null},
    --   {"role": "admin", "scope": null}
    -- ]
    preferences     JSONB NOT NULL DEFAULT '{}',
    auth_provider   VARCHAR(50) DEFAULT 'local',
    auth_subject    VARCHAR(500),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, email)
);

CREATE INDEX idx_user_org ON app_user(organization_id);
CREATE INDEX idx_user_email ON app_user(email);
CREATE INDEX idx_user_roles ON app_user USING gin (roles);

-- RLS policy
ALTER TABLE app_user ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON app_user
    USING (organization_id = current_setting('app.current_org_id')::UUID);
```

---

## Organisational Structure

```sql
-- ============================================================
-- BUSINESS STRUCTURE
-- ============================================================

CREATE TABLE business_unit (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organization(id),
    parent_id       UUID REFERENCES business_unit(id),
    name            VARCHAR(255) NOT NULL,
    code            VARCHAR(50),
    unit_type       VARCHAR(50) NOT NULL DEFAULT 'department',
    path            TEXT,                   -- materialised path: '/root/eng/platform'
    depth           INT NOT NULL DEFAULT 0,
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- metadata example: {
    --   "jurisdiction": "DE",
    --   "regulated_entity": true,
    --   "regulatory_body": "BaFin",
    --   "headcount": 42,
    --   "cost_center": "CC-4200"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_bu_org ON business_unit(organization_id);
CREATE INDEX idx_bu_parent ON business_unit(parent_id);
CREATE INDEX idx_bu_path ON business_unit(path text_pattern_ops);
CREATE INDEX idx_bu_metadata ON business_unit USING gin (metadata);
```

---

## Risk Management

```sql
-- ============================================================
-- RISK MANAGEMENT
-- Core fields relational; variable fields in JSONB
-- ============================================================

CREATE TABLE risk (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organization(id),
    ref_id          VARCHAR(50) NOT NULL,
    title           VARCHAR(500) NOT NULL,
    description     TEXT,

    -- Core relational fields (queried/filtered/sorted constantly)
    category        VARCHAR(100),
    business_unit_id UUID REFERENCES business_unit(id),
    inherent_likelihood INT NOT NULL,
    inherent_impact     INT NOT NULL,
    inherent_score      INT GENERATED ALWAYS AS (inherent_likelihood * inherent_impact) STORED,
    residual_likelihood INT,
    residual_impact     INT,
    residual_score      INT GENERATED ALWAYS AS (residual_likelihood * residual_impact) STORED,
    treatment_type  VARCHAR(20) NOT NULL DEFAULT 'mitigate',
    risk_owner_id   UUID REFERENCES app_user(id),
    status          VARCHAR(30) NOT NULL DEFAULT 'identified',
    next_review_date DATE,

    -- Extended fields in JSONB (vary by industry, jurisdiction, risk type)
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- Generic risk metadata example:
    -- {
    --   "risk_source": "External threat actor exploiting unpatched vulnerability",
    --   "consequence": "Customer data exposure, regulatory fine, reputational damage",
    --   "treatment_plan": "Deploy WAF, implement automated patching, add API rate limiting",
    --   "treatment_due_date": "2026-08-01",
    --   "review_frequency": "quarterly",
    --   "last_review_date": "2026-03-15",
    --   "tags": ["cyber", "data-protection", "api-security"]
    -- }
    --
    -- Financial services (Basel IV) risk metadata example:
    -- {
    --   "risk_source": "...",
    --   "operational_risk_category": "external_fraud",   -- Basel event type
    --   "loss_amount": 45000.00,
    --   "loss_currency": "EUR",
    --   "business_line": "retail_banking",               -- Basel business line
    --   "key_risk_indicator": "fraud_attempts_per_month",
    --   "kri_threshold": 50,
    --   "kri_current": 37,
    --   "regulatory_capital_impact": true
    -- }
    --
    -- AI risk (ISO 42001 / EU AI Act) metadata example:
    -- {
    --   "risk_source": "...",
    --   "ai_system_name": "Credit Scoring Model v3",
    --   "ai_risk_classification": "high_risk",           -- EU AI Act classification
    --   "model_type": "gradient_boosted_trees",
    --   "training_data_source": "internal_credit_bureau",
    --   "bias_assessment_date": "2026-02-20",
    --   "bias_assessment_result": "acceptable",
    --   "human_oversight_mechanism": "loan_officer_review",
    --   "conformity_assessment_status": "pending"
    -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, ref_id)
);

CREATE INDEX idx_risk_org ON risk(organization_id);
CREATE INDEX idx_risk_status ON risk(organization_id, status);
CREATE INDEX idx_risk_residual ON risk(organization_id, residual_score DESC);
CREATE INDEX idx_risk_review ON risk(organization_id, next_review_date);
CREATE INDEX idx_risk_owner ON risk(risk_owner_id);
CREATE INDEX idx_risk_metadata ON risk USING gin (metadata);
-- Targeted JSONB index for common queries:
CREATE INDEX idx_risk_tags ON risk USING gin ((metadata->'tags'));

-- Assets linked to risks
CREATE TABLE asset (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organization(id),
    name            VARCHAR(255) NOT NULL,
    asset_type      VARCHAR(50) NOT NULL,
    business_unit_id UUID REFERENCES business_unit(id),
    owner_id        UUID REFERENCES app_user(id),
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- metadata example: {
    --   "classification": "confidential",
    --   "hosting": "aws-eu-west-1",
    --   "data_types": ["PII", "financial"],
    --   "compliance_scope": ["pci_dss", "gdpr"]
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_asset_org ON asset(organization_id);
CREATE INDEX idx_asset_metadata ON asset USING gin (metadata);

CREATE TABLE risk_asset (
    risk_id         UUID NOT NULL REFERENCES risk(id) ON DELETE CASCADE,
    asset_id        UUID NOT NULL REFERENCES asset(id) ON DELETE CASCADE,
    PRIMARY KEY (risk_id, asset_id)
);
```

---

## Control Library

```sql
-- ============================================================
-- CONTROL LIBRARY
-- ============================================================

CREATE TABLE control (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organization(id),
    ref_id          VARCHAR(50) NOT NULL,
    title           VARCHAR(500) NOT NULL,
    description     TEXT,

    -- Core relational fields
    control_type    VARCHAR(30) NOT NULL DEFAULT 'preventive',
    automation_level VARCHAR(20) NOT NULL DEFAULT 'manual',
    frequency       VARCHAR(20) DEFAULT 'continuous',
    owner_id        UUID REFERENCES app_user(id),
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    effectiveness   VARCHAR(20),
    last_test_date  DATE,
    next_test_date  DATE,

    -- Extended fields
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- metadata example: {
    --   "implementation_evidence": "AWS Config rule enforces encryption at rest",
    --   "testing_procedure": "Verify via AWS Config compliance dashboard monthly",
    --   "compensating_for": "Lack of native application-level encryption",
    --   "tags": ["encryption", "data-at-rest", "aws"],
    --   "evidence_sources": ["aws_config", "cloudtrail"]
    -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, ref_id)
);

CREATE INDEX idx_control_org ON control(organization_id);
CREATE INDEX idx_control_status ON control(organization_id, status);
CREATE INDEX idx_control_metadata ON control USING gin (metadata);

-- Many-to-many: Risk <-> Control
CREATE TABLE risk_control (
    risk_id         UUID NOT NULL REFERENCES risk(id) ON DELETE CASCADE,
    control_id      UUID NOT NULL REFERENCES control(id) ON DELETE CASCADE,
    relationship    VARCHAR(20) NOT NULL DEFAULT 'mitigates',
    PRIMARY KEY (risk_id, control_id)
);

-- Control test results
CREATE TABLE control_test (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    control_id      UUID NOT NULL REFERENCES control(id),
    tested_by       UUID REFERENCES app_user(id),
    test_date       TIMESTAMPTZ NOT NULL DEFAULT now(),
    result          VARCHAR(20) NOT NULL,
    automated       BOOLEAN NOT NULL DEFAULT false,
    details         JSONB NOT NULL DEFAULT '{}',
    -- details example: {
    --   "methodology": "reperformance",
    --   "evidence_notes": "Verified 100% of S3 buckets have encryption enabled",
    --   "sample_size": 47,
    --   "population_size": 47,
    --   "exceptions_found": 0,
    --   "source_system": "aws_config",
    --   "raw_evidence": {"config_rule": "s3-bucket-server-side-encryption-enabled", "compliance": "COMPLIANT"}
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_control_test_control ON control_test(control_id);
CREATE INDEX idx_control_test_date ON control_test(control_id, test_date DESC);
```

---

## Compliance Framework Management

```sql
-- ============================================================
-- COMPLIANCE FRAMEWORKS
-- ============================================================

CREATE TABLE framework (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID,                   -- NULL = global/system framework
    name            VARCHAR(255) NOT NULL,
    version         VARCHAR(50),
    provider        VARCHAR(255),
    framework_type  VARCHAR(30) NOT NULL DEFAULT 'standard',
    jurisdiction    CHAR(2),
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    oscal_catalog_id VARCHAR(255),
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- metadata example: {
    --   "source_url": "https://www.iso.org/standard/27001",
    --   "effective_date": "2022-10-25",
    --   "certification_body": "ISO/IEC JTC 1",
    --   "related_frameworks": ["ISO 27002:2022", "ISO 27701:2025"],
    --   "tags": ["information_security", "certifiable"]
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_framework_org ON framework(organization_id);

CREATE TABLE framework_requirement (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    framework_id    UUID NOT NULL REFERENCES framework(id) ON DELETE CASCADE,
    parent_id       UUID REFERENCES framework_requirement(id),
    ref_id          VARCHAR(100) NOT NULL,
    title           VARCHAR(500) NOT NULL,
    description     TEXT,
    path            TEXT,
    depth           INT NOT NULL DEFAULT 0,
    sort_order      INT NOT NULL DEFAULT 0,
    oscal_control_id VARCHAR(255),
    extended_fields JSONB NOT NULL DEFAULT '{}',
    -- extended_fields example (ISO 27001 Annex A control):
    -- {
    --   "control_objective": "Ensure protection of records",
    --   "implementation_guidance": "Records should be protected from loss...",
    --   "annex_category": "A.5 Organizational controls",
    --   "testing_guidance": "Review records management procedures..."
    -- }
    -- extended_fields example (PCI DSS 4.0 requirement):
    -- {
    --   "testing_procedure": "Examine network diagrams and interview personnel...",
    --   "defined_approach_requirements": "All system components...",
    --   "customized_approach_objective": "The location of all...",
    --   "applicability_notes": "This requirement applies to all in-scope networks"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (framework_id, ref_id)
);

CREATE INDEX idx_fw_req_framework ON framework_requirement(framework_id);
CREATE INDEX idx_fw_req_parent ON framework_requirement(parent_id);
CREATE INDEX idx_fw_req_extended ON framework_requirement USING gin (extended_fields);

-- Control <-> Requirement mapping
CREATE TABLE control_requirement (
    control_id      UUID NOT NULL REFERENCES control(id) ON DELETE CASCADE,
    requirement_id  UUID NOT NULL REFERENCES framework_requirement(id) ON DELETE CASCADE,
    coverage_status VARCHAR(20) NOT NULL DEFAULT 'mapped',
    notes           TEXT,
    PRIMARY KEY (control_id, requirement_id)
);

-- Cross-framework mapping
CREATE TABLE requirement_crosswalk (
    source_requirement_id UUID NOT NULL REFERENCES framework_requirement(id) ON DELETE CASCADE,
    target_requirement_id UUID NOT NULL REFERENCES framework_requirement(id) ON DELETE CASCADE,
    mapping_type    VARCHAR(20) NOT NULL DEFAULT 'equivalent',
    confidence      VARCHAR(10) DEFAULT 'high',
    source          VARCHAR(100),
    PRIMARY KEY (source_requirement_id, target_requirement_id)
);

-- Compliance assessment (combines assessment + results into one table)
CREATE TABLE compliance_assessment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organization(id),
    framework_id    UUID NOT NULL REFERENCES framework(id),
    name            VARCHAR(255) NOT NULL,
    assessment_date DATE NOT NULL,
    assessor_id     UUID REFERENCES app_user(id),
    status          VARCHAR(20) NOT NULL DEFAULT 'in_progress',
    overall_score   DECIMAL(5,2),
    results         JSONB NOT NULL DEFAULT '[]',
    -- results example: [
    --   {
    --     "requirement_id": "uuid",
    --     "requirement_ref": "A.5.1",
    --     "status": "compliant",
    --     "evidence_notes": "Policy approved and published on 2026-01-15",
    --     "assessed_by": "uuid",
    --     "assessed_at": "2026-05-10T10:30:00Z"
    --   },
    --   {
    --     "requirement_id": "uuid",
    --     "requirement_ref": "A.5.2",
    --     "status": "partially_compliant",
    --     "evidence_notes": "Roles defined but not formally communicated to all staff",
    --     "gap_description": "Missing formal role communication process",
    --     "remediation_plan": "Schedule all-hands presentation by 2026-06-30"
    --   }
    -- ]
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_comp_assess_org ON compliance_assessment(organization_id);
CREATE INDEX idx_comp_assess_fw ON compliance_assessment(framework_id);
CREATE INDEX idx_comp_assess_results ON compliance_assessment USING gin (results);
```

---

## Policy & Audit Management

```sql
-- ============================================================
-- POLICY MANAGEMENT
-- ============================================================

CREATE TABLE policy (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organization(id),
    ref_id          VARCHAR(50) NOT NULL,
    title           VARCHAR(500) NOT NULL,
    content         TEXT,
    version         INT NOT NULL DEFAULT 1,
    status          VARCHAR(20) NOT NULL DEFAULT 'draft',
    policy_type     VARCHAR(30) NOT NULL DEFAULT 'policy',
    owner_id        UUID REFERENCES app_user(id),
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- metadata example: {
    --   "approver_id": "uuid",
    --   "approved_at": "2026-04-15T09:00:00Z",
    --   "effective_date": "2026-05-01",
    --   "review_due_date": "2027-05-01",
    --   "review_frequency": "annual",
    --   "audience": ["all_employees"],
    --   "languages": ["en", "de", "fr"],
    --   "linked_control_ids": ["uuid-1", "uuid-2"],
    --   "version_history": [
    --     {"version": 1, "created_at": "2025-01-10", "change_summary": "Initial draft"},
    --     {"version": 2, "created_at": "2026-04-15", "change_summary": "Updated Section 3 for DORA"}
    --   ],
    --   "attestation_summary": {
    --     "total_required": 150,
    --     "attested": 142,
    --     "pending": 8,
    --     "last_updated": "2026-05-10"
    --   }
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, ref_id)
);

CREATE INDEX idx_policy_org ON policy(organization_id);
CREATE INDEX idx_policy_status ON policy(organization_id, status);
CREATE INDEX idx_policy_metadata ON policy USING gin (metadata);

-- Policy attestations (relational — queried frequently for compliance reporting)
CREATE TABLE policy_attestation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    policy_id       UUID NOT NULL REFERENCES policy(id),
    user_id         UUID NOT NULL REFERENCES app_user(id),
    policy_version  INT NOT NULL,
    attested_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    status          VARCHAR(20) NOT NULL DEFAULT 'acknowledged',
    UNIQUE (policy_id, user_id, policy_version)
);

CREATE INDEX idx_attestation_policy ON policy_attestation(policy_id);

-- ============================================================
-- AUDIT MANAGEMENT
-- ============================================================

CREATE TABLE audit_engagement (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organization(id),
    ref_id          VARCHAR(50) NOT NULL,
    title           VARCHAR(500) NOT NULL,
    engagement_type VARCHAR(30) NOT NULL DEFAULT 'assurance',
    status          VARCHAR(30) NOT NULL DEFAULT 'planned',
    lead_auditor_id UUID REFERENCES app_user(id),
    business_unit_id UUID REFERENCES business_unit(id),
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- metadata example: {
    --   "audit_plan_year": 2026,
    --   "planned_start": "2026-06-01",
    --   "planned_end": "2026-07-15",
    --   "actual_start": null,
    --   "actual_end": null,
    --   "scope": "Review of access management controls across production systems",
    --   "objectives": "Assess design and operating effectiveness of logical access controls",
    --   "team_members": ["uuid-1", "uuid-2"],
    --   "budget_hours": 120,
    --   "actual_hours": null
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, ref_id)
);

CREATE INDEX idx_audit_eng_org ON audit_engagement(organization_id);
CREATE INDEX idx_audit_eng_status ON audit_engagement(organization_id, status);

CREATE TABLE audit_finding (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    engagement_id   UUID NOT NULL REFERENCES audit_engagement(id),
    ref_id          VARCHAR(50) NOT NULL,
    title           VARCHAR(500) NOT NULL,
    severity        VARCHAR(20) NOT NULL DEFAULT 'medium',
    status          VARCHAR(30) NOT NULL DEFAULT 'draft',
    finding_owner_id UUID REFERENCES app_user(id),
    due_date        DATE,
    details         JSONB NOT NULL DEFAULT '{}',
    -- details example (IIA-aligned):
    -- {
    --   "condition": "17 of 42 privileged accounts had no documented quarterly access review",
    --   "criteria": "ISO 27001 A.9.2.5 requires periodic review of user access rights",
    --   "cause": "Manual spreadsheet-based tracking with no escalation",
    --   "effect": "Increased risk of unauthorised privileged access",
    --   "recommendation": "Implement automated access review workflow",
    --   "management_response": "Agreed. Will implement Okta access certification by Q3 2026",
    --   "corrective_actions": [
    --     {
    --       "description": "Deploy Okta access certification module",
    --       "assigned_to": "uuid",
    --       "due_date": "2026-09-30",
    --       "status": "in_progress"
    --     },
    --     {
    --       "description": "Complete initial certification cycle for all privileged accounts",
    --       "assigned_to": "uuid",
    --       "due_date": "2026-10-31",
    --       "status": "open"
    --     }
    --   ],
    --   "linked_risks": ["uuid-risk-1"],
    --   "linked_controls": ["uuid-control-1", "uuid-control-2"]
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_finding_eng ON audit_finding(engagement_id);
CREATE INDEX idx_finding_status ON audit_finding(status);
CREATE INDEX idx_finding_severity ON audit_finding(severity);
CREATE INDEX idx_finding_details ON audit_finding USING gin (details);
```

---

## Third-Party Risk & Incidents

```sql
-- ============================================================
-- THIRD-PARTY RISK MANAGEMENT
-- ============================================================

CREATE TABLE vendor (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organization(id),
    name            VARCHAR(255) NOT NULL,
    criticality     VARCHAR(20) NOT NULL DEFAULT 'medium',
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    owner_id        UUID REFERENCES app_user(id),
    risk_score      DECIMAL(5,2),
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- metadata example: {
    --   "vendor_type": "cloud_provider",
    --   "jurisdiction": "US",
    --   "website": "https://aws.amazon.com",
    --   "contract_start": "2024-01-01",
    --   "contract_end": "2027-01-01",
    --   "data_processing": true,
    --   "sub_processors": ["Cloudflare", "Twilio"],
    --   "certifications": ["SOC 2 Type II", "ISO 27001", "PCI DSS"],
    --   "dora_classification": "critical_ict_provider",
    --   "last_assessment": {
    --     "date": "2026-03-15",
    --     "result": "acceptable",
    --     "next_due": "2026-09-15"
    --   },
    --   "assessment_history": [
    --     {"date": "2025-09-15", "result": "acceptable", "assessor": "uuid"},
    --     {"date": "2026-03-15", "result": "acceptable", "assessor": "uuid"}
    --   ]
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_vendor_org ON vendor(organization_id);
CREATE INDEX idx_vendor_status ON vendor(organization_id, status);
CREATE INDEX idx_vendor_metadata ON vendor USING gin (metadata);

-- ============================================================
-- INCIDENT MANAGEMENT
-- ============================================================

CREATE TABLE incident (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organization(id),
    ref_id          VARCHAR(50) NOT NULL,
    title           VARCHAR(500) NOT NULL,
    incident_type   VARCHAR(30) NOT NULL,
    severity        VARCHAR(20) NOT NULL DEFAULT 'medium',
    status          VARCHAR(30) NOT NULL DEFAULT 'reported',
    reported_by     UUID REFERENCES app_user(id),
    assigned_to     UUID REFERENCES app_user(id),
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- metadata example: {
    --   "description": "Customer PII exposed via misconfigured S3 bucket",
    --   "reported_at": "2026-05-10T08:30:00Z",
    --   "business_unit_id": "uuid",
    --   "root_cause": "Terraform misconfiguration in production deployment",
    --   "remediation": "Bucket ACL corrected; AWS Config rule deployed to prevent recurrence",
    --   "financial_impact": 15000.00,
    --   "records_affected": 2400,
    --   "data_types_affected": ["email", "phone_number"],
    --   "regulatory_reportable": true,
    --   "regulatory_reported_at": "2026-05-10T14:00:00Z",
    --   "notification_authority": "ICO",
    --   "gdpr_article_33_deadline": "2026-05-13T08:30:00Z",
    --   "linked_risks": ["uuid-risk-1"],
    --   "linked_controls": ["uuid-control-1"],
    --   "timeline": [
    --     {"event": "detected", "timestamp": "2026-05-10T07:15:00Z", "by": "automated_scanner"},
    --     {"event": "reported", "timestamp": "2026-05-10T08:30:00Z", "by": "uuid-user"},
    --     {"event": "contained", "timestamp": "2026-05-10T09:00:00Z", "by": "uuid-user"}
    --   ]
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, ref_id)
);

CREATE INDEX idx_incident_org ON incident(organization_id);
CREATE INDEX idx_incident_status ON incident(organization_id, status);
CREATE INDEX idx_incident_metadata ON incident USING gin (metadata);
```

---

## Regulatory Change & Evidence

```sql
-- ============================================================
-- REGULATORY CHANGE MANAGEMENT
-- ============================================================

CREATE TABLE regulatory_change (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organization(id),
    title           VARCHAR(500) NOT NULL,
    status          VARCHAR(30) NOT NULL DEFAULT 'identified',
    jurisdiction    CHAR(2),
    effective_date  DATE,
    ai_confidence   DECIMAL(3,2),
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- metadata example: {
    --   "description": "DORA RTS on ICT third-party risk management published in EU OJ",
    --   "source": "EU Official Journal",
    --   "source_url": "https://eur-lex.europa.eu/...",
    --   "change_type": "new_regulation",
    --   "industries": ["financial_services", "insurance"],
    --   "impact_assessment": {
    --     "affected_controls": ["uuid-1", "uuid-2"],
    --     "affected_frameworks": ["uuid-dora"],
    --     "affected_policies": ["uuid-pol-vendor"],
    --     "action_items": [
    --       {"description": "Update vendor risk assessment questionnaire", "due_date": "2026-06-15", "owner": "uuid"},
    --       {"description": "Review critical ICT provider contracts", "due_date": "2026-07-01", "owner": "uuid"}
    --     ]
    --   }
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_regchange_org ON regulatory_change(organization_id);
CREATE INDEX idx_regchange_status ON regulatory_change(organization_id, status);
CREATE INDEX idx_regchange_metadata ON regulatory_change USING gin (metadata);

-- ============================================================
-- EVIDENCE REPOSITORY
-- ============================================================

CREATE TABLE evidence (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organization(id),
    name            VARCHAR(500) NOT NULL,
    evidence_type   VARCHAR(30) NOT NULL DEFAULT 'document',
    file_path       TEXT,
    file_size       BIGINT,
    mime_type       VARCHAR(255),
    hash_sha256     VARCHAR(64),
    collection_method VARCHAR(20) DEFAULT 'manual',
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- metadata example: {
    --   "collected_by": "uuid",
    --   "source_system": "aws_config",
    --   "valid_from": "2026-01-01",
    --   "valid_to": "2026-03-31",
    --   "linked_entities": [
    --     {"type": "control_test", "id": "uuid"},
    --     {"type": "compliance_assessment", "id": "uuid"}
    --   ]
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_evidence_org ON evidence(organization_id);
CREATE INDEX idx_evidence_metadata ON evidence USING gin (metadata);
```

---

## Audit Trail

```sql
-- ============================================================
-- AUDIT TRAIL
-- ============================================================

CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL,
    actor_id        UUID,
    action          VARCHAR(50) NOT NULL,
    entity_type     VARCHAR(50) NOT NULL,
    entity_id       UUID NOT NULL,
    entity_ref_id   VARCHAR(100),
    changes         JSONB,
    ip_address      INET,
    timestamp       TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (timestamp);

CREATE INDEX idx_audit_log_org_time ON audit_log(organization_id, timestamp DESC);
CREATE INDEX idx_audit_log_entity ON audit_log(entity_type, entity_id);
```

---

## JSONB Query Examples

```sql
-- ============================================================
-- COMMON JSONB QUERY PATTERNS
-- ============================================================

-- Q: "Find all risks tagged with 'cyber' in organization X"
SELECT ref_id, title, residual_score, status
FROM risk
WHERE organization_id = 'uuid-of-org'
  AND metadata @> '{"tags": ["cyber"]}';

-- Q: "Find all risks classified as high-risk AI systems (EU AI Act)"
SELECT ref_id, title, metadata->>'ai_system_name' AS ai_system
FROM risk
WHERE organization_id = 'uuid-of-org'
  AND metadata @> '{"ai_risk_classification": "high_risk"}';

-- Q: "Find all vendors with DORA critical ICT provider classification"
SELECT name, criticality, risk_score
FROM vendor
WHERE organization_id = 'uuid-of-org'
  AND metadata @> '{"dora_classification": "critical_ict_provider"}';

-- Q: "Find all audit findings with open corrective actions"
SELECT f.ref_id, f.title, f.severity,
       jsonb_array_elements(f.details->'corrective_actions') AS action
FROM audit_finding f
JOIN audit_engagement e ON f.engagement_id = e.id
WHERE e.organization_id = 'uuid-of-org'
  AND f.details->'corrective_actions' @> '[{"status": "open"}]';

-- Q: "Get compliance assessment results for a specific requirement"
SELECT ca.name, ca.assessment_date,
       result->>'status' AS compliance_status,
       result->>'evidence_notes' AS evidence
FROM compliance_assessment ca,
     jsonb_array_elements(ca.results) AS result
WHERE ca.organization_id = 'uuid-of-org'
  AND ca.framework_id = 'uuid-of-iso27001'
  AND result->>'requirement_ref' = 'A.5.1';

-- Q: "Find incidents that were regulatory reportable and reported to the ICO"
SELECT ref_id, title, severity,
       metadata->>'gdpr_article_33_deadline' AS reporting_deadline,
       metadata->>'regulatory_reported_at' AS reported_at
FROM incident
WHERE organization_id = 'uuid-of-org'
  AND metadata @> '{"regulatory_reportable": true, "notification_authority": "ICO"}';
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Multi-Tenancy & Identity | 2 | organization, app_user (roles in JSONB) |
| Organisational Structure | 1 | business_unit |
| Risk Management | 3 | risk, asset, risk_asset |
| Control Library | 3 | control, risk_control, control_test |
| Compliance Frameworks | 5 | framework, framework_requirement, control_requirement, requirement_crosswalk, compliance_assessment |
| Policy Management | 2 | policy, policy_attestation |
| Audit Management | 2 | audit_engagement, audit_finding (corrective actions in JSONB) |
| Third-Party Risk | 1 | vendor (assessments in JSONB) |
| Incidents | 1 | incident |
| Regulatory Change | 1 | regulatory_change (impacts in JSONB) |
| Evidence | 1 | evidence (links in JSONB) |
| Audit Trail | 1 | audit_log (partitioned) |
| **Total** | **23** | ~40% fewer tables than fully normalised |

---

## Key Design Decisions

1. **Core fields relational, variable fields JSONB** — the dividing line is clear: if a field is used in WHERE clauses, ORDER BY, GROUP BY, or JOINs across tables, it is a relational column. If it varies by industry, jurisdiction, framework, or tenant customisation, it goes in JSONB. This is enforced by code review convention, not by the database.

2. **Roles stored as JSONB array on app_user** — rather than separate role and user_role tables, roles are a JSONB array on the user record. This eliminates two tables and simplifies role queries. The trade-off is that role-based queries use GIN index containment (`roles @> '[{"role": "admin"}]'`) rather than foreign key JOINs. For a mid-market platform with <1000 users per tenant, this is performant.

3. **Compliance assessment results embedded as JSONB array** — individual requirement results are stored as a JSONB array on the compliance_assessment record rather than in a separate results table. This eliminates one table and makes assessment retrieval a single query. For frameworks with 100-200 requirements, the JSONB array is well within PostgreSQL's performance envelope. For frameworks with 1000+ requirements (NIST 800-53), results could be paginated in the JSONB array or optionally normalised.

4. **Vendor assessments and corrective actions embedded in parent JSONB** — vendor assessment history is stored in the vendor `metadata` JSONB, and corrective actions are stored in the audit_finding `details` JSONB. This collocates related data for common read patterns (view a vendor with its assessment history; view a finding with its corrective actions) at the cost of requiring application-level validation for embedded object structure.

5. **Evidence links stored in evidence metadata JSONB** — instead of a separate evidence_link table, linked entities are stored as a JSONB array in the evidence `metadata`. This simplifies the schema but requires application-level enforcement of link integrity.

6. **GIN indexes on all JSONB columns** — every JSONB column has a GIN index to support `@>` (containment), `?` (key existence), and `?|` (any key existence) operators efficiently. Targeted indexes on frequently queried nested paths (e.g., `metadata->'tags'`) provide additional performance for hot queries.

7. **Tenant-level custom field definitions** — the organization.settings JSONB includes a `custom_risk_fields` array that defines additional fields tenants want to track. The application renders these as form fields and stores values in the risk `metadata` JSONB. No schema changes required per tenant.

8. **Incident timeline as JSONB array** — the incident metadata includes a `timeline` array of timestamped events. This provides a rich incident chronology without a separate incident_event table, suitable for DORA Article 19 incident reporting and GDPR Article 33 breach notification timelines.

9. **Framework requirement extended_fields for framework-specific content** — ISO 27001 Annex A controls need implementation guidance, PCI DSS requirements need testing procedures, NIST 800-53 controls need control enhancements. Rather than creating per-framework tables, these live in `extended_fields` JSONB on framework_requirement. The OSCAL integration layer maps OSCAL properties to these JSONB fields.

10. **No separate policy_version table** — version history is stored as a JSONB array in the policy `metadata`. For most policies (2-5 versions over their lifetime), this is efficient. The current version content is always in the relational `content` column for full-text search. If version diffing becomes a hot path, this can be normalised later.

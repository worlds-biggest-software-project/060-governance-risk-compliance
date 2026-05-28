# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Governance, Risk & Compliance (GRC) · Created: 2026-05-12

## Philosophy

This model follows classical third-normal-form (3NF) relational design where every domain concept — risk, control, framework, policy, audit engagement, finding, incident, vendor — occupies its own dedicated table with explicit foreign key relationships and junction tables for many-to-many associations. The source of truth is always the current row state in each table, with a separate audit log table capturing change history.

This approach mirrors how enterprise GRC platforms like MetricStream ("Connected GRC" unified data model) and IBM OpenPages structure their data internally: a rich entity graph of risks, controls, assessments, policies, and findings linked through well-defined foreign keys. It also aligns directly with the entity model used by CISO Assistant (Django ORM with explicit model classes for RiskScenario, AppliedControl, ComplianceAssessment, etc.) and Eramba (MySQL-backed relational tables for risk, compliance, policy, and audit objects).

The normalized approach maximises query flexibility and enforces referential integrity at the database level. Every relationship is explicit, every constraint is enforced, and complex cross-domain queries (e.g., "show all controls linked to risks above appetite that also satisfy ISO 27001 Annex A requirements") are natural SQL JOINs rather than application-level data stitching.

**Best for:** Organisations that need strong data integrity guarantees, complex cross-entity reporting, and a well-understood schema that maps directly to GRC domain vocabulary (ISO 31000, COSO ERM, IIA Standards).

**Trade-offs:**
- (+) Maximum referential integrity — the database enforces every relationship
- (+) Complex cross-domain queries are natural SQL JOINs
- (+) Schema is self-documenting — mirrors GRC domain language directly
- (+) Well-supported by every ORM (Django, SQLAlchemy, Prisma, TypeORM)
- (+) Standards-aligned field naming (ISO 31000 risk vocabulary, OSCAL structure)
- (-) High table count (~45-55 tables) increases migration complexity
- (-) Schema changes for jurisdiction-specific or framework-specific fields require ALTER TABLE or new junction tables
- (-) Many-to-many junction tables add JOIN depth for common queries
- (-) No built-in temporal querying — "what was true on date X?" requires separate history tables or SCD patterns

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO 31000:2018 | Risk register fields (likelihood, impact, risk_source, consequence, treatment_type) map to ISO 31000 vocabulary |
| COSO ERM 2017 | Risk taxonomy and risk appetite tables structured around COSO's five components |
| ISO/IEC 27001:2022 | Framework and control tables support Annex A control set as first-class content |
| NIST OSCAL | Framework, control, and assessment tables align with OSCAL Catalog, Profile, and Assessment Results layers |
| ISO/IEC 42001:2023 | AI management system controls supported as framework content packs |
| IIA Global Standards 2024 | Audit engagement, finding, and corrective action tables align with IPPF lifecycle stages |
| OCSF | Audit event log structured to emit OCSF-compatible security events |
| ISO 3166 | Jurisdiction references use ISO 3166-1 alpha-2 codes |
| NIST SP 800-53 Rev. 5 | Control catalog importable via OSCAL-aligned schema |

---

## Multi-Tenancy & Access Control

```sql
-- ============================================================
-- TENANT & IDENTITY
-- ============================================================

CREATE TABLE organization (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    industry        VARCHAR(100),           -- e.g., 'financial_services', 'healthcare', 'technology'
    jurisdiction    CHAR(2),                -- ISO 3166-1 alpha-2
    settings        JSONB DEFAULT '{}',     -- org-level configuration
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organization(id),
    email           VARCHAR(320) NOT NULL,
    display_name    VARCHAR(255) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'active'
                    CHECK (status IN ('active', 'inactive', 'suspended')),
    auth_provider   VARCHAR(50) DEFAULT 'local',  -- 'local', 'saml', 'oidc'
    auth_subject    VARCHAR(500),                  -- external IdP subject identifier
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, email)
);

CREATE INDEX idx_app_user_org ON app_user(organization_id);
CREATE INDEX idx_app_user_email ON app_user(email);

CREATE TABLE role (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organization(id),
    name            VARCHAR(100) NOT NULL,  -- e.g., 'risk_owner', 'compliance_manager', 'auditor', 'cro', 'admin'
    description     TEXT,
    permissions     JSONB NOT NULL DEFAULT '[]',
    -- Example permissions: ["risk.read", "risk.write", "control.read", "audit.manage", "policy.approve"]
    is_system       BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, name)
);

CREATE TABLE user_role (
    user_id         UUID NOT NULL REFERENCES app_user(id) ON DELETE CASCADE,
    role_id         UUID NOT NULL REFERENCES role(id) ON DELETE CASCADE,
    scope_type      VARCHAR(50),            -- NULL = org-wide, 'business_unit', 'entity'
    scope_id        UUID,                   -- FK to business_unit or entity if scoped
    granted_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    granted_by      UUID REFERENCES app_user(id),
    PRIMARY KEY (user_id, role_id, COALESCE(scope_id, '00000000-0000-0000-0000-000000000000'))
);

-- Row-Level Security policy (applied to all tenant-scoped tables)
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
    unit_type       VARCHAR(50) NOT NULL DEFAULT 'department'
                    CHECK (unit_type IN ('division', 'department', 'team', 'subsidiary', 'branch')),
    jurisdiction    CHAR(2),                -- ISO 3166-1 alpha-2; inherits from org if NULL
    path            TEXT,                   -- materialised path for hierarchy queries: '/root/division/dept'
    depth           INT NOT NULL DEFAULT 0,
    owner_id        UUID REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_bu_org ON business_unit(organization_id);
CREATE INDEX idx_bu_parent ON business_unit(parent_id);
CREATE INDEX idx_bu_path ON business_unit(path text_pattern_ops);

CREATE TABLE asset (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organization(id),
    business_unit_id UUID REFERENCES business_unit(id),
    name            VARCHAR(255) NOT NULL,
    asset_type      VARCHAR(50) NOT NULL
                    CHECK (asset_type IN ('application', 'database', 'server', 'network', 'data_store',
                                          'cloud_service', 'physical', 'process', 'vendor_system', 'other')),
    description     TEXT,
    classification  VARCHAR(50),            -- 'public', 'internal', 'confidential', 'restricted'
    owner_id        UUID REFERENCES app_user(id),
    status          VARCHAR(20) NOT NULL DEFAULT 'active'
                    CHECK (status IN ('active', 'decommissioning', 'decommissioned')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_asset_org ON asset(organization_id);
CREATE INDEX idx_asset_bu ON asset(business_unit_id);
```

---

## Risk Management

```sql
-- ============================================================
-- RISK MANAGEMENT (aligned to ISO 31000:2018)
-- ============================================================

CREATE TABLE risk_category (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organization(id),
    parent_id       UUID REFERENCES risk_category(id),
    name            VARCHAR(255) NOT NULL,  -- e.g., 'Strategic', 'Operational', 'Financial', 'Compliance', 'Cyber'
    description     TEXT,
    path            TEXT,                   -- materialised path for hierarchy
    sort_order      INT NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE risk_matrix (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organization(id),
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    likelihood_scale JSONB NOT NULL,
    -- Example: [{"level": 1, "label": "Rare", "description": "< 5% probability"}, ...]
    impact_scale    JSONB NOT NULL,
    -- Example: [{"level": 1, "label": "Insignificant", "description": "< $10K impact"}, ...]
    risk_levels     JSONB NOT NULL,
    -- Example: [{"min_score": 1, "max_score": 4, "label": "Low", "color": "#22c55e"}, ...]
    is_default      BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE risk (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organization(id),
    ref_id          VARCHAR(50) NOT NULL,   -- human-readable identifier: 'RSK-001'
    title           VARCHAR(500) NOT NULL,
    description     TEXT,
    risk_source     TEXT,                   -- ISO 31000: source of risk
    category_id     UUID REFERENCES risk_category(id),
    business_unit_id UUID REFERENCES business_unit(id),
    risk_matrix_id  UUID REFERENCES risk_matrix(id),

    -- Inherent risk (before controls)
    inherent_likelihood INT NOT NULL,       -- scale value from risk_matrix
    inherent_impact     INT NOT NULL,
    inherent_score      INT GENERATED ALWAYS AS (inherent_likelihood * inherent_impact) STORED,

    -- Residual risk (after controls)
    residual_likelihood INT,
    residual_impact     INT,
    residual_score      INT GENERATED ALWAYS AS (residual_likelihood * residual_impact) STORED,

    -- Target risk (risk appetite)
    target_likelihood   INT,
    target_impact       INT,
    target_score        INT GENERATED ALWAYS AS (target_likelihood * target_impact) STORED,

    -- Treatment (ISO 31000)
    treatment_type  VARCHAR(20) NOT NULL DEFAULT 'mitigate'
                    CHECK (treatment_type IN ('accept', 'mitigate', 'transfer', 'avoid', 'escalate')),
    treatment_plan  TEXT,
    treatment_due_date DATE,

    -- Ownership
    risk_owner_id   UUID REFERENCES app_user(id),
    status          VARCHAR(30) NOT NULL DEFAULT 'identified'
                    CHECK (status IN ('identified', 'assessed', 'treating', 'monitoring',
                                      'accepted', 'closed', 'escalated')),

    -- Review
    last_review_date    DATE,
    next_review_date    DATE,
    review_frequency    VARCHAR(20) DEFAULT 'quarterly'
                        CHECK (review_frequency IN ('monthly', 'quarterly', 'semi_annual', 'annual')),

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, ref_id)
);

CREATE INDEX idx_risk_org ON risk(organization_id);
CREATE INDEX idx_risk_category ON risk(category_id);
CREATE INDEX idx_risk_bu ON risk(business_unit_id);
CREATE INDEX idx_risk_owner ON risk(risk_owner_id);
CREATE INDEX idx_risk_status ON risk(organization_id, status);
CREATE INDEX idx_risk_residual ON risk(organization_id, residual_score DESC);
CREATE INDEX idx_risk_next_review ON risk(organization_id, next_review_date);

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
    ref_id          VARCHAR(50) NOT NULL,   -- 'CTL-001'
    title           VARCHAR(500) NOT NULL,
    description     TEXT,
    control_type    VARCHAR(30) NOT NULL DEFAULT 'preventive'
                    CHECK (control_type IN ('preventive', 'detective', 'corrective', 'directive', 'compensating')),
    automation_level VARCHAR(20) NOT NULL DEFAULT 'manual'
                    CHECK (automation_level IN ('manual', 'semi_automated', 'automated')),
    frequency       VARCHAR(20) DEFAULT 'continuous'
                    CHECK (frequency IN ('continuous', 'daily', 'weekly', 'monthly', 'quarterly', 'annual', 'ad_hoc')),
    business_unit_id UUID REFERENCES business_unit(id),
    owner_id        UUID REFERENCES app_user(id),
    status          VARCHAR(20) NOT NULL DEFAULT 'active'
                    CHECK (status IN ('draft', 'active', 'under_review', 'retired')),
    effectiveness   VARCHAR(20)
                    CHECK (effectiveness IN ('effective', 'partially_effective', 'ineffective', 'not_assessed')),
    last_test_date  DATE,
    next_test_date  DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, ref_id)
);

CREATE INDEX idx_control_org ON control(organization_id);
CREATE INDEX idx_control_status ON control(organization_id, status);
CREATE INDEX idx_control_owner ON control(owner_id);
CREATE INDEX idx_control_next_test ON control(organization_id, next_test_date);

-- Many-to-many: Risk <-> Control
CREATE TABLE risk_control (
    risk_id         UUID NOT NULL REFERENCES risk(id) ON DELETE CASCADE,
    control_id      UUID NOT NULL REFERENCES control(id) ON DELETE CASCADE,
    relationship    VARCHAR(20) NOT NULL DEFAULT 'mitigates'
                    CHECK (relationship IN ('mitigates', 'monitors', 'compensates')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (risk_id, control_id)
);

-- Control testing / evidence
CREATE TABLE control_test (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    control_id      UUID NOT NULL REFERENCES control(id),
    tested_by       UUID REFERENCES app_user(id),
    test_date       TIMESTAMPTZ NOT NULL DEFAULT now(),
    result          VARCHAR(20) NOT NULL
                    CHECK (result IN ('pass', 'fail', 'partial', 'not_applicable')),
    evidence_notes  TEXT,
    methodology     VARCHAR(50),            -- 'inquiry', 'observation', 'inspection', 'reperformance'
    automated       BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_control_test_control ON control_test(control_id);
CREATE INDEX idx_control_test_date ON control_test(control_id, test_date DESC);
```

---

## Compliance Framework Management

```sql
-- ============================================================
-- COMPLIANCE FRAMEWORKS (aligned to NIST OSCAL structure)
-- ============================================================

CREATE TABLE framework (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID,                   -- NULL = global/system framework
    name            VARCHAR(255) NOT NULL,  -- 'ISO/IEC 27001:2022'
    version         VARCHAR(50),            -- '2022'
    provider        VARCHAR(255),           -- 'ISO/IEC', 'NIST', 'PCI SSC', 'EU'
    description     TEXT,
    framework_type  VARCHAR(30) NOT NULL DEFAULT 'standard'
                    CHECK (framework_type IN ('standard', 'regulation', 'guideline', 'internal_policy', 'benchmark')),
    jurisdiction    CHAR(2),                -- NULL = international
    effective_date  DATE,
    sunset_date     DATE,
    oscal_catalog_id VARCHAR(255),          -- OSCAL catalog UUID for interoperability
    source_url      TEXT,
    status          VARCHAR(20) NOT NULL DEFAULT 'active'
                    CHECK (status IN ('draft', 'active', 'superseded', 'retired')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_framework_org ON framework(organization_id);
CREATE INDEX idx_framework_status ON framework(status);

CREATE TABLE framework_requirement (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    framework_id    UUID NOT NULL REFERENCES framework(id) ON DELETE CASCADE,
    parent_id       UUID REFERENCES framework_requirement(id),
    ref_id          VARCHAR(100) NOT NULL,  -- 'A.5.1', '800-53:AC-2', 'Req 1.1.1'
    title           VARCHAR(500) NOT NULL,
    description     TEXT,
    guidance        TEXT,
    depth           INT NOT NULL DEFAULT 0,
    path            TEXT,                   -- materialised path for hierarchy
    sort_order      INT NOT NULL DEFAULT 0,
    oscal_control_id VARCHAR(255),          -- OSCAL control identifier
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (framework_id, ref_id)
);

CREATE INDEX idx_fw_req_framework ON framework_requirement(framework_id);
CREATE INDEX idx_fw_req_parent ON framework_requirement(parent_id);
CREATE INDEX idx_fw_req_path ON framework_requirement(path text_pattern_ops);

-- Many-to-many: Control <-> Framework Requirement (the core GRC link)
CREATE TABLE control_requirement (
    control_id      UUID NOT NULL REFERENCES control(id) ON DELETE CASCADE,
    requirement_id  UUID NOT NULL REFERENCES framework_requirement(id) ON DELETE CASCADE,
    coverage_status VARCHAR(20) NOT NULL DEFAULT 'mapped'
                    CHECK (coverage_status IN ('mapped', 'partial', 'planned', 'not_applicable')),
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (control_id, requirement_id)
);

-- Cross-framework mapping (requirement A satisfies requirement B)
CREATE TABLE requirement_crosswalk (
    source_requirement_id UUID NOT NULL REFERENCES framework_requirement(id) ON DELETE CASCADE,
    target_requirement_id UUID NOT NULL REFERENCES framework_requirement(id) ON DELETE CASCADE,
    mapping_type    VARCHAR(20) NOT NULL DEFAULT 'equivalent'
                    CHECK (mapping_type IN ('equivalent', 'superset', 'subset', 'related')),
    confidence      VARCHAR(10) DEFAULT 'high'
                    CHECK (confidence IN ('high', 'medium', 'low')),
    source          VARCHAR(100),           -- 'nist_cprt', 'manual', 'ai_suggested'
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (source_requirement_id, target_requirement_id)
);

-- Compliance assessment (point-in-time assessment of org against a framework)
CREATE TABLE compliance_assessment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organization(id),
    framework_id    UUID NOT NULL REFERENCES framework(id),
    name            VARCHAR(255) NOT NULL,
    assessment_date DATE NOT NULL,
    assessor_id     UUID REFERENCES app_user(id),
    status          VARCHAR(20) NOT NULL DEFAULT 'in_progress'
                    CHECK (status IN ('planned', 'in_progress', 'under_review', 'completed', 'archived')),
    overall_score   DECIMAL(5,2),           -- percentage compliance score
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_comp_assess_org ON compliance_assessment(organization_id);
CREATE INDEX idx_comp_assess_fw ON compliance_assessment(framework_id);

CREATE TABLE compliance_assessment_result (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    assessment_id   UUID NOT NULL REFERENCES compliance_assessment(id) ON DELETE CASCADE,
    requirement_id  UUID NOT NULL REFERENCES framework_requirement(id),
    status          VARCHAR(30) NOT NULL
                    CHECK (status IN ('compliant', 'partially_compliant', 'non_compliant',
                                      'not_applicable', 'not_assessed')),
    evidence_notes  TEXT,
    assessed_by     UUID REFERENCES app_user(id),
    assessed_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (assessment_id, requirement_id)
);
```

---

## Policy Management

```sql
-- ============================================================
-- POLICY MANAGEMENT
-- ============================================================

CREATE TABLE policy (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organization(id),
    ref_id          VARCHAR(50) NOT NULL,   -- 'POL-001'
    title           VARCHAR(500) NOT NULL,
    description     TEXT,
    content         TEXT,                   -- policy body (Markdown or HTML)
    version         INT NOT NULL DEFAULT 1,
    status          VARCHAR(20) NOT NULL DEFAULT 'draft'
                    CHECK (status IN ('draft', 'pending_review', 'pending_approval',
                                      'approved', 'published', 'retired')),
    policy_type     VARCHAR(30) NOT NULL DEFAULT 'policy'
                    CHECK (policy_type IN ('policy', 'standard', 'procedure', 'guideline')),
    owner_id        UUID REFERENCES app_user(id),
    approver_id     UUID REFERENCES app_user(id),
    approved_at     TIMESTAMPTZ,
    published_at    TIMESTAMPTZ,
    effective_date  DATE,
    review_due_date DATE,
    review_frequency VARCHAR(20) DEFAULT 'annual'
                    CHECK (review_frequency IN ('monthly', 'quarterly', 'semi_annual', 'annual', 'biennial')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, ref_id)
);

CREATE INDEX idx_policy_org ON policy(organization_id);
CREATE INDEX idx_policy_status ON policy(organization_id, status);
CREATE INDEX idx_policy_review_due ON policy(organization_id, review_due_date);

-- Policy version history
CREATE TABLE policy_version (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    policy_id       UUID NOT NULL REFERENCES policy(id) ON DELETE CASCADE,
    version         INT NOT NULL,
    content         TEXT NOT NULL,
    change_summary  TEXT,
    created_by      UUID REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (policy_id, version)
);

-- Policy acknowledgement tracking
CREATE TABLE policy_attestation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    policy_id       UUID NOT NULL REFERENCES policy(id),
    user_id         UUID NOT NULL REFERENCES app_user(id),
    policy_version  INT NOT NULL,
    attested_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    status          VARCHAR(20) NOT NULL DEFAULT 'acknowledged'
                    CHECK (status IN ('pending', 'acknowledged', 'declined', 'expired')),
    ip_address      INET,
    UNIQUE (policy_id, user_id, policy_version)
);

CREATE INDEX idx_attestation_policy ON policy_attestation(policy_id);
CREATE INDEX idx_attestation_user ON policy_attestation(user_id);

-- Many-to-many: Policy <-> Control
CREATE TABLE policy_control (
    policy_id       UUID NOT NULL REFERENCES policy(id) ON DELETE CASCADE,
    control_id      UUID NOT NULL REFERENCES control(id) ON DELETE CASCADE,
    PRIMARY KEY (policy_id, control_id)
);
```

---

## Audit Management

```sql
-- ============================================================
-- AUDIT MANAGEMENT (aligned to IIA Global Standards 2024)
-- ============================================================

CREATE TABLE audit_plan (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organization(id),
    name            VARCHAR(255) NOT NULL,  -- 'FY2026 Internal Audit Plan'
    fiscal_year     INT NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'draft'
                    CHECK (status IN ('draft', 'approved', 'in_progress', 'completed')),
    approved_by     UUID REFERENCES app_user(id),
    approved_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE audit_engagement (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    audit_plan_id   UUID REFERENCES audit_plan(id),
    organization_id UUID NOT NULL REFERENCES organization(id),
    ref_id          VARCHAR(50) NOT NULL,   -- 'AUD-2026-001'
    title           VARCHAR(500) NOT NULL,
    description     TEXT,
    engagement_type VARCHAR(30) NOT NULL DEFAULT 'assurance'
                    CHECK (engagement_type IN ('assurance', 'advisory', 'follow_up', 'special')),
    business_unit_id UUID REFERENCES business_unit(id),
    lead_auditor_id UUID REFERENCES app_user(id),
    status          VARCHAR(30) NOT NULL DEFAULT 'planned'
                    CHECK (status IN ('planned', 'fieldwork', 'reporting', 'review',
                                      'issued', 'closed')),
    planned_start   DATE,
    planned_end     DATE,
    actual_start    DATE,
    actual_end      DATE,
    scope           TEXT,
    objectives      TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, ref_id)
);

CREATE INDEX idx_audit_eng_org ON audit_engagement(organization_id);
CREATE INDEX idx_audit_eng_plan ON audit_engagement(audit_plan_id);
CREATE INDEX idx_audit_eng_status ON audit_engagement(organization_id, status);

CREATE TABLE audit_finding (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    engagement_id   UUID NOT NULL REFERENCES audit_engagement(id),
    ref_id          VARCHAR(50) NOT NULL,   -- 'FND-2026-001'
    title           VARCHAR(500) NOT NULL,
    condition       TEXT,                   -- IIA: what was found
    criteria        TEXT,                   -- IIA: what should be
    cause           TEXT,                   -- IIA: root cause
    effect          TEXT,                   -- IIA: impact/risk
    recommendation  TEXT,
    management_response TEXT,
    severity        VARCHAR(20) NOT NULL DEFAULT 'medium'
                    CHECK (severity IN ('critical', 'high', 'medium', 'low', 'informational')),
    status          VARCHAR(30) NOT NULL DEFAULT 'draft'
                    CHECK (status IN ('draft', 'under_review', 'issued', 'remediation',
                                      'verified_closed', 'accepted')),
    finding_owner_id UUID REFERENCES app_user(id),
    due_date        DATE,
    closed_date     DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_finding_eng ON audit_finding(engagement_id);
CREATE INDEX idx_finding_status ON audit_finding(status);
CREATE INDEX idx_finding_due ON audit_finding(due_date) WHERE status NOT IN ('verified_closed', 'accepted');

-- Corrective Action Plan
CREATE TABLE corrective_action (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    finding_id      UUID NOT NULL REFERENCES audit_finding(id),
    description     TEXT NOT NULL,
    assigned_to     UUID REFERENCES app_user(id),
    due_date        DATE,
    status          VARCHAR(20) NOT NULL DEFAULT 'open'
                    CHECK (status IN ('open', 'in_progress', 'completed', 'verified', 'overdue')),
    completion_date DATE,
    verification_notes TEXT,
    verified_by     UUID REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_cap_finding ON corrective_action(finding_id);
CREATE INDEX idx_cap_status ON corrective_action(status);
```

---

## Third-Party Risk Management

```sql
-- ============================================================
-- THIRD-PARTY RISK MANAGEMENT
-- ============================================================

CREATE TABLE vendor (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organization(id),
    name            VARCHAR(255) NOT NULL,
    vendor_type     VARCHAR(30) DEFAULT 'supplier'
                    CHECK (vendor_type IN ('supplier', 'subprocessor', 'contractor', 'cloud_provider', 'partner')),
    criticality     VARCHAR(20) NOT NULL DEFAULT 'medium'
                    CHECK (criticality IN ('critical', 'high', 'medium', 'low')),
    jurisdiction    CHAR(2),                -- ISO 3166-1 alpha-2
    website         TEXT,
    primary_contact_name  VARCHAR(255),
    primary_contact_email VARCHAR(320),
    contract_start  DATE,
    contract_end    DATE,
    risk_score      DECIMAL(5,2),
    status          VARCHAR(20) NOT NULL DEFAULT 'active'
                    CHECK (status IN ('prospect', 'active', 'under_review', 'suspended', 'terminated')),
    owner_id        UUID REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_vendor_org ON vendor(organization_id);
CREATE INDEX idx_vendor_status ON vendor(organization_id, status);

CREATE TABLE vendor_assessment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    vendor_id       UUID NOT NULL REFERENCES vendor(id),
    assessment_type VARCHAR(30) NOT NULL DEFAULT 'initial'
                    CHECK (assessment_type IN ('initial', 'periodic', 'triggered', 'exit')),
    assessor_id     UUID REFERENCES app_user(id),
    assessment_date DATE NOT NULL,
    overall_risk    VARCHAR(20)
                    CHECK (overall_risk IN ('critical', 'high', 'medium', 'low', 'acceptable')),
    findings_summary TEXT,
    status          VARCHAR(20) NOT NULL DEFAULT 'in_progress'
                    CHECK (status IN ('in_progress', 'completed', 'expired')),
    next_assessment_date DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_vendor_assess_vendor ON vendor_assessment(vendor_id);

-- Link vendor risks to org risk register
CREATE TABLE vendor_risk (
    vendor_id       UUID NOT NULL REFERENCES vendor(id) ON DELETE CASCADE,
    risk_id         UUID NOT NULL REFERENCES risk(id) ON DELETE CASCADE,
    PRIMARY KEY (vendor_id, risk_id)
);
```

---

## Incident Management

```sql
-- ============================================================
-- INCIDENT MANAGEMENT
-- ============================================================

CREATE TABLE incident (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organization(id),
    ref_id          VARCHAR(50) NOT NULL,   -- 'INC-2026-001'
    title           VARCHAR(500) NOT NULL,
    description     TEXT,
    incident_type   VARCHAR(30) NOT NULL
                    CHECK (incident_type IN ('security', 'privacy', 'operational', 'compliance',
                                             'data_breach', 'system_outage', 'fraud', 'other')),
    severity        VARCHAR(20) NOT NULL DEFAULT 'medium'
                    CHECK (severity IN ('critical', 'high', 'medium', 'low')),
    status          VARCHAR(30) NOT NULL DEFAULT 'reported'
                    CHECK (status IN ('reported', 'investigating', 'contained', 'remediated',
                                      'closed', 'post_mortem')),
    reported_by     UUID REFERENCES app_user(id),
    reported_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    assigned_to     UUID REFERENCES app_user(id),
    business_unit_id UUID REFERENCES business_unit(id),
    root_cause      TEXT,
    remediation     TEXT,
    financial_impact DECIMAL(15,2),
    regulatory_reportable BOOLEAN NOT NULL DEFAULT false,
    regulatory_reported_at TIMESTAMPTZ,
    resolved_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, ref_id)
);

CREATE INDEX idx_incident_org ON incident(organization_id);
CREATE INDEX idx_incident_status ON incident(organization_id, status);
CREATE INDEX idx_incident_type ON incident(organization_id, incident_type);

-- Link incidents to risks and controls
CREATE TABLE incident_risk (
    incident_id     UUID NOT NULL REFERENCES incident(id) ON DELETE CASCADE,
    risk_id         UUID NOT NULL REFERENCES risk(id) ON DELETE CASCADE,
    PRIMARY KEY (incident_id, risk_id)
);

CREATE TABLE incident_control (
    incident_id     UUID NOT NULL REFERENCES incident(id) ON DELETE CASCADE,
    control_id      UUID NOT NULL REFERENCES control(id) ON DELETE CASCADE,
    failure_type    VARCHAR(30)             -- 'control_failure', 'control_bypass', 'no_control_existed'
                    CHECK (failure_type IN ('control_failure', 'control_bypass', 'no_control_existed', 'design_gap')),
    PRIMARY KEY (incident_id, control_id)
);
```

---

## Evidence & Document Management

```sql
-- ============================================================
-- EVIDENCE REPOSITORY
-- ============================================================

CREATE TABLE evidence (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organization(id),
    name            VARCHAR(500) NOT NULL,
    description     TEXT,
    evidence_type   VARCHAR(30) NOT NULL DEFAULT 'document'
                    CHECK (evidence_type IN ('document', 'screenshot', 'log_extract', 'api_response',
                                             'configuration', 'attestation', 'report', 'other')),
    file_path       TEXT,                   -- object storage path
    file_size       BIGINT,
    mime_type       VARCHAR(255),
    hash_sha256     VARCHAR(64),            -- integrity verification
    collected_by    UUID REFERENCES app_user(id),
    collection_method VARCHAR(20) DEFAULT 'manual'
                    CHECK (collection_method IN ('manual', 'automated', 'api_sync')),
    source_system   VARCHAR(255),           -- e.g., 'aws', 'github', 'okta'
    collected_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    valid_from      DATE,
    valid_to        DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_evidence_org ON evidence(organization_id);
CREATE INDEX idx_evidence_type ON evidence(organization_id, evidence_type);

-- Polymorphic evidence attachment (evidence can link to controls, findings, assessments, etc.)
CREATE TABLE evidence_link (
    evidence_id     UUID NOT NULL REFERENCES evidence(id) ON DELETE CASCADE,
    entity_type     VARCHAR(50) NOT NULL,   -- 'control_test', 'audit_finding', 'compliance_assessment_result', 'vendor_assessment'
    entity_id       UUID NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (evidence_id, entity_type, entity_id)
);

CREATE INDEX idx_evidence_link_entity ON evidence_link(entity_type, entity_id);
```

---

## Regulatory Change Management

```sql
-- ============================================================
-- REGULATORY CHANGE MANAGEMENT
-- ============================================================

CREATE TABLE regulatory_change (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organization(id),
    title           VARCHAR(500) NOT NULL,
    description     TEXT,
    source          VARCHAR(255),           -- 'EU Official Journal', 'SEC EDGAR', 'NIST'
    source_url      TEXT,
    published_date  DATE,
    effective_date  DATE,
    jurisdiction    CHAR(2),                -- ISO 3166-1 alpha-2; NULL = international
    industries      TEXT[],                 -- array of applicable industries
    change_type     VARCHAR(30) NOT NULL DEFAULT 'new_regulation'
                    CHECK (change_type IN ('new_regulation', 'amendment', 'guidance', 'enforcement',
                                           'sunset', 'consultation')),
    status          VARCHAR(30) NOT NULL DEFAULT 'identified'
                    CHECK (status IN ('identified', 'under_review', 'impact_assessed',
                                      'action_required', 'implemented', 'not_applicable')),
    ai_confidence   DECIMAL(3,2),           -- AI classification confidence (0.00 - 1.00)
    assessed_by     UUID REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_reg_change_org ON regulatory_change(organization_id);
CREATE INDEX idx_reg_change_status ON regulatory_change(organization_id, status);

-- Impact mapping: regulatory change affects which controls/frameworks
CREATE TABLE regulatory_change_impact (
    regulatory_change_id UUID NOT NULL REFERENCES regulatory_change(id) ON DELETE CASCADE,
    entity_type     VARCHAR(30) NOT NULL,   -- 'control', 'framework', 'policy', 'risk'
    entity_id       UUID NOT NULL,
    impact_description TEXT,
    action_required TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (regulatory_change_id, entity_type, entity_id)
);
```

---

## Audit Trail

```sql
-- ============================================================
-- AUDIT TRAIL (OCSF-aligned event log)
-- ============================================================

CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL,
    actor_id        UUID,                   -- user who performed the action; NULL for system actions
    actor_email     VARCHAR(320),
    action          VARCHAR(50) NOT NULL,   -- 'create', 'update', 'delete', 'approve', 'attest', 'login', 'export'
    entity_type     VARCHAR(50) NOT NULL,   -- 'risk', 'control', 'policy', 'audit_finding', etc.
    entity_id       UUID NOT NULL,
    entity_ref_id   VARCHAR(100),           -- human-readable ref (e.g., 'RSK-001')
    changes         JSONB,                  -- {"field": {"old": "...", "new": "..."}} for updates
    ip_address      INET,
    user_agent      TEXT,
    timestamp       TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (timestamp);

-- Create monthly partitions (example for 2026)
CREATE TABLE audit_log_2026_01 PARTITION OF audit_log
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
-- ... additional monthly partitions

CREATE INDEX idx_audit_log_org_time ON audit_log(organization_id, timestamp DESC);
CREATE INDEX idx_audit_log_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_log_actor ON audit_log(actor_id);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Multi-Tenancy & Identity | 4 | organization, app_user, role, user_role |
| Organisational Structure | 2 | business_unit, asset |
| Risk Management | 4 | risk_category, risk_matrix, risk, risk_asset |
| Control Library | 3 | control, risk_control, control_test |
| Compliance Frameworks | 5 | framework, framework_requirement, control_requirement, requirement_crosswalk, compliance_assessment + results |
| Policy Management | 4 | policy, policy_version, policy_attestation, policy_control |
| Audit Management | 4 | audit_plan, audit_engagement, audit_finding, corrective_action |
| Third-Party Risk | 3 | vendor, vendor_assessment, vendor_risk |
| Incident Management | 3 | incident, incident_risk, incident_control |
| Evidence | 2 | evidence, evidence_link |
| Regulatory Change | 2 | regulatory_change, regulatory_change_impact |
| Audit Trail | 1 | audit_log (partitioned) |
| **Total** | **37** | Core tables; junction tables included |

---

## Key Design Decisions

1. **UUID primary keys everywhere** — standard for multi-tenant SaaS; eliminates ID collision across tenants and enables distributed ID generation without coordination.

2. **Row-Level Security (RLS) via `organization_id`** — every tenant-scoped table carries `organization_id` as a non-nullable foreign key. PostgreSQL RLS policies enforce tenant isolation at the database level using `current_setting('app.current_org_id')`, preventing application bugs from leaking cross-tenant data.

3. **Explicit junction tables for every many-to-many** — risk_control, control_requirement, policy_control, incident_risk, vendor_risk. This maximises query flexibility (e.g., "find all controls that both mitigate risk RSK-005 and satisfy ISO 27001 A.8.2") at the cost of additional JOINs.

4. **Materialised path for hierarchies** — business_unit, risk_category, and framework_requirement use a `path` column (e.g., `/root/division/dept`) for fast subtree queries via `LIKE 'prefix%'` or `text_pattern_ops` indexes. This avoids recursive CTEs for common hierarchy traversals while remaining simpler than nested sets.

5. **OSCAL alignment for frameworks** — framework and framework_requirement tables include `oscal_catalog_id` and `oscal_control_id` fields, enabling direct import/export of NIST OSCAL Catalog and Profile data. This positions the platform for FedRAMP and FISMA compliance ecosystems.

6. **ISO 31000 vocabulary for risk fields** — risk table columns (risk_source, likelihood, impact, treatment_type) directly map to ISO 31000:2018 terminology, making the schema self-documenting for GRC professionals.

7. **IIA Standards alignment for audit** — audit_finding uses the IIA condition/criteria/cause/effect structure mandated by the Global Internal Audit Standards 2024 (IPPF).

8. **Polymorphic evidence attachment via entity_type/entity_id** — the evidence_link table uses a type-discriminated pattern rather than separate junction tables per entity. This trades type safety (no FK enforcement on entity_id) for flexibility (evidence can attach to any entity type without schema changes).

9. **Partitioned audit log** — audit_log is partitioned by month using PostgreSQL native range partitioning. This ensures audit trail queries remain fast even with millions of events and enables efficient retention management (drop old partitions).

10. **Generated columns for risk scores** — inherent_score, residual_score, and target_score are `GENERATED ALWAYS AS` computed columns, ensuring score consistency without application logic duplication.

11. **Regulatory change management as first-class entity** — unlike most GRC schemas that treat regulatory tracking as external, this model includes regulatory_change with ai_confidence scoring and impact mapping to controls/frameworks, supporting the AI-native regulatory monitoring differentiator.

# Data Model Suggestion 4: Graph-Relational Hybrid

> Project: Governance, Risk & Compliance (GRC) · Created: 2026-05-12

## Philosophy

This model combines a relational PostgreSQL core for structured CRUD operations with a property graph layer for relationship-heavy queries. The relational tables handle day-to-day operations — creating risks, recording control tests, managing policies — while a graph representation enables the relationship-intensive queries that define GRC: "Which controls mitigate this risk AND satisfy these three framework requirements AND are owned by this business unit?", "Show me the blast radius of a failing control — every risk it mitigates, every framework it satisfies, every vendor it covers", "Find all indirect dependencies between this vendor and our critical business processes."

GRC is fundamentally a graph problem. The core value of a GRC platform lies not in the individual entities (risks, controls, frameworks, policies) but in the dense web of relationships between them. A risk is linked to multiple controls; each control maps to multiple framework requirements across multiple frameworks; frameworks have hierarchical requirement trees; controls are tested by auditors who also manage findings that link back to risks; vendors expose risks that are mitigated by controls that satisfy regulations. In a traditional relational model, traversing these relationships requires multi-table JOINs that become increasingly expensive as the graph deepens. In a graph model, traversal is a constant-time operation per hop.

Neo4j's published case studies show organisations modelling risk registers as property graphs to achieve deeper understanding of organisational risk through relationship analysis. This hybrid approach captures that benefit while keeping PostgreSQL as the transactional backbone, avoiding the operational complexity of running a separate graph database in production. The graph layer is implemented as PostgreSQL tables (`graph_node`, `graph_edge`) with recursive CTE queries, or optionally via Apache AGE (a PostgreSQL extension that adds native Cypher query support).

**Best for:** Organisations with complex risk-control-framework webs, conflict-of-interest analysis, supply chain / vendor dependency mapping, regulatory impact analysis ("if this regulation changes, what is affected?"), and AI-powered risk analytics that benefit from graph traversal.

**Trade-offs:**
- (+) Relationship-heavy queries are dramatically faster — graph traversal vs. multi-table JOINs
- (+) "Blast radius" analysis is natural — traverse from any node to all connected entities
- (+) Cross-framework overlap detection via graph path analysis
- (+) Supply chain and vendor dependency graphs modelled naturally
- (+) AI/LLM analytics benefit from graph context — retrieve connected subgraph for any entity
- (+) Flexible — new relationship types require only new edge types, not new junction tables
- (-) Two representations of the same data — relational for CRUD, graph for analysis — require sync
- (-) Graph query syntax (recursive CTEs or Cypher via Apache AGE) has a steeper learning curve
- (-) PostgreSQL graph queries are less optimised than native graph databases (Neo4j, Amazon Neptune)
- (-) Graph consistency must be maintained when relational data changes — additional application logic
- (-) Apache AGE extension adds a dependency; recursive CTEs add query complexity
- (-) Fewer ORM tools understand the graph layer — custom data access code required

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO 31000:2018 | Risk nodes carry ISO 31000 fields; risk-control edges model the treatment relationship |
| COSO ERM 2017 | Risk taxonomy modelled as hierarchical subgraph; COSO components map to node labels |
| NIST OSCAL | Framework requirement hierarchy is a natural tree graph; OSCAL Profile "import" relationships are graph edges |
| ISO/IEC 27001:2022 | Annex A control hierarchy as a subgraph; control-requirement mappings as typed edges |
| IIA Global Standards 2024 | Audit engagement-finding-corrective action chain as a directed graph path |
| DORA | ICT third-party provider dependency chain modelled as vendor-to-service-to-process graph path |
| BCBS 239 | Risk data lineage via graph traversal addresses BCBS 239 accuracy and completeness principles |
| ISO 3166 | Jurisdiction nodes in the graph enable geographic impact analysis |
| OCSF | Graph events can be projected into OCSF-format findings for SIEM integration |

---

## Relational Core (CRUD Operations)

```sql
-- ============================================================
-- TENANT & IDENTITY (identical to normalised model)
-- ============================================================

CREATE TABLE organization (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    industry        VARCHAR(100),
    jurisdiction    CHAR(2),
    settings        JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organization(id),
    email           VARCHAR(320) NOT NULL,
    display_name    VARCHAR(255) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    auth_provider   VARCHAR(50) DEFAULT 'local',
    auth_subject    VARCHAR(500),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, email)
);

CREATE INDEX idx_user_org ON app_user(organization_id);

CREATE TABLE role (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organization(id),
    name            VARCHAR(100) NOT NULL,
    permissions     JSONB NOT NULL DEFAULT '[]',
    UNIQUE (organization_id, name)
);

CREATE TABLE user_role (
    user_id         UUID NOT NULL REFERENCES app_user(id) ON DELETE CASCADE,
    role_id         UUID NOT NULL REFERENCES role(id) ON DELETE CASCADE,
    scope_type      VARCHAR(50),
    scope_id        UUID,
    granted_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (user_id, role_id, COALESCE(scope_id, '00000000-0000-0000-0000-000000000000'))
);

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
    jurisdiction    CHAR(2),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_bu_org ON business_unit(organization_id);
CREATE INDEX idx_bu_parent ON business_unit(parent_id);

-- ============================================================
-- RISK MANAGEMENT
-- ============================================================

CREATE TABLE risk (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organization(id),
    ref_id          VARCHAR(50) NOT NULL,
    title           VARCHAR(500) NOT NULL,
    description     TEXT,
    risk_source     TEXT,
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
    metadata        JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, ref_id)
);

CREATE INDEX idx_risk_org ON risk(organization_id);
CREATE INDEX idx_risk_status ON risk(organization_id, status);
CREATE INDEX idx_risk_residual ON risk(organization_id, residual_score DESC);

-- ============================================================
-- CONTROL LIBRARY
-- ============================================================

CREATE TABLE control (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organization(id),
    ref_id          VARCHAR(50) NOT NULL,
    title           VARCHAR(500) NOT NULL,
    description     TEXT,
    control_type    VARCHAR(30) NOT NULL DEFAULT 'preventive',
    automation_level VARCHAR(20) NOT NULL DEFAULT 'manual',
    frequency       VARCHAR(20) DEFAULT 'continuous',
    owner_id        UUID REFERENCES app_user(id),
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    effectiveness   VARCHAR(20),
    last_test_date  DATE,
    next_test_date  DATE,
    metadata        JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, ref_id)
);

CREATE INDEX idx_control_org ON control(organization_id);
CREATE INDEX idx_control_status ON control(organization_id, status);

CREATE TABLE control_test (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    control_id      UUID NOT NULL REFERENCES control(id),
    tested_by       UUID REFERENCES app_user(id),
    test_date       TIMESTAMPTZ NOT NULL DEFAULT now(),
    result          VARCHAR(20) NOT NULL,
    methodology     VARCHAR(50),
    automated       BOOLEAN NOT NULL DEFAULT false,
    evidence_notes  TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_control_test_control ON control_test(control_id);

-- ============================================================
-- COMPLIANCE FRAMEWORKS
-- ============================================================

CREATE TABLE framework (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID,
    name            VARCHAR(255) NOT NULL,
    version         VARCHAR(50),
    provider        VARCHAR(255),
    framework_type  VARCHAR(30) NOT NULL DEFAULT 'standard',
    jurisdiction    CHAR(2),
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    oscal_catalog_id VARCHAR(255),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE framework_requirement (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    framework_id    UUID NOT NULL REFERENCES framework(id) ON DELETE CASCADE,
    parent_id       UUID REFERENCES framework_requirement(id),
    ref_id          VARCHAR(100) NOT NULL,
    title           VARCHAR(500) NOT NULL,
    description     TEXT,
    depth           INT NOT NULL DEFAULT 0,
    sort_order      INT NOT NULL DEFAULT 0,
    oscal_control_id VARCHAR(255),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (framework_id, ref_id)
);

CREATE INDEX idx_fw_req_framework ON framework_requirement(framework_id);
CREATE INDEX idx_fw_req_parent ON framework_requirement(parent_id);

-- ============================================================
-- POLICY, AUDIT, VENDOR, INCIDENT (standard relational tables)
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
    approver_id     UUID REFERENCES app_user(id),
    effective_date  DATE,
    review_due_date DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, ref_id)
);

CREATE TABLE audit_engagement (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organization(id),
    ref_id          VARCHAR(50) NOT NULL,
    title           VARCHAR(500) NOT NULL,
    engagement_type VARCHAR(30) NOT NULL DEFAULT 'assurance',
    status          VARCHAR(30) NOT NULL DEFAULT 'planned',
    lead_auditor_id UUID REFERENCES app_user(id),
    business_unit_id UUID REFERENCES business_unit(id),
    planned_start   DATE,
    planned_end     DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, ref_id)
);

CREATE TABLE audit_finding (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    engagement_id   UUID NOT NULL REFERENCES audit_engagement(id),
    ref_id          VARCHAR(50) NOT NULL,
    title           VARCHAR(500) NOT NULL,
    condition       TEXT,
    criteria        TEXT,
    cause           TEXT,
    effect          TEXT,
    recommendation  TEXT,
    severity        VARCHAR(20) NOT NULL DEFAULT 'medium',
    status          VARCHAR(30) NOT NULL DEFAULT 'draft',
    finding_owner_id UUID REFERENCES app_user(id),
    due_date        DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE vendor (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organization(id),
    name            VARCHAR(255) NOT NULL,
    criticality     VARCHAR(20) NOT NULL DEFAULT 'medium',
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    jurisdiction    CHAR(2),
    risk_score      DECIMAL(5,2),
    owner_id        UUID REFERENCES app_user(id),
    metadata        JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

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
    metadata        JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, ref_id)
);
```

---

## Graph Layer

```sql
-- ============================================================
-- GRAPH LAYER — property graph on PostgreSQL
-- ============================================================
-- This layer mirrors all GRC entities and their relationships
-- as a queryable property graph. Nodes reference relational
-- entities by ID; edges capture typed relationships with properties.
--
-- Two implementation options:
--   (a) Native PostgreSQL tables with recursive CTEs (portable, no extensions)
--   (b) Apache AGE extension for Cypher query support (richer query language)
--
-- This schema uses option (a) for maximum portability.

CREATE TABLE graph_node (
    id              UUID PRIMARY KEY,       -- same UUID as relational entity
    organization_id UUID NOT NULL REFERENCES organization(id),
    node_type       VARCHAR(50) NOT NULL,   -- 'Risk', 'Control', 'Requirement', 'Framework',
                                            -- 'Policy', 'AuditEngagement', 'AuditFinding',
                                            -- 'Vendor', 'Incident', 'BusinessUnit', 'Asset', 'User'
    ref_id          VARCHAR(100),           -- human-readable identifier
    label           VARCHAR(500) NOT NULL,  -- display name (title/name from relational entity)
    properties      JSONB NOT NULL DEFAULT '{}',
    -- properties carry a subset of entity data for graph queries
    -- without needing to JOIN back to relational tables:
    -- Risk: {"status": "treating", "residual_score": 15, "treatment_type": "mitigate", "category": "Cyber"}
    -- Control: {"status": "active", "effectiveness": "effective", "control_type": "preventive"}
    -- Requirement: {"framework": "ISO 27001:2022", "depth": 1}
    -- Vendor: {"criticality": "critical", "risk_score": 7.5, "jurisdiction": "US"}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_gn_org ON graph_node(organization_id);
CREATE INDEX idx_gn_type ON graph_node(organization_id, node_type);
CREATE INDEX idx_gn_properties ON graph_node USING gin (properties);

CREATE TABLE graph_edge (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organization(id),
    source_id       UUID NOT NULL REFERENCES graph_node(id) ON DELETE CASCADE,
    target_id       UUID NOT NULL REFERENCES graph_node(id) ON DELETE CASCADE,
    edge_type       VARCHAR(50) NOT NULL,   -- relationship type (see taxonomy below)
    properties      JSONB NOT NULL DEFAULT '{}',
    -- properties carry relationship metadata:
    -- MITIGATES: {"relationship": "mitigates"}
    -- SATISFIES: {"coverage_status": "mapped"}
    -- CROSSWALKS: {"mapping_type": "equivalent", "confidence": "high"}
    -- OWNS: {"since": "2026-01-01"}
    weight          DECIMAL(5,2) DEFAULT 1.0,  -- for weighted graph algorithms
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (source_id, target_id, edge_type)    -- prevent duplicate edges
);

CREATE INDEX idx_ge_org ON graph_edge(organization_id);
CREATE INDEX idx_ge_source ON graph_edge(source_id);
CREATE INDEX idx_ge_target ON graph_edge(target_id);
CREATE INDEX idx_ge_type ON graph_edge(edge_type);
CREATE INDEX idx_ge_source_type ON graph_edge(source_id, edge_type);
CREATE INDEX idx_ge_target_type ON graph_edge(target_id, edge_type);
CREATE INDEX idx_ge_properties ON graph_edge USING gin (properties);
```

---

## Edge Type Taxonomy

```sql
-- ============================================================
-- EDGE TYPE REGISTRY (reference documentation)
-- ============================================================

CREATE TABLE edge_type_registry (
    edge_type       VARCHAR(50) PRIMARY KEY,
    source_types    TEXT[] NOT NULL,         -- allowed source node types
    target_types    TEXT[] NOT NULL,         -- allowed target node types
    description     TEXT NOT NULL,
    is_directional  BOOLEAN NOT NULL DEFAULT true
);

INSERT INTO edge_type_registry (edge_type, source_types, target_types, description) VALUES
-- Risk relationships
('MITIGATES',       ARRAY['Control'],       ARRAY['Risk'],          'Control mitigates this risk'),
('MONITORS',        ARRAY['Control'],       ARRAY['Risk'],          'Control monitors this risk'),
('COMPENSATES',     ARRAY['Control'],       ARRAY['Risk'],          'Control compensates for this risk'),
('EXPOSES',         ARRAY['Asset'],         ARRAY['Risk'],          'Asset is exposed to this risk'),
('CATEGORISED_AS',  ARRAY['Risk'],          ARRAY['RiskCategory'],  'Risk belongs to this category'),

-- Framework relationships
('SATISFIES',       ARRAY['Control'],       ARRAY['Requirement'],   'Control satisfies this framework requirement'),
('PARENT_OF',       ARRAY['Requirement'],   ARRAY['Requirement'],   'Parent requirement contains child requirement'),
('BELONGS_TO',      ARRAY['Requirement'],   ARRAY['Framework'],     'Requirement belongs to this framework'),
('CROSSWALKS_TO',   ARRAY['Requirement'],   ARRAY['Requirement'],   'Cross-framework equivalence mapping'),

-- Policy relationships
('GOVERNS',         ARRAY['Policy'],        ARRAY['Control'],       'Policy governs this control'),
('GOVERNS_RISK',    ARRAY['Policy'],        ARRAY['Risk'],          'Policy addresses this risk domain'),

-- Audit relationships
('AUDITS',          ARRAY['AuditEngagement'], ARRAY['BusinessUnit'], 'Audit engagement covers this business unit'),
('FINDING_FOR',     ARRAY['AuditFinding'],  ARRAY['Control'],       'Finding relates to this control'),
('FINDING_RISK',    ARRAY['AuditFinding'],  ARRAY['Risk'],          'Finding relates to this risk'),
('RAISED_IN',       ARRAY['AuditFinding'],  ARRAY['AuditEngagement'], 'Finding raised during this engagement'),

-- Vendor relationships
('SUPPLIES',        ARRAY['Vendor'],        ARRAY['BusinessUnit'],  'Vendor supplies services to this unit'),
('VENDOR_RISK',     ARRAY['Vendor'],        ARRAY['Risk'],          'Vendor introduces this risk'),
('DEPENDS_ON',      ARRAY['Vendor'],        ARRAY['Vendor'],        'Vendor depends on another vendor (supply chain)'),

-- Incident relationships
('INCIDENT_RISK',   ARRAY['Incident'],      ARRAY['Risk'],          'Incident materialised this risk'),
('CONTROL_FAILURE', ARRAY['Incident'],      ARRAY['Control'],       'Incident caused by control failure'),
('AFFECTED_UNIT',   ARRAY['Incident'],      ARRAY['BusinessUnit'],  'Incident affected this business unit'),

-- Regulatory change relationships
('IMPACTS',         ARRAY['RegulatoryChange'], ARRAY['Control'],    'Regulatory change impacts this control'),
('IMPACTS_FRAMEWORK', ARRAY['RegulatoryChange'], ARRAY['Framework'], 'Regulatory change impacts this framework'),

-- Ownership and organisational relationships
('OWNS',            ARRAY['User'],          ARRAY['Risk', 'Control', 'Policy', 'Vendor'], 'User owns this entity'),
('BELONGS_TO_UNIT', ARRAY['Risk', 'Control', 'Asset'], ARRAY['BusinessUnit'], 'Entity belongs to this business unit'),
('REPORTS_TO',      ARRAY['BusinessUnit'],  ARRAY['BusinessUnit'],  'Organisational reporting line')
;
```

---

## Graph Query Examples

```sql
-- ============================================================
-- GRAPH TRAVERSAL QUERIES (PostgreSQL recursive CTEs)
-- ============================================================

-- Q1: "Blast radius of a failing control — what risks, requirements, and frameworks are affected?"
-- Starting from control CTL-042, traverse all connected entities.

WITH RECURSIVE blast_radius AS (
    -- Start from the control node
    SELECT gn.id, gn.node_type, gn.ref_id, gn.label, 0 AS depth,
           ARRAY[gn.id] AS path
    FROM graph_node gn
    WHERE gn.id = 'uuid-of-ctl-042'

    UNION ALL

    -- Traverse outgoing edges from source
    SELECT gn2.id, gn2.node_type, gn2.ref_id, gn2.label, br.depth + 1,
           br.path || gn2.id
    FROM blast_radius br
    JOIN graph_edge ge ON ge.source_id = br.id
    JOIN graph_node gn2 ON gn2.id = ge.target_id
    WHERE br.depth < 3                     -- limit traversal depth
      AND NOT gn2.id = ANY(br.path)        -- prevent cycles

    UNION ALL

    -- Traverse incoming edges to target
    SELECT gn3.id, gn3.node_type, gn3.ref_id, gn3.label, br.depth + 1,
           br.path || gn3.id
    FROM blast_radius br
    JOIN graph_edge ge ON ge.target_id = br.id
    JOIN graph_node gn3 ON gn3.id = ge.source_id
    WHERE br.depth < 3
      AND NOT gn3.id = ANY(br.path)
      AND ge.edge_type IN ('MITIGATES', 'SATISFIES', 'GOVERNS', 'FINDING_FOR')
)
SELECT node_type, ref_id, label, depth
FROM blast_radius
WHERE id != 'uuid-of-ctl-042'
ORDER BY depth, node_type;

-- Q2: "Cross-framework coverage — which controls satisfy requirements
--      in BOTH ISO 27001 and SOC 2?"

SELECT gn_control.ref_id AS control_ref,
       gn_control.label AS control_title,
       COUNT(DISTINCT gn_fw.id) AS framework_count,
       array_agg(DISTINCT gn_fw.label) AS frameworks
FROM graph_node gn_control
JOIN graph_edge ge_sat ON ge_sat.source_id = gn_control.id
                       AND ge_sat.edge_type = 'SATISFIES'
JOIN graph_node gn_req ON gn_req.id = ge_sat.target_id
                       AND gn_req.node_type = 'Requirement'
JOIN graph_edge ge_bel ON ge_bel.source_id = gn_req.id
                       AND ge_bel.edge_type = 'BELONGS_TO'
JOIN graph_node gn_fw ON gn_fw.id = ge_bel.target_id
                       AND gn_fw.node_type = 'Framework'
WHERE gn_control.organization_id = 'uuid-of-org'
  AND gn_control.node_type = 'Control'
  AND gn_fw.label IN ('ISO/IEC 27001:2022', 'SOC 2')
GROUP BY gn_control.ref_id, gn_control.label
HAVING COUNT(DISTINCT gn_fw.id) = 2;

-- Q3: "Vendor supply chain — find all transitive vendor dependencies
--      for a critical vendor"

WITH RECURSIVE vendor_chain AS (
    SELECT gn.id, gn.label AS vendor_name, 0 AS depth,
           ARRAY[gn.id] AS path
    FROM graph_node gn
    WHERE gn.id = 'uuid-of-vendor'
      AND gn.node_type = 'Vendor'

    UNION ALL

    SELECT gn2.id, gn2.label, vc.depth + 1, vc.path || gn2.id
    FROM vendor_chain vc
    JOIN graph_edge ge ON ge.source_id = vc.id
                       AND ge.edge_type = 'DEPENDS_ON'
    JOIN graph_node gn2 ON gn2.id = ge.target_id
    WHERE vc.depth < 5
      AND NOT gn2.id = ANY(vc.path)
)
SELECT vendor_name, depth
FROM vendor_chain
ORDER BY depth;

-- Q4: "Risk exposure for a business unit — find all risks that affect
--      this unit or any child unit"

WITH RECURSIVE unit_tree AS (
    SELECT gn.id
    FROM graph_node gn
    WHERE gn.id = 'uuid-of-bu'

    UNION ALL

    SELECT gn2.id
    FROM unit_tree ut
    JOIN graph_edge ge ON ge.source_id = ut.id
                       AND ge.edge_type = 'REPORTS_TO'
    JOIN graph_node gn2 ON gn2.id = ge.target_id
)
SELECT DISTINCT gn_risk.ref_id, gn_risk.label,
       gn_risk.properties->>'residual_score' AS residual_score,
       gn_risk.properties->>'status' AS status
FROM unit_tree ut
JOIN graph_edge ge ON ge.target_id = ut.id
                   AND ge.edge_type = 'BELONGS_TO_UNIT'
JOIN graph_node gn_risk ON gn_risk.id = ge.source_id
                        AND gn_risk.node_type = 'Risk'
ORDER BY (gn_risk.properties->>'residual_score')::INT DESC;

-- Q5: "Regulatory impact analysis — if DORA changes, what controls,
--      frameworks, and risks are affected?"

SELECT gn.node_type, gn.ref_id, gn.label, ge.edge_type
FROM graph_node gn_reg
JOIN graph_edge ge ON ge.source_id = gn_reg.id
                   AND ge.edge_type IN ('IMPACTS', 'IMPACTS_FRAMEWORK')
JOIN graph_node gn ON gn.id = ge.target_id
WHERE gn_reg.id = 'uuid-of-dora-change'
ORDER BY gn.node_type;

-- Q6: "Conflict of interest detection — find users who own both a risk
--      and the control that mitigates it"

SELECT gn_user.label AS user_name,
       gn_risk.ref_id AS risk_ref,
       gn_risk.label AS risk_title,
       gn_control.ref_id AS control_ref,
       gn_control.label AS control_title
FROM graph_edge ge_own_risk
JOIN graph_node gn_user ON gn_user.id = ge_own_risk.source_id
                        AND gn_user.node_type = 'User'
JOIN graph_node gn_risk ON gn_risk.id = ge_own_risk.target_id
                        AND gn_risk.node_type = 'Risk'
JOIN graph_edge ge_mitigates ON ge_mitigates.target_id = gn_risk.id
                             AND ge_mitigates.edge_type = 'MITIGATES'
JOIN graph_node gn_control ON gn_control.id = ge_mitigates.source_id
                           AND gn_control.node_type = 'Control'
JOIN graph_edge ge_own_control ON ge_own_control.source_id = gn_user.id
                               AND ge_own_control.target_id = gn_control.id
                               AND ge_own_control.edge_type = 'OWNS'
WHERE ge_own_risk.edge_type = 'OWNS'
  AND gn_user.organization_id = 'uuid-of-org';
```

---

## Graph Synchronisation

```sql
-- ============================================================
-- GRAPH SYNC TRIGGERS
-- Keeps graph layer in sync with relational changes.
-- Fires on INSERT/UPDATE/DELETE of relational entities.
-- ============================================================

-- Example trigger function for risk table
CREATE OR REPLACE FUNCTION sync_risk_to_graph()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        INSERT INTO graph_node (id, organization_id, node_type, ref_id, label, properties)
        VALUES (
            NEW.id,
            NEW.organization_id,
            'Risk',
            NEW.ref_id,
            NEW.title,
            jsonb_build_object(
                'status', NEW.status,
                'category', NEW.category,
                'inherent_score', NEW.inherent_score,
                'residual_score', NEW.residual_score,
                'treatment_type', NEW.treatment_type
            )
        );
    ELSIF TG_OP = 'UPDATE' THEN
        UPDATE graph_node
        SET label = NEW.title,
            ref_id = NEW.ref_id,
            properties = jsonb_build_object(
                'status', NEW.status,
                'category', NEW.category,
                'inherent_score', NEW.inherent_score,
                'residual_score', NEW.residual_score,
                'treatment_type', NEW.treatment_type
            ),
            updated_at = now()
        WHERE id = NEW.id;
    ELSIF TG_OP = 'DELETE' THEN
        DELETE FROM graph_node WHERE id = OLD.id;
        -- graph_edge entries cascade via FK
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_risk_graph_sync
    AFTER INSERT OR UPDATE OR DELETE ON risk
    FOR EACH ROW EXECUTE FUNCTION sync_risk_to_graph();

-- Similar triggers for: control, framework_requirement, policy,
-- audit_engagement, audit_finding, vendor, incident, business_unit

-- Example trigger for risk_control junction table -> graph edges
CREATE OR REPLACE FUNCTION sync_risk_control_edge()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        INSERT INTO graph_edge (organization_id, source_id, target_id, edge_type, properties)
        SELECT r.organization_id, NEW.control_id, NEW.risk_id, 'MITIGATES',
               jsonb_build_object('relationship', 'mitigates')
        FROM risk r WHERE r.id = NEW.risk_id;
    ELSIF TG_OP = 'DELETE' THEN
        DELETE FROM graph_edge
        WHERE source_id = OLD.control_id
          AND target_id = OLD.risk_id
          AND edge_type = 'MITIGATES';
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_risk_control_edge_sync
    AFTER INSERT OR DELETE ON risk_control
    FOR EACH ROW EXECUTE FUNCTION sync_risk_control_edge();
```

---

## Junction Tables (Relational Side)

```sql
-- ============================================================
-- RELATIONAL JUNCTION TABLES
-- These are the relational source of truth for relationships.
-- Graph edges are derived from these via triggers.
-- ============================================================

-- Risk <-> Control
CREATE TABLE risk_control (
    risk_id         UUID NOT NULL REFERENCES risk(id) ON DELETE CASCADE,
    control_id      UUID NOT NULL REFERENCES control(id) ON DELETE CASCADE,
    relationship    VARCHAR(20) NOT NULL DEFAULT 'mitigates',
    PRIMARY KEY (risk_id, control_id)
);

-- Control <-> Framework Requirement
CREATE TABLE control_requirement (
    control_id      UUID NOT NULL REFERENCES control(id) ON DELETE CASCADE,
    requirement_id  UUID NOT NULL REFERENCES framework_requirement(id) ON DELETE CASCADE,
    coverage_status VARCHAR(20) NOT NULL DEFAULT 'mapped',
    PRIMARY KEY (control_id, requirement_id)
);

-- Cross-framework mapping
CREATE TABLE requirement_crosswalk (
    source_requirement_id UUID NOT NULL REFERENCES framework_requirement(id) ON DELETE CASCADE,
    target_requirement_id UUID NOT NULL REFERENCES framework_requirement(id) ON DELETE CASCADE,
    mapping_type    VARCHAR(20) NOT NULL DEFAULT 'equivalent',
    confidence      VARCHAR(10) DEFAULT 'high',
    PRIMARY KEY (source_requirement_id, target_requirement_id)
);

-- Policy <-> Control
CREATE TABLE policy_control (
    policy_id       UUID NOT NULL REFERENCES policy(id) ON DELETE CASCADE,
    control_id      UUID NOT NULL REFERENCES control(id) ON DELETE CASCADE,
    PRIMARY KEY (policy_id, control_id)
);

-- Risk <-> Asset
CREATE TABLE risk_asset (
    risk_id         UUID NOT NULL REFERENCES risk(id) ON DELETE CASCADE,
    asset_id        UUID NOT NULL REFERENCES asset(id) ON DELETE CASCADE,
    PRIMARY KEY (risk_id, asset_id)
);

CREATE TABLE asset (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organization(id),
    name            VARCHAR(255) NOT NULL,
    asset_type      VARCHAR(50) NOT NULL,
    business_unit_id UUID REFERENCES business_unit(id),
    owner_id        UUID REFERENCES app_user(id),
    metadata        JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Evidence & Audit Trail

```sql
-- ============================================================
-- EVIDENCE REPOSITORY
-- ============================================================

CREATE TABLE evidence (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organization(id),
    name            VARCHAR(500) NOT NULL,
    evidence_type   VARCHAR(30) NOT NULL DEFAULT 'document',
    file_path       TEXT,
    hash_sha256     VARCHAR(64),
    collection_method VARCHAR(20) DEFAULT 'manual',
    source_system   VARCHAR(255),
    metadata        JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE evidence_link (
    evidence_id     UUID NOT NULL REFERENCES evidence(id) ON DELETE CASCADE,
    entity_type     VARCHAR(50) NOT NULL,
    entity_id       UUID NOT NULL,
    PRIMARY KEY (evidence_id, entity_type, entity_id)
);

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

## Graph Analytics Views

```sql
-- ============================================================
-- MATERIALISED VIEWS FOR GRAPH ANALYTICS
-- ============================================================

-- Control coverage score: how many frameworks does each control satisfy?
CREATE MATERIALIZED VIEW mv_control_coverage AS
SELECT gn_control.id AS control_id,
       gn_control.ref_id AS control_ref,
       gn_control.label AS control_title,
       COUNT(DISTINCT gn_fw.id) AS framework_count,
       COUNT(DISTINCT gn_req.id) AS requirement_count,
       array_agg(DISTINCT gn_fw.label) AS frameworks
FROM graph_node gn_control
JOIN graph_edge ge ON ge.source_id = gn_control.id AND ge.edge_type = 'SATISFIES'
JOIN graph_node gn_req ON gn_req.id = ge.target_id
JOIN graph_edge ge2 ON ge2.source_id = gn_req.id AND ge2.edge_type = 'BELONGS_TO'
JOIN graph_node gn_fw ON gn_fw.id = ge2.target_id
WHERE gn_control.node_type = 'Control'
GROUP BY gn_control.id, gn_control.ref_id, gn_control.label;

CREATE UNIQUE INDEX idx_mv_control_coverage ON mv_control_coverage(control_id);

-- Risk connectivity: risks ranked by number of connected entities
CREATE MATERIALIZED VIEW mv_risk_connectivity AS
SELECT gn.id AS risk_id,
       gn.ref_id AS risk_ref,
       gn.label AS risk_title,
       gn.properties->>'residual_score' AS residual_score,
       COUNT(DISTINCT ge.id) AS edge_count,
       COUNT(DISTINCT CASE WHEN ge.edge_type = 'MITIGATES' THEN ge.source_id END) AS control_count,
       COUNT(DISTINCT CASE WHEN ge.edge_type = 'VENDOR_RISK' THEN ge.source_id END) AS vendor_count,
       COUNT(DISTINCT CASE WHEN ge.edge_type = 'FINDING_RISK' THEN ge.source_id END) AS finding_count
FROM graph_node gn
LEFT JOIN graph_edge ge ON ge.target_id = gn.id
WHERE gn.node_type = 'Risk'
GROUP BY gn.id, gn.ref_id, gn.label, gn.properties;

CREATE UNIQUE INDEX idx_mv_risk_connectivity ON mv_risk_connectivity(risk_id);

-- Orphaned controls: controls not linked to any risk or requirement
CREATE MATERIALIZED VIEW mv_orphaned_controls AS
SELECT gn.id AS control_id,
       gn.ref_id AS control_ref,
       gn.label AS control_title,
       gn.organization_id
FROM graph_node gn
WHERE gn.node_type = 'Control'
  AND NOT EXISTS (
      SELECT 1 FROM graph_edge ge
      WHERE ge.source_id = gn.id
        AND ge.edge_type IN ('MITIGATES', 'MONITORS', 'COMPENSATES', 'SATISFIES')
  );

-- Refresh materialised views periodically
-- REFRESH MATERIALIZED VIEW CONCURRENTLY mv_control_coverage;
-- REFRESH MATERIALIZED VIEW CONCURRENTLY mv_risk_connectivity;
-- REFRESH MATERIALIZED VIEW mv_orphaned_controls;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Multi-Tenancy & Identity | 4 | organization, app_user, role, user_role |
| Organisational Structure | 1 | business_unit |
| Risk Management | 1 | risk |
| Control Library | 2 | control, control_test |
| Compliance Frameworks | 2 | framework, framework_requirement |
| Policy Management | 1 | policy |
| Audit Management | 2 | audit_engagement, audit_finding |
| Third-Party & Incidents | 2 | vendor, incident |
| Assets | 1 | asset |
| Junction Tables | 5 | risk_control, control_requirement, requirement_crosswalk, policy_control, risk_asset |
| Graph Layer | 3 | graph_node, graph_edge, edge_type_registry |
| Evidence | 2 | evidence, evidence_link |
| Audit Trail | 1 | audit_log (partitioned) |
| Materialised Views | 3 | mv_control_coverage, mv_risk_connectivity, mv_orphaned_controls |
| **Total** | **30** | 27 tables + 3 materialised views |

---

## Key Design Decisions

1. **Relational tables as source of truth, graph as derived index** — the relational tables are the authoritative data store for all CRUD operations. The graph layer (graph_node, graph_edge) is a derived representation kept in sync via PostgreSQL triggers. This means the graph can be rebuilt from scratch at any time by replaying relational data, eliminating consistency concerns.

2. **PostgreSQL-native graph rather than separate graph database** — using PostgreSQL tables for the graph layer avoids the operational complexity of running a separate Neo4j or Amazon Neptune instance. For mid-market GRC platforms with tens of thousands of nodes (not billions), PostgreSQL recursive CTEs perform well. The Apache AGE extension can be added later for Cypher query support if graph query complexity grows.

3. **Typed edge taxonomy with registry** — the edge_type_registry table documents all valid edge types with allowed source/target node types. This provides graph schema documentation and enables application-level validation that edges are structurally correct (e.g., a MITIGATES edge must go from Control to Risk, not the reverse).

4. **Graph node properties carry denormalised entity data** — graph_node.properties includes a subset of the relational entity's fields (status, scores, type) so that graph traversal queries can filter and display results without JOINing back to relational tables. The trigger function selects which fields to copy to properties.

5. **Trigger-based synchronisation** — PostgreSQL AFTER triggers on relational tables and junction tables automatically create, update, and delete corresponding graph_node and graph_edge records. This ensures the graph stays in sync without application-level dual-write logic. The trade-off is slightly slower write operations due to trigger execution.

6. **Materialised views for common graph analytics** — frequently needed graph analytics (control coverage, risk connectivity, orphaned controls) are pre-computed as materialised views and refreshed periodically. This avoids expensive recursive CTEs on every dashboard load.

7. **Conflict-of-interest detection is a natural graph query** — the graph representation enables conflict-of-interest analysis (Q6 above: users who own both a risk and the control mitigating it) that would require complex multi-table self-JOINs in a purely relational model. This is a concrete differentiation for GRC platforms targeting regulated industries.

8. **Vendor supply chain as graph traversal** — DORA requires organisations to map ICT third-party provider dependencies including sub-contractors. The DEPENDS_ON edge type between vendor nodes enables recursive supply chain traversal — a query that is impractical in a flat relational vendor table.

9. **Weight column for graph algorithms** — the graph_edge.weight column enables weighted graph algorithms (shortest path, centrality analysis) for advanced analytics. For example, weighting MITIGATES edges by control effectiveness enables identification of the most critical controls in the risk mitigation network.

10. **Graph enables AI context retrieval** — when an AI agent needs to generate a risk narrative or assess the impact of a regulatory change, it can retrieve the connected subgraph (risk + controls + requirements + frameworks + vendors + incidents within N hops) as structured context for the LLM prompt. This is dramatically more informative than retrieving a single row from a relational table.

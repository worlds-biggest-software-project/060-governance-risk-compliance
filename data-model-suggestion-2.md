# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: Governance, Risk & Compliance (GRC) · Created: 2026-05-12

## Philosophy

This model treats every state change in the GRC system as an immutable event stored in an append-only event store. The event store is the single source of truth. Current state is derived by replaying events into materialised read models (projections) optimised for specific query patterns. This is the Command Query Responsibility Segregation (CQRS) pattern: writes go to the event store, reads come from projections.

In a GRC context, this approach is uniquely powerful because audit trail completeness is not a bolt-on feature — it is the architecture itself. Every risk score change, every control test result, every policy approval, every compliance assessment decision is captured as an immutable event with full actor, timestamp, and payload data. The system can answer questions that are impossible in traditional schemas: "What was our risk posture on March 15th?" or "Show me the complete history of how control CTL-042's effectiveness changed over time." Regulatory examiners can replay the exact sequence of decisions that led to the current state, with cryptographic guarantees that no events were tampered with.

Event sourcing is used in high-compliance financial systems (banking core ledgers, trading platforms, payment processors) where regulators require complete, immutable audit trails. Microsoft Azure Architecture Center, AWS Prescriptive Guidance, and Mia-Platform all document CQRS + Event Sourcing as the preferred pattern for compliance-heavy domains. In the GRC space specifically, the append-only nature eliminates the "who changed this risk score and when?" problem that plagues conventional risk registers.

**Best for:** Organisations with strict regulatory audit requirements (BFSI under BCBS 239/DORA, healthcare under HIPAA), internal audit functions that need point-in-time state reconstruction, and AI-powered analytics that benefit from rich change history data.

**Trade-offs:**
- (+) Complete, immutable audit trail is inherent — not a separate logging layer
- (+) Point-in-time state reconstruction: "what was true on date X?" is a native query
- (+) Rich change history enables AI pattern detection (anomalous risk score changes, control effectiveness trends)
- (+) Event replay enables correction without data mutation — append corrective events instead of UPDATE
- (+) Natural fit for regulatory examination: examiners can replay decision sequences
- (-) Higher implementation complexity — requires event store + projection infrastructure
- (-) Eventual consistency between event store and read models requires careful handling
- (-) Event store grows unboundedly — requires snapshotting strategy for long-lived aggregates
- (-) Schema evolution for events is harder than ALTER TABLE — events are immutable
- (-) More complex for simple CRUD operations that don't need full history
- (-) Read model rebuilds can be slow for large event volumes without snapshots

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO 31000:2018 | Risk events carry ISO 31000 field vocabulary; risk lifecycle events map to the standard's assess/treat/monitor process |
| BCBS 239 | Complete risk data lineage through event replay directly addresses BCBS 239 Principle 3 (accuracy), 4 (completeness), and 6 (adaptability) |
| COSO ERM 2017 | Event taxonomy maps to COSO components; "Review and Revision" component is natively supported by event replay |
| IIA Global Standards 2024 | Audit engagement events follow IPPF lifecycle stages; finding events carry condition/criteria/cause/effect structure |
| OCSF | GRC events can be projected into OCSF-format security events for SIEM integration |
| DORA | ICT incident events structured for DORA Article 19 incident classification and reporting |
| NIST OSCAL | Framework and control events carry OSCAL identifiers for interoperability |
| ISO 3166 | Jurisdiction fields in events use ISO 3166-1 alpha-2 codes |

---

## Event Store (Write Side)

```sql
-- ============================================================
-- EVENT STORE — the single source of truth
-- ============================================================

CREATE TABLE event_store (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id       UUID NOT NULL,          -- aggregate root identifier (e.g., risk_id, control_id)
    stream_type     VARCHAR(50) NOT NULL,   -- 'Risk', 'Control', 'Policy', 'AuditEngagement', etc.
    event_type      VARCHAR(100) NOT NULL,  -- 'RiskCreated', 'RiskScoreUpdated', 'ControlTestRecorded', etc.
    event_version   INT NOT NULL,           -- monotonically increasing per stream
    payload         JSONB NOT NULL,         -- event-specific data
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- metadata example: {"actor_id": "uuid", "actor_email": "...", "ip_address": "...",
    --                     "correlation_id": "uuid", "causation_id": "uuid", "org_id": "uuid"}
    timestamp       TIMESTAMPTZ NOT NULL DEFAULT now(),
    checksum        VARCHAR(64),            -- SHA-256 hash for tamper detection
    UNIQUE (stream_id, event_version)       -- optimistic concurrency control
) PARTITION BY RANGE (timestamp);

-- Monthly partitions
CREATE TABLE event_store_2026_01 PARTITION OF event_store
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
-- ... additional monthly partitions

CREATE INDEX idx_event_stream ON event_store(stream_id, event_version);
CREATE INDEX idx_event_type ON event_store(stream_type, event_type);
CREATE INDEX idx_event_time ON event_store(timestamp DESC);
CREATE INDEX idx_event_org ON event_store USING gin ((metadata->'org_id'));
CREATE INDEX idx_event_actor ON event_store USING gin ((metadata->'actor_id'));

-- ============================================================
-- AGGREGATE SNAPSHOTS — periodic state snapshots to speed replay
-- ============================================================

CREATE TABLE aggregate_snapshot (
    stream_id       UUID NOT NULL,
    stream_type     VARCHAR(50) NOT NULL,
    snapshot_version INT NOT NULL,          -- event_version at snapshot time
    state           JSONB NOT NULL,         -- full aggregate state as JSON
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (stream_id, snapshot_version)
);

-- ============================================================
-- ORGANIZATION & IDENTITY (minimal write-side reference data)
-- ============================================================

CREATE TABLE organization (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organization(id),
    email           VARCHAR(320) NOT NULL,
    display_name    VARCHAR(255) NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, email)
);
```

---

## Event Type Taxonomy

```sql
-- ============================================================
-- EVENT TYPE REGISTRY (documentation table — not enforced by FK)
-- ============================================================

CREATE TABLE event_type_registry (
    event_type      VARCHAR(100) PRIMARY KEY,
    stream_type     VARCHAR(50) NOT NULL,
    description     TEXT NOT NULL,
    payload_schema  JSONB,                  -- JSON Schema for payload validation
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Seed the event type registry:
INSERT INTO event_type_registry (event_type, stream_type, description) VALUES
-- Risk aggregate events
('RiskCreated',              'Risk',             'New risk identified and added to register'),
('RiskUpdated',              'Risk',             'Risk description, source, or category modified'),
('RiskScoreAssessed',        'Risk',             'Inherent or residual risk score updated'),
('RiskTreatmentSet',         'Risk',             'Treatment type and plan defined or modified'),
('RiskOwnerAssigned',        'Risk',             'Risk owner changed'),
('RiskReviewed',             'Risk',             'Periodic risk review completed'),
('RiskStatusChanged',        'Risk',             'Risk status transitioned (identified -> assessed -> treating -> ...)'),
('RiskAccepted',             'Risk',             'Risk formally accepted with rationale'),
('RiskEscalated',            'Risk',             'Risk escalated to senior management/board'),
('RiskClosed',               'Risk',             'Risk closed with closure rationale'),
('RiskControlLinked',        'Risk',             'Control linked to this risk as mitigation'),
('RiskControlUnlinked',      'Risk',             'Control removed from risk mitigation set'),
('RiskAssetLinked',          'Risk',             'Asset linked to this risk'),

-- Control aggregate events
('ControlCreated',           'Control',          'New control defined in the control library'),
('ControlUpdated',           'Control',          'Control description, type, or frequency modified'),
('ControlTestRecorded',      'Control',          'Control test performed with pass/fail/partial result'),
('ControlEffectivenessSet',  'Control',          'Overall control effectiveness rating updated'),
('ControlOwnerAssigned',     'Control',          'Control owner changed'),
('ControlRetired',           'Control',          'Control retired from active use'),
('ControlRequirementMapped', 'Control',          'Control mapped to a framework requirement'),

-- Framework events
('FrameworkImported',        'Framework',        'Compliance framework imported (e.g., from OSCAL catalog)'),
('FrameworkUpdated',         'Framework',        'Framework metadata or requirements modified'),
('RequirementCrosswalkAdded','Framework',        'Cross-framework mapping created between requirements'),

-- Policy aggregate events
('PolicyCreated',            'Policy',           'New policy drafted'),
('PolicyContentUpdated',     'Policy',           'Policy content revised, new version created'),
('PolicySubmittedForReview',  'Policy',          'Policy submitted for approval workflow'),
('PolicyApproved',           'Policy',           'Policy approved by designated approver'),
('PolicyPublished',          'Policy',           'Policy published and effective'),
('PolicyAttested',           'Policy',           'User acknowledged/attested to policy'),
('PolicyRetired',            'Policy',           'Policy retired from active use'),

-- Audit aggregate events
('AuditPlanCreated',         'AuditPlan',        'Annual audit plan created'),
('AuditPlanApproved',        'AuditPlan',        'Audit plan approved by CAE/Board'),
('AuditEngagementCreated',   'AuditEngagement',  'New audit engagement planned'),
('AuditFieldworkStarted',    'AuditEngagement',  'Audit fieldwork commenced'),
('AuditFindingRaised',       'AuditEngagement',  'Audit finding identified during fieldwork'),
('AuditFindingUpdated',      'AuditEngagement',  'Finding details, severity, or response updated'),
('AuditReportIssued',        'AuditEngagement',  'Final audit report issued'),
('CorrectiveActionCreated',  'AuditEngagement',  'Corrective action plan created for a finding'),
('CorrectiveActionCompleted','AuditEngagement',  'Corrective action completed and verified'),

-- Compliance assessment events
('AssessmentStarted',        'ComplianceAssessment', 'Compliance assessment initiated against a framework'),
('RequirementAssessed',      'ComplianceAssessment', 'Individual requirement assessed (compliant/non-compliant/partial)'),
('AssessmentCompleted',      'ComplianceAssessment', 'Compliance assessment finalised with overall score'),

-- Vendor events
('VendorOnboarded',          'Vendor',           'New vendor added to third-party register'),
('VendorAssessed',           'Vendor',           'Vendor risk assessment completed'),
('VendorRiskScoreUpdated',   'Vendor',           'Vendor risk score recalculated'),
('VendorTerminated',         'Vendor',           'Vendor relationship terminated'),

-- Incident events
('IncidentReported',         'Incident',         'Security/compliance incident reported'),
('IncidentInvestigated',     'Incident',         'Incident investigation findings recorded'),
('IncidentContained',        'Incident',         'Incident contained; immediate impact limited'),
('IncidentRemediated',       'Incident',         'Incident root cause addressed'),
('IncidentClosed',           'Incident',         'Incident closed with post-mortem'),

-- Regulatory change events
('RegulatoryChangeIdentified','RegulatoryChange','New regulatory change detected (possibly by AI agent)'),
('RegulatoryChangeAssessed',  'RegulatoryChange','Impact assessment completed'),
('RegulatoryChangeImplemented','RegulatoryChange','Required changes to controls/policies implemented')
;
```

---

## Event Payload Examples

```sql
-- Example: RiskCreated event payload
-- {
--   "ref_id": "RSK-001",
--   "title": "Unauthorised access to customer PII via API",
--   "description": "Risk of customer PII exposure through insufficient API authentication controls",
--   "risk_source": "External threat actor exploiting weak API auth",
--   "category": "Cyber",
--   "business_unit_id": "uuid-of-engineering",
--   "inherent_likelihood": 4,
--   "inherent_impact": 5,
--   "inherent_score": 20,
--   "treatment_type": "mitigate",
--   "risk_owner_id": "uuid-of-ciso"
-- }

-- Example: RiskScoreAssessed event payload
-- {
--   "assessment_type": "residual",
--   "previous_likelihood": null,
--   "previous_impact": null,
--   "previous_score": null,
--   "new_likelihood": 2,
--   "new_impact": 5,
--   "new_score": 10,
--   "rationale": "API gateway rate limiting and OAuth2 controls reduce likelihood from 4 to 2",
--   "controls_considered": ["uuid-of-ctl-001", "uuid-of-ctl-002"]
-- }

-- Example: ControlTestRecorded event payload
-- {
--   "test_date": "2026-05-10T14:30:00Z",
--   "result": "pass",
--   "methodology": "reperformance",
--   "automated": true,
--   "evidence_notes": "Automated test confirmed OAuth2 token validation on all API endpoints",
--   "evidence_ids": ["uuid-of-evidence-1"],
--   "source_system": "github_actions"
-- }

-- Example: PolicyApproved event payload
-- {
--   "version": 3,
--   "approved_by": "uuid-of-cco",
--   "approval_type": "formal",
--   "effective_date": "2026-06-01",
--   "comments": "Approved with minor wording changes to Section 4.2"
-- }

-- Example: AuditFindingRaised event payload (IIA-aligned)
-- {
--   "ref_id": "FND-2026-003",
--   "title": "Incomplete access review process for privileged accounts",
--   "condition": "17 of 42 privileged accounts had no documented quarterly access review",
--   "criteria": "ISO 27001 A.9.2.5 requires periodic review of user access rights",
--   "cause": "Manual spreadsheet-based tracking process with no escalation for overdue reviews",
--   "effect": "Increased risk of unauthorised privileged access and potential regulatory finding",
--   "severity": "high",
--   "recommendation": "Implement automated access review workflow with escalation",
--   "finding_owner_id": "uuid-of-it-director"
-- }

-- Example: RegulatoryChangeIdentified event payload
-- {
--   "title": "DORA RTS on ICT Third-Party Risk Management published",
--   "source": "EU Official Journal",
--   "source_url": "https://eur-lex.europa.eu/...",
--   "jurisdiction": "EU",
--   "effective_date": "2026-07-01",
--   "change_type": "new_regulation",
--   "industries": ["financial_services", "insurance"],
--   "ai_confidence": 0.94,
--   "affected_frameworks": ["uuid-of-dora-framework"],
--   "affected_controls": ["uuid-of-ctl-vendor-001", "uuid-of-ctl-vendor-002"]
-- }
```

---

## Read Models (Projections)

Read models are PostgreSQL tables populated by event handlers that process the event store. They are denormalised for fast query access and can be rebuilt from scratch by replaying all events.

```sql
-- ============================================================
-- READ MODEL: Current Risk Register
-- ============================================================

CREATE TABLE rm_risk (
    id              UUID PRIMARY KEY,       -- same as stream_id
    organization_id UUID NOT NULL,
    ref_id          VARCHAR(50) NOT NULL,
    title           VARCHAR(500) NOT NULL,
    description     TEXT,
    risk_source     TEXT,
    category        VARCHAR(100),
    business_unit_id UUID,
    business_unit_name VARCHAR(255),        -- denormalised for query performance

    inherent_likelihood INT,
    inherent_impact     INT,
    inherent_score      INT,

    residual_likelihood INT,
    residual_impact     INT,
    residual_score      INT,

    target_likelihood   INT,
    target_impact       INT,
    target_score        INT,

    treatment_type      VARCHAR(20),
    treatment_plan      TEXT,
    treatment_due_date  DATE,

    risk_owner_id       UUID,
    risk_owner_name     VARCHAR(255),       -- denormalised
    risk_owner_email    VARCHAR(320),       -- denormalised

    status              VARCHAR(30),
    last_review_date    DATE,
    next_review_date    DATE,
    review_frequency    VARCHAR(20),

    control_count       INT DEFAULT 0,      -- denormalised count of linked controls
    open_finding_count  INT DEFAULT 0,      -- denormalised count of open audit findings

    last_event_id       UUID,               -- last processed event for idempotency
    last_event_version  INT,
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now(),

    UNIQUE (organization_id, ref_id)
);

CREATE INDEX idx_rm_risk_org ON rm_risk(organization_id);
CREATE INDEX idx_rm_risk_status ON rm_risk(organization_id, status);
CREATE INDEX idx_rm_risk_residual ON rm_risk(organization_id, residual_score DESC);
CREATE INDEX idx_rm_risk_review ON rm_risk(organization_id, next_review_date);

-- ============================================================
-- READ MODEL: Current Control Library
-- ============================================================

CREATE TABLE rm_control (
    id              UUID PRIMARY KEY,
    organization_id UUID NOT NULL,
    ref_id          VARCHAR(50) NOT NULL,
    title           VARCHAR(500) NOT NULL,
    description     TEXT,
    control_type    VARCHAR(30),
    automation_level VARCHAR(20),
    frequency       VARCHAR(20),
    owner_id        UUID,
    owner_name      VARCHAR(255),
    status          VARCHAR(20),
    effectiveness   VARCHAR(20),

    last_test_date  DATE,
    last_test_result VARCHAR(20),
    next_test_date  DATE,
    test_count      INT DEFAULT 0,
    pass_rate       DECIMAL(5,2),           -- percentage of tests that passed

    risk_count      INT DEFAULT 0,
    requirement_count INT DEFAULT 0,

    last_event_id   UUID,
    last_event_version INT,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now(),

    UNIQUE (organization_id, ref_id)
);

CREATE INDEX idx_rm_control_org ON rm_control(organization_id);
CREATE INDEX idx_rm_control_effectiveness ON rm_control(organization_id, effectiveness);

-- ============================================================
-- READ MODEL: Compliance Posture (denormalised framework view)
-- ============================================================

CREATE TABLE rm_compliance_posture (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL,
    framework_id        UUID NOT NULL,
    framework_name      VARCHAR(255) NOT NULL,
    total_requirements  INT NOT NULL DEFAULT 0,
    compliant_count     INT NOT NULL DEFAULT 0,
    partial_count       INT NOT NULL DEFAULT 0,
    non_compliant_count INT NOT NULL DEFAULT 0,
    not_assessed_count  INT NOT NULL DEFAULT 0,
    compliance_percentage DECIMAL(5,2),
    last_assessment_date DATE,
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, framework_id)
);

CREATE INDEX idx_rm_compliance_org ON rm_compliance_posture(organization_id);

-- ============================================================
-- READ MODEL: Risk-Control-Requirement Graph (denormalised for dashboard)
-- ============================================================

CREATE TABLE rm_risk_control_link (
    risk_id         UUID NOT NULL,
    risk_ref_id     VARCHAR(50),
    risk_title      VARCHAR(500),
    control_id      UUID NOT NULL,
    control_ref_id  VARCHAR(50),
    control_title   VARCHAR(500),
    relationship    VARCHAR(20),
    control_effectiveness VARCHAR(20),
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (risk_id, control_id)
);

CREATE TABLE rm_control_requirement_link (
    control_id      UUID NOT NULL,
    control_ref_id  VARCHAR(50),
    requirement_id  UUID NOT NULL,
    requirement_ref_id VARCHAR(100),
    framework_name  VARCHAR(255),
    coverage_status VARCHAR(20),
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (control_id, requirement_id)
);

-- ============================================================
-- READ MODEL: Audit Dashboard
-- ============================================================

CREATE TABLE rm_audit_engagement (
    id              UUID PRIMARY KEY,
    organization_id UUID NOT NULL,
    ref_id          VARCHAR(50) NOT NULL,
    title           VARCHAR(500),
    engagement_type VARCHAR(30),
    status          VARCHAR(30),
    lead_auditor_name VARCHAR(255),
    planned_start   DATE,
    planned_end     DATE,
    actual_start    DATE,
    actual_end      DATE,
    finding_count   INT DEFAULT 0,
    open_finding_count INT DEFAULT 0,
    critical_finding_count INT DEFAULT 0,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_audit_org ON rm_audit_engagement(organization_id);

-- ============================================================
-- READ MODEL: Regulatory Change Tracker
-- ============================================================

CREATE TABLE rm_regulatory_change (
    id              UUID PRIMARY KEY,
    organization_id UUID NOT NULL,
    title           VARCHAR(500),
    source          VARCHAR(255),
    jurisdiction    CHAR(2),
    effective_date  DATE,
    change_type     VARCHAR(30),
    status          VARCHAR(30),
    ai_confidence   DECIMAL(3,2),
    affected_control_count INT DEFAULT 0,
    affected_framework_count INT DEFAULT 0,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_regchange_org ON rm_regulatory_change(organization_id);
CREATE INDEX idx_rm_regchange_status ON rm_regulatory_change(organization_id, status);
```

---

## Temporal Query Examples

```sql
-- ============================================================
-- POINT-IN-TIME STATE RECONSTRUCTION
-- ============================================================

-- Q: "What was the residual risk score for RSK-001 on March 15, 2026?"
-- Replay all RiskScoreAssessed events for that stream up to the target date.

SELECT payload->>'new_score' AS residual_score,
       payload->>'new_likelihood' AS residual_likelihood,
       payload->>'new_impact' AS residual_impact,
       payload->>'rationale' AS rationale,
       metadata->>'actor_email' AS assessed_by,
       timestamp AS assessed_at
FROM event_store
WHERE stream_id = 'uuid-of-rsk-001'
  AND event_type = 'RiskScoreAssessed'
  AND payload->>'assessment_type' = 'residual'
  AND timestamp <= '2026-03-15T23:59:59Z'
ORDER BY event_version DESC
LIMIT 1;

-- Q: "Show the complete audit trail for policy POL-005"
-- All events for the policy aggregate in chronological order.

SELECT event_type, payload, metadata->>'actor_email' AS actor, timestamp
FROM event_store
WHERE stream_id = 'uuid-of-pol-005'
ORDER BY event_version ASC;

-- Q: "How many control test failures occurred in Q1 2026?"
-- Aggregate across all ControlTestRecorded events.

SELECT COUNT(*) AS failed_tests,
       COUNT(DISTINCT stream_id) AS controls_with_failures
FROM event_store
WHERE stream_type = 'Control'
  AND event_type = 'ControlTestRecorded'
  AND payload->>'result' = 'fail'
  AND timestamp BETWEEN '2026-01-01' AND '2026-03-31'
  AND metadata->>'org_id' = 'uuid-of-org';

-- Q: "Show risk score trend for RSK-001 over the past 12 months"
-- Time series of score changes for trend analysis and AI pattern detection.

SELECT timestamp::DATE AS assessment_date,
       payload->>'assessment_type' AS assessment_type,
       (payload->>'new_score')::INT AS score,
       (payload->>'new_likelihood')::INT AS likelihood,
       (payload->>'new_impact')::INT AS impact,
       metadata->>'actor_email' AS assessor
FROM event_store
WHERE stream_id = 'uuid-of-rsk-001'
  AND event_type = 'RiskScoreAssessed'
  AND timestamp >= now() - INTERVAL '12 months'
ORDER BY timestamp ASC;
```

---

## Event Processing Infrastructure

```sql
-- ============================================================
-- PROJECTION TRACKING (ensures exactly-once processing)
-- ============================================================

CREATE TABLE projection_checkpoint (
    projection_name VARCHAR(100) PRIMARY KEY,   -- 'rm_risk', 'rm_control', 'rm_compliance_posture'
    last_event_id   UUID NOT NULL,
    last_timestamp  TIMESTAMPTZ NOT NULL,
    processed_count BIGINT NOT NULL DEFAULT 0,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- OUTBOX TABLE (for reliable event publishing to message queues)
-- ============================================================

CREATE TABLE event_outbox (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_id        UUID NOT NULL,          -- FK to event_store (not enforced for performance)
    topic           VARCHAR(100) NOT NULL,  -- 'grc.risk.events', 'grc.control.events'
    payload         JSONB NOT NULL,
    published       BOOLEAN NOT NULL DEFAULT false,
    published_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_outbox_unpublished ON event_outbox(published, created_at)
    WHERE published = false;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 2 | event_store (partitioned), aggregate_snapshot |
| Identity | 2 | organization, app_user |
| Event Metadata | 1 | event_type_registry |
| Read Model: Risk | 1 | rm_risk |
| Read Model: Control | 1 | rm_control |
| Read Model: Compliance | 1 | rm_compliance_posture |
| Read Model: Relationships | 2 | rm_risk_control_link, rm_control_requirement_link |
| Read Model: Audit | 1 | rm_audit_engagement |
| Read Model: Regulatory | 1 | rm_regulatory_change |
| Infrastructure | 2 | projection_checkpoint, event_outbox |
| **Total** | **14** | Write: 5 tables; Read: 7 projections; Infra: 2 |

---

## Key Design Decisions

1. **Single event store table** — all domain events across all aggregate types (Risk, Control, Policy, AuditEngagement, Vendor, Incident, etc.) go into one partitioned table rather than per-aggregate event tables. This simplifies infrastructure, enables cross-aggregate temporal queries, and makes the audit trail a single queryable surface. The `stream_type` and `event_type` columns provide sufficient filtering.

2. **JSONB payloads rather than typed event tables** — event payloads are stored as JSONB, enabling schema evolution without migrations. New event types can be added by inserting into the event_type_registry and emitting events with new payloads. Old events remain unchanged. JSON Schema validation on payloads (stored in event_type_registry) provides structural guarantees at the application layer.

3. **Optimistic concurrency via (stream_id, event_version) unique constraint** — the UNIQUE constraint on (stream_id, event_version) prevents concurrent writes to the same aggregate from creating conflicting events. The application must increment event_version for each event appended to a stream, and a constraint violation indicates a concurrency conflict requiring retry.

4. **Aggregate snapshots for performance** — long-lived aggregates (a risk that has been assessed quarterly for 5 years has 60+ events) are periodically snapshotted. State reconstruction starts from the latest snapshot and replays only subsequent events. Snapshots are not the source of truth — they are an optimisation that can be rebuilt.

5. **Denormalised read models with owner names** — read models like rm_risk carry `risk_owner_name` and `risk_owner_email` as denormalised columns to avoid JOINs in dashboard queries. These are updated when the corresponding user record changes (via a UserUpdated event). This is acceptable because read models are disposable — they can be rebuilt from events.

6. **Projection checkpoint for exactly-once processing** — the projection_checkpoint table tracks the last processed event per projection, enabling resumable, idempotent event processing. If a projection handler crashes, it resumes from the checkpoint without reprocessing or missing events.

7. **Outbox pattern for reliable event publishing** — the event_outbox table implements the transactional outbox pattern: events are written to both event_store and event_outbox in a single transaction. A background publisher reads unpublished outbox rows and publishes them to a message queue (e.g., NATS, RabbitMQ, Kafka). This guarantees at-least-once delivery to external consumers (SIEM, AI pipeline, webhooks).

8. **SHA-256 checksum for tamper detection** — each event carries a `checksum` field containing a SHA-256 hash of the payload + metadata + timestamp. This enables integrity verification during regulatory examinations: an auditor can verify that no events have been modified after the fact.

9. **Time-partitioned event store** — the event_store table is partitioned by month using PostgreSQL native range partitioning. This enables efficient retention management (old partitions can be moved to cold storage) and improves query performance for time-bounded queries. For a mid-market GRC platform generating ~10K-100K events/month, monthly partitions keep each partition manageable.

10. **Read model rebuilds are a feature, not a bug** — because projections are derived from the event store, they can be dropped and rebuilt at any time. This enables schema evolution on the read side: when a new dashboard metric is needed, add it to the projection, rebuild from events, and deploy. No data migration required.

11. **Event taxonomy as a first-class concept** — the event_type_registry table documents all known event types with descriptions and optional JSON Schema for payload validation. This makes the event vocabulary discoverable and self-documenting, critical for a GRC platform where the event taxonomy must be auditable.

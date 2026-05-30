# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: Lab Information Management System (LIMS) · Created: 2026-05-22

## Philosophy

This model treats the immutable event log as the single source of truth. Every state change in the system — sample registered, test assigned, result entered, result verified, instrument calibrated — is recorded as an append-only event in a central event store. Current state is derived by replaying events or reading from materialised read models that are projected from the event stream.

This architecture is a natural fit for regulated laboratory environments where 21 CFR Part 11 and ISO 17025 demand complete, tamper-proof audit trails. In a traditional CRUD model, the audit trail is a secondary concern bolted on via triggers or application-level logging. In an event-sourced model, the audit trail IS the data. There is no separate audit_log table because every table is, in effect, an audit log. The question "what was the status of sample X on March 15th?" is answered by replaying events up to that date — a trivial operation that would require complex temporal queries in a CRUD model.

The read side uses Command Query Responsibility Segregation (CQRS): write operations append events to the event store, while read operations query pre-built materialised views optimised for specific access patterns (sample status dashboard, pending worksheets, QC trending). This separation allows the write path to be simple and fast (append-only) while the read path can be tuned for each use case independently.

**Best for:** Pharmaceutical and clinical labs with strict 21 CFR Part 11 / GLP requirements where tamper-proof audit trails are non-negotiable, temporal queries are frequent ("show me the state of this sample at the time of the audit"), and AI/ML analytics benefit from rich event histories.

**Trade-offs:**
- (+) Audit trail is inherent, not bolted on — every change is an event with who/what/when/why
- (+) Temporal queries are trivial — replay events to any point in time
- (+) Rich event stream is ideal for AI/ML pattern detection (anomaly detection, predictive QC)
- (+) Events are immutable — tamper-proof by design, not by access control
- (+) Easy to add new read models without changing the write path
- (-) Higher storage requirements — events are never deleted, only compacted
- (-) Eventual consistency between event store and read models adds complexity
- (-) Debugging requires understanding event replay, not just SELECT queries
- (-) Initial development requires building both event handlers and projections
- (-) Reporting queries require well-maintained materialised views rather than ad-hoc JOINs

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| 21 CFR Part 11 | The entire event store IS the audit trail — every state change is an immutable event with user, timestamp, and reason; electronic signatures are events themselves |
| ALCOA+ Principles | Events are inherently Attributable (user_id), Contemporaneous (occurred_at), Original (immutable), Accurate (validated before append); Legible via read projections |
| ISO/IEC 17025:2017 | Calibration events, measurement traceability events, and corrective action events form unbroken chains; temporal replay supports assessment queries |
| ASTM E1578 | Event types map to ASTM-defined LIMS functions: sample_registered, test_assigned, result_entered, result_verified |
| HL7 FHIR R4/R5 | Read projections for DiagnosticReport, Specimen, and Observation can be generated directly from event streams for clinical interoperability |
| LOINC / SNOMED CT | Coded identifiers stored as event payload fields, propagated to read projections |
| ISO 9001:2015 | Nonconformance and CAPA workflows are modeled as event chains with full lifecycle traceability |
| GAMP 5 | Event store provides complete change history for computer system validation evidence |

---

## Event Store (Write Side)

```sql
-- ============================================================
-- CORE EVENT STORE
-- ============================================================

CREATE TABLE event_store (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    stream_type     TEXT NOT NULL,                 -- aggregate type: 'sample', 'test_request', 'instrument', etc.
    stream_id       UUID NOT NULL,                 -- aggregate ID (e.g. sample UUID)
    event_type      TEXT NOT NULL,                 -- e.g. 'sample.registered', 'result.entered', 'instrument.calibrated'
    event_version   INTEGER NOT NULL,              -- monotonically increasing per stream
    payload         JSONB NOT NULL,                -- event-specific data
    metadata        JSONB NOT NULL DEFAULT '{}',   -- cross-cutting: correlation_id, causation_id, ip_address
    user_id         UUID NOT NULL,                 -- who caused this event
    reason          TEXT,                          -- 21 CFR Part 11: reason for change
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(stream_type, stream_id, event_version)  -- optimistic concurrency control
);

-- Append-only: prevent any modification
REVOKE UPDATE, DELETE ON event_store FROM PUBLIC;

-- Primary query patterns
CREATE INDEX idx_event_stream ON event_store(stream_type, stream_id, event_version);
CREATE INDEX idx_event_tenant_time ON event_store(tenant_id, occurred_at);
CREATE INDEX idx_event_type ON event_store(event_type);
CREATE INDEX idx_event_user ON event_store(user_id);

-- Partitioning by month for performance at scale
-- In production, use declarative partitioning:
-- CREATE TABLE event_store (...) PARTITION BY RANGE (occurred_at);

-- ============================================================
-- EVENT TYPE REGISTRY (self-documenting event catalogue)
-- ============================================================

CREATE TABLE event_type_registry (
    event_type      TEXT PRIMARY KEY,              -- e.g. 'sample.registered'
    description     TEXT NOT NULL,
    payload_schema  JSONB NOT NULL,                -- JSON Schema for the payload
    version         TEXT NOT NULL DEFAULT '1.0',
    deprecated      BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- SNAPSHOTS (optional performance optimisation)
-- ============================================================

CREATE TABLE event_snapshot (
    stream_type     TEXT NOT NULL,
    stream_id       UUID NOT NULL,
    snapshot_version INTEGER NOT NULL,             -- event_version at snapshot time
    state           JSONB NOT NULL,                -- serialised aggregate state
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (stream_type, stream_id)
);
```

## Event Type Examples

```sql
-- Register the core event types with their payload schemas

INSERT INTO event_type_registry (event_type, description, payload_schema) VALUES

-- Sample lifecycle events
('sample.registered', 'A new sample was registered in the system', '{
    "type": "object",
    "properties": {
        "sample_id_human": {"type": "string"},
        "client_id": {"type": "string", "format": "uuid"},
        "sample_type": {"type": "string"},
        "sample_matrix": {"type": "string"},
        "sample_point": {"type": "string"},
        "priority": {"type": "string", "enum": ["low","normal","high","urgent"]},
        "date_sampled": {"type": "string", "format": "date-time"},
        "remarks": {"type": "string"}
    },
    "required": ["sample_id_human", "sample_type"]
}'::JSONB),

('sample.received', 'Sample was physically received at the lab', '{
    "type": "object",
    "properties": {
        "received_by": {"type": "string", "format": "uuid"},
        "condition": {"type": "string"},
        "storage_location": {"type": "string"},
        "temperature_on_receipt": {"type": "number"}
    },
    "required": ["received_by"]
}'::JSONB),

('sample.custody_transferred', 'Chain of custody transfer occurred', '{
    "type": "object",
    "properties": {
        "from_user_id": {"type": "string", "format": "uuid"},
        "to_user_id": {"type": "string", "format": "uuid"},
        "location": {"type": "string"}
    },
    "required": ["from_user_id", "to_user_id"]
}'::JSONB),

('sample.disposed', 'Sample was disposed of', '{
    "type": "object",
    "properties": {
        "disposal_method": {"type": "string"},
        "disposed_by": {"type": "string", "format": "uuid"}
    }
}'::JSONB),

-- Test lifecycle events
('test.requested', 'A test was requested for a sample', '{
    "type": "object",
    "properties": {
        "sample_id": {"type": "string", "format": "uuid"},
        "test_code": {"type": "string"},
        "test_name": {"type": "string"},
        "loinc_code": {"type": "string"},
        "method_code": {"type": "string"},
        "priority": {"type": "string"}
    },
    "required": ["sample_id", "test_code"]
}'::JSONB),

('test.assigned_to_worksheet', 'Test was assigned to a worksheet', '{
    "type": "object",
    "properties": {
        "worksheet_id": {"type": "string", "format": "uuid"},
        "analyst_id": {"type": "string", "format": "uuid"},
        "instrument_id": {"type": "string", "format": "uuid"},
        "position": {"type": "integer"}
    },
    "required": ["worksheet_id", "analyst_id"]
}'::JSONB),

-- Result events
('result.entered', 'A test result was entered', '{
    "type": "object",
    "properties": {
        "test_request_id": {"type": "string", "format": "uuid"},
        "value_numeric": {"type": "number"},
        "value_text": {"type": "string"},
        "unit": {"type": "string"},
        "instrument_id": {"type": "string", "format": "uuid"},
        "spec_lower": {"type": "number"},
        "spec_upper": {"type": "number"},
        "is_oos": {"type": "boolean"},
        "uncertainty": {"type": "number"}
    }
}'::JSONB),

('result.verified', 'A test result was verified/approved', '{
    "type": "object",
    "properties": {
        "test_request_id": {"type": "string", "format": "uuid"},
        "verifier_id": {"type": "string", "format": "uuid"},
        "verification_comment": {"type": "string"}
    },
    "required": ["test_request_id", "verifier_id"]
}'::JSONB),

('result.amended', 'A previously verified result was amended', '{
    "type": "object",
    "properties": {
        "test_request_id": {"type": "string", "format": "uuid"},
        "previous_value": {"type": "number"},
        "new_value": {"type": "number"},
        "reason": {"type": "string"}
    },
    "required": ["test_request_id", "reason"]
}'::JSONB),

-- Instrument events
('instrument.calibrated', 'Instrument calibration was performed', '{
    "type": "object",
    "properties": {
        "result": {"type": "string", "enum": ["pass","fail","adjusted"]},
        "certificate_number": {"type": "string"},
        "reference_standard": {"type": "string"},
        "uncertainty": {"type": "number"},
        "next_due": {"type": "string", "format": "date"}
    },
    "required": ["result"]
}'::JSONB),

-- E-signature events
('signature.applied', 'An electronic signature was applied', '{
    "type": "object",
    "properties": {
        "target_stream_type": {"type": "string"},
        "target_stream_id": {"type": "string", "format": "uuid"},
        "meaning": {"type": "string", "enum": ["approval","review","authorship","verification"]},
        "full_name": {"type": "string"},
        "title": {"type": "string"},
        "auth_method": {"type": "string"}
    },
    "required": ["target_stream_type", "target_stream_id", "meaning", "full_name"]
}'::JSONB),

-- QC events
('qc.result_recorded', 'A QC result was recorded', '{
    "type": "object",
    "properties": {
        "qc_type": {"type": "string"},
        "test_code": {"type": "string"},
        "value": {"type": "number"},
        "expected": {"type": "number"},
        "recovery_pct": {"type": "number"},
        "is_acceptable": {"type": "boolean"},
        "westgard_violation": {"type": "string"}
    }
}'::JSONB);
```

## Read Models (Materialised Projections)

```sql
-- ============================================================
-- PROJECTION: Current Sample State
-- ============================================================

CREATE TABLE read_sample (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    sample_id_human TEXT NOT NULL,
    client_id       UUID,
    client_name     TEXT,                          -- denormalised for read performance
    sample_type     TEXT NOT NULL,
    sample_matrix   TEXT,
    sample_point    TEXT,
    site_id         UUID,
    status          TEXT NOT NULL,
    priority        TEXT NOT NULL,
    date_sampled    TIMESTAMPTZ,
    date_received   TIMESTAMPTZ,
    date_due        TIMESTAMPTZ,
    received_by     UUID,
    storage_location TEXT,
    batch_id        UUID,
    parent_sample_id UUID,
    total_tests     INTEGER NOT NULL DEFAULT 0,
    completed_tests INTEGER NOT NULL DEFAULT 0,
    has_oos         BOOLEAN NOT NULL DEFAULT false,
    last_event_version INTEGER NOT NULL,           -- for optimistic concurrency
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);
CREATE INDEX idx_read_sample_tenant_status ON read_sample(tenant_id, status);
CREATE INDEX idx_read_sample_client ON read_sample(client_id);
CREATE INDEX idx_read_sample_human_id ON read_sample(tenant_id, sample_id_human);

-- ============================================================
-- PROJECTION: Current Test Request State
-- ============================================================

CREATE TABLE read_test_request (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    sample_id       UUID NOT NULL,
    sample_id_human TEXT NOT NULL,                 -- denormalised
    test_code       TEXT NOT NULL,
    test_name       TEXT NOT NULL,
    loinc_code      TEXT,
    method_code     TEXT,
    status          TEXT NOT NULL,
    assigned_to     UUID,
    analyst_name    TEXT,                          -- denormalised
    instrument_id   UUID,
    instrument_name TEXT,                          -- denormalised
    worksheet_id    UUID,
    priority        TEXT NOT NULL,
    due_date        TIMESTAMPTZ,
    last_event_version INTEGER NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);
CREATE INDEX idx_read_test_sample ON read_test_request(sample_id);
CREATE INDEX idx_read_test_status ON read_test_request(tenant_id, status);
CREATE INDEX idx_read_test_worksheet ON read_test_request(worksheet_id);

-- ============================================================
-- PROJECTION: Current Test Result (latest value)
-- ============================================================

CREATE TABLE read_test_result (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    test_request_id UUID NOT NULL UNIQUE,
    sample_id       UUID NOT NULL,
    sample_id_human TEXT NOT NULL,
    test_code       TEXT NOT NULL,
    test_name       TEXT NOT NULL,
    value_numeric   NUMERIC,
    value_text      TEXT,
    unit            TEXT,
    spec_lower      NUMERIC,
    spec_upper      NUMERIC,
    is_oos          BOOLEAN NOT NULL DEFAULT false,
    is_oot          BOOLEAN NOT NULL DEFAULT false,
    uncertainty     NUMERIC,
    instrument_id   UUID,
    entered_by      UUID,
    entered_at      TIMESTAMPTZ,
    verified_by     UUID,
    verified_at     TIMESTAMPTZ,
    is_verified     BOOLEAN NOT NULL DEFAULT false,
    amendment_count INTEGER NOT NULL DEFAULT 0,
    last_event_version INTEGER NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);
CREATE INDEX idx_read_result_sample ON read_test_result(sample_id);
CREATE INDEX idx_read_result_oos ON read_test_result(tenant_id, is_oos) WHERE is_oos = true;

-- ============================================================
-- PROJECTION: Current Instrument State
-- ============================================================

CREATE TABLE read_instrument (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    site_id         UUID NOT NULL,
    name            TEXT NOT NULL,
    instrument_type TEXT NOT NULL,
    serial_number   TEXT,
    model           TEXT,
    manufacturer    TEXT,
    status          TEXT NOT NULL,
    location        TEXT,
    last_calibrated DATE,
    calibration_result TEXT,
    next_calibration_due DATE,
    calibration_overdue BOOLEAN NOT NULL DEFAULT false,
    last_event_version INTEGER NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);
CREATE INDEX idx_read_instrument_tenant ON read_instrument(tenant_id);
CREATE INDEX idx_read_instrument_cal_due ON read_instrument(next_calibration_due);

-- ============================================================
-- PROJECTION: Worksheet Dashboard
-- ============================================================

CREATE TABLE read_worksheet (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    worksheet_number TEXT NOT NULL,
    analyst_id      UUID NOT NULL,
    analyst_name    TEXT NOT NULL,
    status          TEXT NOT NULL,
    instrument_id   UUID,
    instrument_name TEXT,
    total_tests     INTEGER NOT NULL DEFAULT 0,
    completed_tests INTEGER NOT NULL DEFAULT 0,
    qc_pass_count   INTEGER NOT NULL DEFAULT 0,
    qc_fail_count   INTEGER NOT NULL DEFAULT 0,
    opened_at       TIMESTAMPTZ NOT NULL,
    submitted_at    TIMESTAMPTZ,
    verified_at     TIMESTAMPTZ,
    last_event_version INTEGER NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);
CREATE INDEX idx_read_worksheet_status ON read_worksheet(tenant_id, status);
CREATE INDEX idx_read_worksheet_analyst ON read_worksheet(analyst_id);

-- ============================================================
-- PROJECTION: QC Trending (for Levey-Jennings charts)
-- ============================================================

CREATE TABLE read_qc_trend (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    test_code       TEXT NOT NULL,
    instrument_id   UUID,
    qc_type         TEXT NOT NULL,
    value           NUMERIC NOT NULL,
    expected        NUMERIC,
    recovery_pct    NUMERIC,
    z_score         NUMERIC,
    is_acceptable   BOOLEAN NOT NULL,
    westgard_violation TEXT,
    recorded_at     TIMESTAMPTZ NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL
);
CREATE INDEX idx_read_qc_trend_test ON read_qc_trend(tenant_id, test_code, recorded_at);
CREATE INDEX idx_read_qc_trend_violations ON read_qc_trend(tenant_id, westgard_violation)
    WHERE westgard_violation IS NOT NULL;

-- ============================================================
-- PROJECTION: Nonconformance / CAPA Tracker
-- ============================================================

CREATE TABLE read_nonconformance (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    nc_number       TEXT NOT NULL,
    title           TEXT NOT NULL,
    source          TEXT NOT NULL,
    severity        TEXT NOT NULL,
    status          TEXT NOT NULL,
    raised_by       UUID NOT NULL,
    raised_by_name  TEXT NOT NULL,
    raised_at       TIMESTAMPTZ NOT NULL,
    corrective_actions_count INTEGER NOT NULL DEFAULT 0,
    open_actions_count INTEGER NOT NULL DEFAULT 0,
    closed_at       TIMESTAMPTZ,
    last_event_version INTEGER NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);
CREATE INDEX idx_read_nc_status ON read_nonconformance(tenant_id, status);
```

## Reference Data Tables (Traditional Relational)

```sql
-- Reference data is NOT event-sourced — it changes infrequently
-- and does not require temporal replay. Standard relational tables.

CREATE TABLE tenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE site (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,
    country_code    CHAR(2) NOT NULL,
    timezone        TEXT NOT NULL DEFAULT 'UTC',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    email           TEXT NOT NULL,
    display_name    TEXT NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, email)
);

CREATE TABLE role (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,
    permissions     TEXT[] NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, name)
);

CREATE TABLE test_catalogue (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    test_code       TEXT NOT NULL,
    test_name       TEXT NOT NULL,
    loinc_code      TEXT,
    snomed_code     TEXT,
    method_code     TEXT,
    method_name     TEXT,
    unit            TEXT,
    result_type     TEXT NOT NULL DEFAULT 'numeric',
    detection_limit NUMERIC,
    quantitation_limit NUMERIC,
    default_spec_lower NUMERIC,
    default_spec_upper NUMERIC,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, test_code)
);

CREATE TABLE client (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,
    client_code     TEXT NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, client_code)
);
```

## Temporal Query Examples

```sql
-- ============================================================
-- EXAMPLE: Reconstruct sample state at a specific point in time
-- ============================================================

-- "What was the status of sample X on 2026-03-15 at 14:00?"
SELECT
    event_type,
    payload,
    user_id,
    occurred_at
FROM event_store
WHERE stream_type = 'sample'
  AND stream_id = '550e8400-e29b-41d4-a716-446655440000'
  AND occurred_at <= '2026-03-15 14:00:00+00'
ORDER BY event_version ASC;

-- The application replays these events to reconstruct the sample
-- aggregate at that exact point in time.

-- ============================================================
-- EXAMPLE: Full audit trail for a test result
-- ============================================================

-- "Show me every change to test request X, including who, when, why"
SELECT
    e.event_type,
    e.payload,
    e.reason,
    u.display_name AS changed_by,
    e.occurred_at
FROM event_store e
JOIN app_user u ON u.id = e.user_id
WHERE e.stream_type = 'test_request'
  AND e.stream_id = '550e8400-e29b-41d4-a716-446655440001'
ORDER BY e.event_version ASC;

-- ============================================================
-- EXAMPLE: Find all OOS events in the last 30 days
-- ============================================================

SELECT
    e.stream_id AS test_request_id,
    e.payload->>'test_code' AS test_code,
    e.payload->>'value_numeric' AS result_value,
    e.payload->>'spec_upper' AS spec_upper,
    u.display_name AS entered_by,
    e.occurred_at
FROM event_store e
JOIN app_user u ON u.id = e.user_id
WHERE e.tenant_id = current_setting('app.current_tenant_id')::UUID
  AND e.event_type = 'result.entered'
  AND (e.payload->>'is_oos')::BOOLEAN = true
  AND e.occurred_at >= now() - INTERVAL '30 days'
ORDER BY e.occurred_at DESC;

-- ============================================================
-- EXAMPLE: Instrument calibration history
-- ============================================================

SELECT
    e.payload->>'result' AS calibration_result,
    e.payload->>'certificate_number' AS certificate,
    e.payload->>'reference_standard' AS reference_standard,
    e.payload->>'next_due' AS next_due,
    u.display_name AS calibrated_by,
    e.occurred_at AS calibration_date
FROM event_store e
JOIN app_user u ON u.id = e.user_id
WHERE e.stream_type = 'instrument'
  AND e.stream_id = '550e8400-e29b-41d4-a716-446655440002'
  AND e.event_type = 'instrument.calibrated'
ORDER BY e.occurred_at DESC;

-- ============================================================
-- EXAMPLE: AI/ML feature extraction — event frequency analysis
-- ============================================================

-- "How many result amendments happen per test method per month?"
-- (useful for detecting methods with high rework rates)
SELECT
    payload->>'method_code' AS method_code,
    DATE_TRUNC('month', occurred_at) AS month,
    COUNT(*) AS amendment_count
FROM event_store
WHERE tenant_id = current_setting('app.current_tenant_id')::UUID
  AND event_type = 'result.amended'
  AND occurred_at >= now() - INTERVAL '12 months'
GROUP BY payload->>'method_code', DATE_TRUNC('month', occurred_at)
ORDER BY amendment_count DESC;
```

## Projection Rebuild Process

```sql
-- ============================================================
-- PROJECTION CHECKPOINTS
-- ============================================================

-- Track which events each projection has processed
CREATE TABLE projection_checkpoint (
    projection_name TEXT PRIMARY KEY,              -- e.g. 'read_sample', 'read_test_result'
    last_event_id   UUID NOT NULL,
    last_occurred_at TIMESTAMPTZ NOT NULL,
    events_processed BIGINT NOT NULL DEFAULT 0,
    last_rebuilt_at TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- To rebuild a projection from scratch:
-- 1. TRUNCATE the read model table
-- 2. Reset its checkpoint to the beginning
-- 3. Replay all events through the projection handler
-- 4. Update the checkpoint

-- This makes it safe to add new read models at any time —
-- just create the table, register a checkpoint, and replay.
```

## Row-Level Security

```sql
-- RLS on event store
ALTER TABLE event_store ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation_events ON event_store
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);

-- RLS on all read projections
ALTER TABLE read_sample ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation_read_sample ON read_sample
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);

ALTER TABLE read_test_request ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation_read_test ON read_test_request
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);

ALTER TABLE read_test_result ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation_read_result ON read_test_result
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);

-- Event store is append-only
CREATE POLICY event_store_insert_only ON event_store
    FOR INSERT
    WITH CHECK (tenant_id = current_setting('app.current_tenant_id')::UUID);
REVOKE UPDATE, DELETE ON event_store FROM PUBLIC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 3 | event_store, event_type_registry, event_snapshot |
| Reference Data | 6 | tenant, site, app_user, role, test_catalogue, client |
| Read Projections | 7 | read_sample, read_test_request, read_test_result, read_instrument, read_worksheet, read_qc_trend, read_nonconformance |
| Infrastructure | 1 | projection_checkpoint |
| **Total** | **17** | Far fewer tables than normalized; complexity shifts to event handlers and projections |

---

## Key Design Decisions

1. **Single event_store table for all aggregates** — rather than one event table per aggregate type, a single table with stream_type discrimination simplifies infrastructure while the (stream_type, stream_id, event_version) unique constraint provides optimistic concurrency control.

2. **JSONB payloads with JSON Schema validation** — event payloads are flexible JSONB, but each event_type has a registered JSON Schema in event_type_registry. Validation is enforced at the application layer before append, ensuring structural integrity without rigid column definitions.

3. **Reference data is NOT event-sourced** — tenant, site, user, test catalogue, and client are standard relational tables. These change infrequently, do not require temporal replay, and are simpler to manage as traditional CRUD. The event store is reserved for operational laboratory data where audit trails matter.

4. **Denormalised read projections** — read_sample includes client_name, read_test_request includes analyst_name and instrument_name. This eliminates JOINs on the read path, which is critical for dashboard performance. The trade-off is that projection handlers must update these denormalised fields when reference data changes.

5. **Projection checkpoints enable safe rebuilds** — any read model can be rebuilt from scratch by truncating and replaying. This makes it safe to add new projections (e.g. a new AI analytics view) without modifying the event store.

6. **Event partitioning by time** — the event_store should be partitioned by occurred_at (monthly or quarterly) in production. This keeps individual partitions manageable and enables efficient range queries for temporal analysis.

7. **Electronic signatures as events** — rather than a separate e_signature table, signatures are signature.applied events in the event store. This means the audit trail for "who signed what and when" is part of the same immutable stream, not a separate table that could be manipulated independently.

8. **Rich event stream enables AI/ML** — the event store is the ideal training dataset for predictive models. Event frequency analysis, temporal patterns (how long between result.entered and result.verified), and anomaly detection can all be performed directly on the event stream without ETL.

9. **Eventual consistency is acceptable for LIMS dashboards** — lab analysts do not need sub-millisecond consistency on dashboards. A projection lag of 100-500ms is invisible to users refreshing a sample status page, but it dramatically simplifies the architecture.

10. **Immutability enforced at database level** — REVOKE UPDATE/DELETE on event_store means even a compromised application layer cannot modify historical events. Combined with PostgreSQL's WAL and backup strategy, this provides defence-in-depth for 21 CFR Part 11 compliance.

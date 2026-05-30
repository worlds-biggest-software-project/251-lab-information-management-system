# Data Model Suggestion 4: Graph-Relational Hybrid

> Project: Lab Information Management System (LIMS) · Created: 2026-05-22

## Philosophy

This model combines a relational backbone for operational CRUD with a property graph layer for relationship-heavy queries. The core laboratory workflow (samples, tests, results, instruments) lives in conventional PostgreSQL tables, but a parallel graph structure -- implemented using `graph_node` and `graph_edge` tables in PostgreSQL (no separate graph database required) -- captures the rich web of relationships that define laboratory traceability.

In a laboratory, almost every entity is connected to every other entity through chains that auditors, quality managers, and compliance officers need to traverse. A single test result connects to: the sample it came from, the instrument that produced it, the calibration that validated the instrument, the reference standard that calibrated the instrument, the reagent lots consumed, the analyst who performed the test, the reviewer who approved it, the electronic signature that authorized the release, the report that published it, and the client who received it. In a normalized relational model, answering "show me everything connected to this OOS result" requires knowing the schema intimately and writing complex multi-table joins. In a graph model, it is a single traversal query: "start at this result node and walk all edges up to depth N."

This architecture is particularly powerful for three LIMS-specific use cases: (1) metrological traceability chains (ISO 17025 Section 6.5) where auditors need to follow the unbroken path from a result through instruments, calibrations, and reference standards back to SI units; (2) impact analysis when an instrument fails calibration and the lab needs to identify every result produced since the last valid calibration; (3) conflict-of-interest and independence checks where the reviewer of a result must not be the analyst who produced it. All three are graph traversal problems that are awkward in SQL but natural in a graph model.

The implementation uses PostgreSQL exclusively -- no Neo4j or separate graph database. The `graph_node` and `graph_edge` tables use PostgreSQL's `ltree` extension for hierarchical path queries and recursive CTEs for multi-hop traversals. This avoids the operational complexity of running a separate database while still providing graph query capabilities.

**Best for:** Laboratories with complex traceability requirements (ISO 17025 calibration chains, GLP study traceability), labs that need impact analysis when instruments fail, multi-site labs with cross-site sample transfers, and environments where relationship discovery and visualization are important for quality management.

**Trade-offs:**

- (+) Traceability chain traversal is a single graph query, not a multi-table JOIN cascade
- (+) Impact analysis ("what results are affected by this failed calibration?") is trivial
- (+) Relationship visualization can be generated directly from graph data
- (+) Adding new relationship types requires only new edge types, not schema changes
- (+) Natural fit for ISO 17025 metrological traceability documentation
- (-) Dual storage (relational + graph) means writes are more expensive
- (-) Developers must understand both relational and graph query patterns
- (-) Graph consistency must be maintained alongside relational consistency
- (-) More complex application layer to keep both stores in sync
- (-) Over-engineering for simple labs with straightforward workflows

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO/IEC 17025:2017 | Metrological traceability chain (result -> instrument -> calibration -> reference_standard -> SI) modeled as graph edges; Clause 6.5 traversal is a single recursive query |
| 21 CFR Part 11 | Audit trail remains relational (append-only); electronic signatures create graph edges linking signer to signed entity |
| ALCOA+ | Graph edges capture the "who did what to what" relationships; temporal edge properties record "when" |
| ASTM E1578 | Relational tables use ASTM vocabulary for core entities; graph extends these with relationship semantics |
| HL7 FHIR R4/R5 | FHIR resource references (DiagnosticReport -> Specimen, Observation -> Device) map directly to graph edges |
| LOINC / SNOMED CT | Coded identifiers stored as node properties; enable cross-reference traversals |
| ISO 9001:2015 | CAPA relationships (nonconformance -> root cause -> corrective action -> effectiveness review) are natural graph chains |
| GLP / GALP | Study traceability from raw data through derived results to final report modeled as directed acyclic graph |

---

## Graph Layer

```sql
-- ============================================================
-- GRAPH INFRASTRUCTURE
-- ============================================================
-- The graph layer sits alongside relational tables.
-- Every significant entity gets a node; every significant relationship
-- gets an edge. The relational tables are the source of truth for
-- operational data; the graph layer is the source of truth for
-- relationships and traceability.

CREATE EXTENSION IF NOT EXISTS ltree;

CREATE TABLE graph_node (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    node_type       TEXT NOT NULL,
    -- Node types correspond to relational table names:
    -- 'sample', 'analysis', 'result', 'instrument', 'calibration',
    -- 'reference_standard', 'reagent_lot', 'worksheet', 'report',
    -- 'user', 'client', 'nonconformance', 'corrective_action',
    -- 'e_signature', 'qc_result', 'stability_study'
    entity_id       UUID NOT NULL,                    -- FK to the relational table row
    label           TEXT NOT NULL,                    -- human-readable label (e.g. "Sample WQ-2026-001234")
    properties      JSONB NOT NULL DEFAULT '{}',      -- cached properties for graph queries without JOINing back
    -- Example properties for a 'result' node:
    -- {
    --   "test_code": "LEAD",
    --   "value": 0.023,
    --   "unit": "mg/L",
    --   "is_oos": true,
    --   "status": "verified"
    -- }
    path            LTREE,                            -- hierarchical path for tree-structured entities
    -- Example paths:
    -- 'tenant.site_boston.lab_env'  (for a lab)
    -- 'tenant.site_boston.lab_env.sample_WQ2026001234'  (for a sample)
    -- 'tenant.site_boston.lab_env.sample_WQ2026001234.analysis_lead'  (for an analysis)
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, node_type, entity_id)
);

CREATE INDEX idx_gnode_tenant_type ON graph_node(tenant_id, node_type);
CREATE INDEX idx_gnode_entity ON graph_node(entity_id);
CREATE INDEX idx_gnode_path ON graph_node USING GIST (path);
CREATE INDEX idx_gnode_properties ON graph_node USING GIN (properties);

CREATE TABLE graph_edge (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    from_node_id    UUID NOT NULL REFERENCES graph_node(id),
    to_node_id      UUID NOT NULL REFERENCES graph_node(id),
    edge_type       TEXT NOT NULL,
    -- Edge types define the relationship semantics:
    -- 'tested_on'          : analysis -> instrument
    -- 'calibrated_with'    : calibration -> reference_standard
    -- 'calibration_of'     : calibration -> instrument
    -- 'produced_result'    : analysis -> result
    -- 'belongs_to_sample'  : analysis -> sample
    -- 'performed_by'       : analysis -> user (analyst)
    -- 'reviewed_by'        : result -> user (reviewer)
    -- 'signed_by'          : e_signature -> user
    -- 'signature_on'       : e_signature -> (report | result | ...)
    -- 'used_reagent'       : analysis -> reagent_lot
    -- 'included_in_report' : sample -> report
    -- 'submitted_to'       : report -> client
    -- 'parent_of'          : sample -> sample (aliquot)
    -- 'transferred_custody': sample -> user (with temporal properties)
    -- 'triggered_by'       : corrective_action -> nonconformance
    -- 'assigned_to_worksheet': analysis -> worksheet
    -- 'qc_for_method'      : qc_result -> test_method
    -- 'stability_timepoint': stability_study -> sample
    -- 'traceability_chain' : result -> calibration -> reference_standard (composite)
    properties      JSONB NOT NULL DEFAULT '{}',
    -- Edge properties carry relationship metadata:
    -- {
    --   "role": "primary_analyst",
    --   "effective_from": "2026-05-20T09:00:00Z",
    --   "effective_to": "2026-05-20T17:00:00Z",
    --   "confidence": 1.0,
    --   "notes": "Calibration performed before analysis batch"
    -- }
    weight          NUMERIC DEFAULT 1.0,              -- for weighted traversals (e.g. priority routing)
    is_active       BOOLEAN NOT NULL DEFAULT true,    -- soft-delete for temporal edges
    effective_from  TIMESTAMPTZ NOT NULL DEFAULT now(),
    effective_to    TIMESTAMPTZ,                      -- null = currently active
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_gedge_from ON graph_edge(from_node_id);
CREATE INDEX idx_gedge_to ON graph_edge(to_node_id);
CREATE INDEX idx_gedge_type ON graph_edge(edge_type);
CREATE INDEX idx_gedge_tenant_type ON graph_edge(tenant_id, edge_type);
CREATE INDEX idx_gedge_temporal ON graph_edge(effective_from, effective_to);
CREATE INDEX idx_gedge_properties ON graph_edge USING GIN (properties);

-- Prevent self-loops
ALTER TABLE graph_edge ADD CONSTRAINT no_self_loop
    CHECK (from_node_id != to_node_id);
```

## Relational Core: Operational Tables

```sql
-- ============================================================
-- TENANT & USERS (standard relational)
-- ============================================================

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
CREATE INDEX idx_site_tenant ON site(tenant_id);

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

CREATE TABLE user_role (
    user_id         UUID NOT NULL REFERENCES app_user(id),
    role_id         UUID NOT NULL REFERENCES role(id),
    site_id         UUID REFERENCES site(id),
    granted_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (user_id, role_id)
);

CREATE TABLE client (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,
    client_code     TEXT NOT NULL,
    contact_email   TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, client_code)
);

-- ============================================================
-- SAMPLE MANAGEMENT
-- ============================================================

CREATE TABLE sample_type (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,
    snomed_code     TEXT,
    category        TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, name)
);

CREATE TABLE sample (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    site_id         UUID NOT NULL REFERENCES site(id),
    sample_id_human TEXT NOT NULL,
    client_id       UUID REFERENCES client(id),
    sample_type_id  UUID NOT NULL REFERENCES sample_type(id),
    parent_sample_id UUID REFERENCES sample(id),
    status          TEXT NOT NULL DEFAULT 'registered',
    priority        TEXT NOT NULL DEFAULT 'normal',
    date_sampled    TIMESTAMPTZ,
    date_received   TIMESTAMPTZ,
    date_due        TIMESTAMPTZ,
    received_by     UUID REFERENCES app_user(id),
    storage_location TEXT,
    batch_id        UUID,
    remarks         TEXT,
    created_by      UUID NOT NULL REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, sample_id_human)
);
CREATE INDEX idx_sample_tenant_status ON sample(tenant_id, status);
CREATE INDEX idx_sample_client ON sample(client_id);

-- ============================================================
-- TEST DEFINITIONS & METHODS
-- ============================================================

CREATE TABLE test_method (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,
    method_code     TEXT NOT NULL,
    standard_ref    TEXT,
    version         TEXT NOT NULL DEFAULT '1.0',
    is_accredited   BOOLEAN NOT NULL DEFAULT false,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, method_code, version)
);

CREATE TABLE test_definition (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,
    test_code       TEXT NOT NULL,
    loinc_code      TEXT,
    test_method_id  UUID NOT NULL REFERENCES test_method(id),
    unit            TEXT,
    decimal_places  INTEGER DEFAULT 2,
    result_type     TEXT NOT NULL DEFAULT 'numeric',
    detection_limit NUMERIC,
    quantitation_limit NUMERIC,
    spec_lower      NUMERIC,
    spec_upper      NUMERIC,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, test_code)
);

-- ============================================================
-- ANALYSIS & RESULTS
-- ============================================================

CREATE TABLE analysis (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    sample_id       UUID NOT NULL REFERENCES sample(id),
    test_definition_id UUID NOT NULL REFERENCES test_definition(id),
    instrument_id   UUID,
    analyst_id      UUID REFERENCES app_user(id),
    worksheet_id    UUID,
    status          TEXT NOT NULL DEFAULT 'pending',
    priority        TEXT NOT NULL DEFAULT 'normal',
    due_date        TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_analysis_sample ON analysis(sample_id);
CREATE INDEX idx_analysis_status ON analysis(tenant_id, status);

CREATE TABLE result (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    analysis_id     UUID NOT NULL REFERENCES analysis(id),
    result_numeric  NUMERIC,
    result_text     TEXT,
    unit            TEXT,
    spec_lower      NUMERIC,
    spec_upper      NUMERIC,
    is_oos          BOOLEAN NOT NULL DEFAULT false,
    is_oot          BOOLEAN NOT NULL DEFAULT false,
    uncertainty     NUMERIC,
    entered_by      UUID NOT NULL REFERENCES app_user(id),
    entered_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    reviewed_by     UUID REFERENCES app_user(id),
    reviewed_at     TIMESTAMPTZ,
    verified_by     UUID REFERENCES app_user(id),
    verified_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_result_analysis ON result(analysis_id);
CREATE INDEX idx_result_oos ON result(is_oos) WHERE is_oos = true;

-- ============================================================
-- INSTRUMENTS & CALIBRATION
-- ============================================================

CREATE TABLE instrument (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    site_id         UUID NOT NULL REFERENCES site(id),
    name            TEXT NOT NULL,
    instrument_type TEXT NOT NULL,
    manufacturer    TEXT,
    model           TEXT,
    serial_number   TEXT,
    status          TEXT NOT NULL DEFAULT 'active',
    connection_config JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, name)
);

ALTER TABLE analysis
    ADD CONSTRAINT fk_analysis_instrument
    FOREIGN KEY (instrument_id) REFERENCES instrument(id);

CREATE TABLE reference_standard (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,
    code            TEXT NOT NULL,
    manufacturer    TEXT,
    lot_number      TEXT,
    certified_value NUMERIC,
    uncertainty     NUMERIC,
    unit            TEXT,
    certificate_number TEXT,
    traceability_chain TEXT,                          -- "NIST -> SI via NVLAP Lab #200XXX"
    expiry_date     DATE,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, code)
);

CREATE TABLE calibration (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    instrument_id   UUID NOT NULL REFERENCES instrument(id),
    reference_standard_id UUID REFERENCES reference_standard(id),
    calibration_date DATE NOT NULL,
    next_due_date   DATE NOT NULL,
    calibration_type TEXT NOT NULL,
    performed_by    UUID REFERENCES app_user(id),
    result          TEXT NOT NULL,
    certificate_number TEXT,
    uncertainty     NUMERIC,
    remarks         TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_calibration_instrument ON calibration(instrument_id);
CREATE INDEX idx_calibration_next_due ON calibration(next_due_date);

-- ============================================================
-- WORKSHEETS, QC, BATCHES
-- ============================================================

CREATE TABLE worksheet (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    worksheet_number TEXT NOT NULL,
    analyst_id      UUID NOT NULL REFERENCES app_user(id),
    instrument_id   UUID REFERENCES instrument(id),
    status          TEXT NOT NULL DEFAULT 'open',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, worksheet_number)
);

ALTER TABLE analysis
    ADD CONSTRAINT fk_analysis_worksheet
    FOREIGN KEY (worksheet_id) REFERENCES worksheet(id);

CREATE TABLE batch (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    batch_number    TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'open',
    created_by      UUID NOT NULL REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, batch_number)
);

ALTER TABLE sample
    ADD CONSTRAINT fk_sample_batch
    FOREIGN KEY (batch_id) REFERENCES batch(id);

CREATE TABLE reagent_lot (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,
    lot_number      TEXT NOT NULL,
    manufacturer    TEXT,
    expiry_date     DATE,
    status          TEXT NOT NULL DEFAULT 'in_use',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE qc_result (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    worksheet_id    UUID NOT NULL REFERENCES worksheet(id),
    test_definition_id UUID NOT NULL REFERENCES test_definition(id),
    instrument_id   UUID REFERENCES instrument(id),
    qc_type         TEXT NOT NULL,
    result_value    NUMERIC,
    expected_value  NUMERIC,
    recovery_pct    NUMERIC,
    is_acceptable   BOOLEAN NOT NULL DEFAULT true,
    westgard_violation TEXT,
    entered_by      UUID NOT NULL REFERENCES app_user(id),
    entered_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_qc_worksheet ON qc_result(worksheet_id);

-- ============================================================
-- REPORTING & COMPLIANCE
-- ============================================================

CREATE TABLE report (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    report_number   TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'draft',
    issued_at       TIMESTAMPTZ,
    issued_by       UUID REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, report_number)
);

CREATE TABLE nonconformance (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    nc_number       TEXT NOT NULL,
    title           TEXT NOT NULL,
    description     TEXT NOT NULL,
    source          TEXT NOT NULL,
    severity        TEXT NOT NULL DEFAULT 'minor',
    status          TEXT NOT NULL DEFAULT 'open',
    raised_by       UUID NOT NULL REFERENCES app_user(id),
    raised_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    closed_at       TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, nc_number)
);

CREATE TABLE corrective_action (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    nonconformance_id UUID NOT NULL REFERENCES nonconformance(id),
    description     TEXT NOT NULL,
    assigned_to     UUID NOT NULL REFERENCES app_user(id),
    status          TEXT NOT NULL DEFAULT 'open',
    due_date        DATE,
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- AUDIT TRAIL & E-SIGNATURES (relational, append-only)
-- ============================================================

CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    user_id         UUID NOT NULL,
    user_display    TEXT NOT NULL,
    action          TEXT NOT NULL,
    entity_type     TEXT NOT NULL,
    entity_id       UUID NOT NULL,
    old_values      JSONB,
    new_values      JSONB,
    reason          TEXT,
    ip_address      INET,
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);
REVOKE UPDATE, DELETE ON audit_log FROM PUBLIC;
CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_tenant_time ON audit_log(tenant_id, occurred_at);

CREATE TABLE e_signature (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    signer_id       UUID NOT NULL REFERENCES app_user(id),
    entity_type     TEXT NOT NULL,
    entity_id       UUID NOT NULL,
    meaning         TEXT NOT NULL,
    full_name       TEXT NOT NULL,
    auth_method     TEXT NOT NULL DEFAULT 'password',
    signed_at       TIMESTAMPTZ NOT NULL DEFAULT now()
);
REVOKE UPDATE, DELETE ON e_signature FROM PUBLIC;
CREATE INDEX idx_esig_entity ON e_signature(entity_type, entity_id);
```

## Graph Population Triggers

```sql
-- ============================================================
-- AUTOMATIC GRAPH POPULATION
-- ============================================================
-- These trigger functions maintain the graph layer automatically
-- when relational tables are modified. This ensures the graph
-- stays in sync with the relational source of truth.

-- Helper function: create or update a graph node
CREATE OR REPLACE FUNCTION upsert_graph_node(
    p_tenant_id UUID,
    p_node_type TEXT,
    p_entity_id UUID,
    p_label TEXT,
    p_properties JSONB DEFAULT '{}',
    p_path LTREE DEFAULT NULL
) RETURNS UUID AS $$
DECLARE
    v_node_id UUID;
BEGIN
    INSERT INTO graph_node (tenant_id, node_type, entity_id, label, properties, path)
    VALUES (p_tenant_id, p_node_type, p_entity_id, p_label, p_properties, p_path)
    ON CONFLICT (tenant_id, node_type, entity_id)
    DO UPDATE SET
        label = EXCLUDED.label,
        properties = EXCLUDED.properties,
        path = COALESCE(EXCLUDED.path, graph_node.path),
        updated_at = now()
    RETURNING id INTO v_node_id;
    RETURN v_node_id;
END;
$$ LANGUAGE plpgsql;

-- Helper function: create a graph edge if it doesn't exist
CREATE OR REPLACE FUNCTION ensure_graph_edge(
    p_tenant_id UUID,
    p_from_node_id UUID,
    p_to_node_id UUID,
    p_edge_type TEXT,
    p_properties JSONB DEFAULT '{}'
) RETURNS UUID AS $$
DECLARE
    v_edge_id UUID;
BEGIN
    -- Check if edge already exists
    SELECT id INTO v_edge_id
    FROM graph_edge
    WHERE from_node_id = p_from_node_id
      AND to_node_id = p_to_node_id
      AND edge_type = p_edge_type
      AND is_active = true;

    IF v_edge_id IS NULL THEN
        INSERT INTO graph_edge (tenant_id, from_node_id, to_node_id, edge_type, properties)
        VALUES (p_tenant_id, p_from_node_id, p_to_node_id, p_edge_type, p_properties)
        RETURNING id INTO v_edge_id;
    END IF;

    RETURN v_edge_id;
END;
$$ LANGUAGE plpgsql;

-- Trigger: when a sample is created, create a graph node
CREATE OR REPLACE FUNCTION trg_sample_graph() RETURNS TRIGGER AS $$
DECLARE
    v_node_id UUID;
    v_client_node_id UUID;
    v_parent_node_id UUID;
BEGIN
    -- Create sample node
    v_node_id := upsert_graph_node(
        NEW.tenant_id, 'sample', NEW.id,
        'Sample ' || NEW.sample_id_human,
        jsonb_build_object(
            'sample_id_human', NEW.sample_id_human,
            'status', NEW.status,
            'priority', NEW.priority
        )
    );

    -- Edge: sample -> client (submitted_by)
    IF NEW.client_id IS NOT NULL THEN
        SELECT gn.id INTO v_client_node_id
        FROM graph_node gn
        WHERE gn.node_type = 'client' AND gn.entity_id = NEW.client_id;

        IF v_client_node_id IS NOT NULL THEN
            PERFORM ensure_graph_edge(NEW.tenant_id, v_node_id, v_client_node_id, 'submitted_by');
        END IF;
    END IF;

    -- Edge: sample -> parent_sample (aliquot_of)
    IF NEW.parent_sample_id IS NOT NULL THEN
        SELECT gn.id INTO v_parent_node_id
        FROM graph_node gn
        WHERE gn.node_type = 'sample' AND gn.entity_id = NEW.parent_sample_id;

        IF v_parent_node_id IS NOT NULL THEN
            PERFORM ensure_graph_edge(NEW.tenant_id, v_node_id, v_parent_node_id, 'aliquot_of');
        END IF;
    END IF;

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER sample_graph_trigger
    AFTER INSERT OR UPDATE ON sample
    FOR EACH ROW EXECUTE FUNCTION trg_sample_graph();

-- Trigger: when an analysis is created, create node + edges
CREATE OR REPLACE FUNCTION trg_analysis_graph() RETURNS TRIGGER AS $$
DECLARE
    v_node_id UUID;
    v_sample_node_id UUID;
    v_instrument_node_id UUID;
    v_analyst_node_id UUID;
    v_worksheet_node_id UUID;
    v_test_name TEXT;
BEGIN
    SELECT name INTO v_test_name FROM test_definition WHERE id = NEW.test_definition_id;

    v_node_id := upsert_graph_node(
        NEW.tenant_id, 'analysis', NEW.id,
        'Analysis: ' || COALESCE(v_test_name, 'Unknown'),
        jsonb_build_object('status', NEW.status, 'test_name', v_test_name)
    );

    -- Edge: analysis -> sample
    SELECT gn.id INTO v_sample_node_id
    FROM graph_node gn WHERE gn.node_type = 'sample' AND gn.entity_id = NEW.sample_id;
    IF v_sample_node_id IS NOT NULL THEN
        PERFORM ensure_graph_edge(NEW.tenant_id, v_node_id, v_sample_node_id, 'belongs_to_sample');
    END IF;

    -- Edge: analysis -> instrument
    IF NEW.instrument_id IS NOT NULL THEN
        SELECT gn.id INTO v_instrument_node_id
        FROM graph_node gn WHERE gn.node_type = 'instrument' AND gn.entity_id = NEW.instrument_id;
        IF v_instrument_node_id IS NOT NULL THEN
            PERFORM ensure_graph_edge(NEW.tenant_id, v_node_id, v_instrument_node_id, 'tested_on');
        END IF;
    END IF;

    -- Edge: analysis -> analyst
    IF NEW.analyst_id IS NOT NULL THEN
        SELECT gn.id INTO v_analyst_node_id
        FROM graph_node gn WHERE gn.node_type = 'user' AND gn.entity_id = NEW.analyst_id;
        IF v_analyst_node_id IS NOT NULL THEN
            PERFORM ensure_graph_edge(NEW.tenant_id, v_node_id, v_analyst_node_id, 'performed_by');
        END IF;
    END IF;

    -- Edge: analysis -> worksheet
    IF NEW.worksheet_id IS NOT NULL THEN
        SELECT gn.id INTO v_worksheet_node_id
        FROM graph_node gn WHERE gn.node_type = 'worksheet' AND gn.entity_id = NEW.worksheet_id;
        IF v_worksheet_node_id IS NOT NULL THEN
            PERFORM ensure_graph_edge(NEW.tenant_id, v_node_id, v_worksheet_node_id, 'assigned_to_worksheet');
        END IF;
    END IF;

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER analysis_graph_trigger
    AFTER INSERT OR UPDATE ON analysis
    FOR EACH ROW EXECUTE FUNCTION trg_analysis_graph();

-- Trigger: when a calibration is recorded, create node + edges
CREATE OR REPLACE FUNCTION trg_calibration_graph() RETURNS TRIGGER AS $$
DECLARE
    v_node_id UUID;
    v_instrument_node_id UUID;
    v_refstd_node_id UUID;
    v_tenant_id UUID;
BEGIN
    SELECT i.tenant_id INTO v_tenant_id FROM instrument i WHERE i.id = NEW.instrument_id;

    v_node_id := upsert_graph_node(
        v_tenant_id, 'calibration', NEW.id,
        'Calibration ' || NEW.calibration_date::TEXT || ' (' || NEW.result || ')',
        jsonb_build_object(
            'result', NEW.result,
            'calibration_date', NEW.calibration_date,
            'next_due', NEW.next_due_date,
            'certificate', NEW.certificate_number
        )
    );

    -- Edge: calibration -> instrument
    SELECT gn.id INTO v_instrument_node_id
    FROM graph_node gn WHERE gn.node_type = 'instrument' AND gn.entity_id = NEW.instrument_id;
    IF v_instrument_node_id IS NOT NULL THEN
        PERFORM ensure_graph_edge(v_tenant_id, v_node_id, v_instrument_node_id, 'calibration_of');
    END IF;

    -- Edge: calibration -> reference_standard (traceability chain)
    IF NEW.reference_standard_id IS NOT NULL THEN
        SELECT gn.id INTO v_refstd_node_id
        FROM graph_node gn WHERE gn.node_type = 'reference_standard' AND gn.entity_id = NEW.reference_standard_id;
        IF v_refstd_node_id IS NOT NULL THEN
            PERFORM ensure_graph_edge(v_tenant_id, v_node_id, v_refstd_node_id, 'calibrated_with');
        END IF;
    END IF;

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER calibration_graph_trigger
    AFTER INSERT ON calibration
    FOR EACH ROW EXECUTE FUNCTION trg_calibration_graph();
```

## Graph Query Examples

```sql
-- ============================================================
-- QUERY 1: Full traceability chain for a result
-- "Show me the complete chain from this result back to SI units"
-- ============================================================

-- Starting from a result, walk: result -> analysis -> instrument -> calibration -> reference_standard
WITH RECURSIVE trace AS (
    -- Start at the result node
    SELECT
        gn.id AS node_id,
        gn.node_type,
        gn.label,
        gn.properties,
        0 AS depth,
        ARRAY[gn.id] AS path
    FROM graph_node gn
    WHERE gn.node_type = 'result'
      AND gn.entity_id = '550e8400-e29b-41d4-a716-446655440000'  -- the result UUID

    UNION ALL

    -- Walk edges outward
    SELECT
        gn2.id,
        gn2.node_type,
        gn2.label,
        gn2.properties,
        t.depth + 1,
        t.path || gn2.id
    FROM trace t
    JOIN graph_edge ge ON ge.from_node_id = t.node_id AND ge.is_active = true
    JOIN graph_node gn2 ON gn2.id = ge.to_node_id
    WHERE gn2.id != ALL(t.path)  -- prevent cycles
      AND t.depth < 10           -- max depth safety
      AND ge.edge_type IN (
          'belongs_to_sample', 'tested_on', 'calibration_of',
          'calibrated_with', 'performed_by', 'used_reagent'
      )
)
SELECT depth, node_type, label, properties
FROM trace
ORDER BY depth ASC;

-- ============================================================
-- QUERY 2: Impact analysis for a failed calibration
-- "Which results were produced on this instrument since its
--  last valid calibration?"
-- ============================================================

WITH last_valid_cal AS (
    SELECT calibration_date
    FROM calibration
    WHERE instrument_id = '550e8400-e29b-41d4-a716-446655440001'
      AND result = 'pass'
    ORDER BY calibration_date DESC
    LIMIT 1
),
affected_analyses AS (
    SELECT ge.from_node_id AS analysis_node_id
    FROM graph_edge ge
    JOIN graph_node gn ON gn.id = ge.from_node_id AND gn.node_type = 'analysis'
    WHERE ge.to_node_id = (
        SELECT gn2.id FROM graph_node gn2
        WHERE gn2.node_type = 'instrument'
          AND gn2.entity_id = '550e8400-e29b-41d4-a716-446655440001'
    )
    AND ge.edge_type = 'tested_on'
    AND ge.effective_from >= (SELECT calibration_date FROM last_valid_cal)
)
SELECT
    gn_sample.label AS sample,
    gn_analysis.label AS analysis,
    gn_result.label AS result,
    gn_result.properties->>'value' AS result_value,
    gn_result.properties->>'is_oos' AS is_oos
FROM affected_analyses aa
JOIN graph_node gn_analysis ON gn_analysis.id = aa.analysis_node_id
-- Walk analysis -> sample
JOIN graph_edge ge_sample ON ge_sample.from_node_id = gn_analysis.id
    AND ge_sample.edge_type = 'belongs_to_sample'
JOIN graph_node gn_sample ON gn_sample.id = ge_sample.to_node_id
-- Walk analysis -> result (reverse: result -> analysis)
LEFT JOIN graph_edge ge_result ON ge_result.to_node_id = gn_analysis.id
    AND ge_result.edge_type = 'produced_result'
LEFT JOIN graph_node gn_result ON gn_result.id = ge_result.from_node_id
ORDER BY gn_sample.label;

-- ============================================================
-- QUERY 3: Analyst independence check
-- "Verify that the reviewer of result X is NOT the analyst
--  who performed the analysis"
-- ============================================================

SELECT
    analyst_edge.properties->>'role' AS analyst_role,
    gn_analyst.label AS analyst,
    reviewer_edge.properties->>'role' AS reviewer_role,
    gn_reviewer.label AS reviewer,
    (gn_analyst.entity_id = gn_reviewer.entity_id) AS independence_violation
FROM graph_node gn_result
-- Analyst: result's analysis -> performed_by -> user
JOIN graph_edge ge_analysis ON ge_analysis.from_node_id = gn_result.id
    AND ge_analysis.edge_type = 'produced_result'
JOIN graph_edge analyst_edge ON analyst_edge.from_node_id = ge_analysis.to_node_id
    AND analyst_edge.edge_type = 'performed_by'
JOIN graph_node gn_analyst ON gn_analyst.id = analyst_edge.to_node_id
-- Reviewer: result -> reviewed_by -> user
JOIN graph_edge reviewer_edge ON reviewer_edge.from_node_id = gn_result.id
    AND reviewer_edge.edge_type = 'reviewed_by'
JOIN graph_node gn_reviewer ON gn_reviewer.id = reviewer_edge.to_node_id
WHERE gn_result.node_type = 'result'
  AND gn_result.entity_id = '550e8400-e29b-41d4-a716-446655440003';

-- ============================================================
-- QUERY 4: Sample family tree (aliquots and sub-samples)
-- "Show the complete hierarchy from original sample down through
--  all aliquots and sub-aliquots"
-- ============================================================

WITH RECURSIVE sample_tree AS (
    SELECT
        gn.id AS node_id,
        gn.label,
        gn.properties,
        0 AS depth,
        gn.label AS tree_path
    FROM graph_node gn
    WHERE gn.node_type = 'sample'
      AND gn.entity_id = '550e8400-e29b-41d4-a716-446655440004'

    UNION ALL

    SELECT
        gn2.id,
        gn2.label,
        gn2.properties,
        st.depth + 1,
        st.tree_path || ' > ' || gn2.label
    FROM sample_tree st
    JOIN graph_edge ge ON ge.to_node_id = st.node_id
        AND ge.edge_type = 'aliquot_of'
    JOIN graph_node gn2 ON gn2.id = ge.from_node_id
    WHERE st.depth < 5
)
SELECT depth, label, properties->>'status' AS status, tree_path
FROM sample_tree
ORDER BY depth, label;

-- ============================================================
-- QUERY 5: Neighbourhood exploration
-- "Show me everything connected to this sample within 2 hops"
-- ============================================================

WITH RECURSIVE neighbourhood AS (
    SELECT
        gn.id AS node_id,
        gn.node_type,
        gn.label,
        0 AS depth,
        NULL::TEXT AS edge_type,
        NULL::TEXT AS direction,
        ARRAY[gn.id] AS visited
    FROM graph_node gn
    WHERE gn.node_type = 'sample'
      AND gn.entity_id = '550e8400-e29b-41d4-a716-446655440005'

    UNION ALL

    -- Outbound edges
    SELECT
        gn2.id, gn2.node_type, gn2.label,
        n.depth + 1, ge.edge_type, 'outbound',
        n.visited || gn2.id
    FROM neighbourhood n
    JOIN graph_edge ge ON ge.from_node_id = n.node_id AND ge.is_active = true
    JOIN graph_node gn2 ON gn2.id = ge.to_node_id
    WHERE gn2.id != ALL(n.visited)
      AND n.depth < 2

    UNION ALL

    -- Inbound edges
    SELECT
        gn2.id, gn2.node_type, gn2.label,
        n.depth + 1, ge.edge_type, 'inbound',
        n.visited || gn2.id
    FROM neighbourhood n
    JOIN graph_edge ge ON ge.to_node_id = n.node_id AND ge.is_active = true
    JOIN graph_node gn2 ON gn2.id = ge.from_node_id
    WHERE gn2.id != ALL(n.visited)
      AND n.depth < 2
)
SELECT DISTINCT depth, node_type, label, edge_type, direction
FROM neighbourhood
ORDER BY depth, node_type;

-- ============================================================
-- QUERY 6: LTREE hierarchical queries
-- "Find all entities under the Boston Environmental Lab"
-- ============================================================

SELECT node_type, label, properties
FROM graph_node
WHERE path <@ 'tenant.site_boston.lab_env'
ORDER BY path;

-- "Find all samples under any lab at the Boston site"
SELECT label, properties
FROM graph_node
WHERE node_type = 'sample'
  AND path ~ 'tenant.site_boston.*';
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Graph Infrastructure | 2 | graph_node, graph_edge |
| Tenancy & Organisation | 2 | tenant, site |
| Users & Access Control | 3 | app_user, role, user_role |
| Client Management | 1 | client |
| Sample Management | 2 | sample, sample_type |
| Test & Method | 2 | test_method, test_definition |
| Analysis & Results | 2 | analysis, result |
| Instruments & Calibration | 3 | instrument, reference_standard, calibration |
| Worksheets & QC | 3 | worksheet, batch, qc_result |
| Reagent Tracking | 1 | reagent_lot |
| Reporting & Compliance | 3 | report, nonconformance, corrective_action |
| Audit & Signatures | 2 | audit_log, e_signature |
| **Total** | **26** | Plus 2 graph tables; complexity shifts to graph triggers and traversal queries |

---

## Key Design Decisions

1. **PostgreSQL-only graph implementation** -- the graph layer uses standard PostgreSQL tables (`graph_node`, `graph_edge`) with recursive CTEs for traversal, avoiding the operational complexity of running a separate graph database like Neo4j. The `ltree` extension provides hierarchical path queries for tree-structured data.

2. **Relational tables remain the source of truth** -- graph nodes and edges are derived from relational data via triggers. If the graph becomes inconsistent, it can be rebuilt from the relational tables. This means the relational model can be used independently for simple CRUD operations without the graph layer.

3. **Automatic graph maintenance via triggers** -- when a sample, analysis, calibration, or other entity is created or updated in the relational tables, triggers automatically create/update the corresponding graph nodes and edges. This keeps the graph in sync without requiring application-level dual writes.

4. **Temporal edges with effective_from/effective_to** -- graph edges carry temporal properties, enabling point-in-time graph queries. "What instruments was this analyst authorized to use on March 15th?" becomes a filtered graph traversal rather than a complex temporal join.

5. **Node properties cache relational data** -- `graph_node.properties` stores a JSONB snapshot of key attributes from the relational table (e.g., result value, OOS flag, status). This allows many graph queries to be answered entirely from the graph layer without joining back to relational tables.

6. **Edge types encode domain semantics** -- rather than using generic edge labels, edge types like `tested_on`, `calibrated_with`, `performed_by`, and `aliquot_of` carry domain meaning. This makes graph queries self-documenting and enables type-filtered traversals.

7. **Impact analysis is the killer use case** -- when an instrument fails calibration, the graph query "find all results connected to this instrument via `tested_on` edges since the last `calibration_of` edge with result='pass'" identifies every potentially affected result in a single traversal. In a relational model, this requires knowing the schema and writing careful multi-table joins.

8. **LTREE paths for hierarchical navigation** -- entities that form natural hierarchies (tenant -> site -> lab -> sample -> analysis) get `ltree` path values on their graph nodes. This enables efficient subtree queries like "find all samples in the Boston Environmental Lab" using the `<@` operator.

9. **Bidirectional traversal in neighbourhood queries** -- the neighbourhood exploration query walks both inbound and outbound edges, building a complete "connection map" around any entity. This is the foundation for a visual graph explorer in the UI.

10. **Analyst independence checks are graph queries** -- verifying that a result's analyst and reviewer are different people requires traversing `performed_by` and `reviewed_by` edges from the same result node. This is a natural graph pattern that would require a multi-step relational query with careful join conditions in a normalized model.

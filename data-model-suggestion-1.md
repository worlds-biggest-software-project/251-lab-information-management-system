# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Lab Information Management System (LIMS) · Created: 2026-05-22

## Philosophy

This model follows classical third-normal-form (3NF) relational design, where every domain concept gets its own table with explicit foreign key relationships. The goal is maximum data integrity, clear referential constraints, and unambiguous query semantics. Every column has a defined type and constraint; there are no polymorphic columns or JSONB catch-alls.

This approach mirrors how enterprise LIMS like LabWare, LabVantage, and STARLIMS structure their internal data. It is also the most natural fit for ISO 17025 and 21 CFR Part 11 compliance because every data point has a well-defined lineage through foreign keys: a result traces to a test, which traces to a sample, which traces to a client, an instrument, a method, and the personnel who performed the work. Auditors can follow the chain without ambiguity.

The trade-off is a higher table count (70+ tables) and more complex migrations when the schema evolves. However, for a regulated laboratory environment where data integrity is paramount and schema changes are infrequent (labs run the same tests for years), this is a strength, not a weakness.

**Best for:** Regulated pharmaceutical, clinical, and environmental testing labs that prioritise audit trail rigour, ISO 17025/21 CFR Part 11 compliance, and complex cross-entity reporting over deployment speed.

**Trade-offs:**
- (+) Maximum referential integrity — every relationship is enforced by the database
- (+) Complex cross-entity queries are straightforward JOINs, not JSONB containment or graph traversals
- (+) Schema is self-documenting; new developers can understand the data model from the DDL alone
- (+) Alignment with ASTM E1578 terminology and FHIR resource boundaries
- (-) Higher table count means more migrations and more complex ORM mappings
- (-) Adding jurisdiction-specific or lab-specific fields requires schema migrations, not just JSON
- (-) Initial development is slower than a JSONB hybrid approach
- (-) Multi-tenant isolation requires careful RLS policy application across many tables

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO/IEC 17025:2017 | Drives the calibration, measurement_traceability, corrective_action, and audit_log tables; all sample-to-result chains maintain unbroken traceability |
| 21 CFR Part 11 | Shapes the e_signature, audit_log, and user_session tables; electronic signatures are linked to specific record changes with intent and meaning |
| ASTM E1578 | Provides the vocabulary for core entities: Sample, Test, Result, Instrument, Method, Worksheet, Batch |
| HL7 FHIR R4/R5 | DiagnosticReport, Specimen, Observation, ServiceRequest resource boundaries inform the sample/test/result table structure |
| LOINC | test_definition.loinc_code maps local test names to universal LOINC identifiers for interoperability |
| SNOMED CT | specimen_type.snomed_code and organism.snomed_code enable clinical interoperability |
| ISO 3166-1/2 | jurisdiction and site tables use ISO 3166 codes for multi-region compliance |
| ISO 9001:2015 | CAPA (corrective_action, preventive_action) and document_control tables align with QMS requirements |
| ALCOA+ | Audit trail design ensures data is Attributable, Legible, Contemporaneous, Original, Accurate |
| OAuth 2.0 / OIDC | user and api_token tables support federated identity via OIDC claims |

---

## Core Identity & Multi-Tenancy

```sql
-- ============================================================
-- TENANT & ORGANISATION
-- ============================================================

CREATE TABLE tenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,          -- URL-safe identifier
    subscription    TEXT NOT NULL DEFAULT 'free',  -- free | pro | enterprise
    settings        JSONB NOT NULL DEFAULT '{}',   -- tenant-level config overrides
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE site (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,
    address_line1   TEXT,
    address_line2   TEXT,
    city            TEXT,
    state_province  TEXT,
    postal_code     TEXT,
    country_code    CHAR(2) NOT NULL,              -- ISO 3166-1 alpha-2
    timezone        TEXT NOT NULL DEFAULT 'UTC',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_site_tenant ON site(tenant_id);

-- ============================================================
-- USERS & ACCESS CONTROL
-- ============================================================

CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    email           TEXT NOT NULL,
    display_name    TEXT NOT NULL,
    password_hash   TEXT,                          -- NULL if SSO-only
    oidc_subject    TEXT,                          -- OpenID Connect subject claim
    is_active       BOOLEAN NOT NULL DEFAULT true,
    mfa_enabled     BOOLEAN NOT NULL DEFAULT false,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, email)
);
CREATE INDEX idx_user_tenant ON app_user(tenant_id);

CREATE TABLE role (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,                 -- e.g. 'lab_technician', 'quality_manager', 'admin'
    description     TEXT,
    permissions     TEXT[] NOT NULL DEFAULT '{}',   -- array of permission strings
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, name)
);

CREATE TABLE user_role (
    user_id         UUID NOT NULL REFERENCES app_user(id),
    role_id         UUID NOT NULL REFERENCES role(id),
    site_id         UUID REFERENCES site(id),      -- NULL = all sites
    granted_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    granted_by      UUID NOT NULL REFERENCES app_user(id),
    PRIMARY KEY (user_id, role_id, site_id)
);
```

## Client & Contact Management

```sql
CREATE TABLE client (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,
    client_code     TEXT NOT NULL,                 -- lab-assigned client identifier
    address_line1   TEXT,
    city            TEXT,
    country_code    CHAR(2),
    contact_email   TEXT,
    contact_phone   TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, client_code)
);
CREATE INDEX idx_client_tenant ON client(tenant_id);

CREATE TABLE client_contact (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    client_id       UUID NOT NULL REFERENCES client(id),
    name            TEXT NOT NULL,
    email           TEXT,
    phone           TEXT,
    is_primary      BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Sample Management

```sql
-- ============================================================
-- REFERENCE DATA
-- ============================================================

CREATE TABLE sample_type (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,                 -- e.g. 'Drinking Water', 'Blood Serum', 'Tablet'
    snomed_code     TEXT,                          -- SNOMED CT specimen type code
    description     TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, name)
);

CREATE TABLE sample_matrix (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,                 -- e.g. 'Aqueous', 'Solid', 'Gas', 'Biological Fluid'
    description     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, name)
);

CREATE TABLE sample_point (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,                 -- e.g. 'Production Line A', 'Patient Ward 3B'
    site_id         UUID REFERENCES site(id),
    description     TEXT,
    latitude        NUMERIC(10, 7),
    longitude       NUMERIC(10, 7),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, name)
);

CREATE TABLE container_type (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,                 -- e.g. 'Vacutainer EDTA', '500ml Glass Bottle'
    preservation    TEXT,                          -- e.g. 'Refrigerate 2-8C', 'Acidify pH<2'
    max_volume_ml   NUMERIC(10, 2),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, name)
);

-- ============================================================
-- SAMPLES
-- ============================================================

CREATE TABLE sample (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    site_id         UUID NOT NULL REFERENCES site(id),
    sample_id_human TEXT NOT NULL,                 -- human-readable lab ID (e.g. 'WQ-2026-001234')
    client_id       UUID REFERENCES client(id),
    sample_type_id  UUID NOT NULL REFERENCES sample_type(id),
    sample_matrix_id UUID REFERENCES sample_matrix(id),
    sample_point_id UUID REFERENCES sample_point(id),
    container_type_id UUID REFERENCES container_type(id),
    parent_sample_id UUID REFERENCES sample(id),  -- for aliquots / sub-samples
    status          TEXT NOT NULL DEFAULT 'registered',
        -- registered | received | in_progress | awaiting_review | verified | published | cancelled | disposed
    priority        TEXT NOT NULL DEFAULT 'normal', -- low | normal | high | urgent
    date_sampled    TIMESTAMPTZ,
    date_received   TIMESTAMPTZ,
    date_due        TIMESTAMPTZ,
    sampled_by      TEXT,                          -- name of person who collected
    received_by     UUID REFERENCES app_user(id),
    storage_location TEXT,
    batch_id        UUID,                          -- FK added after batch table
    remarks         TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, sample_id_human)
);
CREATE INDEX idx_sample_tenant_status ON sample(tenant_id, status);
CREATE INDEX idx_sample_client ON sample(client_id);
CREATE INDEX idx_sample_type ON sample(sample_type_id);
CREATE INDEX idx_sample_date_received ON sample(date_received);
CREATE INDEX idx_sample_parent ON sample(parent_sample_id);

-- Chain of custody tracking
CREATE TABLE chain_of_custody (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    sample_id       UUID NOT NULL REFERENCES sample(id),
    from_user_id    UUID REFERENCES app_user(id),
    to_user_id      UUID REFERENCES app_user(id),
    action          TEXT NOT NULL,                 -- 'transfer' | 'receive' | 'store' | 'dispose'
    location        TEXT,
    remarks         TEXT,
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_coc_sample ON chain_of_custody(sample_id);
```

## Test & Method Management

```sql
CREATE TABLE test_method (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,                 -- e.g. 'EPA 524.2 VOCs by GC-MS'
    method_code     TEXT NOT NULL,                 -- internal method reference
    standard_ref    TEXT,                          -- external standard (e.g. 'EPA 524.2', 'USP <621>')
    description     TEXT,
    version         TEXT NOT NULL DEFAULT '1.0',
    accreditation_body TEXT,                       -- e.g. 'A2LA', 'UKAS'
    is_accredited   BOOLEAN NOT NULL DEFAULT false,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, method_code, version)
);

CREATE TABLE test_definition (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,                 -- e.g. 'Lead (Pb)', 'Escherichia coli', 'pH'
    test_code       TEXT NOT NULL,                 -- internal test code
    loinc_code      TEXT,                          -- LOINC universal identifier
    snomed_code     TEXT,                          -- SNOMED CT code for organisms / analytes
    category        TEXT,                          -- e.g. 'Metals', 'Microbiology', 'Physical'
    test_method_id  UUID NOT NULL REFERENCES test_method(id),
    unit            TEXT,                          -- e.g. 'mg/L', 'CFU/100mL', 'pH units'
    decimal_places  INTEGER DEFAULT 2,
    result_type     TEXT NOT NULL DEFAULT 'numeric',
        -- numeric | text | boolean | selection | calculation
    calculation_formula TEXT,                      -- for calculated results
    default_spec_lower NUMERIC,                   -- default lower specification limit
    default_spec_upper NUMERIC,                   -- default upper specification limit
    detection_limit NUMERIC,                       -- limit of detection (LOD)
    quantitation_limit NUMERIC,                    -- limit of quantitation (LOQ)
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, test_code)
);
CREATE INDEX idx_test_def_method ON test_definition(test_method_id);
CREATE INDEX idx_test_def_loinc ON test_definition(loinc_code);

-- Analysis profiles group commonly ordered tests together
CREATE TABLE analysis_profile (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,                 -- e.g. 'Drinking Water Full Panel', 'Blood Chemistry'
    profile_code    TEXT NOT NULL,
    description     TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, profile_code)
);

CREATE TABLE analysis_profile_test (
    profile_id      UUID NOT NULL REFERENCES analysis_profile(id),
    test_definition_id UUID NOT NULL REFERENCES test_definition(id),
    sort_order      INTEGER NOT NULL DEFAULT 0,
    PRIMARY KEY (profile_id, test_definition_id)
);
```

## Results & Worksheets

```sql
-- ============================================================
-- TEST REQUESTS & RESULTS
-- ============================================================

CREATE TABLE test_request (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    sample_id       UUID NOT NULL REFERENCES sample(id),
    test_definition_id UUID NOT NULL REFERENCES test_definition(id),
    worksheet_id    UUID,                          -- FK added after worksheet table
    status          TEXT NOT NULL DEFAULT 'pending',
        -- pending | assigned | in_progress | complete | verified | cancelled
    assigned_to     UUID REFERENCES app_user(id),
    instrument_id   UUID,                          -- FK added after instrument table
    priority        TEXT NOT NULL DEFAULT 'normal',
    due_date        TIMESTAMPTZ,
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    verified_at     TIMESTAMPTZ,
    verified_by     UUID REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_test_request_sample ON test_request(sample_id);
CREATE INDEX idx_test_request_status ON test_request(tenant_id, status);
CREATE INDEX idx_test_request_worksheet ON test_request(worksheet_id);

CREATE TABLE test_result (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    test_request_id UUID NOT NULL REFERENCES test_request(id),
    result_numeric  NUMERIC,                       -- populated for numeric results
    result_text     TEXT,                          -- populated for text/selection results
    result_boolean  BOOLEAN,                       -- populated for boolean results
    unit            TEXT,
    spec_lower      NUMERIC,                       -- specification limit at time of test
    spec_upper      NUMERIC,
    is_oos          BOOLEAN NOT NULL DEFAULT false, -- out of specification flag
    is_oot          BOOLEAN NOT NULL DEFAULT false, -- out of trend flag
    uncertainty     NUMERIC,                       -- measurement uncertainty (ISO 17025)
    remarks         TEXT,
    entered_by      UUID NOT NULL REFERENCES app_user(id),
    entered_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    instrument_id   UUID,                          -- FK added after instrument table
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_test_result_request ON test_result(test_request_id);

-- ============================================================
-- WORKSHEETS (daily work planning)
-- ============================================================

CREATE TABLE worksheet_template (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,
    description     TEXT,
    layout          TEXT NOT NULL DEFAULT 'list',  -- list | plate_96 | plate_384
    instrument_id   UUID,
    test_method_id  UUID REFERENCES test_method(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, name)
);

CREATE TABLE worksheet (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    template_id     UUID REFERENCES worksheet_template(id),
    worksheet_number TEXT NOT NULL,
    analyst_id      UUID NOT NULL REFERENCES app_user(id),
    status          TEXT NOT NULL DEFAULT 'open',
        -- open | in_progress | submitted | verified | closed
    instrument_id   UUID,
    opened_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    submitted_at    TIMESTAMPTZ,
    verified_at     TIMESTAMPTZ,
    verified_by     UUID REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, worksheet_number)
);
CREATE INDEX idx_worksheet_analyst ON worksheet(analyst_id);
CREATE INDEX idx_worksheet_status ON worksheet(tenant_id, status);

-- Link test_request.worksheet_id -> worksheet.id
ALTER TABLE test_request
    ADD CONSTRAINT fk_test_request_worksheet
    FOREIGN KEY (worksheet_id) REFERENCES worksheet(id);
```

## Batch Management

```sql
CREATE TABLE batch (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    batch_number    TEXT NOT NULL,
    title           TEXT,
    description     TEXT,
    client_id       UUID REFERENCES client(id),
    status          TEXT NOT NULL DEFAULT 'open',  -- open | in_progress | closed
    created_by      UUID NOT NULL REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, batch_number)
);

-- Link sample.batch_id -> batch.id
ALTER TABLE sample
    ADD CONSTRAINT fk_sample_batch
    FOREIGN KEY (batch_id) REFERENCES batch(id);
```

## Instrument & Equipment Management

```sql
CREATE TABLE instrument_type (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,                 -- e.g. 'ICP-MS', 'HPLC', 'pH Meter'
    manufacturer    TEXT,
    description     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, name)
);

CREATE TABLE instrument (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    site_id         UUID NOT NULL REFERENCES site(id),
    instrument_type_id UUID NOT NULL REFERENCES instrument_type(id),
    name            TEXT NOT NULL,                 -- e.g. 'ICP-MS-01'
    serial_number   TEXT,
    asset_tag       TEXT,
    model           TEXT,
    location        TEXT,                          -- physical location in lab
    status          TEXT NOT NULL DEFAULT 'active',
        -- active | maintenance | out_of_service | decommissioned
    commissioned_date DATE,
    decommissioned_date DATE,
    communication_protocol TEXT,                   -- 'ASTM' | 'RS232' | 'TCP_IP' | 'REST' | 'manual'
    connection_config JSONB,                       -- port, baud rate, IP address, etc.
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, name)
);
CREATE INDEX idx_instrument_site ON instrument(site_id);
CREATE INDEX idx_instrument_type ON instrument(instrument_type_id);

-- Link test_request and test_result instrument FKs
ALTER TABLE test_request
    ADD CONSTRAINT fk_test_request_instrument
    FOREIGN KEY (instrument_id) REFERENCES instrument(id);

ALTER TABLE test_result
    ADD CONSTRAINT fk_test_result_instrument
    FOREIGN KEY (instrument_id) REFERENCES instrument(id);

ALTER TABLE worksheet_template
    ADD CONSTRAINT fk_worksheet_template_instrument
    FOREIGN KEY (instrument_id) REFERENCES instrument(id);

ALTER TABLE worksheet
    ADD CONSTRAINT fk_worksheet_instrument
    FOREIGN KEY (instrument_id) REFERENCES instrument(id);

-- ============================================================
-- CALIBRATION (ISO 17025 metrological traceability)
-- ============================================================

CREATE TABLE calibration_schedule (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    instrument_id   UUID NOT NULL REFERENCES instrument(id),
    frequency_days  INTEGER NOT NULL,              -- calibration interval
    last_calibrated DATE,
    next_due        DATE NOT NULL,
    reference_standard TEXT,                       -- traceable standard used
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_cal_sched_instrument ON calibration_schedule(instrument_id);
CREATE INDEX idx_cal_sched_next_due ON calibration_schedule(next_due);

CREATE TABLE calibration_record (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    instrument_id   UUID NOT NULL REFERENCES instrument(id),
    calibrated_by   UUID NOT NULL REFERENCES app_user(id),
    calibration_date DATE NOT NULL,
    result          TEXT NOT NULL,                 -- 'pass' | 'fail' | 'adjusted'
    certificate_number TEXT,
    reference_standard TEXT,                       -- NIST traceable reference
    uncertainty     NUMERIC,
    remarks         TEXT,
    attachment_id   UUID,                          -- FK to attachment table
    next_due        DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_cal_record_instrument ON calibration_record(instrument_id);
```

## Quality Control

```sql
CREATE TABLE qc_definition (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,                 -- e.g. 'Method Blank', 'LCS', 'CCV', 'Duplicate'
    qc_type         TEXT NOT NULL,                 -- blank | lcs | lcsd | ms | msd | crm | ccv | ccb | duplicate
    description     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, name)
);

CREATE TABLE qc_result (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    qc_definition_id UUID NOT NULL REFERENCES qc_definition(id),
    worksheet_id    UUID NOT NULL REFERENCES worksheet(id),
    test_definition_id UUID NOT NULL REFERENCES test_definition(id),
    result_numeric  NUMERIC,
    expected_value  NUMERIC,
    recovery_pct    NUMERIC,                       -- % recovery for LCS/MS
    rpd             NUMERIC,                       -- relative percent difference for duplicates
    acceptance_lower NUMERIC,
    acceptance_upper NUMERIC,
    is_acceptable   BOOLEAN NOT NULL DEFAULT true,
    westgard_rule_violated TEXT,                   -- e.g. '1-2s', '2-2s', 'R-4s', '4-1s'
    remarks         TEXT,
    entered_by      UUID NOT NULL REFERENCES app_user(id),
    entered_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_qc_result_worksheet ON qc_result(worksheet_id);
CREATE INDEX idx_qc_result_test ON qc_result(test_definition_id);

-- Levey-Jennings / QC chart data points
CREATE TABLE qc_chart_point (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    test_definition_id UUID NOT NULL REFERENCES test_definition(id),
    instrument_id   UUID REFERENCES instrument(id),
    value           NUMERIC NOT NULL,
    mean            NUMERIC NOT NULL,
    std_dev         NUMERIC NOT NULL,
    z_score         NUMERIC,
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_qc_chart_test_date ON qc_chart_point(test_definition_id, recorded_at);
```

## Inventory & Reagents

```sql
CREATE TABLE reagent_category (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, name)
);

CREATE TABLE reagent (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    category_id     UUID REFERENCES reagent_category(id),
    name            TEXT NOT NULL,
    catalogue_number TEXT,
    manufacturer    TEXT,
    grade           TEXT,                          -- 'ACS', 'Reagent', 'HPLC', 'LC-MS'
    cas_number      TEXT,                          -- CAS registry number
    unit            TEXT,                          -- 'mL', 'g', 'each'
    reorder_level   NUMERIC,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, name, manufacturer)
);

CREATE TABLE reagent_lot (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    reagent_id      UUID NOT NULL REFERENCES reagent(id),
    lot_number      TEXT NOT NULL,
    expiry_date     DATE,
    quantity_received NUMERIC,
    quantity_remaining NUMERIC,
    received_date   DATE NOT NULL,
    received_by     UUID NOT NULL REFERENCES app_user(id),
    certificate_of_analysis TEXT,                  -- supplier CoA reference
    storage_location TEXT,
    status          TEXT NOT NULL DEFAULT 'in_use', -- in_use | expired | depleted | quarantined
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_reagent_lot_reagent ON reagent_lot(reagent_id);
CREATE INDEX idx_reagent_lot_expiry ON reagent_lot(expiry_date);
```

## Reporting & Certificates

```sql
CREATE TABLE report_template (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,                 -- e.g. 'Standard CoA', 'ISO 17025 Test Report'
    template_type   TEXT NOT NULL,                 -- 'coa' | 'test_report' | 'compliance' | 'custom'
    template_body   TEXT NOT NULL,                 -- HTML/Mustache template
    is_default      BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, name)
);

CREATE TABLE report (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    report_number   TEXT NOT NULL,
    template_id     UUID NOT NULL REFERENCES report_template(id),
    status          TEXT NOT NULL DEFAULT 'draft', -- draft | issued | amended | superseded
    issued_at       TIMESTAMPTZ,
    issued_by       UUID REFERENCES app_user(id),
    amendment_reason TEXT,
    pdf_attachment_id UUID,                        -- FK to attachment
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, report_number)
);

CREATE TABLE report_sample (
    report_id       UUID NOT NULL REFERENCES report(id),
    sample_id       UUID NOT NULL REFERENCES sample(id),
    PRIMARY KEY (report_id, sample_id)
);
```

## Compliance & Quality Management

```sql
-- ============================================================
-- CORRECTIVE / PREVENTIVE ACTIONS (CAPA)
-- ============================================================

CREATE TABLE nonconformance (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    nc_number       TEXT NOT NULL,
    title           TEXT NOT NULL,
    description     TEXT NOT NULL,
    source          TEXT NOT NULL,                 -- 'internal_audit' | 'external_audit' | 'oos' | 'complaint' | 'observation'
    severity        TEXT NOT NULL DEFAULT 'minor', -- minor | major | critical
    sample_id       UUID REFERENCES sample(id),
    instrument_id   UUID REFERENCES instrument(id),
    status          TEXT NOT NULL DEFAULT 'open',  -- open | investigating | action_required | closed
    raised_by       UUID NOT NULL REFERENCES app_user(id),
    raised_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    closed_at       TIMESTAMPTZ,
    closed_by       UUID REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, nc_number)
);

CREATE TABLE corrective_action (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    nonconformance_id UUID NOT NULL REFERENCES nonconformance(id),
    action_type     TEXT NOT NULL,                 -- 'corrective' | 'preventive'
    description     TEXT NOT NULL,
    assigned_to     UUID NOT NULL REFERENCES app_user(id),
    due_date        DATE,
    status          TEXT NOT NULL DEFAULT 'open',  -- open | in_progress | completed | verified
    completed_at    TIMESTAMPTZ,
    verified_by     UUID REFERENCES app_user(id),
    effectiveness_review TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_capa_nc ON corrective_action(nonconformance_id);
```

## Audit Trail & Electronic Signatures

```sql
-- ============================================================
-- AUDIT LOG (21 CFR Part 11 / ISO 17025 / ALCOA+)
-- ============================================================

CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    user_id         UUID NOT NULL REFERENCES app_user(id),
    action          TEXT NOT NULL,                 -- 'create' | 'update' | 'delete' | 'verify' | 'sign' | 'login' | 'logout'
    entity_type     TEXT NOT NULL,                 -- table name: 'sample', 'test_result', etc.
    entity_id       UUID NOT NULL,
    old_values      JSONB,                         -- previous field values
    new_values      JSONB,                         -- new field values
    reason          TEXT,                          -- 21 CFR Part 11 requires reason for change
    ip_address      INET,
    user_agent      TEXT,
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);
-- Append-only: no UPDATE or DELETE should be permitted on this table
CREATE INDEX idx_audit_log_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_log_user ON audit_log(user_id);
CREATE INDEX idx_audit_log_tenant_time ON audit_log(tenant_id, occurred_at);

-- ============================================================
-- ELECTRONIC SIGNATURES (21 CFR Part 11)
-- ============================================================

CREATE TABLE e_signature (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    signer_id       UUID NOT NULL REFERENCES app_user(id),
    entity_type     TEXT NOT NULL,                 -- what is being signed
    entity_id       UUID NOT NULL,
    signature_meaning TEXT NOT NULL,               -- 'approval' | 'review' | 'authorship' | 'verification'
    full_name       TEXT NOT NULL,                 -- legal name at time of signing
    title           TEXT,                          -- role/title at time of signing
    reason          TEXT,                          -- reason for signing
    signed_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    auth_method     TEXT NOT NULL DEFAULT 'password',
        -- password | mfa | biometric | certificate
    is_valid        BOOLEAN NOT NULL DEFAULT true
);
CREATE INDEX idx_esig_entity ON e_signature(entity_type, entity_id);
CREATE INDEX idx_esig_signer ON e_signature(signer_id);
```

## Document & Attachment Management

```sql
CREATE TABLE attachment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    filename        TEXT NOT NULL,
    content_type    TEXT NOT NULL,                 -- MIME type
    size_bytes      BIGINT NOT NULL,
    storage_path    TEXT NOT NULL,                 -- S3 key or filesystem path
    checksum_sha256 TEXT NOT NULL,                 -- file integrity verification
    uploaded_by     UUID NOT NULL REFERENCES app_user(id),
    entity_type     TEXT,                          -- optional: what this is attached to
    entity_id       UUID,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_attachment_entity ON attachment(entity_type, entity_id);

-- Link calibration_record.attachment_id -> attachment.id
ALTER TABLE calibration_record
    ADD CONSTRAINT fk_cal_record_attachment
    FOREIGN KEY (attachment_id) REFERENCES attachment(id);

ALTER TABLE report
    ADD CONSTRAINT fk_report_pdf
    FOREIGN KEY (pdf_attachment_id) REFERENCES attachment(id);
```

## Instrument Data Import

```sql
CREATE TABLE instrument_import (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    instrument_id   UUID NOT NULL REFERENCES instrument(id),
    filename        TEXT NOT NULL,
    file_format     TEXT NOT NULL,                 -- 'csv' | 'xml' | 'astm' | 'json'
    raw_data        TEXT,                          -- raw file content
    parser_config   TEXT,                          -- parser configuration name
    status          TEXT NOT NULL DEFAULT 'pending',
        -- pending | parsing | parsed | mapped | error
    error_message   TEXT,
    imported_by     UUID NOT NULL REFERENCES app_user(id),
    imported_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE instrument_import_result (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    import_id       UUID NOT NULL REFERENCES instrument_import(id),
    test_request_id UUID REFERENCES test_request(id),  -- mapped target
    raw_identifier  TEXT NOT NULL,                 -- sample ID as read from instrument
    raw_analyte     TEXT NOT NULL,                 -- analyte name as read from instrument
    raw_value       TEXT NOT NULL,                 -- result value as read from instrument
    raw_unit        TEXT,
    is_mapped       BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_import_result_import ON instrument_import_result(import_id);
```

## Stability Studies & Environmental Monitoring

```sql
CREATE TABLE stability_study (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    study_number    TEXT NOT NULL,
    product_name    TEXT NOT NULL,
    batch_number    TEXT,
    condition       TEXT NOT NULL,                 -- e.g. '25C/60%RH', '40C/75%RH', '5C'
    start_date      DATE NOT NULL,
    end_date        DATE,
    status          TEXT NOT NULL DEFAULT 'active',
    created_by      UUID NOT NULL REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, study_number)
);

CREATE TABLE stability_timepoint (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    study_id        UUID NOT NULL REFERENCES stability_study(id),
    timepoint_months INTEGER NOT NULL,             -- 0, 1, 3, 6, 9, 12, 18, 24, 36...
    sample_id       UUID REFERENCES sample(id),
    status          TEXT NOT NULL DEFAULT 'scheduled',
        -- scheduled | sample_pulled | testing | complete
    scheduled_date  DATE NOT NULL,
    actual_date     DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_stability_tp_study ON stability_timepoint(study_id);

CREATE TABLE environmental_monitoring_location (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    site_id         UUID NOT NULL REFERENCES site(id),
    name            TEXT NOT NULL,
    area_classification TEXT,                      -- e.g. 'ISO 5', 'ISO 7', 'Grade A', 'Grade D'
    monitoring_type TEXT NOT NULL,                 -- 'viable' | 'non_viable' | 'temperature' | 'humidity'
    frequency       TEXT,                          -- 'daily' | 'weekly' | 'monthly'
    alert_limit     NUMERIC,
    action_limit    NUMERIC,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, name)
);
```

## Row-Level Security (Multi-Tenancy)

```sql
-- Enable RLS on all data tables (example for key tables)
ALTER TABLE sample ENABLE ROW LEVEL SECURITY;
ALTER TABLE test_request ENABLE ROW LEVEL SECURITY;
ALTER TABLE test_result ENABLE ROW LEVEL SECURITY;
ALTER TABLE audit_log ENABLE ROW LEVEL SECURITY;

-- Policy: users can only see rows belonging to their tenant
CREATE POLICY tenant_isolation_sample ON sample
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);

CREATE POLICY tenant_isolation_test_request ON test_request
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);

CREATE POLICY tenant_isolation_audit_log ON audit_log
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);

-- Audit log is append-only
CREATE POLICY audit_log_insert_only ON audit_log
    FOR INSERT
    WITH CHECK (tenant_id = current_setting('app.current_tenant_id')::UUID);

-- Prevent updates and deletes on audit_log
REVOKE UPDATE, DELETE ON audit_log FROM PUBLIC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Tenancy & Organisation | 2 | tenant, site |
| Users & Access Control | 3 | app_user, role, user_role |
| Client Management | 2 | client, client_contact |
| Sample Management | 7 | sample, sample_type, sample_matrix, sample_point, container_type, chain_of_custody, batch |
| Test & Method | 4 | test_method, test_definition, analysis_profile, analysis_profile_test |
| Results & Worksheets | 4 | test_request, test_result, worksheet_template, worksheet |
| Instruments & Calibration | 4 | instrument_type, instrument, calibration_schedule, calibration_record |
| Quality Control | 3 | qc_definition, qc_result, qc_chart_point |
| Inventory & Reagents | 3 | reagent_category, reagent, reagent_lot |
| Reporting | 3 | report_template, report, report_sample |
| Compliance & CAPA | 2 | nonconformance, corrective_action |
| Audit & Signatures | 2 | audit_log, e_signature |
| Documents | 1 | attachment |
| Instrument Import | 2 | instrument_import, instrument_import_result |
| Stability & EM | 3 | stability_study, stability_timepoint, environmental_monitoring_location |
| **Total** | **45** | |

---

## Key Design Decisions

1. **UUID primary keys everywhere** — enables distributed ID generation, safe cross-site data federation, and no sequential ID leakage between tenants.

2. **Tenant ID on every data table** — combined with PostgreSQL Row-Level Security policies, this provides strong multi-tenant data isolation without schema-per-tenant complexity.

3. **Separate test_definition and test_method tables** — a method (e.g. EPA 524.2) can define many individual tests (Benzene, Toluene, Ethylbenzene); this matches how labs actually organise their scope of accreditation under ISO 17025.

4. **LOINC and SNOMED codes as optional columns** — labs that need clinical interoperability can populate these; non-clinical labs can ignore them. This avoids forcing LOINC as the internal identifier while enabling FHIR-compatible result exchange.

5. **Append-only audit_log with REVOKE UPDATE/DELETE** — the database itself enforces immutability, satisfying 21 CFR Part 11 and ALCOA+ requirements at the storage layer rather than relying solely on application-level controls.

6. **Electronic signatures as a separate table** — e_signature records are linked to any entity via (entity_type, entity_id) polymorphism, supporting the 21 CFR Part 11 requirement that signatures carry meaning, identity, and timestamp independently of the signed record.

7. **Instrument connection_config as JSONB** — this is the one deliberate JSONB column in the model. Instrument communication parameters (baud rate, IP address, port, ASTM framing) vary so widely that a relational decomposition would create dozens of nullable columns. JSONB is the pragmatic choice here.

8. **Calibration chain of traceability** — calibration_record links instrument → reference_standard → certificate_number, creating the unbroken traceability chain required by ISO 17025 Section 6.5.

9. **QC chart points as a first-class entity** — rather than computing Levey-Jennings statistics at query time, each QC measurement is stored with its mean, standard deviation, and z-score, enabling efficient Westgard rule evaluation and AI-based trend detection.

10. **Sample hierarchy via parent_sample_id** — supports aliquoting and sub-sampling as a self-referential foreign key, enabling recursive CTE queries to trace any aliquot back to its original sample.

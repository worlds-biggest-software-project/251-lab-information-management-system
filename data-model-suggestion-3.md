# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Lab Information Management System (LIMS) · Created: 2026-05-22

## Philosophy

This model uses a conventional relational backbone for the core sample-test-result pipeline and shared compliance infrastructure (audit trails, electronic signatures, calibration records), while storing variable, method-specific, and jurisdiction-specific data in typed JSONB columns. The key insight is that in a LIMS, roughly 70% of the data model is universal — every lab has samples, tests, results, instruments, and users — but the remaining 30% varies enormously: a water chemistry lab needs different result fields than a microbiology lab; European IVD labs face different regulatory metadata requirements than US pharmaceutical labs; each instrument manufacturer outputs data in a different schema.

Rather than modeling every possible variation as nullable relational columns (leading to wide, sparse tables) or splitting into dozens of type-specific sub-tables (leading to complex inheritance hierarchies), the hybrid approach puts the stable core in relational columns and the variable part in JSONB columns with documented JSON Schema contracts. This lets the application validate JSONB content against a schema at write time while allowing schema evolution without database migrations.

This architecture is common in modern SaaS products that serve diverse customer segments. Scispot and QBench, two of the more modern LIMS SaaS platforms, use API-first designs with JSON payloads that imply this kind of hybrid storage. The approach is also well-suited for an AI-native LIMS because the JSONB fields can store structured AI-generated metadata (predicted OOS probability, anomaly scores, classification labels) without schema changes for each new ML feature.

**Best for:** Multi-purpose LIMS deployments serving diverse lab types (environmental, pharmaceutical, food, clinical) where method-specific and jurisdiction-specific data varies widely, and where rapid feature iteration (especially AI features) is prioritised over strict relational purity.

**Trade-offs:**
- (+) Core relational structure provides data integrity for the universal pipeline
- (+) JSONB fields accommodate method-specific, instrument-specific, and jurisdiction-specific data without migrations
- (+) New AI/ML metadata fields can be added without schema changes
- (+) Faster development iteration — new test types don't require DDL changes
- (+) Lower table count than fully normalized (easier to understand for new developers)
- (-) JSONB fields are not referentially constrained by the database; validation is application-level
- (-) Complex queries spanning relational and JSONB columns can be less intuitive
- (-) JSONB content is harder to report on with traditional BI tools
- (-) Risk of "JSONB sprawl" if discipline is not maintained on what goes in JSONB vs. columns
- (-) GIN indexes on JSONB are less efficient than B-tree indexes on relational columns

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO/IEC 17025:2017 | Core relational columns handle traceability chain and audit requirements; jurisdiction-specific accreditation metadata stored in JSONB |
| 21 CFR Part 11 | Audit trail and e-signature tables are fully relational (never JSONB) to ensure strict immutability and constraint enforcement |
| ASTM E1578 | Core entity names follow ASTM vocabulary; ASTM-defined fields are relational columns, not JSONB |
| HL7 FHIR R4/R5 | FHIR resource mapping uses relational columns for mandatory FHIR fields; FHIR extensions map naturally to JSONB |
| LOINC / SNOMED CT | Coded identifiers are relational columns on test and specimen type tables |
| ISO 3166-1/2 | Jurisdiction codes are relational; jurisdiction-specific regulatory requirements stored in `regulatory_config` JSONB |
| JSON Schema Draft 2020-12 | Every JSONB column has a documented JSON Schema; schemas stored in `json_schema_registry` for runtime validation |
| OpenAPI 3.1 | API schema reflects both relational and JSONB fields with full type documentation |

---

## Schema Registry (JSONB Validation)

```sql
-- ============================================================
-- JSON SCHEMA REGISTRY
-- ============================================================
-- Stores the JSON Schema for every JSONB column in the system.
-- The application validates JSONB payloads against these schemas
-- before writing. This replaces database-level column constraints
-- for the flexible parts of the model.

CREATE TABLE json_schema_registry (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    schema_name     TEXT NOT NULL UNIQUE,             -- e.g. 'sample.custom_fields', 'result.method_data'
    table_name      TEXT NOT NULL,
    column_name     TEXT NOT NULL,
    json_schema     JSONB NOT NULL,                   -- JSON Schema definition
    version         TEXT NOT NULL DEFAULT '1.0',
    description     TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Example schema registration:
INSERT INTO json_schema_registry (schema_name, table_name, column_name, json_schema, description) VALUES
('sample.custom_fields', 'sample', 'custom_fields', '{
    "type": "object",
    "properties": {
        "patient_id": {"type": "string", "description": "For clinical labs only"},
        "patient_dob": {"type": "string", "format": "date"},
        "farm_id": {"type": "string", "description": "For agricultural testing"},
        "field_temperature_c": {"type": "number"},
        "field_ph": {"type": "number"},
        "gps_latitude": {"type": "number"},
        "gps_longitude": {"type": "number"},
        "regulatory_submission_id": {"type": "string"},
        "customs_declaration_number": {"type": "string"}
    },
    "additionalProperties": true
}'::JSONB, 'Flexible fields for sample metadata that varies by lab type and jurisdiction');
```

## Core Identity & Multi-Tenancy

```sql
CREATE TABLE tenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    subscription    TEXT NOT NULL DEFAULT 'free',
    config          JSONB NOT NULL DEFAULT '{}',
    -- Example config:
    -- {
    --   "default_timezone": "America/New_York",
    --   "date_format": "YYYY-MM-DD",
    --   "require_reason_for_change": true,
    --   "require_dual_signature": false,
    --   "sample_id_prefix": "ENV",
    --   "sample_id_sequence_format": "ENV-{YYYY}-{SEQ:6}"
    -- }
    regulatory_config JSONB NOT NULL DEFAULT '{}',
    -- Example regulatory_config:
    -- {
    --   "framework": "FDA",
    --   "cfr_part_11_enabled": true,
    --   "iso_17025_accredited": true,
    --   "accreditation_body": "A2LA",
    --   "accreditation_number": "4567.01",
    --   "accreditation_scope": ["drinking_water", "wastewater", "soil"],
    --   "data_retention_years": 10,
    --   "require_esignature_on": ["result_verification", "report_approval"]
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE site (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,
    country_code    CHAR(2) NOT NULL,                 -- ISO 3166-1 alpha-2
    timezone        TEXT NOT NULL DEFAULT 'UTC',
    address         JSONB NOT NULL DEFAULT '{}',
    -- Example address:
    -- {
    --   "line1": "123 Lab Street",
    --   "line2": "Suite 400",
    --   "city": "Boston",
    --   "state": "MA",
    --   "postal_code": "02101",
    --   "country": "US"
    -- }
    site_config     JSONB NOT NULL DEFAULT '{}',      -- site-specific overrides
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_site_tenant ON site(tenant_id);

CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    email           TEXT NOT NULL,
    display_name    TEXT NOT NULL,
    password_hash   TEXT,
    oidc_subject    TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    mfa_enabled     BOOLEAN NOT NULL DEFAULT false,
    qualifications  JSONB NOT NULL DEFAULT '[]',
    -- Example qualifications (for CLIA / ISO 17025 personnel records):
    -- [
    --   {"type": "degree", "name": "B.S. Chemistry", "institution": "MIT", "year": 2018},
    --   {"type": "certification", "name": "ASCP MLS", "number": "123456", "expiry": "2027-12-31"},
    --   {"type": "competency", "method": "EPA 524.2", "assessed_date": "2026-01-15", "assessor": "Jane Smith"}
    -- ]
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
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
    granted_by      UUID NOT NULL REFERENCES app_user(id),
    PRIMARY KEY (user_id, role_id)
);
```

## Sample Management

```sql
CREATE TABLE sample_type (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,
    snomed_code     TEXT,
    category        TEXT,                             -- 'environmental', 'clinical', 'pharma', 'food'
    default_custom_fields_schema TEXT,                -- references json_schema_registry.schema_name
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, name)
);

CREATE TABLE client (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,
    client_code     TEXT NOT NULL,
    contact         JSONB NOT NULL DEFAULT '{}',
    -- Example contact:
    -- {
    --   "primary_name": "John Doe",
    --   "primary_email": "john@example.com",
    --   "primary_phone": "+1-555-0123",
    --   "billing_email": "billing@example.com",
    --   "addresses": [
    --     {"type": "billing", "line1": "...", "city": "...", "country": "US"},
    --     {"type": "shipping", "line1": "...", "city": "...", "country": "US"}
    --   ]
    -- }
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, client_code)
);
CREATE INDEX idx_client_tenant ON client(tenant_id);

CREATE TABLE sample (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    site_id         UUID NOT NULL REFERENCES site(id),
    sample_id_human TEXT NOT NULL,
    client_id       UUID REFERENCES client(id),
    sample_type_id  UUID NOT NULL REFERENCES sample_type(id),
    parent_sample_id UUID REFERENCES sample(id),

    -- Core relational fields (universal across all lab types)
    status          TEXT NOT NULL DEFAULT 'registered',
    priority        TEXT NOT NULL DEFAULT 'normal',
    date_sampled    TIMESTAMPTZ,
    date_received   TIMESTAMPTZ,
    date_due        TIMESTAMPTZ,
    received_by     UUID REFERENCES app_user(id),
    storage_location TEXT,
    batch_id        UUID,

    -- JSONB: lab-type-specific and jurisdiction-specific fields
    custom_fields   JSONB NOT NULL DEFAULT '{}',
    -- Environmental lab example:
    -- {
    --   "sampling_point": "Well MW-3",
    --   "field_ph": 7.2,
    --   "field_temperature_c": 18.5,
    --   "field_conductivity_us": 450,
    --   "gps_latitude": 42.3601,
    --   "gps_longitude": -71.0589,
    --   "weather_conditions": "Clear, 22C",
    --   "sampler_name": "Mike Johnson",
    --   "preservation": "HNO3 to pH<2"
    -- }
    --
    -- Clinical lab example:
    -- {
    --   "patient_id": "MRN-123456",
    --   "patient_name": "Jane Doe",
    --   "patient_dob": "1985-03-15",
    --   "ordering_physician": "Dr. Smith",
    --   "diagnosis_code": "R79.89",
    --   "fasting": true,
    --   "collection_site": "Left antecubital"
    -- }
    --
    -- Pharmaceutical lab example:
    -- {
    --   "product_name": "Ibuprofen 200mg Tablets",
    --   "product_code": "IBU-200",
    --   "batch_number": "MFG-2026-0451",
    --   "manufacturing_date": "2026-04-01",
    --   "expiry_date": "2028-04-01",
    --   "storage_condition": "25C/60%RH"
    -- }

    -- JSONB: chain of custody events (lightweight embedded array)
    custody_chain   JSONB NOT NULL DEFAULT '[]',
    -- [
    --   {"action": "collected", "by": "Mike Johnson", "at": "2026-05-20T09:00:00Z", "location": "Field Site A"},
    --   {"action": "transported", "by": "Mike Johnson", "at": "2026-05-20T11:30:00Z", "temp_c": 4},
    --   {"action": "received", "by_user_id": "uuid...", "at": "2026-05-20T14:00:00Z", "location": "Lab Reception"}
    -- ]

    created_by      UUID NOT NULL REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, sample_id_human)
);
CREATE INDEX idx_sample_tenant_status ON sample(tenant_id, status);
CREATE INDEX idx_sample_client ON sample(client_id);
CREATE INDEX idx_sample_date_received ON sample(date_received);
CREATE INDEX idx_sample_custom_fields ON sample USING GIN (custom_fields);
```

## Test & Method Management

```sql
CREATE TABLE test_method (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,
    method_code     TEXT NOT NULL,
    standard_ref    TEXT,
    version         TEXT NOT NULL DEFAULT '1.0',
    is_accredited   BOOLEAN NOT NULL DEFAULT false,

    -- JSONB: method-specific configuration
    method_config   JSONB NOT NULL DEFAULT '{}',
    -- Example for EPA 524.2 (VOCs by GC-MS):
    -- {
    --   "instrument_type": "GC-MS",
    --   "sample_volume_ml": 25,
    --   "purge_time_min": 11,
    --   "trap_temperature_c": 30,
    --   "desorb_temperature_c": 250,
    --   "internal_standards": ["Fluorobenzene", "1,4-Difluorobenzene"],
    --   "surrogate_standards": ["4-Bromofluorobenzene"],
    --   "qc_requirements": {
    --     "method_blank": {"frequency": "1_per_batch"},
    --     "lcs": {"frequency": "1_per_batch", "recovery_range": [70, 130]},
    --     "ms_msd": {"frequency": "1_per_20_samples", "rpd_limit": 30}
    --   }
    -- }

    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, method_code, version)
);

CREATE TABLE test_definition (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,
    test_code       TEXT NOT NULL,
    loinc_code      TEXT,
    snomed_code     TEXT,
    category        TEXT,
    test_method_id  UUID NOT NULL REFERENCES test_method(id),
    unit            TEXT,
    decimal_places  INTEGER DEFAULT 2,
    result_type     TEXT NOT NULL DEFAULT 'numeric',
    detection_limit NUMERIC,
    quantitation_limit NUMERIC,

    -- JSONB: specification limits per context
    specifications  JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "default": {"lower": 0, "upper": 0.015, "unit": "mg/L"},
    --   "drinking_water_epa": {"lower": 0, "upper": 0.015, "unit": "mg/L", "ref": "EPA MCL"},
    --   "drinking_water_eu": {"lower": 0, "upper": 0.010, "unit": "mg/L", "ref": "EU Directive 2020/2184"},
    --   "wastewater_npdes": {"lower": null, "upper": 0.065, "unit": "mg/L", "ref": "NPDES permit"},
    --   "pharma_usp": {"lower": null, "upper": 0.5, "unit": "ppm", "ref": "USP <232>"}
    -- }
    -- This avoids a specification_limit junction table while supporting
    -- multi-jurisdiction and multi-product-type specifications.

    -- JSONB: calculation configuration for derived results
    calculation     JSONB,
    -- {
    --   "formula": "((raw_value - blank_value) * dilution_factor) / sample_weight",
    --   "inputs": ["raw_value", "blank_value", "dilution_factor", "sample_weight"],
    --   "rounding": "round_half_up"
    -- }

    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, test_code)
);
CREATE INDEX idx_test_def_method ON test_definition(test_method_id);
CREATE INDEX idx_test_def_loinc ON test_definition(loinc_code);

CREATE TABLE analysis_profile (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,
    profile_code    TEXT NOT NULL,
    test_ids        UUID[] NOT NULL,                  -- array of test_definition IDs
    description     TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, profile_code)
);
```

## Analysis & Results

```sql
CREATE TABLE analysis (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    sample_id       UUID NOT NULL REFERENCES sample(id),
    test_definition_id UUID NOT NULL REFERENCES test_definition(id),
    worksheet_id    UUID,
    instrument_id   UUID,
    analyst_id      UUID REFERENCES app_user(id),
    status          TEXT NOT NULL DEFAULT 'pending',
    priority        TEXT NOT NULL DEFAULT 'normal',
    due_date        TIMESTAMPTZ,

    -- Core relational result fields
    result_numeric  NUMERIC,
    result_text     TEXT,
    result_boolean  BOOLEAN,
    unit            TEXT,
    spec_key        TEXT,                             -- which specification set was applied (from test_definition.specifications)
    spec_lower      NUMERIC,                         -- snapshot of limit at time of test
    spec_upper      NUMERIC,
    is_oos          BOOLEAN NOT NULL DEFAULT false,
    is_oot          BOOLEAN NOT NULL DEFAULT false,
    uncertainty     NUMERIC,                          -- measurement uncertainty (ISO 17025)
    uncertainty_k   NUMERIC DEFAULT 2,                -- coverage factor

    -- JSONB: method-specific result data
    method_data     JSONB NOT NULL DEFAULT '{}',
    -- GC-MS example:
    -- {
    --   "retention_time_min": 12.34,
    --   "peak_area": 15234,
    --   "signal_to_noise": 45.2,
    --   "internal_standard_area": 50000,
    --   "internal_standard_recovery_pct": 98.5,
    --   "surrogate_recovery_pct": 95.3,
    --   "dilution_factor": 1,
    --   "qualifier_ions": [91, 92, 65],
    --   "qualifier_ion_ratios": [0.58, 0.23]
    -- }
    --
    -- Microbiology example:
    -- {
    --   "colony_count": 42,
    --   "dilution": "1:100",
    --   "plate_type": "mFC",
    --   "incubation_temp_c": 44.5,
    --   "incubation_hours": 24,
    --   "organism_identified": "E. coli",
    --   "confirmation_test": "Indole positive"
    -- }

    -- JSONB: AI-generated metadata (added without schema migration)
    ai_metadata     JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "oos_probability": 0.12,
    --   "anomaly_score": 0.03,
    --   "trend_direction": "stable",
    --   "predicted_retest_needed": false,
    --   "model_version": "oos-predictor-v2.1",
    --   "inference_timestamp": "2026-05-22T10:30:00Z"
    -- }

    entered_by      UUID REFERENCES app_user(id),
    entered_at      TIMESTAMPTZ,
    reviewed_by     UUID REFERENCES app_user(id),
    reviewed_at     TIMESTAMPTZ,
    verified_by     UUID REFERENCES app_user(id),
    verified_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_analysis_sample ON analysis(sample_id);
CREATE INDEX idx_analysis_tenant_status ON analysis(tenant_id, status);
CREATE INDEX idx_analysis_worksheet ON analysis(worksheet_id);
CREATE INDEX idx_analysis_oos ON analysis(tenant_id, is_oos) WHERE is_oos = true;
CREATE INDEX idx_analysis_method_data ON analysis USING GIN (method_data);
CREATE INDEX idx_analysis_ai_metadata ON analysis USING GIN (ai_metadata);

-- Worksheet for grouping analyses
CREATE TABLE worksheet (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    worksheet_number TEXT NOT NULL,
    analyst_id      UUID NOT NULL REFERENCES app_user(id),
    instrument_id   UUID,
    test_method_id  UUID REFERENCES test_method(id),
    status          TEXT NOT NULL DEFAULT 'open',
    layout          JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "type": "list",
    --   "positions": [
    --     {"position": 1, "type": "blank", "analysis_id": null},
    --     {"position": 2, "type": "standard", "analysis_id": "uuid..."},
    --     {"position": 3, "type": "sample", "analysis_id": "uuid..."},
    --     ...
    --     {"position": 20, "type": "qc", "qc_type": "CCV", "analysis_id": null}
    --   ]
    -- }
    opened_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    submitted_at    TIMESTAMPTZ,
    verified_at     TIMESTAMPTZ,
    verified_by     UUID REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, worksheet_number)
);

-- Add FK from analysis to worksheet and instrument
ALTER TABLE analysis
    ADD CONSTRAINT fk_analysis_worksheet
    FOREIGN KEY (worksheet_id) REFERENCES worksheet(id);
```

## Instruments & Calibration

```sql
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
    location        TEXT,

    -- JSONB: communication configuration (varies per instrument type)
    connection      JSONB NOT NULL DEFAULT '{}',
    -- RS-232 example:
    -- {"protocol": "RS232", "port": "/dev/ttyUSB0", "baud_rate": 9600, "data_bits": 8, "parity": "none"}
    -- TCP/IP example:
    -- {"protocol": "TCP_IP", "host": "192.168.1.50", "port": 9100}
    -- ASTM example:
    -- {"protocol": "ASTM", "host": "192.168.1.51", "port": 15200, "frame_size": 247}

    -- JSONB: instrument-specific metadata
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "firmware_version": "3.2.1",
    --   "detector_type": "FID",
    --   "column_type": "DB-5ms 30m x 0.25mm",
    --   "gas_type": "Helium",
    --   "software_version": "Chromeleon 7.3",
    --   "last_pm_date": "2026-04-15",
    --   "next_pm_date": "2026-10-15"
    -- }

    commissioned_date DATE,
    decommissioned_date DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, name)
);
CREATE INDEX idx_instrument_tenant_status ON instrument(tenant_id, status);

-- Add FK from analysis to instrument
ALTER TABLE analysis
    ADD CONSTRAINT fk_analysis_instrument
    FOREIGN KEY (instrument_id) REFERENCES instrument(id);

ALTER TABLE worksheet
    ADD CONSTRAINT fk_worksheet_instrument
    FOREIGN KEY (instrument_id) REFERENCES instrument(id);

CREATE TABLE calibration (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    instrument_id   UUID NOT NULL REFERENCES instrument(id),
    calibration_date DATE NOT NULL,
    next_due_date   DATE NOT NULL,
    calibration_type TEXT NOT NULL,                    -- 'internal', 'external', 'verification'
    performed_by    UUID REFERENCES app_user(id),
    external_provider TEXT,
    result          TEXT NOT NULL,                     -- 'pass', 'fail', 'adjusted'
    certificate_number TEXT,

    -- JSONB: calibration-specific data
    calibration_data JSONB NOT NULL DEFAULT '{}',
    -- Balance example:
    -- {
    --   "reference_weights": [
    --     {"nominal_g": 1.0, "certified_g": 1.00003, "measured_g": 1.00005, "uncertainty_g": 0.00002},
    --     {"nominal_g": 10.0, "certified_g": 10.0001, "measured_g": 10.0003, "uncertainty_g": 0.00005},
    --     {"nominal_g": 100.0, "certified_g": 100.001, "measured_g": 100.002, "uncertainty_g": 0.0005}
    --   ],
    --   "eccentricity_test": {"max_deviation_mg": 0.2, "pass": true},
    --   "repeatability_test": {"std_dev_mg": 0.05, "pass": true},
    --   "traceability": "NIST weights certified to SI via NVLAP accredited lab"
    -- }

    remarks         TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_calibration_instrument ON calibration(instrument_id);
CREATE INDEX idx_calibration_next_due ON calibration(next_due_date);
```

## Quality Control

```sql
CREATE TABLE qc_result (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    worksheet_id    UUID NOT NULL REFERENCES worksheet(id),
    test_definition_id UUID NOT NULL REFERENCES test_definition(id),
    instrument_id   UUID REFERENCES instrument(id),
    qc_type         TEXT NOT NULL,                    -- 'method_blank', 'lcs', 'lcsd', 'ms', 'msd', 'crm', 'ccv', 'ccb', 'duplicate'

    -- Core relational QC fields
    result_value    NUMERIC,
    expected_value  NUMERIC,
    recovery_pct    NUMERIC,
    rpd             NUMERIC,
    is_acceptable   BOOLEAN NOT NULL DEFAULT true,

    -- JSONB: QC evaluation details
    evaluation      JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "acceptance_range": [80, 120],
    --   "z_score": 1.3,
    --   "bias_pct": 2.1,
    --   "westgard_rules_checked": ["1_2s", "2_2s", "R_4s", "4_1s", "10x"],
    --   "westgard_violations": [],
    --   "levey_jennings": {
    --     "mean": 10.05,
    --     "sd": 0.15,
    --     "n": 45,
    --     "control_chart_position": "within_1sd"
    --   }
    -- }

    entered_by      UUID NOT NULL REFERENCES app_user(id),
    entered_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_qc_result_worksheet ON qc_result(worksheet_id);
CREATE INDEX idx_qc_result_test_instrument ON qc_result(test_definition_id, instrument_id);
CREATE INDEX idx_qc_result_entered ON qc_result(entered_at);
```

## Reporting

```sql
CREATE TABLE report_template (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,
    template_type   TEXT NOT NULL,                    -- 'coa', 'test_report', 'compliance'
    template_body   TEXT NOT NULL,                    -- HTML/Jinja2 template
    template_config JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "header_logo_url": "https://...",
    --   "footer_text": "This report shall not be reproduced...",
    --   "show_uncertainty": true,
    --   "show_method_reference": true,
    --   "accreditation_logo_url": "https://...",
    --   "signatory_fields": ["lab_director", "quality_manager"]
    -- }
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, name)
);

CREATE TABLE report (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    report_number   TEXT NOT NULL,
    template_id     UUID NOT NULL REFERENCES report_template(id),
    sample_ids      UUID[] NOT NULL,                  -- array of sample IDs included
    status          TEXT NOT NULL DEFAULT 'draft',
    issued_at       TIMESTAMPTZ,
    issued_by       UUID REFERENCES app_user(id),
    pdf_url         TEXT,
    amendment_of    UUID REFERENCES report(id),
    amendment_reason TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, report_number)
);
```

## Audit Trail & Electronic Signatures

```sql
-- ============================================================
-- AUDIT LOG — fully relational (never JSONB for compliance data)
-- ============================================================

CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    user_id         UUID NOT NULL,
    user_display    TEXT NOT NULL,                     -- denormalized for immutability
    action          TEXT NOT NULL,
    entity_type     TEXT NOT NULL,
    entity_id       UUID NOT NULL,
    field_name      TEXT,
    old_value       TEXT,
    new_value       TEXT,
    reason          TEXT,                              -- mandatory for regulated changes
    ip_address      INET,
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);
-- Append-only: REVOKE UPDATE, DELETE ON audit_log FROM PUBLIC;
CREATE INDEX idx_audit_log_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_log_tenant_time ON audit_log(tenant_id, occurred_at);
CREATE INDEX idx_audit_log_user ON audit_log(user_id);

CREATE TABLE e_signature (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    signer_id       UUID NOT NULL REFERENCES app_user(id),
    entity_type     TEXT NOT NULL,
    entity_id       UUID NOT NULL,
    meaning         TEXT NOT NULL,
    full_name       TEXT NOT NULL,
    title           TEXT,
    reason          TEXT,
    auth_method     TEXT NOT NULL DEFAULT 'password',
    signed_at       TIMESTAMPTZ NOT NULL DEFAULT now()
);
-- Append-only: REVOKE UPDATE, DELETE ON e_signature FROM PUBLIC;
CREATE INDEX idx_esig_entity ON e_signature(entity_type, entity_id);
```

## Batch, Stability & Environmental Monitoring

```sql
CREATE TABLE batch (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    batch_number    TEXT NOT NULL,
    description     TEXT,
    client_id       UUID REFERENCES client(id),
    status          TEXT NOT NULL DEFAULT 'open',
    metadata        JSONB NOT NULL DEFAULT '{}',
    created_by      UUID NOT NULL REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, batch_number)
);

ALTER TABLE sample
    ADD CONSTRAINT fk_sample_batch
    FOREIGN KEY (batch_id) REFERENCES batch(id);

CREATE TABLE stability_study (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    study_number    TEXT NOT NULL,
    product_name    TEXT NOT NULL,
    condition       TEXT NOT NULL,
    start_date      DATE NOT NULL,
    status          TEXT NOT NULL DEFAULT 'active',

    -- JSONB: study protocol and schedule
    protocol        JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "storage_condition": "25C/60%RH",
    --   "timepoints_months": [0, 1, 3, 6, 9, 12, 18, 24, 36],
    --   "tests_per_timepoint": ["assay", "dissolution", "moisture", "degradation"],
    --   "acceptance_criteria": {
    --     "assay": {"lower_pct": 90, "upper_pct": 110},
    --     "dissolution": {"min_pct_30min": 80}
    --   }
    -- }

    created_by      UUID NOT NULL REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, study_number)
);

CREATE TABLE stability_timepoint (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    study_id        UUID NOT NULL REFERENCES stability_study(id),
    timepoint_months INTEGER NOT NULL,
    scheduled_date  DATE NOT NULL,
    sample_id       UUID REFERENCES sample(id),
    status          TEXT NOT NULL DEFAULT 'scheduled',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_stability_tp_study ON stability_timepoint(study_id);

CREATE TABLE env_monitoring_location (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    site_id         UUID NOT NULL REFERENCES site(id),
    name            TEXT NOT NULL,
    monitoring_type TEXT NOT NULL,
    config          JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "area_classification": "ISO 7",
    --   "frequency": "daily",
    --   "alert_limit": 50,
    --   "action_limit": 100,
    --   "unit": "CFU/m3"
    -- }
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, name)
);

CREATE TABLE env_reading (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    location_id     UUID NOT NULL REFERENCES env_monitoring_location(id),
    value           NUMERIC NOT NULL,
    is_alert        BOOLEAN NOT NULL DEFAULT false,
    is_action       BOOLEAN NOT NULL DEFAULT false,
    recorded_by     UUID REFERENCES app_user(id),
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    source          TEXT NOT NULL DEFAULT 'manual'
);
CREATE INDEX idx_env_reading_location_time ON env_reading(location_id, recorded_at);
```

## Instrument Data Import

```sql
CREATE TABLE instrument_import (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    instrument_id   UUID NOT NULL REFERENCES instrument(id),
    filename        TEXT NOT NULL,
    file_format     TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'pending',
    error_message   TEXT,

    -- JSONB: parsed results before mapping
    parsed_results  JSONB NOT NULL DEFAULT '[]',
    -- [
    --   {"raw_sample_id": "WQ-2026-001234", "analyte": "Lead", "value": 0.0023, "unit": "mg/L", "mapped_analysis_id": "uuid..."},
    --   {"raw_sample_id": "WQ-2026-001234", "analyte": "Copper", "value": 0.045, "unit": "mg/L", "mapped_analysis_id": "uuid..."},
    --   {"raw_sample_id": "BLANK", "analyte": "Lead", "value": 0.0001, "unit": "mg/L", "mapped_analysis_id": null}
    -- ]

    imported_by     UUID NOT NULL REFERENCES app_user(id),
    imported_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_import_instrument ON instrument_import(instrument_id);
```

## Useful Query Examples

```sql
-- ============================================================
-- EXAMPLE: Find samples with specific custom field values
-- ============================================================

-- "Find all water samples from Well MW-3 in the last month"
SELECT s.sample_id_human, s.status, s.date_received, s.custom_fields
FROM sample s
WHERE s.tenant_id = current_setting('app.current_tenant_id')::UUID
  AND s.custom_fields @> '{"sampling_point": "Well MW-3"}'
  AND s.date_received >= now() - INTERVAL '30 days';

-- ============================================================
-- EXAMPLE: Query method-specific result data
-- ============================================================

-- "Find all GC-MS results with surrogate recovery below 70%"
SELECT
    a.id,
    s.sample_id_human,
    td.name AS test_name,
    a.result_numeric,
    a.method_data->>'surrogate_recovery_pct' AS surrogate_recovery
FROM analysis a
JOIN sample s ON s.id = a.sample_id
JOIN test_definition td ON td.id = a.test_definition_id
WHERE a.tenant_id = current_setting('app.current_tenant_id')::UUID
  AND (a.method_data->>'surrogate_recovery_pct')::NUMERIC < 70;

-- ============================================================
-- EXAMPLE: Multi-jurisdiction specification lookup
-- ============================================================

-- "What is the Lead specification for EU drinking water vs EPA?"
SELECT
    td.name,
    td.test_code,
    td.specifications->'drinking_water_epa'->>'upper' AS epa_limit,
    td.specifications->'drinking_water_eu'->>'upper' AS eu_limit,
    td.specifications->'drinking_water_epa'->>'ref' AS epa_reference,
    td.specifications->'drinking_water_eu'->>'ref' AS eu_reference
FROM test_definition td
WHERE td.tenant_id = current_setting('app.current_tenant_id')::UUID
  AND td.test_code = 'LEAD';

-- ============================================================
-- EXAMPLE: AI metadata query
-- ============================================================

-- "Find analyses where AI predicts OOS probability > 50%"
SELECT
    a.id,
    s.sample_id_human,
    td.name AS test_name,
    a.result_numeric,
    (a.ai_metadata->>'oos_probability')::NUMERIC AS oos_probability,
    a.ai_metadata->>'model_version' AS model_version
FROM analysis a
JOIN sample s ON s.id = a.sample_id
JOIN test_definition td ON td.id = a.test_definition_id
WHERE a.tenant_id = current_setting('app.current_tenant_id')::UUID
  AND (a.ai_metadata->>'oos_probability')::NUMERIC > 0.5
ORDER BY (a.ai_metadata->>'oos_probability')::NUMERIC DESC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Schema Registry | 1 | json_schema_registry |
| Tenancy & Organisation | 2 | tenant, site |
| Users & Access Control | 3 | app_user, role, user_role |
| Client Management | 1 | client |
| Sample Management | 2 | sample, sample_type |
| Test & Method | 3 | test_method, test_definition, analysis_profile |
| Analysis & Results | 2 | analysis, worksheet |
| Instruments & Calibration | 2 | instrument, calibration |
| Quality Control | 1 | qc_result |
| Reporting | 2 | report_template, report |
| Audit & Signatures | 2 | audit_log, e_signature |
| Batch & Stability | 3 | batch, stability_study, stability_timepoint |
| Environmental Monitoring | 2 | env_monitoring_location, env_reading |
| Instrument Import | 1 | instrument_import |
| **Total** | **27** | ~40% fewer tables than normalized; JSONB absorbs variation |

---

## Key Design Decisions

1. **Relational core, JSONB periphery** — the sample-test-result pipeline, audit trail, and electronic signatures are strictly relational. JSONB is used only for data that genuinely varies by lab type, method, jurisdiction, or AI model version. This avoids "JSONB everything" anti-patterns while keeping the schema manageable.

2. **JSON Schema registry for validation** — every JSONB column has a registered JSON Schema. The application validates payloads against these schemas before writing, providing the data integrity guarantees that the database cannot enforce on JSONB natively.

3. **Specifications as JSONB on test_definition** — rather than a separate specification_limit junction table, multi-jurisdiction specification limits are stored as a keyed JSONB object. This is a deliberate trade-off: it reduces table count and simplifies the common "look up the limit for this test in this jurisdiction" query, at the cost of not being able to JOIN on specification keys.

4. **AI metadata as a dedicated JSONB column** — `analysis.ai_metadata` provides a clean extension point for ML-generated predictions, anomaly scores, and classification labels. New AI features can be deployed without database migrations, which is critical for rapid ML iteration.

5. **Chain of custody embedded in sample JSONB** — for most labs, the custody chain is a simple sequential log that doesn't need its own table. Embedding it in `sample.custody_chain` as a JSONB array reduces table count. Labs with complex custody requirements can still promote this to a separate table.

6. **Method-specific data in analysis.method_data** — GC-MS retention times, microbiology colony counts, and spectroscopy peak areas are fundamentally different data structures. Storing them in a typed JSONB column avoids dozens of method-specific result sub-tables while preserving the ability to query method-specific fields via GIN indexes.

7. **Worksheet layout as JSONB** — worksheet position arrangements (96-well plates, linear lists, custom layouts) are stored as JSONB rather than separate position tables. This simplifies the common "render the worksheet" operation while keeping the layout definition flexible.

8. **Audit trail and e-signatures remain fully relational** — these tables are never JSONB because 21 CFR Part 11 auditors expect structured, queryable, immutable records. The REVOKE UPDATE/DELETE on these tables provides database-level enforcement.

9. **User qualifications as JSONB array** — CLIA and ISO 17025 require personnel competency records. These vary widely (degrees, certifications, method-specific competencies) and are best modeled as a JSONB array rather than separate qualification tables.

10. **GIN indexes on key JSONB columns** — `sample.custom_fields`, `analysis.method_data`, and `analysis.ai_metadata` all have GIN indexes to support containment queries (@>) without full-table scans. This is essential for the AI metadata queries that will power predictive dashboards.

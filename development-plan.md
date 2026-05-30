# Lab Information Management System (LIMS) — Phased Development Plan

> Project: 251-lab-information-management-system · Created: 2026-05-29
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and the four `data-model-suggestion-*.md` files. The build delivers an open-source, **API-first, AI-native LIMS** with out-of-the-box ISO/IEC 17025:2017 and FDA 21 CFR Part 11 scaffolding, targeting SMB and mid-market pharmaceutical QC, clinical, biotech, food/environmental, and academic labs.

The chosen data architecture is **Data Model Suggestion 3 (Hybrid Relational + JSONB)**: a strict relational backbone for the universal sample → test → result pipeline plus all compliance records (audit trail, e-signatures, calibration), with typed, schema-validated JSONB columns for the ~30% of data that varies by lab type, method, jurisdiction, and AI model version. This is the best fit for a multi-purpose AI-native LIMS because `analysis.ai_metadata` and `analysis.method_data` accept new ML features and instrument formats without migrations, while audit and e-signature tables stay fully relational and immutable for 21 CFR Part 11. Concepts from Suggestion 1 (separate `test_method`/`test_definition`, append-only audit) and Suggestion 4 (calibration traceability chain) are folded in where they strengthen compliance.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary language | Python 3.12 | Dominant ecosystem for the AI-native differentiators (OOS prediction, anomaly detection, LLM compliance narratives, NL query); rich scientific libraries (NumPy, pandas, SciPy) for Westgard/Levey-Jennings statistics; readable for the domain-expert contributors the README invites. |
| API framework | FastAPI 0.115+ | Generates an **OpenAPI 3.1** document automatically (standards.md requirement), giving the "100% API parity" differentiator and free SDK generation for the planned Python/JS/R clients. Async support is essential for instrument I/O and LLM calls. Pydantic v2 enforces request/response schemas and validates JSONB payloads against JSON Schema. |
| ORM / migrations | SQLAlchemy 2.0 + Alembic | Mature typed ORM; Alembic gives versioned, auditable migrations (GAMP 5 / CSV change control). |
| Database | PostgreSQL 16 | Required for the hybrid model: native JSONB + GIN indexes, `ltree` (optional traceability), Row-Level Security for multi-tenant isolation, `REVOKE UPDATE/DELETE` to make audit/event tables immutable at the storage layer (ALCOA+, 21 CFR Part 11). |
| Task queue | Celery 5 + Redis 7 | Async workloads: instrument file parsing, PDF/CoA rendering, HL7/FHIR messaging, batch LLM inference, scheduled calibration/stability reminders. Redis also backs caching and rate limiting. |
| AI / LLM access | LiteLLM + a thin `LLMProvider` abstraction; scikit-learn for ML | LiteLLM keeps the system **cloud-agnostic** (the README's explicit advantage over Sapio's AWS Bedrock lock-in): one interface for OpenAI, Anthropic, Azure, or self-hosted Ollama. scikit-learn (gradient-boosted trees) covers OOS prediction and anomaly detection without GPU dependencies. |
| MCP server | Official `mcp` Python SDK | Exposes LIMS data (sample status, QC trends, traceability) as an MCP server — the early-mover opportunity in standards.md; lets any LLM host act as an "AI co-scientist" without LLM vendor coupling. |
| Frontend | React 18 + TypeScript + Vite + TanStack Query + shadcn/ui (Tailwind) | SPA consuming only the public REST API, proving API parity. TanStack Query handles caching/optimistic updates; shadcn/ui gives a modern, accessible UI (the gap vs. dated Senaite/LabWare UIs). Charts via Recharts for Levey-Jennings/QC. |
| PDF / report rendering | Jinja2 templates → WeasyPrint | HTML/CSS templates render to print-quality PDF Certificates of Analysis and ISO 17025 reports with accreditation logos and signatory blocks. |
| Instrument protocols | `pyserial` (RS-232), asyncio TCP server, custom ASTM E1381/E1394 framer, `hl7apy` (HL7 v2), `fhir.resources` (FHIR R4) | Covers the hardware-level protocols every analytical instrument still speaks (standards.md) plus clinical messaging. |
| Auth | OAuth 2.0 + OIDC (Authlib) + JWT (`python-jose`) + TOTP MFA (`pyotp`) | RFC 6749 / 7519 / OIDC compliance; JWT for stateless API auth; OIDC for enterprise SSO (Azure AD/Okta); TOTP MFA required for 21 CFR Part 11 e-signature re-authentication. |
| Testing | pytest + pytest-asyncio + httpx + Testcontainers (Postgres) + Playwright (E2E) | Unit/integration with a real ephemeral Postgres via Testcontainers (RLS and JSONB behaviour cannot be faithfully mocked); Playwright drives the SPA for end-to-end compliance flows. |
| Code quality | ruff (lint+format) + mypy (strict) + pre-commit | Single fast toolchain; mypy strict gives the type safety regulated software benefits from. |
| Containerisation | Docker + docker-compose | Self-hosted and cloud deployment (README). compose wires api, worker, postgres, redis, frontend. |
| Package manager | uv | Fast, reproducible locking for the Python service. |
| CI | GitHub Actions | Lint, type-check, test matrix, Docker build, OpenAPI spec diff. |

### Project Structure

```
lims/
├── pyproject.toml
├── uv.lock
├── README.md
├── docker-compose.yml
├── Dockerfile                      # API + worker image
├── .pre-commit-config.yaml
├── alembic.ini
├── migrations/                     # Alembic versions
│   └── versions/
├── src/
│   └── lims/
│       ├── __init__.py
│       ├── main.py                 # FastAPI app factory, router registration
│       ├── config.py               # Pydantic Settings (env-driven)
│       ├── db/
│       │   ├── session.py          # engine, session, tenant RLS context
│       │   ├── base.py             # declarative Base, mixins (TenantMixin, TimestampMixin)
│       │   └── models/             # SQLAlchemy models, one file per domain
│       │       ├── tenancy.py      # tenant, site
│       │       ├── identity.py     # app_user, role, user_role
│       │       ├── client.py
│       │       ├── sample.py       # sample, sample_type
│       │       ├── test.py         # test_method, test_definition, analysis_profile
│       │       ├── analysis.py     # analysis, worksheet
│       │       ├── instrument.py   # instrument, calibration, reference_standard
│       │       ├── qc.py           # qc_result
│       │       ├── inventory.py    # reagent, reagent_lot
│       │       ├── reporting.py    # report, report_template
│       │       ├── compliance.py   # nonconformance, corrective_action
│       │       ├── stability.py    # stability_study, stability_timepoint, env_*
│       │       ├── audit.py        # audit_log, e_signature  (immutable)
│       │       ├── attachment.py
│       │       ├── instrument_import.py
│       │       └── schema_registry.py  # json_schema_registry
│       ├── schemas/                # Pydantic request/response DTOs (per domain)
│       ├── services/               # business logic (framework-agnostic)
│       │   ├── sample_service.py
│       │   ├── result_service.py
│       │   ├── spec_engine.py      # OOS evaluation, spec resolution
│       │   ├── calc_engine.py      # configurable calculation rules
│       │   ├── westgard.py         # QC rule evaluation
│       │   ├── audit_service.py    # writes audit_log
│       │   ├── esign_service.py
│       │   ├── report_service.py
│       │   ├── coc_service.py      # chain of custody
│       │   ├── sequence_service.py # human-readable ID generation
│       │   └── jsonschema_service.py  # validate JSONB vs registry
│       ├── api/
│       │   ├── deps.py             # auth, tenant, permission dependencies
│       │   └── routes/             # FastAPI routers (one per domain)
│       ├── auth/
│       │   ├── jwt.py
│       │   ├── oidc.py
│       │   ├── mfa.py
│       │   └── permissions.py      # permission constants + RBAC checks
│       ├── instruments/
│       │   ├── parsers/            # csv, xml, astm, generic
│       │   ├── astm/               # E1381 framing, E1394 records
│       │   ├── serial_listener.py
│       │   └── tcp_listener.py
│       ├── interop/
│       │   ├── hl7v2.py            # ORM/ORU mapping
│       │   ├── fhir.py             # DiagnosticReport/Specimen/Observation
│       │   └── codes.py            # LOINC/SNOMED helpers
│       ├── ai/
│       │   ├── llm.py              # LLMProvider abstraction (LiteLLM)
│       │   ├── nlq.py              # natural-language query → safe SQL/API
│       │   ├── narrative.py        # compliance narrative generation
│       │   ├── oos_predictor.py    # sklearn model train/infer
│       │   ├── anomaly.py          # QC anomaly detection
│       │   └── routing.py          # sample routing optimisation
│       ├── mcp/
│       │   └── server.py           # MCP server exposing LIMS tools/resources
│       ├── tasks/                  # Celery tasks
│       └── reporting/templates/    # Jinja2 CoA / report templates
├── tests/
│   ├── conftest.py                 # Testcontainers Postgres, tenant fixtures
│   ├── unit/
│   ├── integration/
│   └── e2e/                        # Playwright specs
├── sdk/                            # generated/maintained clients
│   ├── python/
│   ├── js/
│   └── r/
└── frontend/
    ├── package.json
    ├── vite.config.ts
    └── src/
        ├── api/                    # generated OpenAPI client
        ├── components/
        ├── pages/
        └── lib/
```

The structure is additive: every phase below populates existing folders rather than restructuring them.

---

## Phase 1: Foundation, Tenancy & Auth

### Purpose
Establish the runnable skeleton: configuration, database with multi-tenant Row-Level Security, the immutable audit/e-signature substrate, authentication, and RBAC. Nothing lab-specific yet, but every later phase depends on tenant isolation, the audit trail, and permission checks being correct from the first commit (retrofitting compliance is the failure mode of the incumbents).

### Tasks

#### 1.1 — Project scaffold, config, and app factory

**What**: A FastAPI app that boots, reads typed config from the environment, and exposes `/health` and `/openapi.json`.

**Design**:
```python
# config.py
class Settings(BaseSettings):
    database_url: str
    redis_url: str = "redis://redis:6379/0"
    jwt_secret: str
    jwt_algorithm: str = "HS256"
    access_token_ttl_minutes: int = 30
    refresh_token_ttl_days: int = 14
    llm_provider: str = "openai"          # openai|anthropic|azure|ollama
    llm_model: str = "gpt-4o-mini"
    llm_api_key: str | None = None
    environment: str = "development"
    model_config = SettingsConfigDict(env_prefix="LIMS_")

# main.py
def create_app() -> FastAPI:
    app = FastAPI(title="LIMS", version="0.1.0", openapi_version="3.1.0")
    register_routers(app)
    register_exception_handlers(app)   # uniform ProblemDetails (RFC 9457) errors
    return app
```
Error responses use a single `ProblemDetail` model (`type`, `title`, `status`, `detail`, `instance`, `errors[]`).

**Testing**:
- `Unit: Settings loads from env vars with LIMS_ prefix → correct typed values`
- `Unit: missing required LIMS_DATABASE_URL → ValidationError naming the field`
- `Integration: GET /health → 200 {"status":"ok"}`
- `Integration: GET /openapi.json → 200, openapi field == "3.1.0"`

#### 1.2 — Database layer, base mixins, tenant RLS context

**What**: SQLAlchemy engine/session, `TenantMixin`/`TimestampMixin`, and a request-scoped mechanism that sets `app.current_tenant_id` so Postgres RLS policies apply.

**Design**:
```python
# db/base.py
class Base(DeclarativeBase): ...

class TimestampMixin:
    created_at: Mapped[datetime] = mapped_column(server_default=func.now())
    updated_at: Mapped[datetime] = mapped_column(server_default=func.now(),
                                                 onupdate=func.now())

class TenantMixin:
    tenant_id: Mapped[UUID] = mapped_column(ForeignKey("tenant.id"), index=True)
```
Session dependency executes `SET LOCAL app.current_tenant_id = :tid` at the start of each tenant-scoped transaction. RLS policy template (applied per data table in migrations):
```sql
ALTER TABLE <t> ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation_<t> ON <t>
  USING (tenant_id = current_setting('app.current_tenant_id')::UUID);
```

**Testing** (Testcontainers Postgres, real RLS):
- `Integration: insert rows for tenant A and B; session set to A → query returns only A's rows`
- `Integration: session with no app.current_tenant_id set → RLS returns zero rows`
- `Integration: attempt to insert a row with mismatched tenant_id under tenant A's context → blocked by WITH CHECK`

#### 1.3 — Tenant, Site, User, Role, UserRole models + migrations

**What**: Core identity tables from the hybrid model (Suggestion 3), with `app_user.qualifications` and `tenant.config`/`regulatory_config` JSONB.

**Design**: Implement `tenant`, `site`, `app_user`, `role`, `user_role` exactly as in data-model-suggestion-3 (lines for `tenant`, `site`, `app_user`, `role`, `user_role`). `role.permissions` is `TEXT[]`. Permission strings are namespaced: `sample:create`, `result:enter`, `result:verify`, `report:issue`, `instrument:calibrate`, `admin:users`, etc. Seed three default roles per tenant: `lab_technician`, `quality_manager`, `lab_director`.

**Testing**:
- `Unit: default roles seeded → permission sets match expected constants`
- `Integration: create user with qualifications JSONB → round-trips, GIN-queryable`
- `Integration: UNIQUE(tenant_id,email) violated → IntegrityError surfaced as 409`

#### 1.4 — Immutable audit log & electronic signatures

**What**: `audit_log` and `e_signature` tables, made append-only at the DB level, plus `AuditService` and `ESignService`.

**Design**: Tables per Suggestion 3 (`audit_log` with `user_display` denormalised, `field_name`/`old_value`/`new_value`/`reason`; `e_signature` with `meaning`/`full_name`/`title`/`reason`/`auth_method`). Migration runs `REVOKE UPDATE, DELETE ON audit_log, e_signature FROM PUBLIC`. `AuditService.record(action, entity_type, entity_id, *, old, new, reason)` is called by all mutating services. `ESignService.sign(user, entity, meaning, reason, *, password|totp)` re-authenticates the signer (21 CFR Part 11 §11.200) before writing.
```python
class SignatureMeaning(StrEnum):
    APPROVAL = "approval"; REVIEW = "review"
    AUTHORSHIP = "authorship"; VERIFICATION = "verification"
```

**Testing**:
- `Integration: UPDATE on audit_log row → permission denied by Postgres`
- `Integration: DELETE on e_signature row → permission denied`
- `Unit: AuditService.record diffs old/new dicts → one audit row per changed field`
- `Integration: ESignService.sign with wrong password → 401, no signature written`
- `Integration: ESignService.sign with valid TOTP → e_signature row with meaning+timestamp`

#### 1.5 — Authentication: JWT, OIDC, MFA

**What**: Password + JWT login, refresh tokens, OIDC SSO, TOTP MFA enrolment.

**Design**: `POST /auth/login` → access+refresh JWT (claims: `sub`, `tenant_id`, `roles`, `exp`). `POST /auth/refresh`, `POST /auth/logout`. `GET /auth/oidc/login` / `/auth/oidc/callback` (Authlib) maps `oidc_subject` to `app_user`. `POST /auth/mfa/enroll` returns provisioning URI; `POST /auth/mfa/verify`. Passwords hashed with Argon2. Every login/logout writes an audit row.

**Testing**:
- `Integration: valid credentials → 200 with access+refresh tokens`
- `Integration: invalid password → 401, audit row action=login_failed`
- `Unit: expired access token → 401; valid refresh → new access token`
- `Integration (mocked OIDC provider): callback with valid code → session for mapped user`
- `Unit: TOTP verify with correct code → enabled; with skewed code outside window → rejected`

#### 1.6 — Permission dependencies (RBAC)

**What**: FastAPI dependencies `current_user`, `require_permission(perm)`, `current_tenant`.

**Design**: `require_permission("result:verify")` resolves the user's effective permissions (union of roles, optionally site-scoped via `user_role.site_id`) and raises 403 otherwise.

**Testing**:
- `Unit: user with role lacking perm → 403`
- `Unit: user with site-scoped role accessing other site's resource → 403`
- `Integration: lab_director can hit an admin-only route; technician cannot`

---

## Phase 2: Reference Data, Samples & Chain of Custody

### Purpose
Deliver the entry point of every LIMS workflow: registering reference data (clients, sample types) and logging samples with human-readable IDs, status lifecycle, custody tracking, and the JSONB `custom_fields` that make the system serve environmental, clinical, and pharma labs from one schema. After this phase a lab can log and track physical samples.

### Tasks

#### 2.1 — JSON Schema registry & validation service

**What**: `json_schema_registry` table and a service that validates any JSONB payload against its registered schema before write.

**Design**: Table per Suggestion 3. `JsonSchemaService.validate(schema_name, payload)` loads the active schema (cached in Redis) and validates with `jsonschema` (Draft 2020-12). Registering invalid JSON Schema is rejected. Seed `sample.custom_fields` schema from the suggestion.

**Testing**:
- `Unit: payload conforming to registered schema → ok`
- `Unit: payload with wrong type for field → ValidationError naming the path`
- `Unit: unknown schema_name → 422 with clear message`

#### 2.2 — Client and Sample-Type management

**What**: CRUD for `client` (JSONB `contact`) and `sample_type` (with `category` and `default_custom_fields_schema`).

**Design**: Routers `/clients`, `/sample-types`; standard list (cursor pagination via RFC 8288 `Link` header), get, create, update, soft-delete (`is_active=false`). All mutations audited.

**Testing**:
- `Integration: create client with nested contact JSONB → 201, round-trips`
- `Integration: list clients paginated → Link header next/prev present`
- `Integration: soft-delete sets is_active=false, excluded from default list`

#### 2.3 — Human-readable ID sequence generator

**What**: Per-tenant, gapless, format-driven sample IDs (e.g. `ENV-2026-000123`).

**Design**: `SequenceService.next(scope)` uses a `sequence_counter(tenant_id, scope, period, value)` table with `SELECT ... FOR UPDATE`; format string from `tenant.config.sample_id_sequence_format` (`{YYYY}`, `{SEQ:6}`). Reset policy (yearly/never) configurable.

**Testing**:
- `Unit: format ENV-{YYYY}-{SEQ:6} → ENV-2026-000001 then ...000002`
- `Integration: 50 concurrent next() calls → 50 unique sequential IDs, no gaps`
- `Unit: yearly reset → counter restarts at new year boundary`

#### 2.4 — Sample registration & lifecycle

**What**: `sample` CRUD with status state machine and `custom_fields` validation.

**Design**: Model per Suggestion 3 (`sample` incl. `custom_fields`, embedded `custody_chain`, `parent_sample_id`, `batch_id` deferred FK). State machine:
```
registered → received → in_progress → awaiting_review → verified → published
             └──────────────→ cancelled    (any non-terminal → cancelled)
   published/verified → disposed
```
Transitions validated in `SampleService.transition(sample, new_status, user, reason)`; illegal transitions → 409. `custom_fields` validated against the schema named on its `sample_type`. Aliquot creation copies tenant/type and sets `parent_sample_id`.

**Testing**:
- `Integration: register sample → 201, ID auto-generated, status=registered, audit row`
- `Unit: transition registered→verified (skipping states) → 409 IllegalTransition`
- `Unit: clinical custom_fields missing patient_id when schema requires it → 422`
- `Integration: create aliquot → parent_sample_id set, shares tenant`

#### 2.5 — Chain of custody

**What**: Custody transfer/receive/store/dispose actions appended immutably.

**Design**: `CocService.transfer(sample, from_user, to_user, action, location)` appends to `sample.custody_chain` JSONB array (and writes an audit row). `GET /samples/{id}/custody` returns the chronological chain.

**Testing**:
- `Integration: transfer custody → entry appended with actor+timestamp`
- `Unit: custody actions are append-only (existing entries never mutated)`
- `Integration: dispose action requires sample in published/verified → else 409`

#### 2.6 — Barcode / QR label generation

**What**: Endpoint returning a printable label (PNG/PDF/ZPL) encoding the sample ID.

**Design**: `GET /samples/{id}/label?format=png|pdf|zpl`. Uses `python-barcode`/`qrcode`; ZPL template for Zebra printers. Encodes `sample_id_human` + a verification URL.

**Testing**:
- `Unit: PNG label decodes back to the sample_id_human`
- `Integration: ?format=zpl → text/plain ZPL with ^FD<id>^FS`

---

## Phase 3: Tests, Methods, Worksheets, Results & OOS Engine

### Purpose
The heart of the product. Define what tests exist (methods, test definitions, multi-jurisdiction specifications), assign them to samples, group work into worksheets, enter results, run configurable calculations, and automatically evaluate out-of-specification status. After this phase the core lab pipeline (login → test → result → OOS flag) is fully operational via the API.

### Tasks

#### 3.1 — Test method & test definition catalogue

**What**: `test_method`, `test_definition` (with JSONB `specifications`, `calculation`, `method_config`), and `analysis_profile`.

**Design**: Models per Suggestion 3. `test_definition.specifications` is the keyed multi-jurisdiction object (`default`, `drinking_water_epa`, `pharma_usp`, …). `loinc_code`/`snomed_code` optional. Separating method from definition mirrors ISO 17025 scope-of-accreditation (Suggestion 1 decision #3). `analysis_profile.test_ids` is `UUID[]`.

**Testing**:
- `Integration: create method then definition referencing it → 201`
- `Unit: specifications JSONB validates against registered schema`
- `Integration: create profile bundling 5 tests → expands to 5 on application`

#### 3.2 — Worksheet management

**What**: `worksheet` with JSONB `layout` (list / 96- / 384-well plate), analyst & instrument assignment, status lifecycle.

**Design**: Model per Suggestion 3. States `open → in_progress → submitted → verified → closed`. `layout.positions[]` records position→analysis mapping including QC slots. `WorksheetService.assign(worksheet, analysis_ids)` sets `analysis.worksheet_id` and positions.

**Testing**:
- `Integration: create plate_96 worksheet → 96 position slots`
- `Unit: submit worksheet with unfilled sample positions → 409`
- `Integration: assign analyses → analysis.worksheet_id updated`

#### 3.3 — Analysis (test request) assignment

**What**: `analysis` rows linking sample × test_definition, with status, analyst, instrument, due date.

**Design**: Model per Suggestion 3 (`analysis` incl. `method_data`, `ai_metadata`, `spec_key`, `spec_lower/upper` snapshot, `uncertainty`/`uncertainty_k`). `AnalysisService.request(sample, test_def | profile)` creates rows; status `pending → assigned → in_progress → complete → verified`.

**Testing**:
- `Integration: request profile on sample → one analysis per test, status=pending`
- `Unit: request test on cancelled sample → 409`

#### 3.4 — Calculation engine

**What**: Evaluate `test_definition.calculation` formulas to derive results.

**Design**: `CalcEngine.evaluate(formula, inputs, rounding)` parses a **restricted** arithmetic grammar (no arbitrary `eval`) via `asteval` with a whitelist of operators/functions. Inputs (`raw_value`, `blank_value`, `dilution_factor`, …) come from `analysis.method_data` and related rows. Rounding modes `round_half_up`/`round_half_even`, decimal places from `test_definition.decimal_places`.

**Testing**:
- `Unit: ((raw-blank)*df)/weight with known inputs → expected value`
- `Unit: formula referencing missing input → clear error, no result`
- `Unit: injection attempt (__import__) in formula → rejected at parse`
- `Unit: round_half_up at 2 dp → 0.125 → 0.13`

#### 3.5 — Result entry & specification (OOS) engine

**What**: Enter results (numeric/text/boolean), resolve the applicable spec, flag OOS.

**Design**: `SpecEngine.resolve(test_def, context)` selects the spec set by `spec_key` (jurisdiction/product), snapshots `spec_lower/upper` onto the `analysis` at entry time (so later spec edits don't rewrite history — ALCOA+ Original). `is_oos = value < spec_lower OR value > spec_upper`. `ResultService.enter(...)` validates `method_data` JSONB, runs the calc engine if configured, sets OOS, transitions `analysis` to `complete`, audits. Result amendment requires a `reason` and writes a new audit row plus increments an amendment counter.

**Testing**:
- `Unit: value above upper spec → is_oos=true`
- `Unit: value within spec → is_oos=false`
- `Integration: enter result → spec snapshot frozen even after spec_def later edited`
- `Integration: amend verified result without reason → 422; with reason → audited amendment`
- `Unit: text/boolean result types bypass OOS numeric logic`

#### 3.6 — Result verification with e-signature

**What**: Second-person verification gated by e-signature and analyst-independence check.

**Design**: `POST /analyses/{id}/verify` requires `result:verify`; rejects if verifier == analyst (independence, Suggestion 4 use case) → 409; calls `ESignService.sign(meaning=VERIFICATION)`; transitions analysis→`verified` and propagates to sample status when all analyses verified.

**Testing**:
- `Integration: analyst attempts to verify own result → 409 IndependenceViolation`
- `Integration: qualified verifier with valid signature → analysis verified, e_signature row`
- `Integration: all analyses verified → sample auto-transitions to awaiting_review/verified`

---

## Phase 4: Instruments, Calibration & Quality Control

### Purpose
Add the equipment side of the lab: instrument registry, ISO 17025 metrological traceability through calibration and reference standards, and statistical QC (Westgard rules, Levey-Jennings). This is what separates a sample tracker from a compliant LIMS, and it provides the structured QC data the AI phase will learn from.

### Tasks

#### 4.1 — Instrument & reference standard registry

**What**: `instrument` (JSONB `connection`, `metadata`) and `reference_standard` (certified value, uncertainty, traceability chain).

**Design**: `instrument` per Suggestion 3; `reference_standard` adopted from Suggestion 4 (certified_value, uncertainty, `traceability_chain` text e.g. "NIST → SI via NVLAP Lab #200XXX", expiry). `analysis.instrument_id`/`worksheet.instrument_id` FKs added.

**Testing**:
- `Integration: register RS-232 instrument with connection JSONB → validates against schema`
- `Integration: reference standard past expiry → flagged in list response`

#### 4.2 — Calibration scheduling & records

**What**: `calibration` with result, certificate, next-due; overdue detection.

**Design**: `calibration` per Suggestion 3 (+ `reference_standard_id` from Suggestion 4 for the traceability link). `CalibrationService.record(...)` writes the calibration, recomputes instrument calibration status; instruments with `next_due_date < today` or last result `fail` are blocked from new analyses (config-gated). A daily Celery task emits due/overdue reminders.

**Testing**:
- `Unit: next_due_date in past → instrument.calibration_overdue=true`
- `Integration: assign analysis to overdue instrument (strict mode) → 409`
- `Integration: failed calibration → impact list of analyses since last passing calibration`

#### 4.3 — Westgard rule engine

**What**: Evaluate QC results against Westgard multi-rules.

**Design**: `westgard.evaluate(series: list[QcPoint], mean, sd) -> list[Violation]` implements `1-2s` (warning), `1-3s`, `2-2s`, `R-4s`, `4-1s`, `10x`. Pure function over a numeric series; returns rule code + offending indices.
```python
@dataclass
class Violation: rule: str; indices: list[int]; severity: str
```

**Testing**:
- `Unit: one point > 3SD → 1-3s violation`
- `Unit: two consecutive points > 2SD same side → 2-2s`
- `Unit: range across two points > 4SD → R-4s`
- `Unit: ten consecutive on one side of mean → 10x`
- `Unit: in-control series → no violations`

#### 4.4 — QC result recording & Levey-Jennings data

**What**: `qc_result` capture with recovery/RPD, acceptance evaluation, and trend points.

**Design**: `qc_result` per Suggestion 3 (JSONB `evaluation` holding acceptance range, z-score, Westgard results, Levey-Jennings mean/sd/n). `QcService.record(...)` computes recovery_pct (LCS/MS), RPD (duplicates), z-score, runs the Westgard engine over the recent series for that test×instrument, sets `is_acceptable`, raises an automatic nonconformance on action-level violations (link to Phase 6 CAPA once available; until then store a flag).

**Testing**:
- `Unit: LCS recovery outside [70,130] → is_acceptable=false`
- `Unit: duplicate RPD > limit → is_acceptable=false`
- `Integration: QC series triggering 1-3s → evaluation.westgard_violations contains 1-3s`

#### 4.5 — QC chart query endpoint

**What**: Return Levey-Jennings series for a test×instrument over a date range.

**Design**: `GET /qc/charts?test_code=&instrument_id=&from=&to=` returns points with value, mean, ±1/2/3 SD bands, and flagged violations — feeds the frontend chart and the anomaly model.

**Testing**:
- `Integration: chart query → ordered points with SD bands and violation markers`
- `Integration: empty range → 200 with empty series`

---

## Phase 5: Reporting, Certificates of Analysis & Attachments

### Purpose
Turn verified results into the lab's deliverable: a Certificate of Analysis or ISO 17025 test report, with controlled templates, amendment lineage, e-signed issuance, and supporting attachments with integrity checksums. After this phase the lab can complete the full revenue workflow: log → test → verify → report.

### Tasks

#### 5.1 — Attachment store with integrity

**What**: `attachment` table + upload/download with SHA-256 checksums.

**Design**: `attachment` per Suggestion 1 (filename, content_type, size_bytes, storage_path, `checksum_sha256`, polymorphic `entity_type`/`entity_id`). Pluggable backend: local filesystem or S3-compatible (config). Checksum verified on download; mismatch → 409.

**Testing**:
- `Integration: upload file → checksum stored; download → bytes match, checksum re-verified`
- `Unit: corrupted stored file → download raises integrity error`

#### 5.2 — Report templates

**What**: `report_template` (Jinja2 body + JSONB `template_config`).

**Design**: Per Suggestion 3. `template_config` controls logos, footer, `show_uncertainty`, `show_method_reference`, accreditation logo, `signatory_fields`. Ship a default ISO 17025 test-report template and a CoA template.

**Testing**:
- `Unit: render template with sample context → HTML containing results table`
- `Unit: show_uncertainty=false → uncertainty column omitted`

#### 5.3 — Report generation & PDF rendering

**What**: `report` creation, async PDF render, and issuance with e-signature.

**Design**: `report` per Suggestion 3 (`sample_ids UUID[]`, `amendment_of`, `amendment_reason`, status `draft → issued → amended → superseded`). `ReportService.generate(template, sample_ids)` renders HTML via Jinja2, queues a Celery task to produce PDF (WeasyPrint), stores it as an attachment. `POST /reports/{id}/issue` requires `report:issue` + e-signature (meaning=APPROVAL), stamps `issued_at`/`issued_by`. Amending an issued report creates a new report with `amendment_of` set and supersedes the original.

**Testing**:
- `Integration: generate report → draft created, PDF attachment produced`
- `Integration: issue report without signature → 401; with signature → issued + audit`
- `Integration: amend issued report → new report amendment_of=original, original=superseded`
- `Unit: report includes only verified results; draft results excluded`

#### 5.4 — Compliance summary endpoint

**What**: Structured data feed (sample, results, QC, calibration, signatures) for one report — the input the AI narrative phase consumes.

**Design**: `GET /reports/{id}/compliance-data` returns a normalised JSON bundle (results with spec snapshots, applicable QC outcomes, instrument calibration status at time of test, signatures). Pure read; no LLM here.

**Testing**:
- `Integration: bundle includes calibration valid-at-test-time for each result's instrument`
- `Integration: bundle lists all e-signatures applied to the report's samples`

---

## Phase 6: CAPA, Stability, Environmental Monitoring & Inventory

### Purpose
Round out the quality-management surface that ISO 9001/17025 and pharma workflows require: nonconformance/CAPA tracking, stability studies with scheduled timepoints, environmental monitoring with alert/action limits, and reagent/lot inventory with expiry. These can be built largely in parallel once Phases 2–4 exist.

### Tasks

#### 6.1 — Nonconformance & corrective action (CAPA)

**What**: `nonconformance` and `corrective_action` workflow, auto-raised from OOS/QC failures.

**Design**: Tables per Suggestion 1/3. NC sources include `oos`, `qc_failure`, `internal_audit`, `complaint`. `CapaService.raise_from_oos(analysis)` is invoked by the OOS engine (Phase 3) and QC engine (Phase 4). Lifecycle `open → investigating → action_required → closed`; closing requires linked corrective actions to be `completed`/`verified`.

**Testing**:
- `Integration: OOS result → nonconformance auto-created linked to analysis`
- `Unit: close NC with open corrective actions → 409`
- `Integration: corrective action completion + effectiveness review → NC closeable`

#### 6.2 — Stability studies

**What**: `stability_study` (JSONB `protocol`) and `stability_timepoint` scheduling.

**Design**: Per Suggestion 3. `protocol.timepoints_months` drives auto-generation of `stability_timepoint` rows on study creation. A daily Celery task marks upcoming pulls due and (optionally) auto-registers the timepoint sample.

**Testing**:
- `Integration: create study with timepoints [0,3,6,12] → 4 timepoint rows scheduled`
- `Integration: due timepoint → reminder task flags it`

#### 6.3 — Environmental monitoring

**What**: `env_monitoring_location` (JSONB `config` with alert/action limits) and `env_reading` capture.

**Design**: Per Suggestion 3. `EnvService.record(location, value)` sets `is_alert`/`is_action` by comparing to `config.alert_limit`/`action_limit`; action breaches raise a nonconformance.

**Testing**:
- `Unit: reading above action_limit → is_action=true, NC raised`
- `Unit: reading between alert and action → is_alert=true, is_action=false`

#### 6.4 — Reagent & inventory tracking

**What**: `reagent`, `reagent_lot` with expiry and reorder levels.

**Design**: Adopt `reagent`/`reagent_lot` from Suggestion 1 (CAS number, grade, lot expiry, quantity_remaining, supplier CoA). Daily task flags expiring/expired lots and below-reorder reagents.

**Testing**:
- `Integration: consume from lot → quantity_remaining decremented`
- `Unit: expired lot used in analysis → warning surfaced`
- `Integration: reagent below reorder_level → appears in reorder report`

---

## Phase 7: Instrument Data Import & Interoperability (ASTM / HL7 / FHIR)

### Purpose
Connect the LIMS to the physical and clinical world: import instrument output files, listen on ASTM/RS-232/TCP, and exchange HL7 v2 / FHIR messages with hospital systems. This realises the integration differentiator and the LOINC/SNOMED coding from standards.md.

### Tasks

#### 7.1 — File import pipeline (CSV / XML / generic)

**What**: `instrument_import` ingestion with configurable parsers and result mapping.

**Design**: `instrument_import` per Suggestion 3 (JSONB `parsed_results[]`). `POST /imports` (file + instrument_id) stores raw, queues a Celery parse task. Parsers implement a common `Parser.parse(bytes) -> list[RawResult]` interface; a per-instrument mapping config maps `raw_sample_id`→sample and `raw_analyte`→test_definition. Mapped rows populate `analysis.result_numeric`/`method_data`. Unmapped rows surface for manual reconciliation.

**Testing**:
- `Fixture: sample CSV → parsed_results populated, status=parsed`
- `Integration: matching raw_sample_id maps to existing analysis → result auto-entered`
- `Unit: malformed XML → status=error with message, no partial results`

#### 7.2 — ASTM E1381/E1394 + serial/TCP listeners

**What**: Real-time instrument result capture over RS-232 and TCP using ASTM framing.

**Design**: `astm/` implements E1381 framing (ENQ/ACK/STX…ETX/checksum) and E1394 record types (H/P/O/R/L). `serial_listener.py` (pyserial) and `tcp_listener.py` (asyncio) run as worker processes, parse incoming records into `RawResult`, and feed the same mapping pipeline as 7.1.

**Testing**:
- `Unit: E1381 frame with valid checksum → ACK; bad checksum → NAK`
- `Unit: E1394 R-record → RawResult with analyte/value/unit`
- `Integration (loopback socket): stream ASTM session → results captured`

#### 7.3 — HL7 v2 messaging (ORM/ORU) & LOINC/SNOMED

**What**: Inbound order (ORM^O01) and outbound result (ORU^R01) messages.

**Design**: `interop/hl7v2.py` (hl7apy) parses ORM into sample+analysis; builds ORU from verified results. `interop/codes.py` maps internal test codes ↔ LOINC and specimen types ↔ SNOMED via `test_definition.loinc_code`/`sample_type.snomed_code`.

**Testing**:
- `Unit: ORM^O01 → sample + analyses created with mapped test codes`
- `Unit: verified results → well-formed ORU^R01 with LOINC OBX segments`
- `Unit: result missing LOINC → ORU uses local code with coding-system flag`

#### 7.4 — FHIR R4 resources

**What**: Expose/accept FHIR `DiagnosticReport`, `Specimen`, `Observation`, `ServiceRequest`.

**Design**: `interop/fhir.py` (`fhir.resources`) maps sample→Specimen, analysis→ServiceRequest, result→Observation (LOINC-coded), report→DiagnosticReport. `GET /fhir/DiagnosticReport/{id}` returns a valid FHIR JSON resource.

**Testing**:
- `Unit: sample → Specimen passes FHIR R4 schema validation`
- `Integration: GET /fhir/DiagnosticReport/{id} → valid resource with contained Observations`

---

## Phase 8: AI-Native Layer

### Purpose
Deliver the differentiators that no open-source LIMS has and incumbents have only nascently: cloud-agnostic LLM access, natural-language query, AI compliance narratives, predictive OOS, and QC anomaly detection. All AI features write to the JSONB `ai_metadata` extension point so they need no migrations, and an MCP server exposes LIMS data to any LLM host.

### Tasks

#### 8.1 — LLM provider abstraction

**What**: A single `LLMProvider` interface over OpenAI/Anthropic/Azure/Ollama via LiteLLM.

**Design**:
```python
class LLMProvider(Protocol):
    async def complete(self, system: str, user: str, *,
                       json_schema: dict | None = None) -> str: ...
```
Config-selected backend (`LIMS_LLM_PROVIDER`); supports structured (JSON-schema-constrained) output; retries with backoff; per-tenant token budgeting in Redis. No vendor lock-in (explicit advantage vs. Sapio/Bedrock).

**Testing**:
- `Integration (mocked LiteLLM): complete returns provider text`
- `Unit: provider switch via config → correct backend selected`
- `Unit: token budget exceeded → 429, no provider call`

#### 8.2 — Natural-language query

**What**: Plain-English questions over LIMS data for non-technical staff (Scibot-style).

**Design**: `ai/nlq.py` translates a question into a **constrained, parameterised** query plan against a whitelisted set of read views (samples, results, QC, instruments) — never raw free SQL. The LLM emits a structured plan (entity + filters + aggregation) validated by Pydantic; the service executes it with the caller's tenant + permissions enforced. Returns rows + a natural-language summary. `POST /ai/query`.

**Testing**:
- `Integration (mocked LLM): "show OOS results last 7 days" → plan filters is_oos + date, returns rows`
- `Unit: plan referencing a non-whitelisted table → rejected`
- `Integration: NLQ respects RLS — cannot reach other tenant's data even if asked`

#### 8.3 — AI compliance narrative generation

**What**: Generate audit-ready ISO 17025 / 21 CFR Part 11 narrative prose from the Phase 5 compliance bundle.

**Design**: `ai/narrative.py` feeds `/reports/{id}/compliance-data` into a templated prompt and returns a narrative for inclusion in reports. Output is clearly labelled AI-generated, requires human review + e-signature before issuance (GAMP 5 / EU Annex 22 explainability), and records `model_version` for change control.

**Testing**:
- `Integration (mocked LLM): bundle → narrative referencing the actual results/QC`
- `Unit: narrative cannot be attached to a report without human-review flag set`
- `Unit: generated narrative records model_version + timestamp`

#### 8.4 — Predictive OOS model

**What**: scikit-learn model predicting OOS probability before final results.

**Design**: `ai/oos_predictor.py` trains a gradient-boosted classifier on historical analyses (features: method, instrument, recent QC z-scores, reagent lot age, analyst, in-process `method_data`). Training is a Celery task per tenant; inference at result entry writes `analysis.ai_metadata.oos_probability` + `model_version`. Versioned model artifacts stored as attachments.

**Testing**:
- `Unit: feature extraction from analysis → fixed-length numeric vector`
- `Integration: train on fixture dataset → model artifact persisted`
- `Integration: inference on new analysis → ai_metadata.oos_probability in [0,1]`

#### 8.5 — QC anomaly detection

**What**: ML detection of QC drift beyond static Westgard rules.

**Design**: `ai/anomaly.py` runs an IsolationForest / EWMA over the Levey-Jennings series per test×instrument; scores written to `read_qc_trend`-equivalent JSONB and surfaced on the QC chart endpoint. Escalates to nonconformance on sustained anomaly.

**Testing**:
- `Unit: injected drift not caught by Westgard → anomaly_score elevated`
- `Unit: stable series → low anomaly scores`

#### 8.6 — MCP server

**What**: Expose LIMS data/actions as MCP tools/resources for any LLM host.

**Design**: `mcp/server.py` (official `mcp` SDK) exposes read tools (`get_sample_status`, `get_qc_trend`, `get_traceability`) and guarded write tools (`request_test`), each enforcing tenant + permission via a scoped token. Read resources expose sample/result documents. First LIMS with an MCP server (standards.md early-mover note).

**Testing**:
- `Integration: MCP get_sample_status with scoped token → status payload`
- `Unit: MCP write tool without permission scope → denied`
- `Unit: MCP tool cannot cross tenant boundary`

---

## Phase 9: Web Frontend (React SPA)

### Purpose
Provide the modern UI that closes the gap with dated open-source incumbents, consuming only the public REST API to prove 100% API parity. Built after the API stabilises so it tracks a fixed contract.

### Tasks

#### 9.1 — App shell, auth & generated API client

**What**: Vite + React + TS shell with login (incl. MFA/OIDC), routing, and an OpenAPI-generated typed client.

**Design**: Generate `frontend/src/api` from `/openapi.json` (openapi-typescript). TanStack Query for data; JWT stored in memory + refresh flow; role-aware navigation.

**Testing**:
- `E2E (Playwright): login with MFA → dashboard loads`
- `E2E: unauthorised route for technician role → hidden/redirect`

#### 9.2 — Sample & worksheet workspace

**What**: Sample login form (dynamic fields from sample-type schema), sample list/detail, worksheet view, result entry grid.

**Design**: Sample form renders fields from the JSON Schema of the selected sample type. Result grid supports keyboard-fast entry, shows OOS flags inline, and surfaces `ai_metadata.oos_probability`.

**Testing**:
- `E2E: log clinical sample → patient fields appear, sample created`
- `E2E: enter OOS result → row highlighted, OOS badge shown`

#### 9.3 — QC charts, calibration & reports UI

**What**: Levey-Jennings charts (Recharts) with Westgard/anomaly markers, calibration dashboard, report generation/issuance with e-signature modal.

**Design**: Chart consumes `/qc/charts`; e-signature modal re-prompts for password/TOTP before issue. Calibration dashboard lists overdue instruments.

**Testing**:
- `E2E: open QC chart → SD bands and violation markers render`
- `E2E: issue report → e-signature modal → PDF downloadable`

#### 9.4 — AI assistant panel

**What**: NL query box + narrative review UI.

**Design**: Calls `/ai/query`; renders result table + summary; narrative review screen with explicit "reviewed by human" checkbox gating attachment.

**Testing**:
- `E2E: ask "samples due today" → results table populated`
- `E2E: attach narrative without review checkbox → blocked`

---

## Phase 10: Client SDKs, Packaging & Deployment Hardening

### Purpose
Make the system consumable and deployable: Python/JS/R SDKs (matching OpenSpecimen/LabKey expectations), container packaging, seed/demo data, and the compliance/operational hardening (backups, retention, validation docs) that regulated buyers require.

### Tasks

#### 10.1 — Client SDKs

**What**: Generated, ergonomic Python, JavaScript, and R clients.

**Design**: Generate from OpenAPI (`openapi-python-client`, `openapi-typescript-codegen`, `rapiclient`/`Rlabkey`-style for R). Add thin convenience wrappers (auth, pagination iterators). Publish to PyPI/npm/CRAN-ready layout under `sdk/`.

**Testing**:
- `Integration: Python SDK round-trips sample create/get against a running API`
- `Unit: JS client types compile against the spec`

#### 10.2 — Docker & docker-compose

**What**: Production images and a one-command local stack.

**Design**: Multi-stage `Dockerfile` (api+worker). `docker-compose.yml` wires api, worker, beat, postgres, redis, frontend, with healthchecks and named volumes. Entrypoint runs Alembic migrations.

**Testing**:
- `E2E: docker-compose up → /health green, migrations applied, frontend served`

#### 10.3 — Seed data & demo tenant

**What**: A seed script creating a demo tenant with roles, sample types, methods, instruments, and sample data across lab types.

**Design**: `python -m lims.seed` idempotently creates the demo dataset for evaluation and tests.

**Testing**:
- `Integration: seed run twice → idempotent, no duplicates`

#### 10.4 — Compliance & operational hardening

**What**: Data retention, backup guidance, and a validation evidence pack.

**Design**: Retention driven by `tenant.regulatory_config.data_retention_years`; nightly backup task/documentation; export an audit-trail and validation document (mapping features → 21 CFR Part 11 / ISO 17025 clauses) generated from the codebase. Configurable `require_reason_for_change` and `require_esignature_on`.

**Testing**:
- `Integration: require_reason_for_change=true → mutating without reason → 422`
- `Integration: retention export → audit trail dumped in machine-readable form`
- `Unit: validation pack lists every e-signed action type`

---

## Phase Summary & Dependencies

```
Phase 1: Foundation, Tenancy & Auth        ─── required by everything
    │
Phase 2: Reference Data, Samples & CoC      ─── requires Phase 1
    │
Phase 3: Tests, Results & OOS Engine        ─── requires Phase 2   (core value ships here)
    │
    ├── Phase 4: Instruments, Calibration & QC      ─── requires Phase 3
    │
    ├── Phase 5: Reporting & CoA                     ─── requires Phase 3 (needs 4 for QC-in-report)
    │
    └── Phase 6: CAPA, Stability, EM & Inventory     ─── requires Phase 3 (CAPA auto-raise needs 4)
         │
Phase 7: Instrument Import & Interop (ASTM/HL7/FHIR) ─── requires Phase 3 (+4 for QC mapping)
    │
Phase 8: AI-Native Layer                    ─── requires Phases 3–7 (learns from results/QC/reports)
    │
Phase 9: Web Frontend                        ─── requires a stable API (Phases 1–8)
    │
Phase 10: SDKs, Packaging & Hardening        ─── requires Phases 1–8 (9 optional for SDK work)
```

**Parallelism opportunities:**
- After Phase 3: **Phases 4, 5, and 6** can be developed concurrently (note 5 and 6 reach full fidelity once 4 lands QC data).
- **Phase 7** can proceed alongside 5/6 once Phase 4 mapping targets exist.
- Within Phase 8, tasks **8.4 (OOS predictor)** and **8.5 (anomaly detection)** can parallel **8.2 (NLQ)** / **8.3 (narratives)** since they share only 8.1.
- **Phase 10.1 (SDKs)** can begin as soon as the OpenAPI contract is frozen, in parallel with Phase 9.

---

## Definition of Done (per phase)

A phase is complete only when all of the following hold:

1. All tasks in the phase are implemented.
2. All unit and integration tests for the phase pass (`pytest`), including the named scenarios above.
3. `ruff check` and `ruff format --check` pass with no findings.
4. `mypy --strict src/lims` passes.
5. `docker build` succeeds and `docker-compose up` brings the stack to a healthy state.
6. The phase's primary capability works end-to-end against a running instance (verified by an integration or E2E test).
7. New configuration options are documented (env vars in `config.py` and README).
8. New/changed API endpoints appear correctly in the generated `/openapi.json` (OpenAPI 3.1) and the spec-diff CI check is reviewed.
9. Alembic migration(s) are created, are reversible, and apply cleanly on an empty database.
10. Every new mutating operation writes an audit-trail entry, and every regulated state change (verify/issue/amend/sign) is gated by an electronic signature where the standards require it.
11. Any new JSONB column has a registered JSON Schema and is validated on write.
12. Multi-tenant isolation is enforced by Row-Level Security on every new data table, with a test proving cross-tenant access is blocked.

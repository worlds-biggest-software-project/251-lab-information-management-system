# Standards & API Reference

> Project: Lab Information Management System (LIMS) · Generated: 2026-05-03

---

## Industry Standards & Specifications

### ISO Standards

**ISO/IEC 17025:2017 — General Requirements for the Competence of Testing and Calibration Laboratories**
- URL: https://www.iso.org/standard/66912.html
- The primary accreditation standard for testing and calibration laboratories. A compliant LIMS must support full audit trails, measurement traceability, calibration scheduling, uncertainty reporting, and records management to satisfy assessors during accreditation audits.

**ISO 15189:2022 — Medical Laboratories: Requirements for Quality and Competence**
- URL: https://www.iso.org/standard/76677.html
- Specifies quality and competence requirements for medical laboratories. LIMS deployments in hospital or clinical diagnostic labs must support patient data confidentiality, sample traceability, QC procedures, and result reporting formats aligned with this standard.

**ISO 20387:2018 — Biotechnology: Biobanking — General Requirements for Biobanking**
- URL: https://www.iso.org/standard/67888.html
- Defines quality and competence requirements for biobanks managing human, animal, fungal, and plant biological materials. A LIMS targeting biobanking must support consent management, specimen lifecycle traceability, ethical data handling, and long-term cryostorage tracking.

**ISO 9001:2015 — Quality Management Systems: Requirements**
- URL: https://www.iso.org/standard/62085.html
- The foundational quality management standard; overlaps with ISO 17025 for document control, non-conformance management, corrective action, and calibration. LIMS should support CAPA workflows and controlled document versioning to enable ISO 9001 compliance.

**ISO/IEC 27001:2022 — Information Security Management Systems**
- URL: https://www.iso.org/standard/27001
- Defines requirements for an information security management system (ISMS). Relevant to any cloud-deployed LIMS handling patient data, proprietary research results, or regulated pharmaceutical records; drives encryption, access control, and incident management requirements.

---

### W3C & IETF Standards

**RFC 7231 — Hypertext Transfer Protocol (HTTP/1.1): Semantics and Content**
- URL: https://www.rfc-editor.org/rfc/rfc7231
- Defines HTTP semantics used by all LIMS REST APIs; correct use of HTTP methods (GET, POST, PUT, DELETE, PATCH) and status codes is required for interoperable API design.

**RFC 6749 — The OAuth 2.0 Authorization Framework**
- URL: https://www.rfc-editor.org/rfc/rfc6749
- De facto standard for delegated authorisation used by QBench, Scispot, and other modern LIMS APIs. Required for secure third-party integrations with ERP, EHR, and instrument control systems.

**RFC 7519 — JSON Web Token (JWT)**
- URL: https://www.rfc-editor.org/rfc/rfc7519
- Token format used by most modern LIMS REST APIs for stateless authentication. Knowledge of JWT structure and validation is essential for secure API implementation.

**RFC 8288 — Web Linking**
- URL: https://www.rfc-editor.org/rfc/rfc8288
- Defines the `Link` header used in paginated REST API responses; relevant for LIMS APIs returning large result sets (sample lists, test batches).

**HL7 FHIR R4 / R5 — Fast Healthcare Interoperability Resources**
- URL: https://hl7.org/fhir/
- The current standard for healthcare data exchange via RESTful APIs. LIMS connecting to EHR/HIS systems (Epic, Cerner) increasingly need FHIR-compliant DiagnosticReport, Specimen, and Observation resources. 71% of countries already use FHIR in some capacity as of 2025.

**HL7 v2.x — Health Level Seven Version 2 Messaging Standard**
- URL: https://www.hl7.org/implement/standards/product_brief.cfm?product_id=185
- Legacy but still widely deployed messaging standard for lab orders (ORM) and results (ORU). Most clinical LIMS must support HL7 v2 interfaces for hospital information system connectivity alongside or instead of FHIR.

---

### Data Model & API Specifications

**ASTM E1578-18 — Standard Guide for Laboratory Information Management Systems**
- URL: https://www.astm.org/e1578-18.html
- Defines terminology, functional requirements, and system validation guidance for LIMS architecture. Sets the foundational vocabulary for what a LIMS must do; directly cited in many accreditation and procurement frameworks.

**ASTM E1381 / E1394 — Standard Specifications for Low-Level Protocol and High-Level Protocol for Instrument-to-Computer Communication**
- URL: https://www.astm.org/e1394-97.html (E1394); https://www.astm.org/e1381-95.html (E1381)
- Define the ASTM instrument communication protocols (a predecessor/complement to HL7) used by analytical instruments to send results to LIMS. Still widely implemented in clinical and food-testing lab instruments.

**OpenAPI Specification 3.1**
- URL: https://spec.openapis.org/oas/v3.1.0
- The de facto standard for documenting REST APIs. Any new LIMS should publish its API as a machine-readable OpenAPI 3.1 document to enable SDK generation, interactive documentation, and automated testing.

**JSON Schema (Draft 2020-12)**
- URL: https://json-schema.org/specification
- Standard for defining the structure and validation rules of JSON data payloads. Relevant for defining LIMS data models (samples, tests, results) in a portable, interoperable way.

**SNOMED CT — Systematized Nomenclature of Medicine Clinical Terms**
- URL: https://www.snomed.org/
- The preferred terminology standard for clinical observations and specimen types in healthcare-adjacent LIMS deployments. Used to encode test names, specimen types, and organism identifiers in a globally interoperable way.

**LOINC — Logical Observation Identifiers Names and Codes**
- URL: https://loinc.org/
- Standard coding system for laboratory test names and observations; required for FHIR-based lab result exchange and interoperability with electronic health records.

---

### Security & Authentication Standards

**FDA 21 CFR Part 11 — Electronic Records; Electronic Signatures**
- URL: https://www.ecfr.gov/current/title-21/chapter-I/subchapter-A/part-11
- US regulation mandating that electronic records and signatures in pharmaceutical, biotech, and clinical labs have the same legal standing as paper records. Requires LIMS to implement access control, audit trails, time-stamped entries, and validated electronic signature workflows.

**GAMP 5 (Second Edition, 2022) — Good Automated Manufacturing Practice**
- URL: https://ispe.org/publications/guidance-documents/gamp-5-guide-2nd-edition
- ISPE guidance that is the de facto framework for computer system validation (CSV) in pharmaceutical and medical device manufacturing. Defines risk-based validation approaches for LIMS and ELN systems. The 2022 second edition adds guidance on AI/ML, cloud, and open-source software validation.

**ALCOA+ Principles (FDA / WHO / OECD)**
- URL: https://www.fdaguidelines.com/alcoa-plus-data-integrity-principles-explained-for-gmp-glp-and-gcp-environments/
- Framework requiring that all lab data be Attributable, Legible, Contemporaneous, Original, Accurate, Complete, Consistent, Enduring, and Available. LIMS must enforce these principles through audit trails, timestamping, and immutable result storage.

**OECD GLP / GALP — Good (Automated) Laboratory Practice**
- URL: https://www.oecd.org/en/topics/sub-issues/good-laboratory-practice-glp.html
- OECD principles for non-clinical safety testing; GALP (Good Automated Laboratory Practice) extends GLP to computer systems managing regulated data. Relevant to LIMS used in toxicology and environmental testing supporting regulatory submissions.

**CLIA — Clinical Laboratory Improvement Amendments (42 CFR Part 493)**
- URL: https://www.cms.gov/medicare/quality/clinical-laboratory-improvement-amendments
- US federal regulations for clinical laboratory quality; LIMS must support QC data recording, proficiency testing tracking, personnel qualification records, and equipment calibration logs to satisfy CLIA requirements.

**OAuth 2.0 + OpenID Connect 1.0**
- URL: https://openid.net/specs/openid-connect-core-1_0.html
- Standard protocol for federated identity and SSO. LIMS increasingly expected to integrate with enterprise identity providers (Azure AD, Okta) via OIDC for single sign-on and delegated access.

---

### MCP Server Specifications

Model Context Protocol (MCP) is potentially relevant for an AI-native LIMS where an AI assistant needs to query structured LIMS data (samples, results, QC charts) as a context provider for LLM-powered analysis and report generation.

**MCP Specification**
- URL: https://spec.modelcontextprotocol.io/
- Anthropic's open protocol for connecting LLMs to structured data sources and tools. An AI-native LIMS could expose an MCP server that allows LLM assistants to query sample status, retrieve QC trends, and trigger workflow actions via natural language — enabling the "AI co-scientist" use case without tight LLM vendor coupling.

---

## Similar Products — Developer Documentation & APIs

### OpenSpecimen

- **Description:** Open-source biobanking LIMS with 100% REST API coverage; deployed at 70+ research centres globally including Johns Hopkins, Oxford, and Cambridge.
- **API Documentation:** https://openspecimen.atlassian.net/wiki/spaces/OA/pages/1116035/REST+APIs
- **SDKs/Libraries:** Python SDK (OpenSpecimenAPIConnector) — https://openspecimenapiconnectorpy.readthedocs.io/
- **Developer Guide:** https://openspecimen.atlassian.net/wiki/spaces/CAT/pages/81821723/REST+API+Invoker+Plugin
- **Standards:** REST/JSON; token-based authentication via sessions API
- **Authentication:** Session token (POST /rest/ng/sessions with credentials; use returned token in subsequent request headers)

---

### Senaite LIMS

- **Description:** Open-source enterprise LIMS built on the Plone CMS framework; widely deployed in environmental, food, and clinical testing labs.
- **API Documentation:** https://github.com/senaite/senaite.core (integrated JSON API via plone.jsonapi.routes)
- **SDKs/Libraries:** No official SDK; Python/Plone add-on ecosystem; community REST client libraries available
- **Developer Guide:** https://www.senaite.com/ — developer documentation embedded in GitHub wiki and community forums
- **Standards:** REST/JSON; Plone-based content API conventions
- **Authentication:** HTTP Basic Auth or token authentication via Plone PAS framework

---

### QBench

- **Description:** Cloud LIMS for commercial testing labs (food, environmental, cannabis, clinical) with integrated QMS and fast time-to-value.
- **API Documentation:** https://junctionconcepts.zendesk.com/hc/en-us/articles/360030760992-QBench-REST-API-Documentation-Full (also available in-app under Configuration > Developer)
- **SDKs/Libraries:** No official SDK; REST API with standard JSON responses suitable for any HTTP client library
- **Developer Guide:** https://junctionconcepts.zendesk.com/hc/en-us/articles/35386327147277-A-Guide-to-QBench-API-v2
- **Standards:** REST/JSON; OpenAPI-compatible; also supports HL7, ASTM, RS-232, TCP/IP instrument protocols
- **Authentication:** OAuth 2.0 token-based authentication

---

### Scispot

- **Description:** API-first AI-native LIMS/ELN/SDMS platform for biotech startups and scale-ups; includes Scibot natural language assistant and GLUE instrument integration engine.
- **API Documentation:** https://docs.scispot.com/
- **SDKs/Libraries:** No official SDK; REST API with JSON; Labsheets (LIMS), ELN, Manifests, and Sequences endpoints
- **Developer Guide:** https://www.scispot.com/blog/api-first-lims-for-data-science
- **Standards:** REST/JSON; API-first design
- **Authentication:** Personal API tokens (generated under Account > Personal Tokens)

---

### LabKey Server

- **Description:** Open-source research data management platform with strong genomics, sequencing, and clinical research capabilities; includes Biologics LIMS module for biopharma.
- **API Documentation:** https://www.labkey.org/Documentation/wiki-page.view?name=lims
- **SDKs/Libraries:** Java client library, R client (Rlabkey on CRAN), JavaScript client, Python client
- **Developer Guide:** https://www.labkey.com/products-services/lims-software/lims-data-management/
- **Standards:** REST web services; R/Python integration for in-platform analytical pipelines
- **Authentication:** Session-based; supports LDAP, SSO via enterprise identity providers

---

### Thermo Fisher SampleManager LIMS

- **Description:** Enterprise LIMS and SDMS suite with deep instrument integration; widely deployed in pharmaceutical, chemical, and food manufacturing environments.
- **API Documentation:** Astrix white paper: https://astrixinc.com/wp-content/uploads/2022/09/Streamline-Data-Management-with-Simplified-Data-Exchange-using-Thermo-Scientific%E2%84%A2-SampleManager%E2%84%A2-LIMS-REST-API-final.pdf
- **SDKs/Libraries:** No public SDK; REST API via SampleManager WCF service (port 56105); Chromeleon CDS integration library
- **Developer Guide:** https://www.thermofisher.com/blog/connectedlab/introducing-the-latest-advancements-in-samplemanager-lims-software/
- **Standards:** REST/JSON (since v11.2); SAP Remote Function Calls (RFC) for ERP integration; ASTM, RS-232, TCP/IP for instruments
- **Authentication:** Token-based authentication with hardened session handling (v21.x+)

---

### LabVantage

- **Description:** All-in-one LIMS + ELN + LES + SDMS + analytics platform; targets regulated pharmaceutical and manufacturing environments.
- **API Documentation:** Not publicly available; documentation provided to licensed customers only
- **SDKs/Libraries:** Proprietary REST API; third-party integration connectors for SAP, Oracle, Tableau, PowerBI
- **Developer Guide:** https://www.labvantage.com/informatics/
- **Standards:** REST/JSON; SAP/Oracle ERP connectors; ASTM instrument protocols
- **Authentication:** Session-based with enterprise SSO support

---

### STARLIMS (Abbott Informatics)

- **Description:** Enterprise LIMS + ELN + SDMS platform with deep regulatory compliance features; widely deployed in pharmaceutical, clinical, and public health labs.
- **API Documentation:** Not publicly available; provided to licensed customers
- **SDKs/Libraries:** Proprietary integration layer; HL7 v2/FHIR connectors; ASTM instrument interfaces
- **Developer Guide:** https://www.starlims.com/resources/starlims-lab-informatics-software/
- **Standards:** HL7 v2, FHIR, REST/JSON; 21 CFR Part 11, ISO 15189, ISO 17025 compliance
- **Authentication:** LDAP/AD integration; enterprise SSO; role-based access control

---

## Notes

- **FHIR vs. HL7 v2**: Most clinical LIMS still rely on HL7 v2 for instrument and HIS connectivity because most analysers speak ASTM or HL7 v2 rather than FHIR. FHIR R4/R5 is the strategic direction but lab-specific FHIR profiles (DiagnosticReport, Specimen) are still maturing. New LIMS implementations should support both.
- **ASTM instrument protocols**: Despite REST APIs being standard for system-to-system integration, most physical analytical instruments (balances, pH meters, spectrophotometers, analysers) still communicate via ASTM E1394 / RS-232 / TCP/IP at the hardware level. A LIMS must support these for instrument interfacing, even if its system APIs are REST-based.
- **MCP opportunity**: No current LIMS exposes an MCP server, representing an early-mover opportunity for an AI-native LIMS to become a data context provider for LLM-powered lab assistants across any host application.
- **GAMP 5 AI validation**: The 2022 GAMP 5 second edition and the forthcoming EU Annex 22 (AI in regulated environments, final expected 2026) will require AI-native LIMS to document AI model validation, explainability, and change control — design decisions should account for this regulatory trajectory from the start.

# Lab Information Management System (LIMS)

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An open-source, AI-native LIMS for sample tracking, test management, result reporting, and instrument integration — without enterprise pricing or vendor lock-in.

This project aims to deliver a modern Laboratory Information Management System that combines the regulatory rigour of enterprise incumbents with the deployment speed of cloud-native SaaS and the cost profile of open source. It targets pharmaceutical QC, clinical, biotech, and commercial testing labs that need ISO 17025 / 21 CFR Part 11 readiness without six-figure implementation projects.

---

## Why LIMS?

- Enterprise LIMS (LabWare, LabVantage, STARLIMS, Thermo SampleManager) cost $550–$2,500/month plus $10,000–$60,000 implementation, with 6–18 month deployments and customisation locked behind specialist consultants.
- The top four vendors hold roughly 80% market share, and most enterprise tools have only nascent AI features (e.g. LabVantage 8.9 voice commands) bolted onto legacy architectures.
- Existing open-source options (Senaite, OpenSpecimen, OpenELIS, LabKey) carry no licence fee but ship with no AI capabilities and significant Python/Plone or domain-specific limitations.
- AI-native commercial entrants (Sapio, Scispot) are proprietary, often cloud-locked (e.g. AWS Bedrock), and priced for biotech enterprise rather than SMB or academic labs.
- There is currently no open-source LIMS that combines full API parity, AI-assisted workflows, and out-of-the-box ISO 17025 / 21 CFR Part 11 compliance scaffolding.

---

## Key Features

### Sample & Workflow Management

- Sample login, chain-of-custody tracking, and barcode/QR label printing
- Test request management and result entry with configurable calculation rules
- Batch and plate management for high-throughput labs
- Inventory and reagent/consumable tracking

### Compliance & Quality

- Role-based access control with full audit trail and electronic signatures (21 CFR Part 11 / ISO 17025 ready)
- Configurable Certificate of Analysis and compliance report generation
- Out-of-specification (OOS) and out-of-trend (OOT) alerting with Westgard rule enforcement
- Stability study and environmental monitoring workflows

### Instrument & System Integration

- Instrument file import (CSV, XML, ASTM) with configurable parsers
- ASTM, RS-232, TCP/IP, and HL7 / FHIR protocol support
- RESTful API with 100% functional parity to the UI
- Connectors for ERP, EHR, and clinical research platforms

### AI-Assisted Lab Operations

- Natural language query interface for non-technical lab staff
- AI-generated compliance narratives from structured LIMS data
- Predictive OOS/OOT alerts trained on historical QC data
- Anomaly detection in QC trend charts beyond static Westgard rules

### Extended Capabilities

- Electronic Lab Notebook (ELN) module integrated with sample records
- Predictive instrument scheduling and sample routing optimisation
- Multi-site federation with centralised governance and per-site data segregation
- Cross-lab benchmarking dashboard using anonymised aggregate quality metrics

---

## AI-Native Advantage

AI is embedded in the core data model rather than retrofitted as a chatbot layer. LLM-assisted review of instrument outputs replaces manual transcription, natural-language compliance reporting generates audit-ready ISO 17025 / 21 CFR Part 11 narratives from structured LIMS data, and ML models trained on historical measurement and QC data flag likely OOS results before failures occur. Intelligent sample routing balances instrument queues and technician workloads to reduce turnaround time — capabilities that are absent or manual in every current open-source LIMS and only nascent in commercial tools.

---

## Tech Stack & Deployment

The system is designed for self-hosted and cloud deployment with an API-first architecture: every UI action is available via REST, following the model proven by OpenSpecimen. Instrument connectivity uses established protocols (ASTM, RS-232, TCP/IP) alongside HL7 v2 and FHIR for clinical settings. Compliance scaffolding aligns with ISO/IEC 17025:2017, 21 CFR Part 11, GLP/GALP, ISO 9001:2015, ASTM E1578, and CLIA. Client SDKs are planned for Python, JavaScript, and R to support both lab operations and downstream analytics pipelines.

---

## Market Context

The global LIMS market was approximately USD 2.50 billion in 2024 and is projected to reach USD 3.67 billion by 2029 at ~10% CAGR, within a broader laboratory software market growing from USD 6.31 billion (2025) to USD 10.12 billion by 2030 (Business Research Insights; MarketsandMarkets). Buyers include pharma and biotech lab directors, hospital and clinical lab managers, food/environmental/cannabis testing labs, and academic research operations. Enterprise pricing of $550–$2,500/month plus large implementation fees leaves a clear gap for an open-source AI-native alternative at the SMB and mid-market tier.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.

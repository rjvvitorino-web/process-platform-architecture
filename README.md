# Process Platform Architecture
### Enterprise Process Management · Microsoft Power Platform · Government Digital Transformation

![Architecture](https://img.shields.io/badge/Architecture-6--Layer%20Platform-0ea5e9?style=flat-square)
![Stack](https://img.shields.io/badge/Stack-Power%20Platform%20%2B%20Fabric-7c3aed?style=flat-square)
![Context](https://img.shields.io/badge/Context-Government%20Digital%20Transformation-065f46?style=flat-square)
![Status](https://img.shields.io/badge/Status-Architecture%20%26%20Design-f59e0b?style=flat-square)

---

## Overview

This repository documents the architecture of a configurable process management platform designed for a Portuguese central government agency responsible for financial and budgetary oversight across the public sector.

The platform consolidates multiple fragmented operational processes — previously managed through disconnected tools, spreadsheets, and ad hoc applications — into a single, reusable infrastructure that scales to new processes through configuration rather than development.

The architecture was designed to solve three problems simultaneously: **eliminate recurring licensing costs for external users**, **establish a unified data layer** that enables advanced analytics and AI, and **create a foundation** that does not require rebuilding when new processes are added.

---

## The Problem

The agency coordinates financial and budgetary processes involving **117+ external entities and 200+ users** across the Portuguese public sector. At the time of design, these processes presented four compounding limitations:

- **Licensing cost at scale.** The existing solution required individual Power Apps licences for every external user — a significant and growing recurring cost with no architectural ceiling.
- **No relational data layer.** Data lived in SharePoint lists with no relations, no native business rules, no row-level security, and no structured audit trail.
- **Information silos.** Each process operated independently, preventing cross-process analysis or any unified view of the agency's operational output.
- **No interoperability.** Critical external systems (budgetary execution, procurement, reporting) had no integration pathway, forcing manual data transfers between platforms.

---

## Architecture

The platform is structured in six functional layers, each with clearly defined responsibilities and technology choices. The guiding principle throughout is **separation of concerns**: every layer is autonomous and communicates with adjacent layers through well-defined interfaces.

→ **[View the interactive architecture diagram](https://rjvvitorino-web.github.io/process-platform-architecture/architecture/platform-diagram.html)**

### Layer 1 — External Staging (SharePoint Online)
For processes involving users from external entities, a dedicated SharePoint Online site serves as the entry point. Forms are built using the native *Customize with Power Apps* capability directly on SharePoint lists — a specific approach that allows external guest users (provisioned via Entra ID B2B) to submit and track requests **without requiring individual Power Apps licences**. This is the platform's most significant cost decision. See [Architecture Notes](docs/architecture-notes.md#licensing-architecture) for the full rationale.

### Layer 2 — Internal Entry (Power Apps)
For internal-only processes, staff access the platform directly through Power Apps — Canvas for high-customisation interfaces, Model-Driven for standard workflow management. Internal licences are covered under existing Microsoft 365 agreements, with no additional cost.

### Layer 3 — Transactional Core (Dataverse)
Microsoft Dataverse is the single source of truth for all processes across the platform. It provides a relational data model with native business rules, row-level security, structured audit logging, and full versioning — none of which were available in the prior SharePoint-list approach. Power Automate orchestrates bidirectional synchronisation between the SharePoint staging layer and Dataverse, keeping external and internal views consistent in under 60 seconds.

The data model is deliberately **generic and configurable**: adding a new process type means creating a configuration record — defining SLAs, approval flows, mandatory fields, and whether the process involves external users — not modifying schema or building new applications.

### Layer 4 — Interoperability (Azure API Management + OData)
A REST/OData API layer, managed through Azure API Management, exposes the platform to external systems. This ensures the platform is not confined to the Microsoft ecosystem: budgetary execution systems, procurement platforms, and reporting tools can integrate via documented, versioned APIs with OAuth2 authentication and rate limiting. Webhooks enable event-driven notifications to subscribed external systems.

### Layer 5 — Analytics (Microsoft Fabric / OneLake)
Analytical workloads are deliberately separated from the transactional layer. Dataverse synchronises natively to Microsoft Fabric via Dataverse Link, populating a OneLake lakehouse in Delta/Parquet format. This allows heavy reporting queries, cross-process analysis, and historical data joins to run without impacting operational performance. External data sources (budgetary execution, procurement records) are also ingested into OneLake, enabling cross-system analysis that was previously impossible.

Power BI operates in Direct Lake mode against OneLake, providing dashboards for operational monitoring, institutional reporting, and cross-agency analysis.

### Layer 6 — Intelligence (RAG / Copilots)
The platform's AI layer is built on three complementary grounding sources, each serving a distinct query type:

| Source | Type | Used for |
|---|---|---|
| Dataverse | Structured, real-time | Process status, assignments, decisions |
| OneLake | Documents + history | Precedent search, historical analysis |
| **BSO Knowledge Graph** | **Semantic** | **Strategic alignment, objective mapping** |

The third source — the **Balanced Scorecard Ontology (BSO) Knowledge Graph** — is what makes this AI layer architecturally distinctive, and is discussed in detail below.

Copilots are differentiated by user profile: external requesters see only their own entity's processes; internal technicians have access to precedent search and workload tools; management copilots connect operational data to strategic objectives.

---

## The BSO Connection — Research Meets Platform

> *This is the thread that connects this repository to the [Research Track](https://github.com/rjvvitorino-web) of this portfolio.*

The BSO Knowledge Graph embedded in Layer 6 is not a generic AI feature — it is the direct operational application of the **Balanced Scorecard Ontology** methodology implemented as the centrepiece of the MSc thesis in Public Administration Digitalisation (ISCTE-IUL, 2024).

That research project implemented a BSO at the same agency using semantic web technologies: an LLM-to-ontology transformation pipeline that ingested strategic plans, quarterly activity reports, and performance data, structured them as a knowledge graph in GraphDB, and made them queryable for strategic analysis. The results — 96% completion rate on strategic indicators, 72% reduction in strategic analysis time — demonstrated the methodology's operational value.

This platform architecture closes the loop. The same BSO Knowledge Graph that structures strategic objectives and performance indicators becomes a **grounding source for the operational AI layer**: when a technician or manager asks a question that touches on strategic alignment, the copilot can reason across both the operational process data (Dataverse) and the strategic knowledge structure (BSO graph) in a single response.

This is the concrete realisation of a research thesis in production infrastructure — connecting academic output to platform design in a way that is rare in either the public sector or the Power Platform ecosystem.

---

## Key Design Decisions

Four decisions shaped everything else about this architecture:

**1. Licensing elimination as a first-class architectural concern.** The choice to use SharePoint Online as an external staging layer was not a workaround — it was the primary driver of the architecture. Eliminating Power Apps licensing for 200+ external users is a recurring, scalable saving that grows with every new process added to the platform.

**2. Configuration over development.** Every new process is a configuration record, not a development project. The investment in the platform's generic data model and configurable workflow engine means the marginal cost of each additional process is substantially lower than the first.

**3. Transactional and analytical separation.** Running analytical workloads against the operational database is a common architectural mistake that degrades both. Separating Dataverse (transactional, consistent, secure) from Fabric/OneLake (analytical, scalable, flexible) allows each to be optimised independently — including pausing analytical capacity outside working hours.

**4. AI-ready by design.** The two-database architecture, combined with indexed document storage in OneLake and the BSO semantic graph, creates the grounding infrastructure for contextually rich AI without retrofitting. The RAG architecture is a consequence of the data layer decisions, not an addition on top of them.

---

## Outcomes

| Dimension | Target |
|---|---|
| External users served | 200+ concurrent, zero incremental licence cost |
| Sync latency (SharePoint → Dataverse) | < 60 seconds |
| API response time (CRUD) | < 2 seconds |
| Availability (business hours) | 99.5% |
| Implementation timeline | 22–32 weeks (three processes, full platform) |
| Scalability | Each additional process = configuration, not development |

---

## Technology Stack

`SharePoint Online` · `Power Apps (Canvas + Model-Driven)` · `Microsoft Dataverse` · `Power Automate` · `Azure API Management` · `Microsoft Fabric` · `OneLake` · `Power BI (Direct Lake)` · `Copilot Studio` · `Azure OpenAI Service` · `Azure AI Search` · `Entra ID B2B` · `GraphDB` · `OData v4` · `REST / OpenAPI 3.0`

---

## Repository Contents

```
/
├── README.md                        ← this document
├── architecture/
│   └── platform-diagram.html        ← interactive architecture diagram
└── docs/
    └── architecture-notes.md        ← deeper technical rationale
```

Full design documentation, data model specifications, and implementation materials are maintained privately. Available on request.

---

## Contact

If this architecture is relevant to work you're doing — whether in government digital transformation, Power Platform implementation, or AI/semantic web integration — I'd welcome the conversation.

**[Connect on LinkedIn →](https://www.linkedin.com/in/rui-jv-vitorino)**


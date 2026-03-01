# Architecture Notes
## Process Platform — Technical Design Rationale

*This document provides deeper technical context on selected design decisions. It is intended for readers with a technical background in enterprise architecture, Microsoft Power Platform, or government systems design.*

---

## Table of Contents

1. [Licensing Architecture](#1-licensing-architecture)
2. [The Configurable Platform Model](#2-the-configurable-platform-model)
3. [Transactional vs Analytical Separation](#3-transactional-vs-analytical-separation)
4. [The RAG Grounding Strategy](#4-the-rag-grounding-strategy)
5. [FinOps Principles](#5-finops-principles)
6. [Multi-Environment Strategy](#6-multi-environment-strategy)
7. [API Design Philosophy](#7-api-design-philosophy)

---

## 1. Licensing Architecture

### The Core Problem

Power Platform licensing for external users is one of the most frequently misunderstood cost centres in Microsoft 365 deployments. The default assumption — that any Power Apps interface requires a per-user Power Apps licence — leads organisations to either accept significant recurring costs or avoid Power Platform for external-facing scenarios entirely.

This platform was designed around a specific and well-documented exception in Microsoft's licensing model.

### The SharePoint Staging Approach

When a Power Apps form is created using the **Customize with Power Apps** capability directly on a SharePoint list (as opposed to a standalone Canvas App), and that form uses only Standard connectors (SharePoint, Office 365 Users), external guest users provisioned via **Entra ID B2B** can interact with the form using only their Microsoft 365 guest access — **no individual Power Apps licence required**.

This is not a grey area or a workaround. It is documented in the Power Platform Licensing Guide and reflects the architectural intent of the SharePoint-embedded form model.

### Conditions for Compliance

Four conditions must hold simultaneously for this model to remain compliant:

| Condition | Implementation |
|---|---|
| Form creation method | *Customize with Power Apps* on the SharePoint list — not a standalone Canvas App |
| Connector scope | Standard connectors only in the external-facing form (SharePoint, Office 365 Users) |
| User provisioning | Entra ID B2B Guest Users — not anonymous or username/password access |
| Tenant configuration | External sharing enabled on the SharePoint site and tenant; *Allow Microsoft Power Platform internal consent plans* policy active |

Premium connectors, direct Dataverse access, or standalone Canvas Apps would each independently invalidate this model. The architecture addresses this by keeping the external-facing layer strictly within SharePoint, with all integration to Dataverse handled server-side via Power Automate (which runs under service account licences, not user licences).

### The Financial Case

The saving is both significant and compounding. The base case involved 200+ users across 117 external entities. Under a standard per-user Power Apps licence model, this would represent a fixed, recurring annual cost that scales with user growth and with each new external-facing process added. Under the SharePoint staging model, the marginal licensing cost of each additional external user is zero, and the marginal cost of each new external process is similarly zero.

This architectural decision effectively removes licensing as a constraint on the platform's growth.

---

## 2. The Configurable Platform Model

### The Anti-Pattern Being Avoided

The standard approach to building operational process applications in the public sector — and in many enterprises — is to build one application per process. Each application has its own data model, its own UI, its own workflows, and its own maintenance burden. The result is a proliferation of disconnected tools that cannot share data, cannot be analysed together, and each require independent development effort when they need to change.

### The Configuration Record Pattern

This platform inverts that pattern. A single **Process Type** configuration record defines everything variable about a given process:

- Whether it involves external users (determining which entry path applies)
- The SLA for each lifecycle stage
- The approval flow structure (single approver, sequential, parallel)
- Mandatory fields and document types
- Notification rules and escalation logic

The tables, relationships, security roles, and application interfaces are built once. Adding a new process to the platform means creating a configuration record, not writing new code or designing a new data model.

The practical implication: the third and fourth process on the platform cost substantially less to implement than the first, because the platform investment has already been made. This is the architectural equivalent of a product line rather than a collection of bespoke projects.

### Generic Data Model Design

The data model achieves genericity through a deliberate structural choice: **core tables are shared across all processes, while configuration tables define process-specific behaviour**. Process records reference their type through a lookup, and business rules, security filters, and workflows branch based on that lookup rather than being duplicated per process.

This creates a single analytical surface across all process types — a capability that was impossible under the previous fragmented architecture and that directly enables the cross-process dashboards and AI grounding described in later sections.

---

## 3. Transactional vs Analytical Separation

### Why They Cannot Share an Infrastructure

Dataverse is optimised for transactional workloads: consistent reads, low-latency writes, row-level security enforcement, and audit logging on every operation. These characteristics make it an excellent operational database. They also make it an expensive and performance-constrained analytical database.

Analytical queries — historical aggregations, cross-process joins, trend calculations, external data correlations — have different characteristics. They are read-heavy, they operate on large data volumes, they benefit from columnar storage formats, and they are tolerant of slightly stale data. Running them against Dataverse degrades operational performance and incurs unnecessary cost per API call.

### The Separation Architecture

**Dataverse Link to Microsoft Fabric** provides native, no-code synchronisation from Dataverse tables to a OneLake lakehouse in Delta/Parquet format. This is not an ETL pipeline requiring maintenance — it is a platform-native capability that keeps OneLake current without manual intervention.

Once in OneLake, data can be joined with external sources (budgetary execution systems, procurement records, activity reports) that cannot be ingested directly into Dataverse. The combined dataset supports analysis that crosses the boundaries of what the operational system can see.

Power BI operates in **Direct Lake** mode against OneLake — a storage mode introduced with Microsoft Fabric that combines the query performance of import mode with the freshness guarantees of DirectQuery, without the per-query Dataverse API cost.

### FinOps Implications

The separation has an important cost management benefit: Fabric capacity can be **scaled independently** of Dataverse and can be **paused outside working hours**. Analytical workloads are not continuous — reports are generated during the working day, and overnight processing (if any) can be scheduled. This allows the Fabric layer to be right-sized for actual demand rather than peak capacity.

---

## 4. The RAG Grounding Strategy

### Why Grounding Architecture Matters

A language model without grounding knows only what it was trained on. For an operational copilot to be useful — rather than plausible-sounding but unreliable — it must retrieve current, contextually relevant information at query time and reason over it. The architecture of the retrieval layer determines the quality, scope, and trustworthiness of every response.

This platform's AI layer is designed around three complementary grounding sources, each of which answers a different class of question:

### Grounding Source 1: Dataverse (Structured, Real-Time)

Dataverse is queried directly for precise, current operational facts. *What is the status of process #427? Which requests are pending assignment? What is the current SLA position on open items in Division X?*

This requires the copilot to understand the data model and to generate appropriate OData queries — a structured retrieval task rather than semantic search. The advantage is precision and freshness: the answer reflects the state of the system at query time.

### Grounding Source 2: OneLake Documents (Semantic, Historical)

Dispatches, opinions, technical assessments, and correspondence are stored as files in OneLake, processed with chunking and embeddings, and indexed for semantic search via Azure AI Search. This enables precedent retrieval: *Have similar requests been handled before? What was the technical assessment in comparable cases? What is the historical processing time for this type of request?*

This is conventional RAG — embedding similarity search over a document corpus — but the corpus is rich because it accumulates with every process the platform handles.

### Grounding Source 3: BSO Knowledge Graph (Semantic, Strategic)

This is the architecturally distinctive grounding source, and the one that most directly connects this platform to the research track.

The **Balanced Scorecard Ontology (BSO) Knowledge Graph** is a structured semantic representation of the agency's strategic objectives, performance indicators, targets, and their relationships. It was constructed through an LLM-to-ontology transformation pipeline that ingests strategic plans, quarterly reporting documents, and performance data, and outputs a queryable knowledge graph in GraphDB.

The research project that produced this graph (MSc thesis, ISCTE-IUL, 2024) demonstrated 96% completion rate on strategic indicators and a 72% reduction in strategic analysis time at the same agency. The graph encodes not just the indicators and their values, but the **semantic relationships between them** — which objectives support which strategic themes, which indicators are leading versus lagging, which organisational units are responsible for which targets.

When a manager asks the platform copilot a question that involves strategic alignment — *How does current processing volume relate to our service delivery objectives? Which process backlogs are creating risk against strategic commitments?* — the copilot can reason across Dataverse (operational state), OneLake (historical pattern), and the BSO graph (strategic context) in a single response.

This is the concrete operational realisation of the thesis research. The knowledge graph moves from a standalone analytical tool to an embedded component of the agency's intelligence infrastructure.

### Copilot Differentiation by Profile

The three grounding sources are not uniformly exposed to all users. The copilot layer is differentiated by profile precisely because different users need different information and have different data access rights:

| Profile | Grounding sources | Scope |
|---|---|---|
| External requester | OneLake (own documents only), Dataverse (own processes only) | Filtered to own entity |
| Internal technician | Dataverse (assigned processes), OneLake (full document corpus) | Filtered to assigned work |
| Management | All three sources | Cross-process, strategic view |

Row-level security in Dataverse propagates through the copilot layer — the AI cannot surface information the user is not authorised to see.

---

## 5. FinOps Principles

### The Context: Public Sector Cloud Governance

Cloud cost management in the public sector carries a specific accountability dimension. Unlike private sector organisations where budget overruns affect company performance, public sector cloud spend is subject to audit, parliamentary oversight, and public accountability. The architecture was designed with this in mind — not as a compliance exercise, but as a genuine design constraint.

### Architectural Decisions with FinOps Impact

Several platform design choices directly reduce or control cost, independent of any monitoring tooling:

**SharePoint staging (zero licensing cost).** The largest single cost optimisation in the architecture. See [Section 1](#1-licensing-architecture).

**Transactional/analytical separation (independent scaling).** Dataverse is billed per record storage and API calls. Fabric is billed on capacity. Separating them allows each to be sized for its actual workload and allows Fabric capacity to be paused when not in use.

**Archival to OneLake.** Completed processes should be archived from Dataverse (higher per-GB cost, optimised for transactional access) to OneLake (lower per-GB cost, optimised for analytical access) on a defined schedule. Historical data remains queryable via the analytical layer; the operational layer stays lean.

**Non-production environment right-sizing.** Development and test environments have substantially lower usage than production. Power Platform environments, Fabric workspaces, and API Management tiers for non-production can be reduced or paused outside working hours without impacting delivery.

**Progressive AI rollout.** Copilot Studio and Azure OpenAI consumption scales with query volume. Starting with the lowest-complexity copilot profile (external requester) and expanding to higher-complexity profiles (technician, management) after validating actual usage and cost allows the AI investment to be calibrated against demonstrated value.

### Monitoring Architecture

A dedicated Power BI cost dashboard, sourcing from Azure Cost Management APIs ingested into OneLake, provides unified visibility across all platform components. Cost is tagged by environment (Dev/Test/Prod) and by process/division, enabling both oversight and internal chargeback where appropriate.

---

## 6. Multi-Environment Strategy

Three isolated environments — Development, Test, Production — with unidirectional promotion via managed solutions.

**Development** is the only environment where schema changes, new flow configurations, or application modifications are made. It operates with synthetic data and is accessible only to the development team.

**Test** receives exported managed solutions from Development. Functional and acceptance testing, including full end-to-end tests with external-facing SharePoint staging, occurs here using a dedicated test SharePoint site distinct from production. No changes are made directly in this environment.

**Production** receives promoted solutions from Test only. Direct modifications to production are prohibited. Every solution carries semantic versioning (major.minor.patch) with a documented changelog, enabling rollback if a promotion introduces a regression.

This discipline extends to all platform components: Power Automate flows are part of the solution and promoted with it; connection references and environment variables are configured per environment rather than hardcoded; Fabric workspaces are isolated per environment; APIM exposes environment-specific endpoints.

---

## 7. API Design Philosophy

The API layer is designed for longevity — interfaces that external systems can depend on without being coupled to the platform's internal evolution.

**Single gateway.** Azure API Management provides one entry point for all external consumers. Authentication (OAuth2, API Key), rate limiting, logging, and payload transformation are handled at the gateway layer, not in individual connectors or flows. This means the underlying platform can evolve without changing the external API contract.

**OData as the native surface.** Dataverse's native OData v4 API is exposed through APIM rather than duplicated with a custom REST layer. This gives external systems full query capability ($filter, $expand, $orderby, batch operations) with the security model of Dataverse (including row-level security) enforced at the data layer.

**OpenAPI 3.0 documentation.** All endpoints are documented in OpenAPI 3.0 format, versioned alongside the platform solutions. This enables external teams to integrate without requiring access to the platform's internal documentation, and supports future automation of client SDK generation.

**Webhooks over polling.** For event-driven scenarios — process state changes, SLA breaches, approval completions — the platform exposes webhooks rather than requiring external systems to poll for updates. This reduces API call volume, improves real-time responsiveness, and simplifies integration logic in consuming systems.

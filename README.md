# ArchSys

> **Status: early planning – no code yet.** This README captures the current vision and the decisions made so far. Open questions will be resolved and recorded as ADRs.

**An open, self-hostable architecture platform that connects strategy and technology in one continuous model.** Pick a strategic goal and instantly see everything that matters for it – technical data captured automatically from the bottom up, business context maintained by people from the top down, with Git-style versioning of the current and target architecture.

## Why

- **Most enterprise architecture tools stop at the application layer.** Servers, software and containers usually live in separate tools and are only connected through integrations.
- **Architecture data goes stale.** It is maintained by hand, and that maintenance depends on a handful of people.
- **Interfaces and data flows are poorly represented** in many tools and rarely kept up to date.
- **Dashboards look the same for everyone,** regardless of what the organization is actually trying to achieve.

## What makes ArchSys different

| # | Principle | Meaning |
|---|---|---|
| 0 | **Strategy first** | The UI is built around strategic goals (“lenses”), not around an inventory. |
| 1 | **One model from strategy to hardware** | Click from a business capability down to the server it runs on – and see which capabilities are affected when a server fails. |
| 2 | **Automated at the bottom, curated at the top** | Technical data comes from CI pipelines, imports or agents; business data is maintained by people. The value lies in connecting the two. |
| 3 | **Git-style versioning** | Current and target architecture as branches, changes as proposals with approval, full history. |
| 4 | **Dedicated modules instead of one generic object type** | Each element type gets its own menu, fields, validation and UI. |
| 5 | **Open and self-hostable** | No dependency on a SaaS vendor; your data stays with you. |

Principles 0–3 are the core; 4 and 5 shape how it is built.

## Strategic lenses

ArchSys is organized around strategic goals. Selecting a lens shows everything relevant to that goal – KPIs, views, rules and missing data. All lenses work on the same underlying model.

| Lens | Goal | Example KPIs | Behaviour in the app |
|---|---|---|---|
| **Cloud-first** | Move new systems to the cloud, reduce on-premise | Cloud vs. on-premise share, migration progress | Warn on new on-premise servers, require justification |
| **Consolidation / cost** | Fewer, shared systems | Applications per business capability, duplicates, cost per capability | Suggest an existing application with the same capability when creating a new one |
| **Modernization** | Reduce technical debt | Share of technologies past end of support, system age | Highlight outdated technology, require a retirement plan |
| **Security & compliance** | Manage risk and obligations | Critical capabilities and their applications, protection needs, systems without an owner | Mandatory fields (protection need, data category), security approval |
| **Resilience** | Withstand outages | Single points of failure, critical dependencies | Impact analysis front and centre, warn on missing redundancy |
| **Buy before build / SaaS-first** | Prefer standard software | Share of custom-built software | Require justification for new custom builds |
| **Agility / innovation** | Change quickly | Decoupled interfaces, release frequency (from CI) | Focus on interfaces and dependencies |
| **Sustainability** | Reduce resource use and emissions | Estimated CO₂ emissions per hardware, application and capability; utilization | Flag underutilized systems |

**How lenses work**

- **Combinable.** Organizations usually pursue several goals at once, so multiple lenses can be active, optionally weighted.
- **A lens is a package** of required data, computed KPIs, views and activated rules. New lenses can be added without changing the core (plug-in approach).
- **Missing data is made visible.** Each lens shows what it is missing (e.g. “power draw is unknown for 40 % of servers”), so strategy drives data quality instead of data maintenance being an end in itself.
- **Templates plus customization.** ArchSys ships with predefined lenses whose rules can be adapted.
- **Per-application assessment** using an established model such as Gartner's TIME (Tolerate, Invest, Migrate, Eliminate), which the lenses can evaluate.

*Sustainability data:* power draw or device type, utilization and the carbon intensity of the local power grid. Results are estimates. Candidate methodologies to evaluate: the Software Carbon Intensity specification (Green Software Foundation) and Cloud Carbon Footprint.

## Foundation: a subset of ArchiMate

- ArchSys adopts the **layers** (business, application, technology) and a **small selection** of element and relationship types (e.g. *realizes*, *serves*, *runs on*, *composed of*).
- It is **not** a full ArchiMate editor – Archi already covers that.
- Terminology stays ArchiMate-compatible so that import/export via the Open Group exchange format is possible later.

## Modules

| Module | Contents |
|---|---|
| **Strategy** | Business capabilities, goals |
| **Applications** | Applications with owner and lifecycle; **services/microservices are components of an application**, not a separate module |
| **Integration** | Interfaces and data flows |
| **Technology** | Software products and versions, servers, hardware, deployments |
| **Organization** | Teams and owners |

Cross-cutting: search, relationship graph, top-down drill-down and bottom-up impact analysis.

**Drill-down chain:** capability → application → service → deployment (container/server)

## Data model

A hybrid of a shared core and dedicated types:

- **Shared core** for every element: ID, name, owner, lifecycle, history, relationships, permissions, search, completeness.
- **Dedicated types** per module with their own fields, validation and UI.
- Implementation idea: a shared element table plus one table per type (JPA inheritance strategy `JOINED`).

Cross-cutting features are built once, while modules stay clearly separated.

## Views

- **The model is the source of truth.** Views (landscape maps, dependency graphs, drill-downs) are generated from the data.
- Free-form drawing (e.g. an embedded draw.io) may come later, strictly for sketches that do not flow back into the model.

## Automated interface discovery

1. **Provider side from specifications.** A CI step (e.g. a GitHub Action) reads OpenAPI (REST) or AsyncAPI (events) and reports: *“Application X provides interface Y in version Z.”*
2. **Consumer side from a manifest.** Each repository contains a small file with an **application ID** and the interfaces it consumes. The ID links code to the model.
3. **Runtime reconciliation (later stage).** Runtime data (e.g. OpenTelemetry traces, API gateway logs) shows actual calls, revealing interfaces that are *declared but never used* or *used but never declared*.

**Aggregation to the application level**

- Calls *within* an application are internal and only visible when zooming in.
- Calls *across* application boundaries are automatically aggregated into interfaces at the enterprise architecture level.

## Versioning

- Current and target architecture as branches of the model: plan, compare, merge.
- Changes as proposals with approval, similar to pull requests.
- Full history and rollback – also as protection against faulty or destructive edits.

## Users

- **Architects and management:** capabilities, landscape maps, target architecture, analytics
- **Application owners and teams:** maintaining their applications, approving changes
- **IT operations and development:** technology layer, deployments, interfaces from CI

## Technical considerations

- An idempotent ingestion API that stamps every report with commit and timestamp
- A queue between ingestion and processing, since many pipelines report concurrently
- Provenance for every piece of information (CI, manual, runtime)
- Detection of interface changes that could break consumers
- Very different data volumes per layer: a few hundred curated elements at the top, thousands of automatically captured ones at the bottom
- Runs locally with Docker Compose first; a deliberate migration to the cloud later

## Open decisions (ADR candidates)

- **Shared services:** Does a service used by several applications belong to one of them or to a dedicated “platform” application?
- **Applications without code:** Purchased software and SaaS have no repositories – modelled without a service layer and maintained manually.
- **Scope of the metamodel:** Which ArchiMate elements and relationships go into the first version?
- **Graph storage:** Postgres with relationship tables or a graph database?
- **Branching model:** How are branches, conflicts and merges implemented?
- **draw.io:** Embed or not; check the license.
- **Rule engine for lenses:** Simple custom rules or an existing library?
- **Sustainability model:** Which methodology and data sources for CO₂ estimates?

## Non-goals

- SaaS management
- A survey suite
- Cost management
- A full ArchiMate editor

## MVP scope

1. A metamodel with a few types per layer: capability, application, service, server
2. Relationships across layers
3. Drill-down from capability to server; bottom-up impact analysis
4. A GitHub Action that reads `openapi.yaml` and the manifest and sends them to the server
5. Per-application view of provided and consumed interfaces; dependency graph
6. The first three lenses: cloud-first, consolidation, modernization

All development uses fictional sample data. The first real test data will come from my own repositories.

## Roadmap

- [ ] Finalize the MVP metamodel
- [ ] First ADRs (graph storage, data model, branching)
- [ ] Backend skeleton and CI pipeline
- [ ] MVP features 1–6
- [ ] Runtime reconciliation of interfaces
- [ ] Additional lenses (security & compliance, resilience, sustainability)
- [ ] Cloud deployment and migration

## License

To be decided.

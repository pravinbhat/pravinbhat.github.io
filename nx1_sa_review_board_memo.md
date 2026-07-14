---
layout: default
title: "NexusOne (NX1) Regulatory Reporting Pipeline Migration"
description: "Architecture Design Memo"
permalink: /nx1-sa-review/
date: 2026-07-13
author: Pravin Bhat
---

# NexusOne (NX1) Regulatory Reporting Pipeline Migration — Architecture Design Memo

**Classification:** Confidential

**Prepared by:** Principal Solutions Architect, NexusOne

**Version:** 1.1

**Client:** Tier-1 US Retail & Investment Bank, Charlotte NC

**Program:** Basel III & CCAR Pipeline Modernization — 18-Month

**Audience:** SA Review Board & InfoSec Guardians

**Distribution:** Restricted — Program Principals Only

---

## Section 1 — Executive Summary

This memo presents the proposed NexusOne (NX1) target architecture for the Bank's Basel III and CCAR regulatory reporting pipeline migration.

The Bank's current end-of-quarter regulatory reporting cycle spans **5 weeks** from quarter-end to final sign-off. Three compounding bottlenecks drive this latency: human-in-the-loop validation queues that hold batches for analyst sign-off before downstream stages can proceed; slow HDFS disk I/O and MapReduce job scheduling overhead; and sequential batch dependencies that cannot exploit the fact that over 90% of the quarter's transaction data is already present days before quarter-end.

The proposed NX1 architecture compresses this cycle to **3 weeks** — a **40% reduction**. This improvement is driven by continuous CDC ingestion that pre-stages and pre-calculates data throughout the quarter, Trino interactive queries that replace hour-long Hive scan jobs for analyst validation, and automated NX1 Sentinel quality assertions that replace manual sign-off gates. The remaining 3-week window covers regulated parallel-run validation and final regulatory submission in line with the Fed/OCC submission calendar.

> **Program Target:** Full production migration, dual-run validation, and legacy Hadoop and Informatica PowerCenter decommission completed within 18 months, with no disruption to the existing Tableau reporting layer used by the CFO's office.

---

## Section 2 — Hard Constraints & Non-Negotiables

The following constraints are treated as design boundaries for the program. Each is addressed within the proposed NX1 architecture as described in Section 3.

| Constraint | Requirement | NX1 Mechanism |
|---|---|---|
| **Hard — Zero Raw PII in Data Lake** | No raw, unmasked, or non-tokenized PII value may land on persistent object storage at any point. Column-level tokenization for identifier columns (e.g., account IDs, counterparty identifiers); column-level irreversible masking for descriptive PII columns (e.g., customer name, date of birth) — both applied inline before the first write, with no intermediate staging of raw values. | NX1 Mutation Hooks intercept every incoming record before any write to Iceberg object storage. The hook executor runs in a dedicated, isolated thread pool. A circuit breaker halts ingestion and raises an alert if hook processing latency breaches the agreed threshold, preventing unbounded accumulation of unmasked records in transit. |
| **Hard — Zero Tableau Disruption** | No breaking schema changes, workbook rewrites, column renames, or type-cast regressions at any phase. The Tableau layer is live during quarter-end cycles used by the CFO's office. | NX1 Prism endpoint presents a schema signature — column names, data types, nullable flags, and partition descriptors — identical to the legacy Hive metastore Tableau currently interrogates. Iceberg schema evolution with full backward compatibility provides a secondary safety net against upstream Oracle schema changes propagating to Tableau. |
| **Hard — Fixed Aggregator API Contract** | Egress to the third-party regulatory data aggregator must conform to their fixed REST API contract. NX1 does not control or modify the external contract. The contract requires: (a) a POST to fetch a session-scoped pre-signed upload URL; (b) a binary PUT of the submission package to that URL; (c) a metadata confirmation POST with structured payload fields (entity ID, submission period, package checksum, data classification). | NX1 orchestration adapts internally to produce this output sequence. Zero changes to the external interface. The pre-signed URL pattern means submission bytes go directly to the aggregator's storage, preserving chain of custody. A versioned internal schema contract, checked into source control, absorbs future aggregator format changes without ad-hoc pipeline changes. |
| **Hard — Compliance Retention & Lineage** | Automated end-to-end data lineage (source Oracle record → every transformation → final reported figure). Immutable 7-year WORM retention on all regulatory submission artifacts and their supporting lineage manifests. Object lock must be Compliance mode — no user, including storage admins, may delete or overwrite before the retention period expires. | NX1 Manifest for end-to-end lineage tracking and auditor-consumable export in structured and human-readable formats. Compliance-mode object lock on dedicated WORM archive prefixes (`regulatory-submissions` and `lineage-manifests`). Two-prefix topology isolates WORM archive prefixes from Iceberg maintenance operations. |
| **Hard — 18-Month Program Window** | Full production migration, dual-run validation, and legacy Hadoop and Informatica PowerCenter decommission complete within 18 months. | Four-phase execution plan with defined, non-negotiable gate exit criteria. See Section 6. |

---

## Section 3 — Proposed NX1 Architecture

### 3.1 Current-State Footprint

| Component                       | Role                                                                                                                       | Migration Disposition                                                                                                                                                                                                                                                                                                                                                                                  |
| ---------------------------------| ----------------------------------------------------------------------------------------------------------------------------| --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Oracle Data Warehouse**       | Source of record — regulatory & compliance datamart for Basel III RWA and CCAR scenario data                               | Retained. CDC reads continuously from Oracle transaction log.                                                                                                                                                                                                                                                                                                                                          |
| **IBM MQ**                      | Operational event backbone — batch-complete triggers, risk-update pushes, system notifications. Not a bulk data transport. | Retained. Events consumed by NX1 IBM MQ HA Connector.                                                                                                                                                                                                                                                                                                                                                  |
| **Informatica PowerCenter**     | Scheduled batch extraction from Oracle; on-prem PII tokenization & masking; HDFS load.                                     | **Decommissioned at M18, concurrent with Hadoop.** Continues as the authoritative HDFS writer for the legacy pipeline through the Phase 4 dual-run period. PII protection role transitions to NX1 Mutation Hooks in shadow certification from M4; Informatica remains the live production PII mechanism until M18 joint decommission. |
| **Apache Hadoop / HDFS + Hive** | Bulk staging; MapReduce RWA calculations; CCAR aggregations; Hive query layer for Tableau.                                 | **Decommissioned at M18.** Replaced by Apache Iceberg + Apache Spark + NX1 Prism.                                                                                                                                                                                                                                                                                                                      |
| **Tableau Server**              | CFO's office and analyst reporting layer — regulatory validation and executive dashboards.                                 | Retained, untouched. Re-pointed to NX1 Prism endpoint. No workbook edits required.                                                                                                                                                                                                                                                                                                                     |

#### Current-State Data Flow

```mermaid
flowchart TB
    %% ── INGESTION (batch, end-of-quarter) ───────────────────
    subgraph INGEST["① INGESTION  ⚠ end-of-quarter batch accumulation"]
        direction TB
        ORA["Oracle Data Warehouse\n(Basel III RWA · CCAR scenario data)"]
        MQ["IBM MQ\n(event signals — not bulk data)"]
        INFO["Informatica PowerCenter\n(scheduled batch extraction\non-prem PII tokenization & masking)"]
        HDFS["Apache Hadoop / HDFS\n(bulk staging)"]

        ORA -->|"scheduled batch\nextract — quarter-end"| INFO
        INFO -->|"PII-protected\nbatch load"| HDFS
        MQ -.->|"batch-complete trigger\n→ Informatica job"| INFO
    end

    %% ── COMPUTATION (MapReduce, slow I/O) ────────────────────
    subgraph COMPUTE["② COMPUTATION  ⚠ slow HDFS I/O & MapReduce overhead"]
        direction TB
        MR["MapReduce Jobs\n(RWA calculations\nCCAR aggregations)"]
        GATE["⚠ Human-in-the-Loop\nValidation Queue\n(analyst sign-off gate\nbefore downstream proceeds)"]

        MR -->|"calculation\noutput"| GATE
    end

    %% ── QUERY & REPORTING ────────────────────────────────────
    subgraph REPORT["③ QUERY & REPORTING  ⚠ hour-long Hive scan jobs"]
        direction TB
        HIVE["Apache Hive\n(query layer over HDFS)"]
        TABLEAU["Tableau Server\n(CFO's Office · analyst dashboards)"]

        HIVE -->|"Hive metastore\nschema & queries"| TABLEAU
    end

    %% ── CROSS-STAGE EDGES ────────────────────────────────────
    HDFS  -->|"HDFS disk read"| MR
    GATE  -->|"signed-off batch\nreleased downstream"| HIVE
```

> **Bottlenecks driving the 5-week cycle:** ⚠ marks the three compounding constraints — end-of-quarter batch accumulation, slow HDFS I/O and MapReduce job scheduling overhead, and sequential human-in-the-loop sign-off gates that block each downstream stage until an analyst manually approves.

### 3.2 End-to-End Data Flow — Visual Summary

The target-state pipeline flows through four stages: Ingestion (Section 3.3) → Computation (Section 3.4) → Query & Visualization (Section 3.5) → Regulatory Submission (Section 3.6).

```mermaid
flowchart TB
    %% ── INGESTION ──────────────────────────────────────────
    subgraph INGESTION["① INGESTION"]
        direction TB
        ORA["Oracle Data Warehouse\n(CDC — continuous log read)"]
        MQ["IBM MQ\n(MQ HA Connector — event signals)"]
        HOOKS["NX1 Mutation Hooks\n(inline PII tokenization & masking)"]
        ICE["Apache Iceberg\nCoW · unlocked working prefix\n(partitioned by period & domain)"]

        ORA -->|"CDC stream"| HOOKS
        HOOKS -->|"protected records"| ICE
        MQ -.->|"batch-complete &\nrisk-update events"| ICE
    end

    %% ── COMPUTATION ─────────────────────────────────────────
    subgraph COMPUTE["② COMPUTATION"]
        direction TB
        SPARK["Apache Spark\n(RWA chains · CCAR aggregations)\nversioned business rules"]
        SENTINEL["NX1 Sentinel\n(data quality assertions\nrow counts · null rates · totals)"]
        MANIFEST["NX1 Manifest\n(end-to-end lineage graph\nJSON + PDF export)"]

        SPARK -->|"quality assertions\nat every stage"| SENTINEL
        SPARK -->|"transformation\nlineage events"| MANIFEST
    end

    %% ── QUERY & VISUALIZATION ────────────────────────────────
    subgraph QUERY["③ QUERY & VISUALIZATION"]
        direction TB
        PRISM["Trino / NX1 Prism\n(Hive-compatible schema endpoint)"]
        TABLEAU["Tableau Server\n(CFO's Office — unchanged)"]

        PRISM -->|"Hive-identical\nschema & queries"| TABLEAU
    end

    %% ── REGULATORY SUBMISSION ────────────────────────────────
    subgraph SUBMIT["④ REGULATORY SUBMISSION"]
        direction TB
        ORCH["NX1 Orchestration\n(session POST → PUT upload\n→ metadata POST)"]
        AGG["Third-Party Regulatory\nData Aggregator\n(fixed REST API contract)"]
        WORM["WORM Object Store\nCompliance-mode lock · 7-year TTL\nregulatory-submissions/\nlineage-manifests/"]

        ORCH -->|"pre-signed URL →\nbinary PUT →\nmetadata POST"| AGG
        ORCH -->|"finalized submission\nartifact"| WORM
    end

    %% ── CROSS-STAGE EDGES ────────────────────────────────────
    ICE   -->|"Iceberg snapshots"| SPARK
    SPARK -->|"calculated Iceberg\nsnapshots"| PRISM
    SPARK -->|"submission package\n+ SHA-256 checksum"| ORCH
    MANIFEST -.->|"immutable lineage\nmanifest"| WORM
```

### 3.3 Ingestion Path

1. **Oracle CDC → NX1 Ingestion Plane.** Change Data Capture reads the Oracle Data Warehouse transaction log continuously, streaming incremental changes throughout the quarter. This keeps the NX1 data lake current at all times and eliminates the end-of-quarter batch accumulation that drives the 5-week cycle in the current state.

2. **IBM MQ → NX1 IBM MQ HA Connector.** Operational event signals from IBM MQ (batch-complete triggers, risk-update pushes) are consumed by the NX1 HA Connector and routed into the NX1 event stream. This eliminates polling and ensures dependent pipeline stages trigger deterministically. IBM MQ in this environment is an operational event-signalling backbone — not a bulk data transport (see Assumption A1 in Section 4).

3. **NX1 Mutation Hooks — inline PII protection.** Before any data touches persistent storage, NX1 Mutation Hooks intercept every incoming record and apply: (a) column-level tokenization to identifier columns that must retain referential integrity across tables (e.g., account IDs, counterparty identifiers); and (b) column-level irreversible masking to descriptive PII columns carrying no analytical value downstream (e.g., customer name, date of birth). No raw PII value is written to object storage at any point.

   The hook executor runs in a dedicated, isolated thread pool with reserved CPU allocation — it does not compete for compute with the CDC reader or the Iceberg writer. A circuit breaker halts CDC reader progress and triggers an operational alert if hook processing latency breaches the agreed SLA threshold, preventing unbounded accumulation of unmasked records in transit.

4. **Apache Iceberg Object Storage — working prefix.** Protected records land in NX1-managed object storage formatted as Apache Iceberg tables, partitioned by reporting period (quarter-end date) and data domain (Basel III capital / CCAR scenario). Copy-on-Write (CoW) mode is used — producing fully self-contained, immutable snapshot files that are the cleanest pattern for WORM compliance and point-in-time audit recall. Active ingestion and computation tables live under an *unlocked* working prefix to permit routine Iceberg maintenance operations (see WORM topology note in Section 3.7).

### 3.4 Computation Path

5. **Apache Spark — RWA & CCAR calculations.** Decoupled Spark jobs perform Basel III RWA calculation chains (credit risk, market risk, operational risk weightings) and CCAR scenario-based stress projection aggregations against the Iceberg tables. Business rules are managed as versioned, code-controlled configuration artifacts — not embedded in Hive scripts — satisfying change-control requirements. Because data is continuously pre-staged throughout the quarter, Spark jobs run progressively rather than in a single end-of-quarter burst. This is the primary mechanism for compressing the reporting cycle.

6. **NX1 Sentinel (data quality) & NX1 Manifest (lineage).** These are distinct engines with distinct responsibilities. NX1 **Sentinel** enforces automated data quality assertions at each pipeline stage: row count reconciliation, null rate checks, and aggregate total cross-checks against Oracle source records — replacing the manual human-in-the-loop validation queues in the current process. NX1 **Manifest** captures end-to-end lineage for every transformation, producing a machine-readable and human-auditable graph from source Oracle record → CDC event → Iceberg table version → Spark calculation → reported figure. Manifest exports lineage in structured JSON (for programmatic audit queries by the Bank's audit function) and human-readable PDF (for submission to Fed/OCC examiners).

### 3.5 Query & Visualization Path

7. **Trino / NX1 Prism Endpoint → Tableau.** The NX1 Prism endpoint is a Trino-compatible query abstraction that presents a schema signature — column names, data types, nullable flags, and partition descriptors — identical to the legacy Hive metastore that Tableau Server currently interrogates. Tableau's connection or extract refresh configuration is re-pointed at the NX1 Prism connection alias. No workbook edits, no column renames, no recalculations are required. Financial analysts and the CFO's office experience no visible change.

### 3.6 Regulatory Submission Path

8. **Spark → Submission Package Assembly.** At the close of the regulated calculation cycle, a Spark job produces the final submission artifact (structured Parquet or schema-compliant CSV as required by the aggregator contract) and computes a SHA-256 checksum. The assembly job is built against a versioned internal schema contract — a machine-readable schema definition (JSON Schema or Avro IDL) owned jointly by the Bank's integration team and NX1, checked into source control, and used to validate the output package before transmission. This versioned contract is the change surface that absorbs any future aggregator format updates without requiring ad-hoc pipeline modifications.

9. **NX1 Orchestration → Third-Party Aggregator REST API.** NX1 orchestration calls the aggregator's fixed REST API in the required sequence: (a) POST to the session endpoint to receive the pre-signed upload URL; (b) PUT the submission package binary to the pre-signed URL; (c) POST the structured metadata confirmation payload (entity ID, submission period, package checksum, data classification). The external API contract is consumed as-is — zero changes to the external interface. The pre-signed URL pattern ensures submission bytes travel directly to the aggregator's storage, preserving chain of custody without routing through any NX1 intermediate layer.

### 3.7 Governance & Security Envelope

- **Identity & Access Management:** Native integration with the Bank's central Okta (OIDC/SAML) identity provider for all user authentication and RBAC group mapping across the NX1 plane.
- **Workload Security:** Service accounts use OIDC Workload Identity Federation (or SPIFFE/SVID) to obtain short-lived, cryptographically bound tokens scoped to specific NX1 job contexts, with a maximum TTL of 1 hour. This limits blast radius if an automated job credential is compromised.
- **PII Protection:** NX1 Mutation Hooks enforce column-level tokenization on identifier columns and column-level irreversible masking on descriptive PII columns, applied before any write to persistent storage. NX1 Mutation Hooks run in parallel shadow certification mode alongside Informatica from M4, validating 100% column-level PII output equivalence throughout the shadow period against the Informatica PII column map delivered at Gate 1. Informatica remains the live production PII mechanism until M18 joint decommission, at which point NX1 Mutation Hooks become the sole PII protection mechanism.
- **Parallel Operation Governance Boundary (M4–M18):** From M4 onward, NX1/Iceberg is the authoritative store for all NX1 pipeline consumers; Informatica/HDFS remains the authoritative store for all legacy pipeline consumers. No consumer reads from both stores simultaneously. This boundary is enforced at the query layer: NX1 Prism serves NX1 consumers; Hive metastore serves legacy consumers. After Tableau is re-pointed to NX1 Prism at the Phase 3 cutover, the Hive metastore path is retired as a consumer-facing endpoint while the Hadoop compute environment remains live through the Phase 4 dual-run period.
- **WORM Retention & Two-Prefix Topology:** Compliance-mode object lock (7-year TTL) applies exclusively to the `regulatory-submissions` and `lineage-manifests` prefixes, which receive finalized, immutable artifacts only. All active ingestion tables, working computation partitions, and Iceberg metadata files live under a separate unlocked working prefix to preserve routine Iceberg maintenance operations.
- **Lineage & Audit:** NX1 Manifest tracks end-to-end data lineage from source Oracle record to every reported figure, with exportable audit reports in structured JSON and human-readable PDF for Fed/OCC examination cycles. See Section 3.4, item 6 for the full capability description.

### 3.8 Why Apache Iceberg

Apache Iceberg replaces HDFS as the analytical storage layer for four reasons that are particularly relevant in this regulatory context:

- **Copy-on-Write mode** produces self-contained, append-only snapshot files — cleaner for WORM enforcement and auditor review than Merge-on-Read delta chains, and directly compatible with the Compliance-mode object lock requirement.
- **Time-travel queries** allow the Bank to reconstruct the exact data state used for any prior regulatory submission, supporting audit challenges and resubmissions without restoring backups. This is a direct audit capability that cannot be replicated by the current HDFS architecture.
- **Partition evolution** allows the Bank to restructure storage partitioning by reporting period without rewriting historical data — essential as regulatory calendar requirements change across program phases.
- **Schema evolution** with full backward compatibility ensures the Trino/Prism endpoint can absorb upstream Oracle schema changes without breaking downstream Tableau — a secondary safety net for the zero-disruption guarantee.

---

## Section 4 — Design Assumptions

The following assumptions underpin the architecture. Each will be validated during the M1 discovery workstream. If any assumption changes materially, the impact will be assessed and the design will be updated accordingly before Gate 1.

| # | Assumption | Basis | If Invalidated |
|---|---|---|---|
| A1 | **IBM MQ Boundary** | IBM MQ functions strictly as an operational event-signalling backbone — batch-complete triggers, risk-update pushes, system notifications. It is not used for bulk file staging or high-volume transaction streaming. | Connector architecture and throughput requirements change materially. To be validated in M1 discovery with the DBA team. |
| A2 | **Tableau Connection Mode** | Tableau dashboards query the Hive layer via Live Connection or structured extract refresh — not via embedded direct JDBC with custom Hive-specific SQL functions. | If custom Hive SQL is embedded in workbooks, the Prism endpoint abstraction will be insufficient and workbook-level remediation will be required. To be validated in M1 by auditing the Tableau connection registry with the DBA team. |
| A3 | **Oracle CDC Availability** | Oracle Data Warehouse has supplemental logging enabled, or a CDC read-replica can be provisioned without disrupting the production warehouse operational SLA. | Architecture falls back to a micro-batch extraction pattern — workable but the continuous pre-staging benefit is partially reduced. To be validated in M1 with the DBA team. |

---

## Section 5 — Stakeholder Landscape

> **Note:** The SA Review Board & InfoSec Guardians row below summarizes the approvals and governance inputs required from this audience.

| Stakeholder                                                            | What They Own                                                                                                                                                                                               | What Is Required From Them                                                                                                                                                                                                                                                                                       |
| ------------------------------------------------------------------------| -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **CFO's Office & Business Partners** *(Executive Sponsors)*            | Regulatory compliance outcome; executive mandate; budget authority. Focused on SLA compression and mitigating regulatory fine exposure.                                                                     | Executive sponsorship letter confirming the 18-month program mandate; historical data on current SLA breach frequency and regulatory penalty exposure (quantifies the business case); approval authority at each program gate.                                                                                   |
| **Financial Analysts & Domain Experts** *(Consumer / UAT Owners)*      | Day-to-day power users of the Tableau reporting layer. Define what "correct" means for RWA and CCAR output figures. Own UAT sign-off on data parity.                                                        | Documented sign-off criteria for parallel run reconciliation — specifically the acceptable aggregate variance threshold; UAT participation during M10–M14; catalogue of every Tableau workbook and calculated field that must remain bit-for-bit identical after the Prism cutover.                              |
| **Application Development & DBA Teams** *(Technical Partners)*         | System maintainers of Oracle Data Warehouse and Informatica estate. Control Oracle CDC read-replica configuration. Own Tableau server administration.                                                       | Oracle CDC read-replica provisioning; Informatica job inventory and PII column mapping documentation (to verify NX1 Mutation Hooks cover every protected column); Tableau server schema documentation and connection string registry; agreement on the coordinated Informatica and Hadoop decommission timeline. |
| **SA Review Board & InfoSec Guardians** *(Approvers)*                  | Enterprise security approval; network topology sign-off; compliance framework alignment; TCO reduction validation.                                                                                          | Network topology approval and firewall rule sign-off; two-prefix WORM topology design review before any storage bucket is provisioned; pen-test scheduling for the NX1 connectivity boundaries; formal security review of OIDC workload token scoping.                                                           |
| **Aggregator Contract Owner** *(Integration Lead — to be named in M1)* | Monitoring of third-party aggregator release notes and change notifications for the regulatory submission API contract. Single point of contact when the aggregator changes its expected submission format. | Formal appointment confirmed in M1; subscribed to aggregator change notification channels; committed to timely notification before the next submission cycle when a format change is detected.                                                                                                                   |

---

## Section 6 — 18-Month Execution Plan

> Gate exit criteria define the completion conditions for each phase and require sign-off from the designated approvers.

### Phase 1 — Foundation, Discovery & Governance Setup — Months 1–3

**Goal:** Establish the technical, security, and governance foundations required for M4 ingestion activation.

- Activate network topologies, firewall rules, and VPN/private connectivity between the Bank's on-prem environment and the NX1 software plane — pending InfoSec Review Board approval.
- Design and obtain InfoSec Guardians sign-off on the two-prefix storage topology (unlocked working prefix + WORM archive prefix) before any storage buckets are provisioned — Gate 1 prerequisite.
- Configure object storage buckets with Compliance-mode object lock on the `regulatory-submissions` and `lineage-manifests` prefixes (7-year TTL); leave working and ingestion prefixes unlocked for Iceberg maintenance.
- Stand up the core NX1 software plane integrated with the Bank's Okta directory for RBAC group mapping and OIDC workload identity federation.
- **Discovery workstream (parallel):** Audit Oracle CDC availability with DBA team; catalogue all Tableau workbooks and connection strings; document Informatica PII column mapping inventory; confirm IBM MQ throughput profile; confirm Aggregator Contract Owner appointment; align the coordinated Informatica and Hadoop decommission timeline.

> **Critical Dependencies — Phase 1:** Two workstreams should run in parallel from Day 1: (1) InfoSec network topology approval and (2) DBA team Oracle CDC read-replica provisioning. Both are on the critical path for M4 ingestion activation and should be tracked with separate RACI owners.

#### Gate 1 (M3 exit)
- InfoSec network approval signed off
- Two-prefix storage topology reviewed and signed off by InfoSec
- Object storage buckets provisioned with correct lock configuration
- Oracle CDC read-replica provisioned
- IBM MQ throughput profile documented
- Informatica PII column map delivered
- Aggregator Contract Owner named
- Coordinated Informatica and Hadoop decommission timeline agreed

---

### Phase 2 — Ingestion Pipeline Build, PII Handover & Shadow Calculations — Months 4–9

**Goal:** Establish full CDC ingestion, complete PII shadow validation alongside Informatica, and accumulate the shadow calculation evidence required for cutover readiness.

- Deploy the NX1 IBM MQ HA Connector.
- Activate Oracle CDC ingestion into NX1 Mutation Hooks → Iceberg. Both NX1 CDC and Informatica operate in parallel from M4 — Informatica continues as the live production PII mechanism and authoritative HDFS writer; NX1 Mutation Hooks run in shadow certification mode throughout Phase 2 and Phase 3 with no live switchover until M18.
- Run NX1 Mutation Hooks in continuous shadow mode against Informatica's PII protection output — confirm 100% column-level output equivalence across every column in the Informatica PII column map (delivered at Gate 1). Shadow parity evidence accumulates throughout Phase 2 and is carried into Phase 3; no decommission proceeds without full equivalence sign-off by the Bank's DBA team and InfoSec Guardians.
- Activate Spark shadow calculation jobs for RWA chains and CCAR scenario aggregations, running quarterly outputs in parallel with legacy Hadoop MapReduce jobs.
- NX1 Sentinel data quality assertions active at every pipeline stage — row counts, null rates, aggregate totals — with automated alerting on variance.

> **Parallel Operation Governance Boundary:** Active from M4 ingestion activation. NX1/Iceberg is authoritative for NX1 consumers; Informatica/HDFS is authoritative for legacy consumers. No consumer reads from both stores simultaneously. This boundary must be documented in the operational runbook and signed off by the Bank's DBA team and InfoSec Guardians before M4. See Section 3.7 for the full governance boundary definition.

- Begin a structured enablement program for DBA and AppDev teams on Spark, Trino, and Iceberg operational tooling, running in parallel with the build phases.

#### Gate 2 (M9 exit)
- Two consecutive quarterly Spark shadow runs showing ≤0.01% aggregate variance against Hadoop MapReduce outputs, validated and signed off by the Bank's Risk function. A third shadow run is planned in Phase 3 (M12 — see Gate 3); all three runs must be signed off before Gate 3.
- NX1 Mutation Hooks shadow parity evidence documented across both completed quarterly cycles and signed off by the Bank's DBA team and InfoSec Guardians
- IBM MQ HA Connector sustained at peak throughput for 30 consecutive days

---

### Phase 3 — NX1 Prism Activation & Zero-Disruption Tableau Cutover — Months 10–14

**Goal:** Complete the Tableau cutover to NX1 Prism with no analyst-visible disruption and bring the regulatory aggregator integration live in staging.

- Expose the NX1 Prism Trino-compatible endpoint with a schema signature certified against the legacy Hive metastore DDL for the Bank's specific Tableau Server version.
- Re-point Tableau Server connection aliases to the NX1 Prism endpoint — no workbook edits, no schema changes. Financial Analyst UAT team validates every dashboard and calculated field against sign-off criteria defined in M1.
- Stand up the regulatory aggregator integration in staging: Spark submission package assembly → NX1 orchestration → REST API call sequence (pre-signed URL fetch → PUT upload → metadata POST). End-to-end tested against the aggregator's staging environment.
- Continue and complete the structured enablement program for DBA and AppDev teams.

#### Gate 3 (M14 exit)
- All three consecutive quarterly Spark shadow runs at ≤0.01% aggregate variance signed off by the Bank's Risk function (third run must complete by M12 — prerequisite for this gate)
- Financial Analyst UAT sign-off on all Tableau dashboards
- Regulatory aggregator end-to-end integration test passed in staging
- The Bank's DBA team certified on NX1 operational procedures

---

### Phase 4 — Dual-Run, Production Handover & Hadoop Decommission — Months 15–18

**Goal:** Demonstrate production parity across live quarter-end close cycles, complete operational handover, and decommission legacy Hadoop and Informatica PowerCenter in a coordinated event.

- Execute a minimum of **two complete live quarter-end close cycles** concurrently on both the legacy Hadoop environment and the NX1 environment. Three cycles preferred if the Fed/OCC regulatory calendar permits within the window.
- Produce formal parity reports for each dual-run cycle — mathematical equivalence of RWA calculations, CCAR scenario outputs, and submission package contents — reviewed and signed off by the Bank's Risk and Audit committees (Audit added at this stage as sign-off authority given live production submission scope).
- Transfer operational ownership of NX1 to the Bank's DBA and AppDev teams. NX1 SA remains on embedded support through final Hadoop decommission.
- Decommission Informatica PowerCenter and the legacy Hadoop cluster together after the final dual-run cycle receives Risk and Audit sign-off, ensuring a coordinated transition from the legacy pipeline.
- NX1 Manifest lineage export used to produce the first full audit-ready lineage report for the Fed/OCC submission record.

> **Regulatory Calendar Dependency:** The dual-run in M15–M18 must be sequenced to capture at least one live quarter-end close cycle on the NX1 environment before Hadoop decommission. This program must be calendared against the Federal Reserve's actual reporting cycle.

#### Gate 4 (M18 exit)
- Minimum two live quarter-end dual-run cycles with Risk and Audit sign-off on parity reports
- Legacy Hadoop cluster decommissioned
- Informatica PowerCenter decommissioned
- Operational ownership formally transferred to the Bank's DBA and AppDev teams
- First Fed/OCC audit-ready lineage report produced and filed

---

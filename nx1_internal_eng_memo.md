# Basel III & CCAR Engagement — Engineering & Product Risk Assessment

**Classification:** NexusOne Internal — Not for Customer Distribution  
**From:** Principal SA, Embedded — Charlotte NC  
**Version:** 1.1 — Internal  
**To:** NX1 Engineering Leadership, Product Management, Account Team  
**Engagement:** Tier-1 US Bank — Basel III & CCAR Pipeline Migration  
**Re:** Pre-conditions for SA Review Board design sign-off & 18-month program commitment
**Distribution:** NX1 Internal Only  

---

## Section 1 — Why This Memo Exists

I have prepared and submitted a design memo to this Bank's Solutions Architecture Review Board and InfoSec Guardians proposing migration of their Basel III and CCAR regulatory reporting pipelines to the NX1 platform. The memo commits NX1 to a specific architecture and an 18-month delivery program. Several of those commitments are contingent on platform capabilities that are either unverified at the required scale, not yet generally available, or carry known engineering gaps.

This memo documents those gaps directly, maps each one to the specific customer commitment it threatens, and states what I need from Engineering and Product before I can either defend the design to the SA Review Board or allow the program to proceed past Gate 1. This is not a wishlist. Every item below is a prerequisite or a risk that will materialize on the critical path.

Sections 1–4 are addressed to Engineering and Product together — they cover the operational gaps and the specific deliverables I need. **Section 5 is addressed to Product leadership specifically** — it contains four structural platform observations this engagement has surfaced that are relevant beyond this single account.

> **Stakes:** This is a Tier-1 US bank in a highly regulated environment. Basel III and CCAR filings are Federal Reserve mandates. A missed filing deadline or a PII exposure event at any phase does not produce a service credit conversation — it produces a regulatory enforcement action against the Bank and a reputational event for NX1. The engineering requirements below reflect that exposure, not abstract product quality standards.

---

## Section 2 — What Was Committed to the Customer

The following commitments are live in the SA Review Board design memo. Engineering and Product need to understand exactly what has been proposed so they can assess the gap against current platform state.

| Ref    | Commitment                                                                                                                                                                                                                                                                                                    | Contingency / Where It Can Break                                                                                                                                                                                                                                         |
| --------| ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------| --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **C1** | Zero raw PII on persistent object storage at any point, enforced inline by NX1 Mutation Hooks before every Iceberg write, with a dedicated isolated thread pool and a circuit breaker that halts CDC ingestion if hook latency exceeds the agreed SLA threshold.                                              | Hooks have not been certified at Oracle CDC peak throughput for this Bank. If they introduce per-record latency that backs up the CDC stream, unmasked records accumulate in transit. InfoSec will not sign off the design without a benchmark at 2× peak load.          |
| **C2** | Zero disruption to Tableau Server — connection re-pointed to NX1 Prism endpoint with schema signature identical to the legacy Hive metastore. No workbook edits, no column renames, no type-cast regressions.                                                                                                 | Prism DDL compatibility against the Bank's specific Tableau Server version has not been certified. Any implicit type drift — INT32/INT64 mismatches, nullable flag differences — will silently break Tableau calculated fields and produce incorrect regulatory figures. |
| **C3** | IBM MQ operational event signals consumed by the NX1 IBM MQ HA Connector with exactly-once delivery semantics, transactional checkpoint tracking, and DLQ failover — GA before Month 4 (start of ingestion pipeline build).                                                                                   | The existing open-source MQ connector variants in the NX1 software plane are known to exhibit instability under high concurrency. There is no enterprise-grade HA connector currently available at the reliability level this engagement requires.                       |
| **C4** | NX1 Manifest exports end-to-end lineage in structured JSON (with documented public schema) and human-readable PDF for submission to Fed/OCC examiners during audit examinations.                                                                                                                              | The SA Review Board memo cannot be signed off without confirmation that these export formats are GA, or a committed roadmap date before M10. Without this, the Bank cannot meet its regulatory compliance mandate and the design fails the InfoSec gate.                 |
| **C5** | Two-prefix Iceberg storage topology — unlocked working prefix for active ingestion and computation; WORM archive prefix for finalized submission artifacts and lineage manifests. Iceberg maintenance disabled on the WORM archive prefix. Design reviewed and signed off by InfoSec Guardians before Gate 1. | Iceberg maintenance operations issue `DeleteObject` calls that Compliance-mode lock returns as `AccessDenied`, leaving Iceberg metadata in an inconsistent state. This design must be specified and certified before storage is provisioned — it cannot be retrofitted.  |
| **C6** | Informatica PowerCenter decommissioned at M18 jointly with Hadoop — not M9. Informatica remains the authoritative HDFS writer and live production PII mechanism through the Phase 4 dual-run period. NX1 Mutation Hooks run in shadow certification mode from M4 onward with no live PII switchover until M18. This is a deliberate architectural decision to eliminate three design loopholes (Phase 4 HDFS data gap, impossible three-quarter shadow run calendar, unquantified PII switchover window). Parallel operation M4–M18 is governed by the data governance boundary defined in SA Review Board §3.7. | The original M9 contractual decommission term requires formal amendment with the Bank's DBA team before Gate 1 (M3 exit). If the amendment is declined, the three loopholes resurface and a revised design must be brought back to the SA Review Board before Gate 1. |

---

## Section 3 — Engineering Items — Prioritized

P0 items block Gate 1 or Phase 2 activation outright. P1 items must be resolved or formally risk-accepted before the SA Review Board design memo is signed off. P2 items are delivery risks that must be actively managed from M1.

---

### P0 — Blocking: IBM MQ HA Connector — Must GA Before M4

> `Commitment C3` · `SA Review Board §7 item 2` · `Phase 2 gate dependency`

The Bank's operational event backbone is IBM MQ. The existing open-source streaming connector variants within the NX1 software plane are not fit for purpose at this engagement's reliability requirement: they fail to checkpoint gracefully under message surge conditions, lose positional state on restart, and have no Dead Letter Queue failover. These are not edge-case failure modes — they are the documented baseline behavior of the current connectors under sustained load.

The Phase 2 ingestion pipeline build cannot begin without a production-grade connector. If this slips past M4, the entire ingestion activation phase shifts right and the M15–M18 dual-run window collapses — potentially below the minimum two live quarter-end close cycles required for the Fed/OCC parallel run evidence.

**Engineering requirement:** Deliver an enterprise-hardened NX1 IBM MQ HA Connector with: (a) transactional checkpoint tracking with persistent offset storage that survives process restart; (b) exactly-once delivery semantics end-to-end; (c) automated DLQ failover with operational alerting; (d) tested behavior at 2× the Bank's confirmed peak MQ throughput (to be documented in M1 discovery). **Required GA: before Month 4.**

**Account & Commercial ask:** If there is any risk of the GA date slipping past M4, I need a contractual backstop clause agreed with the Bank that specifies remediation obligations and a program extension mechanism. The critical path cannot absorb a connector slip without commercial protection.

---

### P0 — Blocking: Mutation Hooks Throughput Certification & Compute Isolation

> `Commitment C1` · `SA Review Board §7 item 1` · `InfoSec Guardians gate prerequisite`

The PII zero-exposure guarantee — the hardest constraint in this engagement — depends entirely on NX1 Mutation Hooks completing all column-level tokenization and irreversible masking operations fast enough that they do not back-pressure the CDC stream. If they introduce enough per-record latency at peak Oracle CDC throughput, the CDC reader blocks and unmasked records accumulate in the in-process buffer. That is a PII exposure window. The InfoSec Guardians will ask about this explicitly. I need a benchmark in hand before the SA Review Board presentation — which is the Gate 1 (M3 exit) event — not after.

**Engineering requirement (benchmark):** Throughput certification confirming Mutation Hooks complete all PII protection operations — both tokenization and irreversible masking — with sub-10ms per-record latency at ≥50K records/sec sustained ingestion. This is the estimated Oracle CDC peak rate for this Bank; confirm and adjust with M1 discovery data. The benchmark must be run at sustained 2× peak load, not burst.

**Engineering requirement (isolation):** The benchmark is necessary but not sufficient. Under sustained peak load, if the hook executor shares compute with the CDC reader or the Iceberg writer, CPU starvation will degrade hook throughput progressively — the benchmark passes in isolation but fails in production. Engineering must demonstrate: (a) the hook executor runs in a dedicated, isolated thread pool with a reserved CPU allocation that cannot be preempted by adjacent pipeline stages; (b) a circuit breaker halts CDC reader progress and triggers an operational alert if hook latency breaches the agreed SLA threshold — preventing unbounded accumulation of unmasked records in transit. Both must be demonstrated under sustained 2× peak load in the same benchmark run.

If the benchmark cannot be delivered before the SA Review Board presentation (Gate 1), I cannot defend the PII constraint to the InfoSec Guardians and the design cannot proceed.

---

### P0 — Blocking: NX1 Manifest Lineage Export Format — GA Confirmation Required

> `Commitment C4` · `SA Review Board §7 item 4` · `Compliance mandate`

The Bank's compliance mandate for Basel III and CCAR audit examinations requires the ability to produce a human-readable lineage report demonstrating end-to-end traceability from source Oracle records to every reported figure. This is not a nice-to-have. Fed/OCC examiners will request this during examination cycles. If NX1 Manifest only produces internal graph visualizations and does not support an exportable, auditor-consumable format, the compliance mandate fails on its own terms and the SA Review Board cannot sign off the design.

**Product requirement:** Confirm whether NX1 Manifest currently supports export to: (a) structured JSON with a documented public schema — machine-readable, for programmatic queries by the Bank's audit function; (b) human-readable PDF — for submission to Fed/OCC examiners. If both are GA, provide the product documentation reference for the SA Review Board. If either is not GA, provide a committed roadmap delivery date before M10. If there is no roadmap commitment, this item blocks design sign-off — I cannot propose a compliance architecture to a Tier-1 bank that depends on a capability with no delivery date.

---

### P1 — High Risk: Iceberg Two-Prefix Storage Topology & WORM Boundary Design

> `Commitment C5` · `SA Review Board §7 item 5` · `Gate 1 (M3 exit) prerequisite`

The 7-year WORM Compliance-mode object lock requirement creates an architectural hazard with Iceberg that must be resolved before any storage bucket is provisioned — it cannot be retrofitted. Iceberg's routine maintenance procedures (`expire_snapshots`, `remove_orphan_files`, `rewrite_data_files`) issue `DeleteObject` calls against the underlying object store. Compliance-mode lock returns `AccessDenied` on any locked object. If maintenance runs against the WORM archive prefix, Iceberg's internal metadata tables end up in an inconsistent state — manifest lists referencing files that cannot be deleted, orphaned metadata accumulating without bound. This is a silent correctness hazard: it will not surface until a maintenance job runs against the wrong prefix, at which point the metadata corruption may be irreversible.

**Engineering requirement:** Design and certify the two-prefix storage topology before M1 storage configuration begins. The design specification must cover: (a) which Iceberg table configurations are applied per prefix — specifically `write.metadata.delete-after-commit.enabled = true` and scheduled maintenance on the working prefix; all maintenance procedures explicitly disabled on the WORM archive prefix; (b) the post-submission promotion job that copies finalized artifacts to the WORM archive prefix as single-write immutable objects; (c) automated prefix-routing tests confirming no maintenance job can target a locked object. This design must be reviewed and signed off by InfoSec Guardians before Gate 1 (M3 exit) — it is a prerequisite to storage bucket provisioning.

---

### P1 — High Risk: Trino / NX1 Prism DDL Compatibility Certification

> `Commitment C2` · `SA Review Board §7 item 3` · `M10 Tableau cutover gate`

The zero-Tableau-disruption guarantee is the most visible commitment in the SA Review Board memo — the CFO's office will be using dashboards during live quarter-end close cycles throughout the migration. The entire guarantee rests on the NX1 Prism endpoint returning column-level metadata signatures — column names, data types, nullable flags, partition descriptors — that are bit-for-bit identical to the legacy Hive metastore DDL that Tableau Server currently interrogates.

The failure mode is silent and dangerous. If there is a type drift — even an implicit INT32/INT64 widening that Trino performs transparently — Tableau calculated fields that rely on that column will produce incorrect numerical results without surfacing an error. In a regulatory reporting context, incorrect figures that pass through validation silently are worse than an error that surfaces immediately.

**Engineering requirement:** A certified compatibility matrix between NX1 Prism endpoint metadata responses and the Hive metastore DDL spec for the Bank's specific Tableau Server version (to be confirmed in M1 discovery). Engineering must begin compatibility verification work in M1, as soon as the Tableau version is confirmed — running in parallel with Phase 2 build work, not as a Phase 3 sprint item. Verification must be completed and the compatibility matrix delivered before M10 cutover — starting in M1 is not sufficient; completion is the gate condition. Any gaps discovered late will cause the M10 cutover to slip and compress the dual-run window.

---

### P2 — Delivery Risk: Informatica Decommission Contract Amendment — M9 Term Must Be Extended to M18

> `Commitment C6` · `SA Review Board §7 item 7` · `SA Review Board §7 item 9` · `Gate 1 (M3 exit) contractual prerequisite`

The SA Review Board design memo has been revised to defer Informatica PowerCenter decommission from M9 to M18 — a joint decommission with the Hadoop cluster. This is a deliberate architectural decision, not a slip. Informatica remains the authoritative HDFS writer and live production PII mechanism through the Phase 4 dual-run period. The Parallel Operation Governance Boundary (SA Review Board §3.7) governs this: from M4 onward, NX1/Iceberg is authoritative for all NX1 pipeline consumers; Informatica/HDFS remains authoritative for all legacy pipeline consumers. No consumer reads from both stores simultaneously — this boundary is enforced at the query layer, eliminating the reconciliation ambiguity that a true dual-write conflict would create.

This deferral eliminates three critical architectural loopholes that the original M9 decommission date created: (1) the Phase 4 dual-run HDFS data gap — without Informatica writing to HDFS through M18, the legacy pipeline has no data during the dual-run period and parity comparison is impossible; (2) the impossible Gate 2 three-quarter shadow run calendar — three consecutive quarterly shadow runs cannot complete before M9 given a M4 ingestion start; (3) the unquantified PII switchover window — switching production PII protection from Informatica to NX1 Mutation Hooks mid-program, before shadow parity is fully proven, creates an InfoSec exposure window.

**Contractual action required:** The original M9 contractual decommission term must be formally amended with the Bank's DBA team before Gate 1 (M3 exit). This amendment must be agreed and confirmed before Phase 2 build begins. If the Bank's DBA team declines the amendment, the original M9 term stands, the three loopholes resurface, and a revised design must be brought back to the SA Review Board before Gate 1 — the program cannot proceed on the current architecture without this amendment.

**Engineering note:** NX1 Mutation Hooks must run in continuous shadow certification mode against Informatica's PII output from M4 activation, validating 100% column-level output equivalence across every column in the Informatica PII column map (delivered at Gate 1). Shadow parity evidence must accumulate across Phase 2 and Phase 3 — three consecutive quarterly cycles, with the third completing by M12 as a Gate 3 entry condition. No decommission of Informatica proceeds without full equivalence sign-off by the Bank's DBA team and InfoSec Guardians — this is the handover gate evidence, not a testing afterthought.

---

### P2 — Delivery Risk: Bank Enablement Gap — Operational Handover Risk

> `SA Review Board §7 item 8` · `Gate 3 (M14 exit) completion dependency`

The Bank's DBA and AppDev teams have no current structural familiarity with Spark, Trino, or Iceberg metadata architectures. The M15–M18 dual-run phase requires the Bank's team to operate the NX1 platform independently alongside legacy Hadoop. If they cannot do so, the dual-run cannot transition to full Bank ownership and the Hadoop decommission cannot proceed — which means NX1 SA remains on embedded support indefinitely and the 18-month program overruns.

A structured enablement program must be built into the program plan as a parallel workstream starting in Phase 2 (M4), not as a post-delivery training package. It must be scoped, resourced, and confirmed before Phase 2 start (M4). This needs Product to confirm what self-service enablement materials, documentation, and runbook content exist for Spark/Trino/Iceberg operations on the NX1 software plane, and whether NX1 professional services has an enablement delivery capability or whether this falls entirely to the embedded SA.

---

### P2 — Delivery Risk: Regulatory Calendar Dependency — Dual-Run Window

> `SA Review Board §6 Phase 4 callout`

Basel III and CCAR submissions follow the Federal Reserve's mandated reporting calendar. The Phase 4 dual-run (M15–M18) must capture at least one live quarter-end close cycle — preferably two — on the NX1 environment before Hadoop decommission. If the M15–M18 window does not align with the Fed's reporting calendar, we either cannot collect the required parallel run evidence or we must extend the program beyond 18 months. Neither outcome is acceptable under the current commercial terms.

Program Management must calendar the 18-month program against the Federal Reserve's actual reporting cycle from M1. This is not a discovery item — it is a scheduling prerequisite. If the M15–M18 window falls between quarter-end dates, the dual-run window contains no live cycle evidence and the program cannot close.

---

### P2 — Delivery Risk: Aggregator Contract Drift — No Current Monitoring Owner

> `Commitment C3 (adjacent)` · `SA Review Board §7 item 6` · `SA Review Board §5.1 Aggregator Contract Owner`

The third-party regulatory data aggregator's submission REST API is described in the Bank's brief as a "fixed" contract. In practice, regulatory aggregators routinely add required fields, deprecate optional fields, or change validation rules in response to evolving Fed/OCC guidance — frequently with short notice windows. If the Spark submission assembly job is built against a point-in-time snapshot of the contract with no change monitoring and no named owner, any undiscovered format change will cause a submission to be rejected at the aggregator boundary. In a regulatory filing context, a rejected submission is a missed filing deadline.

The SA Review Board memo requires three mitigations to be in place: (a) a versioned internal schema contract (JSON Schema or Avro IDL) checked into source control — the Spark submission assembly job is built against this versioned contract, not a hardcoded snapshot; (b) a named Aggregator Contract Owner on the Bank side, appointed in M1, who monitors aggregator change notification channels; (c) a change control process that routes any detected aggregator format changes to NX1 Engineering with sufficient lead time to update the assembly job before the next submission cycle. All three must be active before the Phase 3 aggregator integration goes into staging — missing any one of them leaves a gap through which a rejected submission can occur without warning.

---

## Section 4 — What I Need Before I Can Proceed

The table below is the minimum set of responses I need from Engineering and Product before the SA Review Board design memo can be formally defended and signed off.

| # | What I Need | From | Required By |
|---|---|---|---|
| 1 | Mutation Hooks throughput benchmark — sub-10ms at ≥50K rec/sec, sustained 2× peak load, with demonstrated CPU isolation and circuit breaker behavior. Written report, not verbal assurance. | Engineering | **Blocking** — Before SA Review Board presentation (Gate 1) |
| 2 | IBM MQ HA Connector GA commitment with confirmed delivery date. If there is any schedule risk, I need to know now — not at M3. | Engineering / Product | **Blocking** — Before Phase 2 start (M4) |
| 3 | NX1 Manifest lineage export: confirm GA status for structured JSON (with public schema) and PDF. If not GA, provide a committed roadmap date before M10. | Product | **Blocking** — Before SA Review Board sign-off |
| 4 | Two-prefix Iceberg / WORM topology specification — covering per-prefix Iceberg table configurations, maintenance disablement on archive prefix, and promotion job design. Must be ready for InfoSec review before any storage bucket is provisioned. | Engineering | Gate 1 (M3 exit) |
| 5 | Trino / NX1 Prism DDL compatibility matrix against the Bank's Tableau Server version (confirmed in M1). Engineering must begin compatibility verification in M1, as soon as the Tableau version is known, and complete it before M10 — not as a Phase 3 sprint item. | Engineering | **Completed before M10 cutover** |
| 6 | Confirmation of what NX1 enablement materials, runbooks, and professional services capacity exist for Bank DBA/AppDev team onboarding to Spark, Trino, and Iceberg operations on the NX1 software plane. | Product / Professional Services | **Confirmed by Phase 2 start (M4)** |
| 7 | Account team contractual backstop clause if IBM MQ HA Connector GA slips. The critical path cannot be protected technically if it is not also protected commercially. | Account & Commercial Team | **Blocking** — Before SA Review Board sign-off |
| 8 | Informatica PowerCenter decommission contract amendment — formal agreement with the Bank's DBA team to extend the decommission date from M9 to M18 (joint decommission with Hadoop). Written contractual amendment, not a verbal agreement or project plan item. Required before Phase 2 build begins. | NX1 SA / Bank DBA Lead | **Gate 1 (M3 exit)** |

---

## Section 5 — What This Engagement Reveals About Platform Readiness

*This section is addressed to Product leadership. Beyond the immediate items above, this engagement surfaces four structural observations relevant to NX1's positioning in the Tier-1 financial services segment beyond this account.*

### 5.1 The IBM MQ Gap Is a Tier-1 Financial Services Market Access Blocker

IBM MQ is the messaging backbone of a substantial share of Tier-1 banks, insurance carriers, and capital markets firms. Any NX1 engagement in that segment that involves operational event signalling will encounter this connector requirement. The current open-source connectors are not credible at enterprise reliability standards. Until a production-grade HA connector exists, NX1 cannot credibly sell into IBM MQ environments — which is most of the Tier-1 financial services market. This is not a feature gap; it is a market access gap.

### 5.2 Mutation Hooks Throughput Has Not Been Characterized at Enterprise CDC Volumes

Inline PII protection before data lake ingestion is a mandatory architectural pattern for any bank with a cloud or hybrid data migration. The fact that Mutation Hooks have not been load-tested at enterprise Oracle CDC peak throughput means every financial services engagement that proposes inline PII protection is carrying unquantified delivery risk. The benchmark required for this engagement should become standard product certification evidence — not a per-engagement custom deliverable.

### 5.3 NX1 Manifest Lacks Auditor-Grade Export Formats

Data lineage tracking is table stakes for regulated financial services. However, lineage that exists only as internal graph visualizations is not actionable for external auditors. Fed/OCC examiners expect structured, exportable lineage evidence. If NX1 Manifest cannot produce structured JSON with a documented public schema and human-readable PDF output, every compliance-driven financial services engagement will hit this gap. It should be treated as a product capability gap, not a configuration item.

### 5.4 WORM + Iceberg Requires a Documented Reference Architecture

The two-prefix topology requirement to prevent Iceberg maintenance from conflicting with Compliance-mode object lock is a non-obvious engineering hazard. Left to individual SAs to re-derive per engagement, it will be implemented inconsistently — and an inconsistent implementation that runs Iceberg maintenance against a locked prefix will produce silent metadata corruption in a production regulatory archive. This should be documented as a reference architecture in the NX1 knowledge base so it is not re-derived as a new discovery on every regulated-data engagement.

---

## Appendix — Risk Register Summary

| # | Risk | Priority | Owner | If Unresolved |
|---|---|---|---|---|
| R1 | IBM MQ HA Connector not GA by M4 | **P0 — Blocking** | Engineering / Product | Phase 2 ingestion activation blocked; dual-run window collapses; 18-month program at risk. |
| R2 | Mutation Hooks throughput/isolation not certified before SA Review Board presentation (Gate 1) | **P0 — Blocking** | Engineering | InfoSec Guardians will not sign off the PII constraint. Design cannot be presented to SA Review Board. |
| R3 | NX1 Manifest lineage export not GA and no committed roadmap date | **P0 — Blocking** | Product | Compliance mandate for Fed/OCC audit examinations cannot be met. SA Review Board sign-off blocked. |
| R4 | Two-prefix Iceberg/WORM topology not designed before M1 storage provisioning | **P1 — High Risk** | Engineering | Silent Iceberg metadata corruption in production regulatory archive. Potentially irreversible. |
| R5 | Trino/Prism DDL compatibility not certified against Bank's Tableau version before M10 | **P1 — High Risk** | Engineering | Tableau calculated fields silently produce incorrect regulatory figures. M10 cutover fails. |
| R6 | Informatica decommission contract amendment not confirmed by Gate 1 (M3 exit) | **P2 — Delivery Risk** | NX1 SA / Bank DBA | Original M9 term stands; three architectural loopholes resurface (Phase 4 HDFS data gap, impossible three-quarter shadow run calendar, unquantified PII switchover window); revised design required before Gate 1. |
| R7 | Bank DBA/AppDev enablement not confirmed by Phase 2 start (M4) | **P2 — Delivery Risk** | NX1 SA / Product | M15–M18 handover cannot complete; NX1 SA on extended embedded support; program overrun. |
| R8 | Program not calendared against Federal Reserve reporting cycle from M1 | **P2 — Delivery Risk** | Program Management | M15–M18 dual-run window contains no live quarter-end close cycle; program cannot close. |
| R9 | No named Aggregator Contract Owner; submission assembly built against point-in-time contract snapshot with no change control process | **P2 — Delivery Risk** | Bank Integration Lead / NX1 SA | Undetected contract change causes submission rejection at aggregator boundary — equivalent to a missed filing deadline. |

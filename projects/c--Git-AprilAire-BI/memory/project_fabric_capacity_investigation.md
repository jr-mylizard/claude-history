---
name: project-fabric-capacity-investigation
description: AprilAire BI Fabric ETL — investigating Lakehouse vs Warehouse landing and Dataflow Gen2 vs Pipeline Copy Activity to reduce Fabric capacity (CU) usage
metadata: 
  node_type: memory
  type: project
  originSessionId: 9f2d05e3-5612-4a4e-ab0b-1f7ba220a6a3
---

Evaluating whether to switch some/all Dataflow Gen2 landing tables from Lakehouse to Warehouse destination, and separately whether to move DB2 ingestion off Dataflow Gen2 onto Pipeline Copy Activity — both as part of a broader push toward incremental loads to cut Fabric capacity (CU) usage on the `ETL_DEV` branch of the AprilAire BI repo (`c:/Git/AprilAire BI`).

**Why:** Team is wary of added architectural complexity (real possibility raised earlier of scaling back or abandoning some of the built automation), so any change here needs a clear, evidence-based capacity payoff, not just a latency/performance win.

**Key findings so far (research-backed, not yet empirically tested against this repo's actual workload):**
- Warehouse ingestion is *less* CU-efficient than Lakehouse ingestion. Structural reason: Fabric's CU-to-compute ratio differs by engine — 1 CU = 2 Spark vCores, but 1 CU = 0.5 Warehouse vCores — so Warehouse compute costs roughly 4x the CU rate of Spark compute per unit of work. Warehouse transformations can be faster in wall-clock time but cost *more* CU, not less — speed and CU cost are not the same axis.
- Bigger lever than Lakehouse-vs-Warehouse: Dataflow Gen2 itself (used for every current DB2 pull) is reported at 4-7x more CU-expensive than Fabric Pipeline Copy Activity for equivalent SQL-source ingestion, per community benchmarks (Alphabold ingestion-cost comparison; Fabric Community threads on Gen2 vs Gen1/Copy Activity CU usage).
- DB2 for i (iSeries/AS400) **is now** a supported Fabric Pipeline Copy Activity source via on-premises data gateway, with a dedicated native connector. This was **not** true when this ETL was originally built in Azure Data Factory (pre-Fabric) — Dataflow was the only viable path to DB2 via gateway at the time, so the original choice wasn't wrong, just constrained by what existed then.
- Current dataflows pull DB2 via simple native SQL query pass-through (`[Query = "..."]` embedded directly in `DB2.Database(...)`, e.g. `ORD_HEAD_Hourly` in the Orders dataflow) — essentially no real Power Query-side transformation happening. This makes them strong candidates for a fairly direct migration to Copy Activity's custom-query source option, since none of Dataflow's transformation capability is actually being used.

**How to apply:** Before recommending a specific direction, the actual next step (not yet started) is to empirically test Copy Activity against the real DB2 source and measure true CU consumption via the Fabric Capacity Metrics app — general benchmarks above are for different source types (SQL Server, OData) and may not transfer exactly to this DB2-via-gateway pattern. Treat the Lakehouse-vs-Warehouse question as secondary to the Dataflow-vs-Copy-Activity question when prioritizing capacity-reduction work. See also [[project-fabric-lakehouse-maintenance]] for the related Lakehouse OPTIMIZE/VACUUM workstream.

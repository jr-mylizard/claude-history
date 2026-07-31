---
name: project-fabric-lakehouse-maintenance
description: "AprilAire BI Fabric ETL — Lakehouse OPTIMIZE/VACUUM maintenance plan (no schemas enabled), and how it differs from Warehouse's automatic compaction"
metadata: 
  node_type: memory
  type: project
  originSessionId: 9f2d05e3-5612-4a4e-ab0b-1f7ba220a6a3
---

Building a Notebook-based script to run Delta table maintenance across all tables in the (currently schema-less) landing Lakehouse in the AprilAire BI repo. Enumerate tables with `spark.catalog.listTables()` (no schema qualifier needed — this Lakehouse doesn't have schemas enabled), then loop `OPTIMIZE` and `VACUUM` per table. `display()` truncates at 1,000 rows by default in Fabric notebooks — use `.count()` on the DataFrame for true totals, not `display()`.

**Why:** Lakehouse requires manual OPTIMIZE (small-file compaction, most relevant for incremental/append-loaded tables which accumulate small files continuously) and VACUUM (storage reclamation, most relevant for truncate-and-reload tables since every reload orphans a full copy of the previous table's files) — Fabric does **not** do this automatically for Lakehouse. Fabric Warehouse, by contrast, handles equivalent compaction and garbage collection automatically in the background (configurable retention, default 30 days, whole-warehouse scope) — this asymmetry is a real operational-burden difference between the two destination types, separate from the CU-cost findings in [[project-fabric-capacity-investigation]].

**How to apply:**
- VACUUM does not affect current-version query performance — queries only ever read the current version's active file manifest, never the orphaned files VACUUM cleans up. VACUUM's only benefit is storage cost (OneLake billed separately from compute, ~$0.023/GB/month) and it caps how far back Delta time-travel/RESTORE can recover from an accidental bad overwrite (default retention 7 days for data files, ~30 days for the transaction log — recoverability is capped by whichever expires first, regardless of whether VACUUM ever runs).
- Pure append-only tables are safe from data loss under VACUUM by construction — VACUUM can only delete files no longer referenced by the current version, and a pure append never removes anything from the current version. Files only become VACUUM-eligible after something else (OPTIMIZE compaction, or an actual UPDATE/DELETE/MERGE/overwrite) supersedes them.
- Recommend running `VACUUM ... DRY RUN` before the real run the first time on a new table, and always specify `RETAIN` explicitly rather than relying on the implicit default, so retention policy is visible in code.

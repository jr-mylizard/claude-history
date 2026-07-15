---
name: project-prod-migration
description: Manual data changes on dev that must be replicated to prod during the prod migration session
metadata: 
  node_type: memory
  type: project
  originSessionId: a67b079d-bb60-4c6e-ad0e-ae7cb2626126
---

Prod migration is the next planned session. Schema migrations come from the PR diff. These are the manual data changes on dev that need to be applied to prod separately:

**`ios_apps` — bundle ID case corrections**
- 16 system app `package_id` values were corrected for case sensitivity (e.g. `com.apple.maps` → `com.apple.Maps`, `com.apple.mobilesms` → `com.apple.MobileSMS`, etc.)
- Prod has the same wrong-case values; fix must be applied there too

**`ios_apps` — duplicate row deletions**
- Two duplicate rows deleted on dev: Music (id=4796) and Wallet (id=4801)
- Prod likely has the same duplicates (IDs may differ — query by bundle ID, not row id)

**`ios_device_apps` — `allowed_in_mosyle` resets**
- After fixing the 16 bundle IDs on dev, `allowed_in_mosyle` was reset to `false` on affected rows to force CDA re-assignment in Mosyle
- Same reset must be done on prod after the bundle ID fix, so the sync re-pushes correct CDA values

**Why:** These were direct SQL edits during debugging, not captured in migrations.

**How to apply:** In the prod migration session, run the bundle ID fix SQL first, then delete duplicates (find by bundle ID), then reset `allowed_in_mosyle`. Schema migrations from the PR must run before the data changes.

---
name: project-mylizard-ios-test
description: ios.test.ts rewrite — full spec written 2026-07-03 at supabase/functions/tests/ios/REWRITE-SPEC.md; ready for a Sonnet session to implement
metadata: 
  node_type: memory
  type: project
  originSessionId: 191987b2-c4f0-4535-9b29-21f90f77e69e
---

**DONE 2026-07-04**: the rewrite was implemented and passes — 13 steps, ~38s, two consecutive
green runs against the remote dev branch, with the real device identity (WI_608A_JR_001_P_001)
verified restored after every run including real failures. Key implementation facts:
snapshot/clear/restore lives in `_shared/testing/snapshot-device-relations.ts`
(`snapshotDeviceIdentity`/`clearDeviceIdentity`/`restoreDeviceIdentity`); sanitizeOps/Resources
are off (fire-and-forget logging_api_calls inserts); `MOSYLE_TEST_DEVICE_IMEI` replaced
`MOSYLE_DEVICE_NAME`/`MOSYLE_TEST_DEVICE_GROUP_ID` in .env; tests run against the REMOTE dev
branch (functions/.env SUPABASE_URL), so deployed dev functions must be current. Gotcha that
cost an hour: new-API-key secret keys are PER-PROJECT — a key created on prod fails on the dev
branch with "Invalid API key... owned by another Supabase project". The weekly scheduled
Mosyle-profile test (spec follow-up section) is still open.

Stale notes resolved since the original memory: `product_id` migration is applied and
`db-types.ts` regenerated (the `as unknown as` cast can go); the design questions this memory
deferred to a "Fable session" were settled in that session — the answers are in the spec.

Design decisions appended to the spec 2026-07-04 ("Decisions added 2026-07-04" section):
one lifecycle test + no-device satellite files (not per-function); snapshot→clear→run→restore
protocol because the test IMEI is Jeremiah's daily phone with a real identity in dev
(WI_608A_JR_001_P_001 — a test run on 2026-07-02/03 consumed the real rows and they had to be
manually restored); a tripwire assertion that the dispatch suppression held (Mosyle profiles
can't be deleted via API); and a **separate follow-up work item**: a weekly scheduled
Mosyle-profile test against the dev branch (catches Mosyle web-UI drift, the Playwright
automation's dominant failure mode).

**Also open:** `enrollment-lifecycle.test.ts` (Android) has not been run recently.

---
name: project-ios-enrollment
description: iOS enrollment pipeline — unified flow verified end-to-end 2026-07-03 on dev; operational gotchas for Mosyle; Android registration is next
metadata: 
  node_type: memory
  type: project
  originSessionId: 191987b2-c4f0-4535-9b29-21f90f77e69e
---

iOS enrollment (MyLizard-Supabase + MyLizard_NodeJS) — full pipeline **verified end-to-end on
2026-07-03** on the dev branch (`xxinwdxprvkvpmhdelnp`): registration form → pre-enrollment
(SearchAPI app resolution, categories, zip→timezone) → 3-min cron enrollment sweep → CDA sync →
capped workflow dispatch → Playwright profile create (GitHub Actions) → adminlogs verification →
mosyle_status flush → enrollment emails. The old `Test_Unified_iOS_Android_Structure` branch
project (`papyhxdkbadzqckuqmhb`) has been **deleted**. Architecture is documented in the repo
(`ios.md`, rewritten 2026-07-03) — this memory holds the non-obvious operational facts:

## Mosyle gotchas (hard-won 2026-07-03)
- **Profile save silently refuses duplicate names** (no POST, no error, no log entry) — this
  masqueraded as a "silent save failure on headless" bug for days. The script now detects an
  existing same-name profile and updates it instead of creating.
- **Logs API**: endpoint is `/v1/adminlogs` (not `/logs`); send a **body-less POST** — date
  filters silently exclude rows; normal bearer auth works. Response nests `logs` under
  `response` (object), not `response[0]` as documented.
- **Rate limit**: 500 calls/10 min documented; 429s have an empty body and no Retry-After.
  All Mosyle calls now log to `logging_api_calls` (api_name='mosyle') for volume tracking.
- Deleted-looking profiles can persist: check the Install App list carefully — device profiles
  blend in with Mosyle's category-named profiles (alphabetical).

## Test-cycle ritual (re-run a profile create on dev)
Delete the profile in Mosyle, then clear `mosyle_status`/`mosyle_synced_at` on
`ios_device_apps` and reset `install_profile_dispatch_attempts` +
`install_profile_last_dispatched_at` on `customer_devices`; the 3-min cron
(`sync-apps-and-websites-ios`, cron `1-59/3`, created manually — never in migrations)
re-dispatches automatically. Dispatch is capped at 3 attempts, ≥60 min apart, alert email on
the final attempt.

## Deployment notes
- `supabase functions deploy <name> --project-ref xxinwdxprvkvpmhdelnp --import-map supabase/functions/deno.json --no-verify-jwt`; Docker not required.
- Always ask before deploying ([[feedback-deploy-approval]]); user manages all git operations.
- Pushes to `dev` trigger `set-branch-secrets` + Supabase branch redeploy; the
  mosyle-profile-manager workflow checks out the branch at dispatch time.

## Remaining (from 2026-07-02 architecture review)
- Rewrite `tests/ios/ios.test.ts` for the unified flow — note JS `getDay()` is 0=Sunday but
  `device_schedule_blocks.day_of_week` is 0=Monday (ISO, via `date_trunc('week')`).
- Narrow anon RLS policies + gate internal pages; no-secrets CI test subset.
- Automated `update` dispatch when apps are added to an already-created profile (only
  "create" is dispatched automatically today).
- `enrollment-summary.html` still loaded via `Deno.readTextFile` — unbundleable on deploy;
  inline it.
- **Android User Registration page is the next major work item.**

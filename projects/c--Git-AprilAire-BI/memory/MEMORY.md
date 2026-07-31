# Memory Index

- [**NEVER DEPLOY**](feedback_no_deploy.md) — Never call apply_migration, deploy_edge_function, git push/commit — user manages all deployments
- [Run scripts freely](feedback_run_scripts.md) — OK to run local PowerShell/Python commands during iterative dev; no-deploy rule still applies
- [User Profile](user_profile.md) — Jeremiah Robinson, MyLizard founder; works across BI, Node/Next.js, Supabase, Stripe
- [Feedback: Stripe source of truth](feedback_stripe_source_of_truth.md) — Sandbox is source of truth; always update Live to match Sandbox, never the reverse
- [Project: MyLizard Stripe](project_mylizard_stripe.md) — Stripe migration scripts, product structure, and live/sandbox alignment status
- [Feedback: Session titles](feedback_session_titles.md) — First message is often a session title; do not act on it, just wait for instructions
- [Project: iOS Enrollment](project_ios_enrollment.md) — unified pipeline verified end-to-end 2026-07-03; Mosyle gotchas (duplicate-name silent save refusal, /adminlogs API, rate limits) and dev test-cycle ritual; Android registration is next
- [Project: iOS Test Rewrite](project_mylizard_ios_test.md) — spec ready at supabase/functions/tests/ios/REWRITE-SPEC.md (2026-07-03); implement in a Sonnet session
- [Feedback: Supabase MCP over Docker](feedback_supabase_mcp_over_docker.md) — use MCP against branch DB (dev = xxinwdxprvkvpmhdelnp) instead of starting Docker; patch device_id when regenerating types
- [Project: RLS Narrowing](project_rls_narrowing.md) — anon lockdown + edge fn gating on dev branches 2026-07-03; vault token rotation + Vercel env vars pending; prod waits for WeWeb retirement
- [Project: Mosyle DNS & App API](project_mosyle_dns_app_api.md) — 33 DNS profiles built in Mosyle; internal API (MosyleDNSController, InstallAppController) understood; next session wires device_management page → dns_profiles → vw_desired_dns_state_mosyle → Playwright sync
- [Project: Device App Audit](project_device_audit.md) — GHA workflow + mosyle_device_check.py; SCHEDULED_PROJECT_REF repo var = dev until ~2026-07-14, then flip to prod
- [Project: Prod Migration](project_prod_migration.md) — Manual data changes on dev (bundle ID case fixes, duplicate deletes, allowed_in_mosyle resets) that must be replicated to prod; schema migrations come from the PR
- [Project: Enrollment Email](project_enrollment_email.md) — enrollment-summary.html moved to email_templates/ (underscore); Deno.readTextFile bundling hazard; follow-up is to inline as template literal
- [Project: Playwright Suite](project_playwright_suite.md) — 55 E2E tests in MyLizard_NodeJS/tests/e2e/; iOS lifecycle ready now, Android lifecycle skip-guarded until PLAYWRIGHT_TEST_ANDROID_IMEI set (~2026-07-14)
- [Project: Mosyle CDA Pipeline](project_mosyle_cda_pipeline.md) — render race (30s bounce fix), lpad truncation, "Installed" badge lies, free installs must use Playwright not App Store; cleanup list pending deploys
- [Project: Fabric Capacity Investigation](project_fabric_capacity_investigation.md) — AprilAire BI ETL: Dataflow Gen2 found 4-7x more CU-costly than Copy Activity; DB2/iSeries now supported via Copy Activity+gateway; testing not yet started
- [Project: Fabric Lakehouse Maintenance](project_fabric_lakehouse_maintenance.md) — AprilAire BI ETL: OPTIMIZE/VACUUM notebook loop plan for schema-less Lakehouse; Warehouse auto-compacts, Lakehouse doesn't

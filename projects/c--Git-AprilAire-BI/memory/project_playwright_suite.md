---
name: project-playwright-suite
description: "Playwright E2E test suite added to MyLizard_NodeJS covering registration, shop, admin, and device lifecycle tests"
metadata: 
  node_type: memory
  type: project
  originSessionId: 81510530-7f6e-431c-9486-8c6449c8788b
---

Playwright test suite created in `c:\Git\MyLizard_NodeJS\tests\e2e\` (2026-07-11).

**Why:** Validate frontend + backend integration end-to-end; physical iOS device available now, Android available 2026-07-14.

**Files created:**
- `playwright.config.ts` — loads `.env.local` + `.env.test.local`; serial execution (workers=1); chromium only
- `tests/e2e/helpers/db.ts` — Supabase service-role helpers (get/wait/cleanup for customer, device, apps, schedules, DNS, email log)
- `tests/e2e/helpers/mosyle.ts` — Mosyle API login + device lookup/enrollment validation
- `tests/e2e/helpers/hexnode.ts` — Hexnode API device + policy + app group lookup
- `tests/e2e/smoke.spec.ts` — 8 public page smoke tests (no auth)
- `tests/e2e/registration-form.spec.ts` — 11 form UI tests (formatting, validation, clear, URL params; no submission)
- `tests/e2e/registration-submission.spec.ts` — Actual submission + DB validation; skip-guarded on PLAYWRIGHT_TEST_IOS_IMEI / PLAYWRIGHT_TEST_ANDROID_IMEI
- `tests/e2e/shop.spec.ts` — Products, cart, checkout (Stripe redirect capture)
- `tests/e2e/admin-auth.spec.ts` — 401 guard + authenticated device management UI
- `tests/e2e/device-management.spec.ts` — Calendar, tabs, Add App, Recategorize, Check Devices, modals
- `tests/e2e/ios-lifecycle.spec.ts` — Full iOS enrollment lifecycle (pre→enroll→post); uses `page.pause()` for manual MDM install step
- `tests/e2e/android-lifecycle.spec.ts` — Full Android/Hexnode lifecycle (same pattern); all tests skip until PLAYWRIGHT_TEST_ANDROID_IMEI is set
- `.env.test.local.example` — template for test-specific env vars

**npm scripts:**
- `npm test` — all tests
- `npm run test:smoke` — fast smoke (no auth, no DB)
- `npm run test:forms` — form UI only
- `npm run test:admin` — admin + device management (needs INTERNAL_USER/INTERNAL_PASSWORD)
- `npm run test:ios` — iOS lifecycle headed (needs PLAYWRIGHT_TEST_IOS_IMEI)
- `npm run test:android` — Android lifecycle headed (needs PLAYWRIGHT_TEST_ANDROID_IMEI)

**How to apply:** When adding new frontend pages or backend edge functions, add corresponding specs in `tests/e2e/`. Use `helpers/db.ts` for DB validation and `helpers/mosyle.ts` / `helpers/hexnode.ts` for MDM validation.

**Key env var:** `PLAYWRIGHT_TEST_IOS_IMEI` — set this in `.env.test.local` to enable submission + lifecycle tests.

**TEST BACKLOG (2026-07-14):** `tests/e2e/MOSYLE-TEST-BACKLOG.md` — prioritized new tests from the slot-CDA cutover + install-model findings (free-app add e2e with slot/CDA/no-window assertions, paid-app window + stored-price re-add, read-back honesty/tombstone canary, removal flow, duplicate-assignment regression, badge states, audit reconciliation, Step-3 self-heal, manual device cards). Golden rule baked in: never trust Mosyle success responses, only read-back; mosyle_status='added' now means grid-verified. Also updated 2026-07-13: helpers/mosyle.ts login was FIXED (was broken Basic-auth flow); mosyle-integrity + mosyle-cda-roundtrip converted to slot model (custom_AS from ios_device_apps.slot).

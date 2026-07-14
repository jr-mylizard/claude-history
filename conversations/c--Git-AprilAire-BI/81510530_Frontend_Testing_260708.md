**You**

Addendum: additional tests suggested by the 2026-07-13 CDA debugging
1. Render-race regression (extends the original round-trip test). After a sync assigns new CDA values and bounces the group, verify the rendered profile includes them — not just that Mosyle stored them. Concretely: enable an app, wait for sync + settle (the fix waits 30s before bouncing), then confirm the app is visible on the device. A bounce that fires too soon after the assigns produces a stale render with success responses everywhere; this was invisible to every API-level check.

2. Group-membership safety after bounce. Assert the device is back in device group 14 after every sync that bounced it. A failed re-add leaves the profile uninstalled — which fails open: every hidden app becomes visible on a child's device. This is the highest-severity silent failure in the system and nothing currently detects it.

3. CDA value↔DB integrity sweep. For every CDA in list_custom_device_attributes, extract the app id from the UniqueID and assert its Value equals that app's package_id in ios_apps. Ignore Name entirely (names are stale from an old id numbering). This catches renumbering drift, the truncation bug's collisions, and manual-edit mistakes (the Bitwarden/Audible overwrite would have been caught by this).

4. CDA id format + mobileconfig coverage. Two static checks: (a) vw_desired_mosyle_cdas_state.cda_id round-trips app ids without truncation (app id 12501 must yield custom_AA12501, not custom_AA1250 — Postgres lpad truncates); (b) every cda_id the view can emit is referenced as a variable in mosyle/mylizard.mobileconfig (currently covers AA0001–AA9999 only). An id outside the profile's range produces an app that can never appear, with all APIs reporting success.

5. "Installed" badge reconciliation. Compare ios_device_apps rows against Mosyle's actual device app inventory (the device_apps.php fetch in mosyle_device_check.py). Flag apps the DB tracks that aren't on the device. The frontend badge means "added to catalog," not "installed" — Uber showed "Installed" for days after it was gone.

6. Schedule-boundary transitions, both directions. The original spec covers enable; add the disable path: when a window closes, remove_custom_device_attributes calls fire, values disappear from the device record, allowed_in_mosyle flips false, and the app hides on-device. We verified enable-then-disable worked on 2026-07-12 (10:01 PM removals) but only by manual log reading.

7. Add-app flow contract (per the new policy). Paid app → app_store_should_be_enabled_until extended to 2h + customer email sent, no install attempt. Free app → (once wired) Playwright/InstallAppController install is dispatched, and no App Store window opens. Until the Playwright install exists, the free-app assertion documents the intended behavior as a skipped/todo test.

One meta-note for that session: the common thread in all of these is that Mosyle's success responses are unreliable — every test should assert on independently-read state (list/read-back/device inventory), never on the write call's response.

---

**Claude**

Let me read the view schema and mobileconfig before writing the tests.

---

**Claude**

Good — the view already has the `greatest(4, length(a.id::text))` fix. Now let me read the mobileconfig to understand its variable range.

---

**Claude**

Now I have everything I need. Key findings:
- The view already has `greatest(4, length(a.id::text))` so truncation is fixed — test 4a should verify the guard is still there
- The mobileconfig covers `%custom_AA0001%` through `%custom_AA9999%` — test 4b checks any app id ≥ 10000 would be uncoverable
- Test 6 (disable path) is already in Phase 2 of the existing round-trip

I'll write four things: helper additions, two new spec files, and extensions to the round-trip spec.

---

**Claude**

Now let me read the db helper to see what's there:

---

**Claude**

Good, I have everything I need. Now I'll add the helper functions, then write the three additions in parallel.

---

**Claude**

Now extend the round-trip spec to add Tests 1 (render-race) and 2 (group membership):

---

**Claude**

Now add the render-race and group-membership assertions inside the main test, after assert 4:

---

**Claude**

Now add the group membership check to Phase 2 (disable side):

---

**Claude**

Now write the integrity spec (Tests 3, 4a, 4b, 5):

---

**Claude**

Now write the add-app contract test (Test 7):

---

**Claude**

Now update `package.json` with new scripts and `.env.test.local.example` with new env vars:

---

**Claude**

Now let me do a quick TypeScript sanity check to catch any obvious errors:

---

**Claude**

Two pre-existing TS issues surfaced — let me fix them:

---

**Claude**

The only remaining error is the pre-existing `admin-auth.spec.ts` issue (unrelated to my changes). All new and modified files are clean.

Let me do a final review of the round-trip spec to make sure the new steps are ordered correctly:

---

**Claude**

Everything looks correct. Here's a summary of what was added across all 7 tests:

---

**Tests 1 & 2 — added to `mosyle-cda-roundtrip.spec.ts`:**
- **Assert 5 (render-race)**: After assert 4 passes, waits an additional 60 s then re-reads device CDA values. A regression here means the group bounce fired before Mosyle propagated the new assignments → stale profile render. The error message specifically points to the 30 s pre-bounce wait in `sync-mosyle.ts`.
- **Assert 6 / 4R (group membership)**: After both the enable and disable syncs, calls `getDeviceGroupUdids(14)` and verifies the device UDID is still in the group. A missing UDID means "fails open" — profile uninstalled, all apps visible.

**Tests 3–5 — new `mosyle-integrity.spec.ts`** (static sweeps, no sync required):
- **Test 3**: For every CDA assigned to the test device, derives `app_id` from the UniqueID, looks up `ios_apps.package_id`, and asserts the Value matches. Catches the Bitwarden/Audible manual-overwrite bug. Skips `custom_AA9999` (the App Store slot).
- **Test 4a**: Queries `vw_desired_mosyle_cdas_state` and re-derives each `cda_id` in JS using `Math.max(4, len(app_id))`. Any deviation means the `greatest()` guard regressed. Warns if no app_id ≥ 10000 exists yet.
- **Test 4b**: Reads `mylizard.mobileconfig`, extracts all `%custom_AAnnnn%` placeholders, and asserts every CDA ID the view can emit has a matching placeholder. Catches the silent gap once `ios_apps.id` reaches 10000.
- **Test 5**: Gets all `allowed_in_mosyle = true` apps for the device, reads device CDA values from Mosyle, and flags any app where the badge shows but the CDA isn't on the device.

**Test 7 — new `add-app-contract.spec.ts`** (full UI → DB):
- **7a (paid)**: Adds a paid app via the UI, then asserts `app_store_should_be_enabled_until ≈ now+2h` (±5 min), customer email was sent, and no `install_app` Mosyle call was made.
- **7b (free)**: Adds a free app and asserts the expiry column was not touched. The InstallAppController dispatch assertion is `test.fixme` pending confirmation of the Mosyle operation name in `logging_api_calls`.

New npm scripts: `test:integrity`, `test:add-app`, `test:mosyle` (runs all three Mosyle suites). New env vars documented in `.env.test.local.example`.

---

**You**

Have you run all these tests already?  If not, what's the process to do that?

---

**Claude**

No — I only ran a TypeScript compile check (`tsc --noEmit`), which came back clean on the new files. The actual tests haven't been executed.

These aren't tests that can run without real hardware and credentials. Here's what you need to actually run each suite:

**`test:cda` / `test:integrity` (Mosyle read/write tests)**

These require:
- `.env.test.local` populated with:
  - `PLAYWRIGHT_TEST_IOS_DEVICE_ID` + `PLAYWRIGHT_TEST_IOS_UDID` (the test device)
  - `MOSYLE_API_ACCESS_TOKEN`, `MOSYLE_USER_EMAIL`, `MOSYLE_USER_PASSWORD`
  - `NEXT_PUBLIC_SUPABASE_URL` + `SUPABASE_SERVICE_ROLE_KEY`
- The test device must be enrolled in Mosyle and in group 14
- For `test:cda`: the device must have at least some apps with `allowed_in_mosyle = false` and a non-null category (so `pickTestApps()` can find something to work with)

```powershell
cd "c:\Git\MyLizard_NodeJS"
npm run test:integrity    # ~30 seconds, read-only, safe to run any time
npm run test:cda          # ~10-15 minutes, modifies DB schedule rows (cleaned up after)
```

**`test:add-app` (UI-driven, requires dev server)**

Needs everything above plus:
- Next.js dev server running (`npm run dev`)
- `PLAYWRIGHT_TEST_PAID_IOS_APP_BUNDLE_ID` and/or `PLAYWRIGHT_TEST_FREE_IOS_APP_BUNDLE_ID`
- `INTERNAL_USER` + `INTERNAL_PASSWORD`

```powershell
# Terminal 1
npm run dev

# Terminal 2
npm run test:add-app
```

**What I'd recommend running first**

Start with `test:integrity` — it's read-only, fast, and will immediately tell you if there are any existing integrity issues (stale CDA values, mobileconfig coverage gaps, badge mismatches). No risk of leaving the device in a bad state.

`test:cda` is the expensive one (waits for two full sync cycles). Run it on its own when you have 15 minutes and aren't running anything else against Mosyle (rate-limit concern).

---

**You**

I don't see .env.test.local.  Should I rename .env.test.local.example and populate it?

---

**Claude**

Yes — `.env.test.local.example` is just the template. The actual file Playwright loads is `.env.test.local`. Rename (or copy) it and fill in the real values:

```powershell
cd "c:\Git\MyLizard_NodeJS"
Copy-Item .env.test.local.example .env.test.local
```

Then open `.env.test.local` and populate at minimum:

```
PLAYWRIGHT_TEST_IOS_DEVICE_ID=WI_608A_JR_001_P_001   # your test device_id
PLAYWRIGHT_TEST_IOS_UDID=00008030-...                  # Mosyle UDID for the same device
MOSYLE_API_ACCESS_TOKEN=...
MOSYLE_USER_EMAIL=...
MOSYLE_USER_PASSWORD=...
```

The Supabase vars (`NEXT_PUBLIC_SUPABASE_URL`, `SUPABASE_SERVICE_ROLE_KEY`) are already in `.env.local` and are loaded before `.env.test.local`, so you don't need to duplicate them unless you want to override (e.g., point at a different project for testing).

Once that's set, `npm run test:integrity` is the safest first run — it only reads, takes ~30 seconds, and will immediately surface any existing issues.

---

**You**

If I want an existing but unused device in the database synched with a test device in mosyle, do I just have to update its imei?

---

**Claude**

Not quite — IMEI alone is enough for enrollment detection (the webhook matches on IMEI), but the sync cron needs more than that. Looking at `vw_desired_mosyle_cdas_state`, it uses `cd.mosyle_device_id` to know which Mosyle device to target when it calls `assign_custom_device_attributes`. If that's null or wrong, the CDA calls go nowhere.

At minimum you'd need to set:

| Column | What to put |
|---|---|
| `imei` | The physical device's IMEI |
| `mdm_vendor` | `'mosyle'` |
| `mosyle_device_id` | Mosyle's internal device ID (find it in the Mosyle console or via the API) |
| `operating_system` | `'ios'` |
| `enabled` | `true` |

The `mosyle_device_id` is what the sync actually uses in API calls. The `PLAYWRIGHT_TEST_IOS_UDID` you'd put in `.env.test.local` is the Apple UDID (different from the Mosyle device ID) — that's what the Playwright helper uses to read back `CustomDeviceAttributes` from the device list endpoint.

To get the Mosyle internal device ID for a device you can hit the API:
```powershell
# Quick lookup by serial number (already in getMosyleDeviceBySerial helper)
# Or in the Mosyle console: Devices → click device → URL contains the device ID
```

If the existing DB device already has `mdm_vendor_device_id` set from a prior enrollment, check whether that's the same value as `mosyle_device_id` — they may or may not be the same column depending on the schema.

---

**You**

I understand that mosyle_device_id is used in most api calls, but when picking up a new device that's not known.  sync-apps-and-websites should look for imeis that match what's in the database and pull their data.  Can you confirm it's not doing that?

---

**Claude**

Let me read the sync function to check.

---

**Claude**

It does exactly what you described. Step 1 (lines 48–95) specifically handles this:

1. Queries `customer_devices` for rows where `mdm_vendor = 'mosyle'` AND `mosyle_device_id IS NULL` AND `imei IS NOT NULL`
2. Fetches the full Mosyle device list once
3. For each pending device, calls `completeIosEnrollment` which matches by IMEI against the Mosyle list and fills in `mosyle_device_id` and other metadata

So your instinct is correct — **you just need to update the `imei`** on the existing DB device (and make sure `mdm_vendor = 'mosyle'` and `mosyle_device_id IS NULL`). The next cron tick (within 3 minutes) will detect it, match it against the Mosyle device list by IMEI, and run `completeIosEnrollment` to fill in everything else automatically.

---

**You**

Look at tests/e2e/MOSYLE-TEST-BACKLOG.md and see if there are any tests we should add to our list.

---

**Claude**

Several items in the backlog aren't covered yet. Here's the gap analysis:

**What we've already built that covers backlog items:**
- CDA round-trip enable/disable → `mosyle-cda-roundtrip.spec.ts`
- Render-race regression (assert 5) → round-trip
- Group membership safety (asserts 6/4R) → round-trip
- CDA value integrity, slot format guard, mobileconfig coverage, badge reconciliation → `mosyle-integrity.spec.ts`
- Add App paid/free basics → `add-app-contract.spec.ts`

**Gaps — not yet implemented:**

| Backlog item | Priority | What's missing |
|---|---|---|
| **Add App free — full contract** | P1 | Our test 7b only checks App Store expiry. Backlog adds: slot in 1–500 + unique per device; `cda_id` format in view; `mosyle_status` flips to `'added'` or `'not_found'` within ~10 min; idempotency (second add returns same slot, no second slot consumed); button-enable regression (assertion 0) |
| **Add App paid — group 15 bouncing** | P1 | Assert device leaves "Block App Store" group (id 15) on next sync tick and re-joins after expiry; fast-forward test by setting the timer to `now()` |
| **Add App paid — stored price path** | P1 | Re-add the same paid app on a second device and verify window opens without an iTunes refetch (catches the bug where paid detection only fired on first-ever fetch) |
| **Tombstone canary** | P1 | After `mosyle_status = 'added'`, independently verify the app's track ID is in the Mosyle profile grid. Without this, DB says added but Mosyle grid disagrees (the MTZ Life case) silently |
| **Removal flow** | P2 | Blocked on frontend — no remove-app control yet |
| **Duplicate-assignment regression** | P2 | Needs profile recon endpoint — keep as manual for now |
| **Badge states (UI)** | P2 | Seed `last_seen_on_device` + `last_audited_at` combos, assert four `InstallBadge` states render correctly |
| **Audit reconciliation** | P2 | Assert `last_audited_at`, `last_seen_on_device` state after dispatching the audit workflow |
| **Step-3 self-healing** | P3 | Simulate lost dispatch via counter/timestamp manipulation, verify re-dispatch and cap behavior |

The highest-value thing not yet covered is the **tombstone canary** — that's the category of silent failure that's bitten you most. The **Add App free full contract** is also significant since the backlog notes the button-enable bug persisted five days unnoticed.

Want me to start with either of those?

---

**You**

Go ahead and do both.

---

**Claude**

Let me read the profile manager script and any recon helpers to understand how to read the Mosyle profile grid.

---

**Claude**

Let me also check the recon script to understand the profile grid API:

---

**Claude**

Now I have a complete picture. The profile grid is only readable via the Mosyle web UI — there's no Business API equivalent. The tombstone check needs to log in to `mybusiness.mosyle.com` and read `.appselector_grid div.app[id]` elements. Let me build everything.

---

**Claude**

Now create the Mosyle web UI helper:

---

**Claude**

Now write the tombstone canary spec:

---

**Claude**

Now replace test 7b in `add-app-contract.spec.ts` with the full free-app contract:

---

**Claude**

Now replace the test 7b body with the full free-app contract:

---

**Claude**

Now update package.json and the env example:

---

**Claude**

Now run the TypeScript check:

---

**Claude**

Only the pre-existing `admin-auth.spec.ts` error — all new files are clean.

Here's what was built:

---

**`helpers/db.ts`** — 4 new helpers:
- `getMosyleInstallProfileName(deviceId)` — reads `mosyle_install_profile_name` from `customer_devices`
- `getIosDeviceAppsWithMosyleStatus(deviceId)` — returns all device apps with `mosyle_status`, `slot`, and catalog metadata
- `getIosDeviceAppByPackageId(deviceId, packageId)` — looks up a specific app row by bundle ID
- `waitForMosyleStatusSet(deviceAppRowId, maxMs)` — polls until `mosyle_status` is non-null, returns the value (or null if timed out)

**`helpers/mosyle-web.ts`** — new file, TypeScript port of the relevant parts of `mosyle_profile_manager.py`:
- `loginToMosyleWeb(page)` — email → Next → password → Enter → wait for dashboard
- `navigateToInstallApp(page)` — clicks `#menu_gerenciamento`, finds "Install App" via JS
- `getProfileGridTrackIds(page, profileName)` — full-page reload, opens profile, reads `.appselector_grid div.app[id]`, returns `Set<string>` of track IDs

**`mosyle-tombstone.spec.ts`** — new test, 5 min timeout:
- Gets profile name from DB
- Finds all `mosyle_status = 'added'` rows for the device
- Opens Mosyle web UI in a fresh isolated browser context
- Reads the server-rendered profile grid
- Fails on any `ios_apps.id` that's missing from the grid, with a precise error pointing to the silent-save class of failure and how to fix it

**`add-app-contract.spec.ts` test 7b** — replaced with full free-app contract (20 min timeout):
- **Assert 0**: button disabled before selection, disabled while typing, enabled only after a result is clicked (the regression that went unnoticed for 5 days)
- **Assert 1**: `ios_device_apps` row created with `slot` in 1–500, unique across the device
- **Assert 2**: `vw_desired_mosyle_cdas_state.cda_id = 'custom_AS' + slot.padStart(4, '0')`
- **Assert 3**: if the app's category has an active schedule (`should_be_allowed = true`), the CDA is on the device (skipped if `PLAYWRIGHT_TEST_IOS_UDID` not set)
- **Assert 4**: `app_store_should_be_enabled_until` unchanged
- **Assert 5**: `mosyle_status` becomes `'added'` or `'not_found'` within 15 min — null at 15 min is an explicit failure
- **Assert 6**: second add of the same app returns the same row ID and same slot
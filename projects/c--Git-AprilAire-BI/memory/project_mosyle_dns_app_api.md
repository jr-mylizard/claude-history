---
name: project-mosyle-dns-app-api
description: "Mosyle DNS profile builder and app install API — architecture, files, pending wiring to device_management page"
metadata: 
  node_type: memory
  type: project
  originSessionId: 897c1e31-3958-4b52-a181-087ed7e7e3ac
---

## What was built (sessions ending 2026-07-07)

**Files:**
- `c:\Git\MyLizard-Supabase\scripts\mosyle\mosyle_dns_manager.py` — Playwright-based Mosyle DNS API wrapper. Key functions: `login`, `open_dns_section`, `open_profile_settings`, `action_duplicate`, `set_filtering`, `set_security`, `create_custom_filter`, `delete_custom_filter`, `list_custom_filters`, `check_url`, `hard_refresh`, `mosyle_post`.
- `c:\Git\MyLizard-Supabase\scripts\mosyle\mosyle_batch_builder.py` — Builds all 33 DNS profiles idempotently. Run once; already ran live (Created: 33, Skipped: 0, Failed: 0).
- `c:\Git\MyLizard-Supabase\supabase\migrations\20260706000001_dns_category_map.sql` — Reference table `dns_category_map` with CleanBrowsing (Android) and Mosyle (iOS) category → MyLizard grouping mappings. **Not yet applied to dev branch.**
- `c:\Git\MyLizard-Supabase\supabase\schemas\10 Tables\102 Device\10217 dns_category_map.sql` — Schema file for above.

**33 Mosyle DNS profiles now exist in Mosyle Business.** Named "ML Open", "ML G", "ML M", "ML Mu", ..., "ML M-G-V-S-A". Profile IDs: ~75–235. 5 account-level custom filters created: Always Blocked Baseline, Music, Messaging, AI, Video Detail.

**Mosyle internal API discovered:**
- Endpoint: `POST https://mybusiness.mosyle.com/Controller/mapping.php`
- Auth: Two session cookies (`credentials_business` JWT + session ID). HttpOnly, session-scoped.
- CSRF: `NotesToken` required in POST body for ALL write operations. Extracted from page HTML per form load. In Playwright this is free: `page.evaluate("() => $('input[name=\"NotesToken\"]').val()")`. In raw HTTP (Postman) it requires a page load first.
- `mapping=MosyleDNSController`: list_profiles, save_settings, save_dns_customfilter, delete_profile, action_duplicate, dns_check_url
- `mapping=InstallAppController`, `operation=save_appinstall_school`: create/update app install profiles. Full payload structure understood (see conversation 2026-07-07). `idprofile=0` = create, existing ID = update.

---

## What the next session needs to wire up (device_management page → Playwright)

### 1. Add `mosyle_profile_name` to `dns_profiles`
The `dns_profiles` table already has 6 boolean columns (music_allowed, messaging_allowed, gaming_allowed, video_questionable_search_allowed, social_media_allowed, artificial_intelligence_allowed) and `hexnode_policy_id` for Android. Add a `mosyle_profile_name varchar` column and populate all 33 rows with the correct "ML ..." names. The mapping is deterministic from the 6 booleans — use the same logic as `profile_name()` in `mosyle_batch_builder.py`.

### 2. Create `vw_desired_dns_state_mosyle` (or extend existing view)
`vw_desired_dns_state` currently has `WHERE cd.mdm_vendor != 'mosyle'` — Mosyle devices are fully excluded. The Mosyle variant of the view should:
- Filter `WHERE cd.mdm_vendor = 'mosyle'`
- Join `dns_profiles` on the 6 booleans (same as existing view)
- Output `device_id`, `mosyle_profile_name` (desired), current assignment, desired_action

### 3. Check `dns_website_categories` vendor scope
The current view cross-joins `dns_website_categories` without a vendor filter. Check whether existing rows have `dns_filter_vendor = 'CleanBrowsing'` on all of them or are vendor-agnostic. If vendor-specific, the Mosyle view will need to filter or add 6 matching Mosyle rows. Either way it's still just the 6 parent-facing category names — no new categories.

### 4. Resolve per-device vs per-group for Mosyle DNS assignment
**This is an open architectural question.** The existing data model is per-device (`dns_device_categories.device_id`). Mosyle DNS profile assignment is per-device-group (or possibly per-device — not confirmed). Options:
- Each device has its own Mosyle device group (1:1) → per-device model works
- Groups are shared → need a group-to-devices mapping layer
Ask on the Mosyle call: does DNS filtering support per-device assignment, or only per-group?

### 5. Playwright sync job
Once the view exists, a Playwright script reads it, iterates devices/groups with `desired_action = 'switch'`, and calls `save_settings` on the correct Mosyle DNS profile for each. This is the same pattern as the Android Hexnode sync.

### 6. Webpage → data flow
The parent-facing webpage (device_management) writes to `dns_device_categories` — the same table Android uses. No new UI data model needed. The 6 toggle states → `dns_device_categories` rows → view computes desired Mosyle profile → Playwright sync applies it.

---

## Pending cleanup (do before beta)
- Delete "Test" app install profile in Mosyle (created during API exploration)
- Delete any "ZZ Test Conflict" DNS profiles if they still exist
- Apply `20260706000001_dns_category_map.sql` migration to dev branch
- Verify CleanBrowsing spelling: "Extremism & Hate" (flagged in migration as unverified)

## Why: motivation
**How to apply:** MyLizard parents will control their child's internet access via 6 category toggles on the device_management page. The 6 toggles deterministically select 1 of 33 Mosyle DNS filtering profiles already built in Mosyle Business. The sync job is the missing link.

---
name: project-ios-enrollment
description: iOS pre-enrollment registration page and edge function — current state and pending work
metadata: 
  node_type: memory
  type: project
  originSessionId: e689312d-cd2d-4759-97ad-866a944081e1
---

iOS pre-enrollment is complete and deployed on the `Test_Unified_iOS_Android_Structure` Supabase branch (`papyhxdkbadzqckuqmhb`).

## What's built

**Registration page**: `mylizard.net/register` (Next.js, `c:\Git\MyLizard_NodeJS\src\app\register\page.tsx`)
- Accepts `?imei=` and `?nickname=` query params to prepopulate fields
- Uses `RegistrationForm` component (`src\components\internal\RegistrationForm.tsx`)
- After successful iOS submission: button changes to orange "Registered", shows amber note for unresolved apps
- Old path `/internal/user-registration-ios` redirects to `/register`

**Edge function**: `pre-enrollment-device-setup` (iOS path in `supabase/functions/pre-enrollment-device-setup/pre-enrollment-ios.ts`)
- Two enrollment paths: Mac path (staged record with UDID/serial/bundle IDs from `ideviceinstaller`) and Web path (display names from screenshots resolved to bundle IDs via SearchAPI)
- Sends enrollment summary email via Resend to customer's email address after registration
- Email built by `_shared/util/buildAppListEmail.ts` (template inlined as string — do NOT use Deno.readTextFile for edge function templates)
- Unresolved app names included in amber note block in email
- Bundle IDs without a dot (e.g. `CFBundleIdentifier` plist keys) are filtered out on Mac path

**Email template** (`buildAppListEmail`):
- iOS intro: "Here are the list of apps that you had previously installed on your device. You will need to re-install them yourself — the App Store will be unblocked for 24 hours to allow you to do so."
- Unresolved apps copy: "...have been forwarded to MyLizard support and they will investigate whether there's a way to add them and follow up with you:"

## Pending / known issues
- `timezone` field not yet added to iOS form — wait for `Test_Unified_iOS_Android_Structure` to merge to dev first
- MyLizard_NodeJS changes (RegistrationForm, /register page) are on a branch — need to be merged
- `staged_device_enrollments` and `dns_device_categories` tables not yet in `db-types.ts` — using `// @ts-ignore` workaround

## Deployment notes
- Deploy edge functions via CLI: `supabase functions deploy <name> --project-ref papyhxdkbadzqckuqmhb --import-map supabase/functions/deno.json --no-verify-jwt`
- Docker not required for remote deploys
- Always ask before deploying ([[feedback-deploy-approval]])

## Post-enrollment (complete)

**Edge function**: `post-enrollment-device-setup` (iOS path in `supabase/functions/post-enrollment-device-setup/post-enrollment-ios.ts`)
- Scans all pending iOS enrollments (`mosyle_device_id IS NULL`), fetches Mosyle device list once, calls `completeIosEnrollment` for each match
- Shared utility: `_shared/util/complete-ios-enrollment.ts` — matches by IMEI (checks `imei`, `imeiOne`, `imeiTwo`), updates `customer_devices`, sends enrollment summary email
- Email built by `_shared/email_templates/buildIosEnrollmentEmail.ts` — internal checklist sent to jr@mylizard.net (6 checks: customer, device, schedule, DNS categories, iOS apps, Mosyle enrollment)
- Duplicate-email guard: DB update uses `WHERE mosyle_device_id IS NULL` + checks returned rows — only the first writer sends the email
- `testEmailDeviceId` param skips Mosyle and sends email for an already-enrolled device (for preview/testing)
- `Content-Type: application/json` required on Mosyle API calls — added to `_shared/apis/mosyleApi/index.ts`

**Cron**: `sync-apps-and-websites-ios` runs every 3 min (`1-59/3`), handles enrollment sweep (Step 1) + CDA sync (Step 2). Staggered with Android cron (`*/3`). Created manually in DBeaver — never in migrations.

## Next up
- Android User Registration page (migrating from WeWeb)

---
name: project-rls-narrowing
description: Anon RLS narrowing + edge function gating shipped to dev branches 2026-07-03; deploy steps pending; prod waits for WeWeb retirement
metadata: 
  node_type: memory
  type: project
  originSessionId: 32d1d047-b0d4-48b7-8b0d-b0602baf1dbb
---

Architecture-review finding #2 implemented 2026-07-03 on the `dev` branches of MyLizard-Supabase and MyLizard_NodeJS (code only — nothing deployed, per repo rules).

Key facts:
- Migration `20260703200000_narrow_anon_rls`: drops all `anon_all`/`authenticated_all` policies, enables RLS on the three scheduling tables, revokes anon on all views except `vw_stripe_products`/`vw_featured_listings`, revokes anon EXECUTE on public functions, flips default privileges, drops the anon storage-upload policy.
- 21 edge functions gated via `_shared/util/require-internal-auth.ts`. Accepts BOTH key generations on either header: new `sb_secret_...` keys (validated against the `SUPABASE_SECRET_KEYS` JSON-dict env var, checked on `apikey`) and the legacy service_role JWT (`SUPABASE_SERVICE_ROLE_KEY`, checked on `Authorization`; legacy keys work until end of 2026). Jeremiah's projects use the new publishable/secret keys. Public by design: `unsubscribe`, `resend-webhook`, `sms-webhook` (the last verifies nothing inbound — open follow-up).
- Next.js: all DB writes/internal reads moved to server actions using `src/lib/supabase/admin.ts` (service key, server-only); `/internal/*` gated by HTTP Basic auth in `src/proxy.ts` (Next 16 renamed middleware→proxy; Node runtime).
- Deploy prerequisites per environment: rotate vault secret `supabase_bearer_token` to the service_role key (it held the publishable key — pg_cron and `.ml_call_edge_function` use it); set `SUPABASE_SERVICE_ROLE_KEY` + `INTERNAL_USER`/`INTERNAL_PASSWORD` in Vercel; local `supabase/functions/.env` needs the local *secret* key (template previously said publishable); Mac app `config.env` needs `SUPABASE_SERVICE_KEY`.
- Production applies only when dev merges to main — at that point WeWeb (points at prod with the anon key for enrollment + schedule updates) must be retired and the prod anon key rotated (it is baked into the WeWeb repo export).

**Why:** anon key previously had blanket CRUD on 38 tables, full access to 3 RLS-disabled scheduling tables, SELECT on every view (incl. vw_purchases revenue data), and could invoke all 22 edge functions.

**How to apply:** New tables/views/functions follow the posture documented in MyLizard-Supabase CLAUDE.md ("Anon access posture" section). Related: [[project-ios-enrollment]], [[feedback-supabase-mcp-over-docker]].

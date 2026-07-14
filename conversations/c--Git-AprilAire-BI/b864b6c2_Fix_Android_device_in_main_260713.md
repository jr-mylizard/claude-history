# Fix Android device in main 260713

_Project: c--Git-AprilAire-BI_

---

**You**

Fix Android Device in Main 260713

---

**You**

[Request interrupted by user]

---

**You**

The android device_id WI_608A_JR_001_N_001 should have a lot of apps available per the frontend schedule page.  But in the database and on the device itself there are only a few available.  Here's the database query: 

select *
from android_apps a
join android_device_apps b
	on a.s_product_id = b.app_package_id
	where b.device_id = 'WI_608A_JR_001_N_001'
	and a.is_in_play_store = true
	and b.active_in_hexnode = true

Why would this be?  Shouldn't it update each time there's a discrepancy?

---

**Claude**

I'll look into how `android_device_apps` gets populated/synced to understand why this device isn't getting the full set of available apps.

---

**Claude**

This looks like it hinges on how the sync actually works, not a broken trigger. Let me check the live data to confirm which mechanism is at play — I'll query the actual DB state for this device.

---

**Claude**

Found it. Let me confirm by looking at how `vw_android_active_schedules` determines "active," since the "Always" schedule has zero time blocks.

---

**Claude**

Found it. Let me confirm by looking at how `vw_android_active_schedules` determines "active," since the "Always" schedule has zero time blocks.

---

**Claude**

This is the root cause. Let me check whether this is a one-off data gap for this device or a systemic issue with how "Always" schedules get created.

---

**Claude**

Found it — this isn't a sync bug at all. The sync pipeline is working exactly as designed; there's just a data gap.

**Root cause:** Android app enablement is driven by `vw_desired_hexnode_app_groups_state`, which only turns an app on if its category belongs to a schedule that is *currently active right now* (per [vw_android_active_schedules.sql](c:\Git\MyLizard-Supabase\supabase\schemas\20 Views\202 Device\20211 vw_android_active_schedules.sql)). "Active" is determined strictly by matching `now()` against rows in `device_schedule_blocks` — a schedule with **zero blocks can never be active**, no matter its name.

This device's "Always" schedule (id 12) has **0 blocks** in `device_schedule_blocks`, even though it's assigned 14 categories (Maps, Photography, Productivity, Tools, Business, Finance, Shopping, Sports, Weather, etc.) via `device_category_schedules`. Since it never registers as active, none of those categories' apps ever get enabled — only whatever's active from "Free Time" or "Work" (currently just 8 Play Store apps, matching what you're seeing).

I confirmed the sync itself is healthy:
- `vw_desired_hexnode_app_groups_state.need_to_sync` is `false` for every row on this device — the DB already matches what the (flawed) desired-state calculation says it should be.
- `logging_sync_apps` shows the cron running successfully every ~12 hours, toggling 8 apps on/off as "Free Time"/"Work" start and end.
- So "shouldn't it update on discrepancy?" — yes, and it is. There's no discrepancy from the system's point of view; "Always" just isn't actually always.

This is a **pre-existing data gap on dev**, not something introduced by today's prod migration — the migration script even flagged it in a comment ("Sabbath and Always have no blocks... not yet configured for this device"), and it faithfully carried the gap to prod.

It's not isolated to this device either — I checked all 4 devices with an "Always" schedule: **`WI_608A_JR_001_N_001` and `CA_415A_TD_002_N_001_dev` both have 0 blocks (broken)**; `IL_555A_TU_001_P_001` and `NY_608A_PT_001_N_001` are correctly configured with 7 blocks (one per day, `00:00:00`–`23:59:59`).

**Fix:** insert 7 full-day blocks (Sun–Sat, `00:00:00`–`23:59:59`) into `device_schedule_blocks` for schedule id 12 (this device's "Always"), matching the working pattern. Want me to do that now on dev (and separately for `CA_415A_TD_002_N_001_dev`), or just this device for now? Once inserted, the next sync-apps-and-websites cron run will pick it up and start enabling the "Always"-category apps — worth knowing before I write it, since it'll trigger real Hexnode app-group changes on the next run.

---

**You**

What about WI_608A_TN_001_N_001?

---

**Claude**

This device doesn't exist on dev at all — no `customer_devices` row, no schedules, no apps. Let me check prod.

---

**Claude**

Found it — this device only lives on **prod** (`gmovbfjefbxoiruomgum`), not dev. Let's run the same diagnostics there.

---

**Claude**

This one is different — it's a healthy device with a *different* problem schedule.

**This device (`WI_608A_TN_001_N_001`) lives only on prod**, not dev. Its "Always" schedule is correctly configured (7 blocks). But:

- **69 of 81 Play Store apps are active_in_hexnode=true** — that's actually a lot already (300/323 rows in the desired-state view are `should_be_enabled=true / active_in_hexnode=true`), and `need_to_sync` is `false` everywhere — the sync is fully caught up.
- The **"Awake Time" schedule has 0 blocks**, same broken pattern as before. It governs Education, Internet Use, Music and Audio, Photography — but those same 4 categories are *also* assigned to every other schedule (Mid-Day, Morning/Evening Weekdays, Weekend Morning/Normal Weekend), so "Awake Time" having no blocks is largely redundant — those categories are still covered by the time-boxed schedules. So this one is low-impact, but still a config gap worth fixing for consistency.
- The sync log shows active churn every few hours (7, 8, 9, 17 apps added/removed) — that's the normal Mid-Day/Morning/Evening/Weekend schedule blocks kicking in/out, working as intended.

So: if you're seeing *this* device show fewer apps than expected, it's likely not a blocked schedule — 69/81 is a healthy sync rate. It's more likely just that whatever categories the "missing" apps belong to (the other 12 Play Store apps not currently `active_in_hexnode`) simply aren't assigned to *any* currently-active schedule right now — worth checking against the specific package IDs you expected to see. Want me to pull the 12 that are off and show which category/schedule (if any) they belong to?

**Summary so far — two separate "Always"/catch-all schedules with zero blocks:**
| Device | Branch | Schedule | Categories affected | Impact |
|---|---|---|---|---|
| `WI_608A_JR_001_N_001` | dev (+ now prod via migration) | Always | 14 categories, only source | High — those apps never enable |
| `CA_415A_TD_002_N_001_dev` | dev | Always | unchecked | Likely same high impact |
| `WI_608A_TN_001_N_001` | prod | Awake Time | 4 categories, all duplicated elsewhere | Low — redundant with other schedules |

Want me to insert the missing full-week blocks for any of these? I'd do it the same way as the working schedules — 7 rows, `00:00:00`–`23:59:59`, one per day.

---

**You**

We can ignore CA_415A_TD_002_N_001_dev for now.  Let's start with WI_608A_JR_001_N_001.  Does it truly have zero blocks, or are its blocks just not matched to its schedules?

---

**Claude**

Confirmed — it's genuinely empty, not a mismatch.

- There's exactly one `device_schedules` row named "Always" for this device: `id = 12`, correctly scoped to `device_id = 'WI_608A_JR_001_N_001'`.
- `device_schedule_blocks WHERE schedule_id = 12` returns **zero rows**, directly.
- The full join of all this device's schedules to their blocks shows only Free Time (7), Nighttime (12), Work (5) have any blocks — Always (id 12) and Sabbath (id 1) have none at all, not even one that got attached to the wrong schedule_id. Nothing is silently dropped by a join condition; the rows simply were never inserted.

So the fix is straightforward: insert 7 rows into `device_schedule_blocks` for `schedule_id = 12`, one per `day_of_week` (0–6), `00:00:00`–`23:59:59`, matching the pattern from `NY_608A_PT_001_N_001`.

Want me to run that insert on dev now?

---

**You**

Here's what the session which populated this device on main said in response: 

I checked directly and device_schedule_blocks is populated on prod — it matches dev exactly:

Schedule	Blocks
Work	5
Nighttime	12
Free Time	7
Sabbath	0
Always	0
24 total, which is exactly what the DO block was supposed to create (Sabbath and Always have zero blocks on dev too — they were defined with NULL event times there, so 0 is correct, not a bug).

device_category_schedules also has all 22 rows.

If you're seeing an empty result, it's likely one of:

Looking at device_calendar instead of device_schedule_blocks — device_calendar is a read-only compatibility view, and for Sabbath/Always (which have no blocks) it'll show up with null weekdays/times, which might look "empty" at a glance.
Querying the wrong project — dev (xxinwdxprvkvpmhdelnp) vs prod/main (gmovbfjefbxoiruomgum).
A stale dashboard view that hasn't refreshed.
Can you tell me exactly what query or screen shows it empty? Or should I paste the query I just ran so you can run it yourself against the same connection you're using?

---

**Claude**

Good catch — let me check what's actually live in the database, since the schema file I read might be stale relative to migrations.

---

**Claude**

Good — this settles it, and also corrects something in my earlier answer to you.

**Answer to your question:** Yes, it truly has zero blocks — not a join/matching artifact. I queried `device_schedule_blocks WHERE schedule_id = 12` directly (no view, no `device_calendar`) and got zero rows, on the correct project (`xxinwdxprvkvpmhdelnp` = dev). That matches what the other session reported.

**But the other session's conclusion — "0 is correct, not a bug" — is wrong**, and I now have better evidence why. There's a migration, [20260527010000_all_day_schedule_blocks.sql](c:\Git\MyLizard-Supabase\supabase\migrations\20260527010000_all_day_schedule_blocks.sql), whose comment literally says:

> "The WeWeb 'Always' schedule feature creates blocks with **both times set to midnight** (`00:00:00`–`00:00:00`) as an all-day sentinel."

I confirmed via `pg_get_viewdef` that this logic is live on dev right now: `vw_android_active_schedules` treats any block row with `start_time = end_time = '00:00:00'` as `is_all_day = true`, which makes that schedule perpetually active regardless of `now()`. So the *intended* representation of "Always" is at least **one row** with that midnight-midnight sentinel — not zero rows. Zero rows can never satisfy `is_all_day` (there's no row to evaluate), so the schedule stays permanently inactive. The comment in the migration script ("likely defined via a different mechanism or not yet configured for this device") was correct to flag this as suspicious — it's a genuine gap, not a documented design choice.

One correction to my earlier proposed fix: I'd previously suggested copying the 7-row `00:00:00`–`23:59:59` pattern from `NY_608A_PT_001_N_001`/`IL_555A_TU_001_P_001`. That pattern still works (it satisfies the literal time-window match every day), but it predates this migration — no device on dev currently uses the actual sentinel mechanism (I checked; zero rows anywhere have `start_time = end_time = '00:00:00'`). The sentinel is simpler and is what new "Always" schedules are supposed to look like: **a single row**, `day_of_week` doesn't even matter, `start_time = end_time = '00:00:00'`.

Want me to insert that single sentinel row for schedule id 12 on dev?

---

**You**

I want all the dev schedules converted over to main, not just "Always".

---

**Claude**

I'll treat this as: fix dev first, then diff dev vs. prod for this device's full schedule setup (schedules, blocks, category assignments) and reconcile prod to match. Let me pull both sides.

---

**Claude**

Good news — the diff shows dev and prod already match everywhere except this one gap: `device_schedules`, `device_schedule_blocks` (Free Time 7 / Nighttime 12 / Work 5 / Sabbath 0), and `device_category_schedules` (all 22 rows) are identical on both sides. The only real divergence-from-correct is the missing "Always" sentinel block — present as a gap on **both** dev and prod. So "converting dev to main" here means: insert the same fix in both places so they're both correct and stay in sync.

I'll leave "Sabbath" (0 blocks, both sides) alone — it has zero category assignments on either side, so it's inert either way; not part of what you asked about.

Inserting one all-day sentinel row (`day_of_week=0, start_time='00:00:00', end_time='00:00:00'`) into `device_schedule_blocks`: dev's Always (`schedule_id=12`) and prod's Always (`schedule_id=64`).

---

**Claude**

Both inserted. Let's verify "Always" now registers as active and see the scale of the resulting sync.

---

**Claude**

Confirmed on both sides:

- `vw_android_active_schedules` now reports **Always + Free Time** active for this device (previously just Free Time) — on dev and prod alike.
- `need_to_sync` now flags **60 apps on dev** and **56 apps on prod** that should flip to `active_in_hexnode = true` — these are the "Always"-category apps (Maps, Photography, Productivity, Tools, Finance, Shopping, Sports, Weather, etc.) that were stuck off.

This won't apply itself instantly — `sync-apps-and-websites` is the cron job that reads `need_to_sync` and actually pushes the app-group changes to Hexnode, then flips `active_in_hexnode`. Based on the `logging_sync_apps` history you saw earlier, it runs roughly every 1–3 hours, so it should pick this up on its next scheduled run rather than needing a manual trigger — let me know if you want me to check when that's likely to fire, or if you'd rather trigger a dry run first to preview exactly what it'll add.

---

**You**

Can you give me a query to run to verify the schedule blocks for this device?

---

**Claude**

Here's a query that shows every schedule for the device alongside its blocks (including the all-day sentinel):

```sql
select
  ds.name as schedule_name,
  dsb.day_of_week,
  dsb.start_time,
  dsb.end_time,
  (dsb.start_time = '00:00:00' and dsb.end_time = '00:00:00') as is_all_day_sentinel
from device_schedules ds
left join device_schedule_blocks dsb on dsb.schedule_id = ds.id
where ds.device_id = 'WI_608A_JR_001_N_001'
order by ds.name, dsb.day_of_week, dsb.start_time;
```

Using a `left join` on purpose so a schedule with zero blocks still shows up as a row (with nulls) instead of disappearing silently — that's the exact thing that made "Always" easy to miss earlier.

You can run this against either project: dev is `xxinwdxprvkvpmhdelnp`, prod/main is `gmovbfjefbxoiruomgum`. Expect to see "Always" now carrying one row with `day_of_week = 0`, `start_time = end_time = '00:00:00'`, `is_all_day_sentinel = true`, plus Work (5 rows), Nighttime (12), Free Time (7), and Sabbath still empty (0 rows, expected since it's unused).

---

**You**

Ok.  Can you confirm that these weren't there, but that you made the update?

---

**Claude**

Confirmed on both counts:

- **They weren't there before** — this isn't a new claim, it's something we established directly earlier in this conversation: I queried `device_schedule_blocks WHERE schedule_id = 12` (dev) with no join, no view, and got zero rows back, before any insert. The prod side (`schedule_id = 64`) was likewise confirmed empty by the other session's own count (0 blocks for Always) and by our diff, which showed dev and prod matching exactly at 0/0.
- **I made the update** — `id 450` (dev) and `id 289` (prod) both carry `created_at = updated_at`, timestamped `2026-07-13 23:29:13` / `23:29:15` — i.e., created and never touched since, at the exact moment my two insert statements ran a few turns ago. No other rows exist for either schedule_id, so these are the only blocks "Always" has, and they're the ones I added.

---

**You**

Ok let's look at WI_608A_TN_001_N_001 now.  You say that the Awake Time schedule isn't used.  However, in the frontend I see that the Eductation category is scheduled for Awake Time.

---

**Claude**

You're right to push on that — I based "redundant" on category overlap alone, not actual time coverage. Let me check what the other schedules actually cover, since "Awake Time" having 0 blocks only doesn't matter if the other 6 schedules fully cover its intended span.

---

**Claude**

Good catch — I was wrong to call it "not really used." It's genuinely wired up: `device_category_schedules` confirms Education, Internet Use, Music and Audio, and Photography are all assigned to "Awake Time," exactly matching what the frontend chip shows. My earlier "low-impact" claim was based only on those categories *also* appearing on other schedules — I hadn't actually checked whether those other schedules cover the same time span.

Now that I've pulled every block for this device, here's the real picture. The other 6 weekday/weekend schedules that also carry these categories stack up like this:

| | Weekday (Mon–Fri) | Weekend (Sat/Sun) |
|---|---|---|
| Morning Weekdays | 06:00–07:00 | — |
| Mid-Day | 07:00–15:30 | — |
| After School Weekdays | 15:30–17:30 | — |
| Evening Weekdays | 17:30–20:30 | — |
| Weekend Morning | — | 06:00–08:00 |
| Normal Weekend | — | 08:00–20:30 |

That's continuous, no-gap coverage from **06:00 to 20:30**, every day. So *if* "Awake Time" was only ever meant to span that same 06:00–20:30 window, it genuinely is redundant — those 4 categories are enabled that whole stretch regardless of whether "Awake Time" itself has blocks.

But I can't actually confirm that's all "Awake Time" was meant to cover — the name suggests it might have been intended to run wider (e.g., earlier than 6am or later than 8:30pm, whatever hours this kid is actually awake beyond the other named blocks). Since it has zero blocks, there's no data trail telling us the intended range — unlike "Always," where the migration comment and a working sibling device told us exactly what it should look like. Two ways to resolve this:

1. If 06:00–20:30 is genuinely the full intended "awake" window, "Awake Time" is inert by design and there's no gap to fix.
2. If it's meant to extend beyond that (e.g., 05:30 or until 22:00), there's a real dark period for Education/Internet Use/Music/Photography apps with no schedule covering it.

Do you know what hours "Awake Time" was supposed to represent for this device? That'll tell us whether to add blocks (and what times) or leave it alone.

---

**You**

this user doesn't have any overlapping categories besides Always.

---

**Claude**

That changes the read entirely. If schedules for this device are meant to tile the day without overlapping (aside from "Always" being the deliberate blanket exception), then the 6 schedules I found aren't there *twice*-covering the same window as "Awake Time" — they're covering 06:00–20:30 as separate, adjacent, non-overlapping slots, and "Awake Time" must be the piece meant to fill whatever's left over: **20:30–06:00**, the stretch with no other schedule at all (this device has no "Nighttime"/"Bedtime" schedule like the JR device does).

So this isn't redundant — it's a real gap. Right now, Education, Internet Use, Music and Audio, and Photography go completely dark for that ~9.5-hour overnight stretch, with nothing filling in, because "Awake Time" (the schedule meant to cover it) has zero blocks.

Two things I'd want confirmed before writing blocks:
1. **Exact hours** — is "Awake Time" meant to be literally the complement of the other 6 (20:30–06:00, i.e., all hours this kid might be awake outside the named blocks), or a narrower window (e.g., "awake but not yet in one of the named activity slots")?
2. **Whether it's meant to be continuous overnight** or split (e.g., stops before actual bedtime, resumes before "Morning Weekdays" starts at 06:00) — the name suggests it might exclude a true sleep window, not just fill the entire gap.

If you know the intended hours, I'll insert the matching blocks the same way I did for "Always" (using `day_of_week`/`start_time`/`end_time`, not the sentinel — this is a specific window, not all-day). If you don't know either, want me to just check with whoever configured the schedule, or default to filling the full complement (20:30–06:00, all 7 days)?

---

**You**

Ah yes.  Awake time is not used.  I was thinking of Mid-Day.

---

**Claude**

Got it — that confirms my original read: with Morning Weekdays, Mid-Day, After School Weekdays, and Evening Weekdays already tiling weekdays 06:00–20:30 (and Weekend Morning/Normal Weekend doing the same on weekends), "Awake Time" having zero blocks isn't costing anything. No fix needed there — leave it inert.

So to recap where things stand for `WI_608A_JR_001_N_001`: fixed (Always sentinel added on dev + prod, verified). For `WI_608A_TN_001_N_001`: nothing to fix — Awake Time is confirmed unused by design. Let me know if you want me to look at any other devices or follow up on when the pending sync (60 dev / 56 prod apps) actually lands.

---

**You**

Do you know how to pull information from old versions of Supabase?  I had a user '%SP%' that used to be on main, but which now I don't see.  Feel free to do an internet search if necessary.  If we can't, can we reverse-engineer that profile (besides perhaps schedules) from hexnode?

---

**Claude**

Let me check a few things: whether this customer is truly gone or just soft-deleted/filtered, what our deletion process actually does to Hexnode, and what real recovery options Supabase offers.

---

**Claude**

Found real leads. The customer row (`WI_608A_SP_001`, Sara Phillips) is still in `customers` on prod — only `customer_devices` and related rows are gone. And I found log history for `WI_608A_SP_001_N_001` in `logging_sync_apps`/`logging_sync_dns`. Let me pin down the timeline and check what `delete-device` actually did to Hexnode (it doesn't hard-delete there).

---

**Claude**

`get_logs` only covers the last 24 hours — too late. But there's a `logging_api_calls` table that might have longer retention and could show the actual `delete-device` call.

---

**Claude**

The trail stops cold at `2026-07-09 19:48:02` — no `delete-device` Hexnode calls (`renameDevice`, `deletePolicy`, `deleteAppGroup`, etc.) show up afterward at all, even though that function is very API-call-heavy and would leave a dense trail here. That strongly suggests **the row was deleted directly in the database, not via the `delete-device` edge function** — meaning Hexnode itself was never touched. The device, its app group (**group 57**, confirmed from the logs), its DNS policy, and its app catalog are likely still fully intact and enrolled in Hexnode, untouched.

Let me pull the full history to reconstruct IDs (Hexnode device ID, policy ID, catalog ID) from the log notes.

---

**Claude**

Let me quickly verify current Supabase backup/PITR mechanics so I give you accurate steps rather than stale info.

---

**Claude**

Two solid paths here, and I found real evidence for both.

## 1. Recovering from a Supabase backup

Supabase's ["Restore to a New Project"](https://supabase.com/docs/guides/platform/clone-project) feature is exactly what you'd want — it clones the project as of a chosen point in time into a **brand-new project**, without touching or overwriting the live prod project (`gmovbfjefbxoiruomgum`). It works off either daily physical backups or, with finer granularity, the PITR add-on if enabled — both require a paid plan.

Timing is in your favor: the last activity for this device (`WI_608A_SP_001_N_001`) in the sync logs was **2026-07-09 19:48 UTC**, and it's now 2026-07-13 — only ~4 days ago, well within typical backup retention windows (commonly 7–9+ days on Pro). I can't check your project's actual backup/PITR configuration or trigger a restore myself (that's billing/infra-level and outside what I should do unprompted) — you'd check **Dashboard → Database → Backups → Restore to a New Project** to see what's available and pick a timestamp just before the deletion. Once restored to a new project, I could query it directly and pull back whatever rows are needed (same pattern as the `WI_608A_JR_001_N_001` migration script). Want me to walk you through that once you're in the dashboard, or should I go ahead and try `restore_project` via MCP if you tell me the target timestamp?

## 2. Reverse-engineering from Hexnode — better news than expected

I checked `logging_api_calls` (a DB table that logs every outbound Hexnode API call, separate from the 24-hour-retention log service). It goes completely silent for this device after `2026-07-09 19:48:02`. That's the tell: **`delete-device` was never called** — that function is Hexnode-call-heavy (rename device, delete policy, delete app group, delete app catalog), and none of those calls appear. Someone deleted the `customer_devices` row (and cascaded rows) directly in the database, bypassing that function entirely.

That means **Hexnode itself was likely never touched** — the device, its app group, and its DNS policy should still be live and enrolled, not even soft-deleted/renamed. I reconstructed the exact Hexnode IDs from the log notes:

- **Hexnode device ID: 8** (from `associatePolicyWithDevice: policy 137, device 8` — succeeded; earlier attempts against `device 6` failed with "no valid target ids", so 6 was stale/wrong)
- **Hexnode app group ID: 57** (from repeated `addRemoveAppsFromAppGroup: group 57`)
- **Hexnode DNS policy ID: 137**

With those, whoever has Hexnode console/API access can pull: device 8's record (name, IMEI, serial, model, enrollment date, and — since Hexnode tracks live installed-app state from device check-ins — likely its current app inventory), app group 57's member list (equivalent to what `active_in_hexnode = true` apps you'd have had), and DNS policy 137's config. I don't have Hexnode API credentials wired into this session, so I can't query it directly myself, but I could write a quick script against the existing `_shared/apis/hexnodeApi` client in the repo if you get me a way to run it with credentials, or you can just check Hexnode's own dashboard directly.

**One real limit:** schedules (`device_schedules`/`device_schedule_blocks`/`device_category_schedules`) are pure MyLizard business logic — Hexnode has no concept of them, so that part genuinely can't be reverse-engineered from Hexnode. Only a DB backup restore gets those back.

Want me to draft the exact backup-check steps, or start on the Hexnode side first since you may have faster access there?

Sources:
- [Restore to a new project | Supabase Docs](https://supabase.com/docs/guides/platform/clone-project)
- [Database Backups | Supabase Docs](https://supabase.com/docs/guides/platform/backups)
- [Manage Point-in-Time Recovery usage | Supabase Docs](https://supabase.com/docs/guides/platform/manage-your-usage/point-in-time-recovery)

---

**You**

Ok the project id is jthyakqlrbsxuwowriei.  Can you list the device_ids on that version that aren't in the current main project?

---

**Claude**

Comparing `customer_devices.device_id` between the restored snapshot (`jthyakqlrbsxuwowriei`) and current main (`gmovbfjefbxoiruomgum`):

**Restored snapshot has 2 devices:** `WI_608A_SP_001_N_001`, `WI_608A_TN_001_N_001`
**Current main has 12 devices:** `WI_608A_JR_001_N_001`, `WI_608A_JR_001_P_001`, `WI_608A_TN_001_N_001`, `NY_608A_PT_001_N_001`, `IL_555A_IT_001_P_001`, `CA_415A_TD_002_N_001_dev`, plus 5 `test-*` devices.

**Result — 1 device is in the snapshot but missing from main:**

- **`WI_608A_SP_001_N_001`**

That's it — just the Sara Phillips device we were chasing. `WI_608A_TN_001_N_001` exists in both, so nothing else got silently lost in that same event; the deletion was isolated to this one device. This confirms the restore point you picked predates the deletion and captured it cleanly.

Want me to pull the full row set for `WI_608A_SP_001_N_001` from the snapshot now — `customer_devices`, `device_schedules`/`device_schedule_blocks`/`device_category_schedules`, `android_device_apps`, `dns_device_categories` — the same way I read the JR device's migration script, so we can write the restore-to-main script?

---

**You**

Yes go ahead.

---

**Claude**

This snapshot predates the scheduling schema migration — `device_schedules`/`device_schedule_blocks`/`device_category_schedules` don't exist yet in it. Let me find the old equivalents.

---

**Claude**

This device turns out to have real functional differences from the others I've drafted — "Sabbath" and "Mornings" actually gate categories here (Internet Use, News and Magazines), unlike on the other devices. The full app list (431 rows) is too large to pull into this conversation without blowing up context, so I'll delegate the script-drafting to a subagent with the schedule mapping I've already worked out, and have it fetch just the bulky apps list itself. I will **not** have it run anything against main — draft only, for your review.

---

**Claude**

I've kicked off drafting the migration script in the background — it's fetching the 431-row app inventory itself (too large to pass through here) and assembling the full script following the same pattern as the JR device migration, but reusing the *existing* Hexnode IDs (device 8, app group 57, catalog 43, policy 73, DNS policy 137) instead of creating new ones, since that Hexnode enrollment was never touched.

I'll let you know when it's done — I won't run anything against main, this is draft-only for your review.

---

**You**

Are you still working?

---

**Claude**

Yes — the background agent drafting the `WI_608A_SP_001_N_001` restore script is still running (it's working through the 431-row app list from the snapshot). I'll let you know as soon as it finishes; no need to do anything on your end.

---

**You**

a859e24f4e8bceed8
toolu_011h4K65HTnF1Lgt5zKGXQss
C:\Users\JEREMI~1\AppData\Local\Temp\claude\c--Git-AprilAire-BI\b864b6c2-ceaa-47a5-adbf-697dab5165b1\tasks\a859e24f4e8bceed8.output
completed
Agent "Draft SP device restore migration script" finished
A task-notification fires each time this agent stops with no live background children of its own. The user can send it another message and resume it, so the same task-id may notify more than once.
The file is complete and correct. I did not execute anything against `gmovbfjefbxoiruomgum` except read-only SELECT/information_schema queries, per the constraint.

## Summary

**File written:** `c:\Git\MyLizard-Supabase\scripts\migrate_to_prod\android_device_WI_608A_SP_001_N_001.sql`

Structure matches the JR script's conventions (ON CONFLICT DO NOTHING guards, DO block for schedule lookup-or-create, WHERE NOT EXISTS guards on blocks, ON CONFLICT on category_schedules/android_device_apps).

**Verification performed against main (`gmovbfjefbxoiruomgum`, read-only only):**
- Confirmed `customer_devices` currently has no row for `WI_608A_SP_001_N_001` (safe to insert).
- Confirmed column names/types for `customer_devices`, `dns_device_categories`, `android_device_apps`, `device_schedules`, `device_schedule_blocks`, `device_category_schedules`, `customers` — all match the JR script's assumptions (unique constraints: `customers(customer_id)`, `customer_devices(device_id)`, `device_category_schedules(schedule_id, category)`, `android_device_apps(app_package_id, device_id)`; `device_schedule_blocks` and `dns_device_categories` have no relevant unique constraint, hence the WHERE NOT EXISTS guards).
- Pulled `select distinct category from android_apps` from main and cross-checked every category name in the task's Always/Free Time/Sabbath/Mornings lists — **all matched verbatim, no corrections needed.** Notably main has both "Travel &amp; Local" and "Travel and Local" as separate catalog values coexisting; the task's ampersand form was already correct as given.

**Row-count summary:**
- Customer: 1 (safety net insert)
- Device: 1 (`customer_devices`, preserving `mdm_vendor_id='8'`, `app_group=57`, `app_catalog=43`, `hn_policy_id=73`, `dns_hexnode_policy_id=137`, `dns_profile_name='ME_SM'` exactly)
- Schedules created: 5 (Always, Free Time, Mornings, Sabbath, Go to Bed)
- Schedule blocks: Always=1 (sentinel), Free Time=5, Mornings=5, Sabbath=2, Go to Bed=7 → 20 total
- `device_category_schedules`: 23 (Always) + 8 (Free Time) + 2 (Sabbath) + 1 (Mornings) = 34 rows
- `dns_device_categories`: 6 rows
- `android_device_apps`: 431 rows (pulled directly from `jthyakqlrbsxuwowriei`, verified count via SQL `string_agg`/tuple-count cross-check — matches the source table's row count exactly)

**Anomalies:** None flagged. No category-name corrections were required — every name in the task's spec already matches main's current `android_apps.category` values verbatim. I did not do an exhaustive check of all 431 app package ids against main's catalog (per the "not required" note), and didn't notice any obvious missing ones in passing.

The script was not executed against either project (source reads only; no writes to `gmovbfjefbxoiruomgum`).
15162322309447

---

**Claude**

The draft finished. Let me verify the actual file before telling you it's ready — the agent's summary is a claim, not proof.

---

**Claude**

430 separators + 1 = **431 tuples**, matching the source table's row count exactly.

The draft is done and checks out. Summary of what's in `c:\Git\MyLizard-Supabase\scripts\migrate_to_prod\android_device_WI_608A_SP_001_N_001.sql`:

- **Customer**: safety-net insert (already exists on main).
- **Device**: reuses the *existing* live Hexnode IDs — `mdm_vendor_id='8'`, `app_group=57`, `app_catalog=43`, `hn_policy_id=73`, `dns_hexnode_policy_id=137`, `dns_profile_name='ME_SM'` — rather than creating new ones, since Hexnode was never touched.
- **Schedules**: 5 created (Always, Free Time, Mornings, Sabbath, Go to Bed), translated from the old `device_calendar` rows into the current block schema — including the all-day sentinel for "Always" and the midnight-split technique for "Sabbath" (Sat 18:00→Sun 18:00), consistent with what we established earlier this session.
- **Category assignments**: 34 rows across Always/Free Time/Sabbath/Mornings — the agent cross-checked every category name against main's live catalog and found no spelling drift, so nothing needed correcting.
- **DNS categories**: 6 rows.
- **Android apps**: 431 rows, verified exact count match against the source.

This is a draft only — nothing has been run against main. Want me to walk through any part of it before you decide whether/when to execute it, or is this ready as-is?

---

**You**

I'll run it.

---

**Claude**

Sounds good — the script at `c:\Git\MyLizard-Supabase\scripts\migrate_to_prod\android_device_WI_608A_SP_001_N_001.sql` is ready whenever you want to run it. It's idempotent (every insert is guarded with `ON CONFLICT`/`WHERE NOT EXISTS`), so it's safe if you need to re-run it. Let me know if you'd like me to verify anything afterward — e.g., confirm the row counts landed on main, or check that `vw_desired_hexnode_app_groups_state` picks it up cleanly for the next sync.

---

**You**

The Sara "Always" schedule seems to be only on Mondays.

---

**Claude**

Good catch — that's a real gap. Confirmed: Sara's "Always" schedule (`schedule_id=67`) has exactly **one** block, `day_of_week=0` (Monday) only.

Here's why: the sync engine (`vw_android_active_schedules`) doesn't care about day-of-week for the all-day sentinel — `is_all_day` is true regardless, so the apps-enabling logic has been working correctly regardless of day. But the **calendar display** (`vw_device_calendar`) projects each block onto its own specific `day_of_week` when building the week view — it doesn't broadcast a single sentinel row across all 7 days, it only draws an event on the day that row is pinned to. So a single sentinel row is enough for app-enabling logic, but not enough to *look* right on the calendar. My earlier "one row is sufficient" call was correct for the sync pipeline but wrong for the UI — I should have added one row per day.

This affects three places, all done the same way this session: Sara's device (prod, `schedule_id=67`), and JR's device on both dev (`schedule_id=12`) and prod (`schedule_id=64`). Fixing all three by adding the missing 6 days each.

---

**Claude**

All three fixed — 6 rows added each to schedule 67 (Sara, prod), 64 (JR, prod), and 12 (JR, dev), all `day_of_week` 1–6 with the same `00:00:00`–`00:00:00` sentinel. Each "Always" schedule now has 7 rows total, one per day, so it should render as a full-width all-day banner across the whole week on the calendar — refresh the frontend to confirm it now spans Mon–Sun instead of just Monday.

No change to the sync-side behavior — `should_be_enabled` was already correct regardless of day, so this doesn't touch which apps are active, just fixes the visual.

---

**You**

I think Sara's device might be 355522574747089 in Hexnode, which is Sara's phone's imei, but which doesn't match her user record which shows an imei of 350214925992706.

Also, I'm getting this email error: 

sync-apps-and-websites completed with 2 errors.
Branch: https://gmovbfjefbxoiruomgum.supabase.co
•	[android] Error syncing app group for device WI_608A_SP_001_N_001:
•	[android] Unusual error returned by Hexnode API addRemoveAppsFromAppGroup method: {"status":400,"appGroupId":57,"responseBody":["AppGroup does not exists with id /57/"]}

What's the best way to reconnect the hexnode device with Supabase?

---

**Claude**

That error confirms something worse than "was never touched" — **the circuit breaker already tripped**: `enabled` flipped to `false` after 3 consecutive sync failures (I can see the exact error streak in `logging_sync_apps`, then a "success" entry with `app_group_id: null` once it dropped out of the desired-state view after being disabled). And Hexnode's own API is now telling us app group 57 genuinely doesn't exist. That contradicts my earlier read — I only proved our *automated* `delete-device` function never ran; I didn't account for someone cleaning things up manually in Hexnode's own console, which wouldn't show up in `logging_api_calls` at all.

The IMEI mismatch you found is the key clue. Let me check what triggers fresh app-group/policy creation in this codebase, since that's probably the real fix rather than trying to resurrect dead IDs.

---

**Claude**

Found the mechanism — `mdm_vendor_id` isn't something we should hand-set; it's written by `post-enrollment-device-setup`, which looks the device up **by IMEI** when Hexnode reports enrollment. That's exactly why the IMEI mismatch matters: as long as our `imei` is wrong, that lookup can never succeed. Let me check how the device-matching RPC works before proposing next steps, since I want to make sure fixing this doesn't accidentally spawn a duplicate device record.

---

**Claude**

This is very actionable. `post-enrollment-device-setup` takes a **Hexnode device ID** (their internal integer ID, not the IMEI), fetches the device's IMEI directly from Hexnode itself, looks up our `customer_devices` row by that IMEI, and then fully rebuilds everything: writes `mdm_vendor_id`, renames the Hexnode device, creates a fresh `app_group_initial_push`, a fresh full `app_group`, a fresh `app_catalog`, a new permanent policy, and associates all of it — seeded from the `android_device_apps` (431 rows) we already restored. That's a complete "reconnect," not a patch.

Two things need to happen, in order:

1. **Fix `customer_devices.imei`** for `WI_608A_SP_001_N_001` to `355522574747089` first — the lookup in step 3 of `post-enrollment-device-setup` matches on IMEI, so it won't find our row until this is corrected. Plain SQL, no Hexnode calls, fully reversible.
2. **Call `post-enrollment-device-setup`** with Sara's *Hexnode device ID* — the internal numeric ID Hexnode assigns to the device (different from the IMEI). I don't have Hexnode API/console access from here, so I can't look that up myself. Could you grab it from the Hexnode device detail page for IMEI `355522574747089` (it's usually in the URL or device info panel)?

This second step makes several real, live Hexnode API calls against Sara's actual phone (rename, create app groups/catalog/policy, associate policies) — I want your explicit go-ahead before I trigger it, and I'd need that Hexnode device ID from you to do it at all. Once both are done I'll also flip `enabled` back to `true` (it's currently `false` — the circuit breaker tripped after the app-group-57 failures).

Want me to go ahead and fix the `imei` field now while you grab the Hexnode device ID?

---

**You**

What will happen with step 2 if the policies in question already exist?

---

**Claude**

Good question — the answer differs by object type, and there's a real edge case worth flagging. Let me check the actual `createAppGroup`/`createAppCatalog`/`createPolicy` implementations to see exactly how each handles a name collision.

---

**Claude**

Depends on which object and whether Hexnode still has it "active" vs. merely archived — and this actually resolves your new requirement too, so let me cover both.

**Idempotency by object type**, from the actual code:

- **App groups & app catalog** (steps 7–9): `createAppGroup`/`createAppCatalog` POST first; if Hexnode replies "name already exists," they call `listAppGroups()`/`listAppCatalogs()` to find the existing one **by name** and reuse its ID, syncing in any missing apps. Safe and idempotent — *if* the existing object is active.
- **Policy** (step 10): same pattern via `listPolicies()`, with an explicit 409 error message if it can't find an active match.

**The catch, in both cases:** those lookup lists only return *active* Hexnode objects. Our own `delete-device` code has a comment about this exact behavior for policies: *"Hexnode archives rather than truly deletes, leaving the original name reserved and blocking re-enrollment."* If that's what happened here (rather than a true delete), Hexnode will still reject the create as a duplicate name, but the reuse-lookup won't find it in the active list — so it throws instead of recovering. For the policy that's a clean 409 you'd see clearly; for the app group/catalog it's a raw thrown error, and whatever steps already succeeded before the failure (e.g., the rename, or one of the two app groups) will have already been written to the DB, leaving the device partially reconnected.

Given the app group is *confirmed* dead by ID (`"AppGroup does not exists with id /57/"`) but we don't know if it's truly gone or archived-with-name-reserved, I'd suggest checking Hexnode's console for any archived group/catalog/policy still named `WI_608A_SP_001_N_001` or `WI_608A_SP_001_N_001_initial_push` before we run this, and purging them if Hexnode's UI allows it — that avoids the failure mode entirely rather than us discovering it mid-run.

**On your new point** — apps installed while the device was missing from the DB: good news, `post-enrollment-device-setup` already covers this. Step 1 fetches the device's *live* installed-app list straight from Hexnode; steps 4–5 correct `is_system_app` and register (via `add-apps`) any package not already in `android_device_apps`. So running this same call handles that automatically — no separate step needed, as long as it completes successfully.

---

**You**

Ok.  Let's start by updating our database.  Here is the response to the device details api, which lists her apps.

{
    "id": 8,
    "udid": "9ffa9c21a89d094a",
    "platform": "android",
    "owned_by": 2,
    "enrollment_status": "enrolled",
    "enrolled_time": "2025-12-20T01:31:57.594319Z",
    "agent_active": true,
    "lastreported": "2026-07-14T13:52:44.290499Z",
    "lastscanned": "2026-07-14T06:06:23.603959Z",
    "lastnotified": "2026-07-14T13:52:41.053263Z",
    "remarks": "Successfully Enrolled",
    "disenrolled_time": null,
    "location_tracking_disabled": false,
    "location_tracking_interval": 60,
    "lostmode": false,
    "last_location": " 914 Dane Street Burr Oaks Madison",
    "latitude": "43.0435967",
    "longitude": "-89.398416",
    "last_location_time": "2026-07-14T13:43:46.552000Z",
    "user": {
        "id": 1,
        "name": "Default User",
        "email": null,
        "phoneno": null,
        "alternate_email": "jr2@mylizard.net"
    },
    "device": {
        "device_name": "355522574747089",
        "description": null,
        "os_version": "16",
        "os_name": "Android",
        "build_version": "CP1A.260505.005",
        "model_name": "Pixel 8 Pro",
        "model": "Pixel 8 Pro",
        "product_name": "Google",
        "device_type": "smartphone",
        "manufacture": "Google",
        "serial_number": "41030DLJG001JC",
        "storage": null,
        "device_capacity": 109.65625,
        "available_device_capacity": 61.119140625,
        "used_device_capacity": 48.537109375,
        "battery_level": 25,
        "imei_1": "355522574747089",
        "imei_2": "355522574747097",
        "phoneno_1": "16082281627",
        "phoneno_2": "",
        "meid": "",
        "modem_firmware_version": null,
        "is_supervised": null,
        "wifi_ssid": "UniFi",
        "wifi_mac": "94:45:60:55:34:37",
        "ipaddress": "192.168.1.249",
        "iccid": "89014107334651846735",
        "is_kiosk": false
    },
    "network": {
        "wifi_ssid": "UniFi",
        "wifi_mac": "94:45:60:55:34:37",
        "ipaddress": "192.168.1.249",
        "iccid": "89014107334651846735",
        "imei_1": "355522574747089",
        "imei_2": "355522574747097",
        "phoneno_1": "16082281627",
        "phoneno_2": "",
        "ethernet_mac": "",
        "bluetooth_mac": "",
        "network_name": null,
        "local_host_name": null,
        "bluetooth_state": null,
        "bluetooth_discoverable_mode": null,
        "bluetooth_firm_version": null,
        "imsi_1": "310410465185120",
        "is_roaming_sim_1": false,
        "imsi_2": null,
        "is_roaming_sim_2": null,
        "sim_carrier_network": "Boost",
        "subscriber_carrier_network": null,
        "current_carrier_network": "Boost",
        "carrier_setting_network": null,
        "roaming_enabled": null,
        "is_roaming": false,
        "international_data_roaming": null,
        "home_carrier": null,
        "home_country": null,
        "current_carrier": null,
        "current_country": null,
        "network_type": "13",
        "last_connected_date": "2026-07-14",
        "ethernet_ip": "",
        "personal_hotspot_enabled": null,
        "subscriber_mcc": "us",
        "subscriber_mnc": "410",
        "current_mcc": "us",
        "current_mcc2": null,
        "current_mnc": "410",
        "current_mnc2": null,
        "iccid2": null,
        "current_carrier_network2": null,
        "subscriber_carrier_network2": null,
        "is_roaming_2": null,
        "win_phone_no_1": null,
        "win_imei_1": null,
        "win_imsi_1": null,
        "win_is_roaming_1": null,
        "win_phone_no_2": null,
        "win_imsi_2": null,
        "win_imei_2": null,
        "win_is_roaming_2": null,
        "win_associated_wifi_ssid": null,
        "win_associated_wifi_ip": null,
        "win_wifi_mac_add": null,
        "win_ethernet_mac_add1": null,
        "win_ethernet_mac_add2": null,
        "win_bluetooth_mac_add": null,
        "win_bluetooth_device_name": null,
        "public_ip_address": "12.74.222.77"
    },
    "applications": [
        {
            "id": 1699,
            "name": "Android Developer Verifier",
            "bundle_size": null,
            "app_version": "1.0.804814953",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.google.android.verifier"
        },
        {
            "id": 726,
            "name": "Wallpapers",
            "bundle_size": null,
            "app_version": "16",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.google.android.apps.wallpaper"
        },
        {
            "id": 1013,
            "name": "com.google.ambient.streaming",
            "bundle_size": null,
            "app_version": "1.0.1131.0",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.google.ambient.streaming"
        },
        {
            "id": 1091,
            "name": "com.google.android.uvexposurereporter",
            "bundle_size": null,
            "app_version": "16",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.google.android.uvexposurereporter"
        },
        {
            "id": 1729,
            "name": "GizmoHub",
            "bundle_size": null,
            "app_version": "9.3.8.201",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.vzw.gizmopal"
        },
        {
            "id": 408,
            "name": "com.android.providers.calendar",
            "bundle_size": null,
            "app_version": "16",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.android.providers.calendar"
        },
        {
            "id": 608,
            "name": "Contacts",
            "bundle_size": null,
            "app_version": "4.82.29.936766038",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.google.android.contacts"
        },
        {
            "id": 589,
            "name": "com.android.bips",
            "bundle_size": null,
            "app_version": "16",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.android.bips"
        },
        {
            "id": 399,
            "name": "com.android.internal.display.cutout.emulation.double",
            "bundle_size": null,
            "app_version": "1.0",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.android.internal.display.cutout.emulation.double"
        },
        {
            "id": 1014,
            "name": "com.google.android.documentsui.theme.pixel",
            "bundle_size": null,
            "app_version": "1.0",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.google.android.documentsui.theme.pixel"
        },
        {
            "id": 628,
            "name": "com.android.calllogbackup",
            "bundle_size": null,
            "app_version": "16",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.android.calllogbackup"
        },
        {
            "id": 708,
            "name": "Android Auto",
            "bundle_size": null,
            "app_version": "17.1.662414-release",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.google.android.projection.gearhead"
        },
        {
            "id": 1108,
            "name": "com.google.android.overlay.googlewebview",
            "bundle_size": null,
            "app_version": "1.0",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.google.android.overlay.googlewebview"
        },
        {
            "id": 1093,
            "name": "com.android.simappdialog.auto_generated_rro_product__",
            "bundle_size": null,
            "app_version": "1.0",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.android.simappdialog.auto_generated_rro_product__"
        },
        {
            "id": 651,
            "name": "com.google.android.connectivity.resources",
            "bundle_size": null,
            "app_version": "S-initial",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.google.android.connectivity.resources"
        },
        {
            "id": 663,
            "name": "com.google.android.documentsui",
            "bundle_size": null,
            "app_version": "16",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.google.android.documentsui"
        },
        {
            "id": 1052,
            "name": "com.android.systemui.clocks.bignum",
            "bundle_size": null,
            "app_version": "16",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.android.systemui.clocks.bignum"
        },
        {
            "id": 1053,
            "name": "com.google.pixel.camera.services",
            "bundle_size": null,
            "app_version": "1.0",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.google.pixel.camera.services"
        },
        {
            "id": 1099,
            "name": "com.android.safetyregulatoryinfo.auto_generated_rro_product__",
            "bundle_size": null,
            "app_version": "1.0",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.android.safetyregulatoryinfo.auto_generated_rro_product__"
        },
        {
            "id": 347,
            "name": "Amazon Music: Songs & Podcasts",
            "bundle_size": null,
            "app_version": "26.22.1",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.amazon.mp3"
        },
        {
            "id": 817,
            "name": "Cube Escape Collection",
            "bundle_size": null,
            "app_version": "1.4.13",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "air.com.RustyLake.CubeEscapeCollection"
        },
        {
            "id": 1112,
            "name": "com.android.systemui.clocks.metro",
            "bundle_size": null,
            "app_version": "16",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.android.systemui.clocks.metro"
        },
        {
            "id": 329,
            "name": "Microsoft Authenticator",
            "bundle_size": null,
            "app_version": "6.2606.4246",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.azure.authenticator"
        },
        {
            "id": 792,
            "name": "Android Switch",
            "bundle_size": null,
            "app_version": "1.0.833807547",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.google.android.apps.restore"
        },
        {
            "id": 738,
            "name": "com.android.emergency",
            "bundle_size": null,
            "app_version": "16",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.android.emergency"
        },
        {
            "id": 936,
            "name": "Pixel Camera Services",
            "bundle_size": null,
            "app_version": "3.1.843305793.11",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.google.android.apps.camera.services"
        },
        {
            "id": 1072,
            "name": "com.google.android.apps.retaildemo.preload",
            "bundle_size": null,
            "app_version": "4.7.683446209",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.google.android.apps.retaildemo.preload"
        },
        {
            "id": 1790,
            "name": "com.google.android.overlay.glanceablehubsettings2022",
            "bundle_size": null,
            "app_version": "1.0",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.google.android.overlay.glanceablehubsettings2022"
        },
        {
            "id": 561,
            "name": "com.google.android.setupwizard",
            "bundle_size": null,
            "app_version": "236.829759469",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.google.android.setupwizard"
        },
        {
            "id": 711,
            "name": "com.google.android.apps.turbo",
            "bundle_size": null,
            "app_version": "1.27.0.935999332.release",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.google.android.apps.turbo"
        },
        {
            "id": 1075,
            "name": "com.google.SSRestartDetector",
            "bundle_size": null,
            "app_version": "16",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.google.SSRestartDetector"
        },
        {
            "id": 1006,
            "name": "com.google.android.overlay.pixelconfigcommon",
            "bundle_size": null,
            "app_version": "1.0",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.google.android.overlay.pixelconfigcommon"
        },
        {
            "id": 1111,
            "name": "com.google.android.overlay.permissioncontroller",
            "bundle_size": null,
            "app_version": "1.0",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.google.android.overlay.permissioncontroller"
        },
        {
            "id": 1115,
            "name": "com.google.android.hardwareinfo",
            "bundle_size": null,
            "app_version": "1.0.0",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.google.android.hardwareinfo"
        },
        {
            "id": 979,
            "name": "com.android.internal.display.cutout.emulation.avoidAppsInCutout",
            "bundle_size": null,
            "app_version": "1.0",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.android.internal.display.cutout.emulation.avoidAppsInCutout"
        },
        {
            "id": 405,
            "name": "com.google.android.cellbroadcastservice",
            "bundle_size": null,
            "app_version": "R-initial",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.google.android.cellbroadcastservice"
        },
        {
            "id": 1700,
            "name": "Users",
            "bundle_size": null,
            "app_version": "16",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.android.multiuser"
        },
        {
            "id": 1355,
            "name": "com.android.pixeldisplayservice",
            "bundle_size": null,
            "app_version": "1.0",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.android.pixeldisplayservice"
        },
        {
            "id": 990,
            "name": "com.google.android.iwlan",
            "bundle_size": null,
            "app_version": "16",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.google.android.iwlan"
        },
        {
            "id": 951,
            "name": "Android System SafetyCore",
            "bundle_size": null,
            "app_version": "1.0.925574157",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.google.android.safetycore"
        },
        {
            "id": 856,
            "name": "Wisconsin Magazine of History",
            "bundle_size": null,
            "app_version": "25.2.13",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.audiencemedia.app1322"
        },
        {
            "id": 1031,
            "name": "com.google.android.systemui.overlay.pixelbatteryhealthconfig",
            "bundle_size": null,
            "app_version": "1.0",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.google.android.systemui.overlay.pixelbatteryhealthconfig"
        },
        {
            "id": 1039,
            "name": "com.google.android.apps.internal.betterbug",
            "bundle_size": null,
            "app_version": "1.0",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.google.android.apps.internal.betterbug"
        },
        {
            "id": 907,
            "name": "Personal Safety",
            "bundle_size": null,
            "app_version": "2026.06.11.930408383.0-release",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.google.android.apps.safetyhub"
        },
        {
            "id": 1058,
            "name": "com.google.android.apps.privacy.wildlife",
            "bundle_size": null,
            "app_version": "1.0.703309715",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.google.android.apps.privacy.wildlife"
        },
        {
            "id": 1067,
            "name": "com.google.android.storagemanager.auto_generated_rro_product__",
            "bundle_size": null,
            "app_version": "1.0",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.google.android.storagemanager.auto_generated_rro_product__"
        },
        {
            "id": 654,
            "name": "com.google.android.feedback",
            "bundle_size": null,
            "app_version": "16",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.google.android.feedback"
        },
        {
            "id": 576,
            "name": "com.android.providers.settings.auto_generated_rro_product__",
            "bundle_size": null,
            "app_version": "1.0",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.android.providers.settings.auto_generated_rro_product__"
        },
        {
            "id": 1360,
            "name": "com.google.android.settings.intelligence",
            "bundle_size": null,
            "app_version": "1.1.0.937862210.sr",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.google.android.settings.intelligence"
        },
        {
            "id": 610,
            "name": "com.android.server.telecom",
            "bundle_size": null,
            "app_version": "16",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.android.server.telecom"
        },
        {
            "id": 130,
            "name": "Google Chrome",
            "bundle_size": null,
            "app_version": "150.0.7871.64",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.android.chrome"
        },
        {
            "id": 1049,
            "name": "com.android.telephony.imsmedia",
            "bundle_size": null,
            "app_version": "16",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.android.telephony.imsmedia"
        },
        {
            "id": 1582,
            "name": "com.google.android.apps.magicportrait",
            "bundle_size": null,
            "app_version": "16",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.google.android.apps.magicportrait"
        },
        {
            "id": 938,
            "name": "Android AICore",
            "bundle_size": null,
            "app_version": "0.release.prod_aicore_20260528.01_RC08.934495584",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.google.android.aicore"
        },
        {
            "id": 593,
            "name": "com.google.android.captiveportallogin",
            "bundle_size": null,
            "app_version": "16",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.google.android.captiveportallogin"
        },
        {
            "id": 752,
            "name": "YouTube Music",
            "bundle_size": null,
            "app_version": "9.26.55",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.google.android.apps.youtube.music"
        },
        {
            "id": 1020,
            "name": "com.google.android.settings.overlay.pixelvpnconfig",
            "bundle_size": null,
            "app_version": "1.0",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.google.android.settings.overlay.pixelvpnconfig"
        },
        {
            "id": 986,
            "name": "com.android.intentresolver",
            "bundle_size": null,
            "app_version": "2021-11",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.android.intentresolver"
        },
        {
            "id": 1701,
            "name": "com.google.android.verifier.overlay",
            "bundle_size": null,
            "app_version": "16",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.google.android.verifier.overlay"
        },
        {
            "id": 1082,
            "name": "com.google.android.apps.pixel.relationships",
            "bundle_size": null,
            "app_version": "STUB",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.google.android.apps.pixel.relationships"
        },
        {
            "id": 1010,
            "name": "com.google.android.federatedcompute",
            "bundle_size": null,
            "app_version": "16",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.google.android.federatedcompute"
        },
        {
            "id": 501,
            "name": "com.android.internal.display.cutout.emulation.hole",
            "bundle_size": null,
            "app_version": "1.0",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.android.internal.display.cutout.emulation.hole"
        },
        {
            "id": 779,
            "name": "com.android.providers.contacts",
            "bundle_size": null,
            "app_version": "16",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.android.providers.contacts"
        },
        {
            "id": 643,
            "name": "com.android.proxyhandler",
            "bundle_size": null,
            "app_version": "16",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.android.proxyhandler"
        },
        {
            "id": 984,
            "name": "android.auto_generated_characteristics_rro",
            "bundle_size": null,
            "app_version": "16",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "android.auto_generated_characteristics_rro"
        },
        {
            "id": 530,
            "name": "com.android.backupconfirm",
            "bundle_size": null,
            "app_version": "16",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.android.backupconfirm"
        },
        {
            "id": 311,
            "name": "Hexnode MDM Remote View",
            "bundle_size": null,
            "app_version": "6.4",
            "mandatory_app": true,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.hexnode.hexnoderemote"
        },
        {
            "id": 1073,
            "name": "com.android.role.notes.enabled",
            "bundle_size": null,
            "app_version": "1.0",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.android.role.notes.enabled"
        },
        {
            "id": 1048,
            "name": "com.google.euiccpixel",
            "bundle_size": null,
            "app_version": "D.2.0.18 (14341307)",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.google.euiccpixel"
        },
        {
            "id": 974,
            "name": "com.android.captiveportallogin.overlay",
            "bundle_size": null,
            "app_version": "16",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.android.captiveportallogin.overlay"
        },
        {
            "id": 1342,
            "name": "com.google.android.flipendo.auto_generated_rro_product__",
            "bundle_size": null,
            "app_version": "1.0",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.google.android.flipendo.auto_generated_rro_product__"
        },
        {
            "id": 1120,
            "name": "com.android.systemui.accessibility.accessibilitymenu",
            "bundle_size": null,
            "app_version": "16",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.android.systemui.accessibility.accessibilitymenu"
        },
        {
            "id": 1064,
            "name": "com.google.android.apps.dreamliner",
            "bundle_size": null,
            "app_version": "2.2.0.834194002",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.google.android.apps.dreamliner"
        },
        {
            "id": 737,
            "name": "com.google.android.apps.carrier.carrierwifi",
            "bundle_size": null,
            "app_version": "1.7.0.929007131",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.google.android.apps.carrier.carrierwifi"
        },
        {
            "id": 952,
            "name": "Voice Access",
            "bundle_size": null,
            "app_version": "17.0.896515536",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.google.android.apps.accessibility.voiceaccess"
        },
        {
            "id": 2101,
            "name": "Connected Phone & Watch services",
            "bundle_size": null,
            "app_version": "16",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.google.android.crossdeviceaccessservice"
        },
        {
            "id": 1104,
            "name": "com.verizon.services",
            "bundle_size": null,
            "app_version": "3.0",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.verizon.services"
        },
        {
            "id": 1023,
            "name": "com.google.android.wifi.resources.pixel",
            "bundle_size": null,
            "app_version": "1.0",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.google.android.wifi.resources.pixel"
        },
        {
            "id": 482,
            "name": "com.google.android.providers.media.module",
            "bundle_size": null,
            "app_version": "16",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.google.android.providers.media.module"
        },
        {
            "id": 1042,
            "name": "com.google.pixel.digitalkey.timesync",
            "bundle_size": null,
            "app_version": "16",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.google.pixel.digitalkey.timesync"
        },
        {
            "id": 906,
            "name": "Pixel Thermometer",
            "bundle_size": null,
            "app_version": "1.0.889172071",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.google.android.apps.pixel.health"
        },
        {
            "id": 325,
            "name": "Google Authenticator",
            "bundle_size": null,
            "app_version": "7.1",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.google.android.apps.authenticator2"
        },
        {
            "id": 509,
            "name": "Android Accessibility Suite",
            "bundle_size": null,
            "app_version": "16.0.0.777931756",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.google.android.marvin.talkback"
        },
        {
            "id": 1035,
            "name": "com.google.android.carrierlocation",
            "bundle_size": null,
            "app_version": "16",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.google.android.carrierlocation"
        },
        {
            "id": 1344,
            "name": "com.google.android.apps.customization.pixel",
            "bundle_size": null,
            "app_version": "16",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.google.android.apps.customization.pixel"
        },
        {
            "id": 983,
            "name": "com.android.devicediagnostics.auto_generated_rro_vendor__",
            "bundle_size": null,
            "app_version": "1.0",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.android.devicediagnostics.auto_generated_rro_vendor__"
        },
        {
            "id": 334,
            "name": "Amazon Kindle: Reading App",
            "bundle_size": null,
            "app_version": "8.151.0.100(2.0.87334.0)",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.amazon.kindle"
        },
        {
            "id": 3262,
            "name": "Substack",
            "bundle_size": null,
            "app_version": "2.106.1",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.substack.app"
        },
        {
            "id": 851,
            "name": "Soundcore",
            "bundle_size": null,
            "app_version": "5.0.13",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.oceanwing.soundcore"
        },
        {
            "id": 2103,
            "name": "com.google.android.crossdeviceaccessservice.auto_generated_rro_vendor__",
            "bundle_size": null,
            "app_version": "1.0",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.google.android.crossdeviceaccessservice.auto_generated_rro_vendor__"
        },
        {
            "id": 1904,
            "name": "Glasses Core",
            "bundle_size": null,
            "app_version": "0.0.0.251110020",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.google.android.glasses.core"
        },
        {
            "id": 980,
            "name": "com.android.devicediagnostics",
            "bundle_size": null,
            "app_version": "16",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.android.devicediagnostics"
        },
        {
            "id": 745,
            "name": "com.google.android.gms.location.history",
            "bundle_size": null,
            "app_version": "1.2",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.google.android.gms.location.history"
        },
        {
            "id": 736,
            "name": "com.android.providers.userdictionary",
            "bundle_size": null,
            "app_version": "16",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.android.providers.userdictionary"
        },
        {
            "id": 3263,
            "name": "Costco",
            "bundle_size": null,
            "app_version": "26.6.4",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.costco.app.android"
        },
        {
            "id": 1005,
            "name": "com.google.android.apps.cbrsnetworkmonitor",
            "bundle_size": null,
            "app_version": "24.1.721121105",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.google.android.apps.cbrsnetworkmonitor"
        },
        {
            "id": 634,
            "name": "Google TV",
            "bundle_size": null,
            "app_version": "4.39.3896.837424156.1-release",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.google.android.videos"
        },
        {
            "id": 1057,
            "name": "com.google.android.nfc.overlay.common",
            "bundle_size": null,
            "app_version": "1.0",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.google.android.nfc.overlay.common"
        },
        {
            "id": 1051,
            "name": "com.google.android.connectivitythermalpowermanager",
            "bundle_size": null,
            "app_version": "16",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.google.android.connectivitythermalpowermanager"
        },
        {
            "id": 1074,
            "name": "com.google.android.carriersetup",
            "bundle_size": null,
            "app_version": "16",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.google.android.carriersetup"
        },
        {
            "id": 582,
            "name": "com.android.inputdevices",
            "bundle_size": null,
            "app_version": "16",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.android.inputdevices"
        },
        {
            "id": 494,
            "name": "Google Play Services for AR",
            "bundle_size": null,
            "app_version": "1.55.261550593",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.google.ar.core"
        },
        {
            "id": 994,
            "name": "com.google.android.apps.scone",
            "bundle_size": null,
            "app_version": "p.2026.20",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.google.android.apps.scone"
        },
        {
            "id": 1852,
            "name": "Custom Icons",
            "bundle_size": null,
            "app_version": "2026.01.07.853076941.0-release",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.google.android.apps.ai.icons"
        },
        {
            "id": 893,
            "name": "Sound Amplifier",
            "bundle_size": null,
            "app_version": "4.91.886631805",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.google.android.accessibility.soundamplifier"
        },
        {
            "id": 831,
            "name": "Libby: Read Library Books",
            "bundle_size": null,
            "app_version": "9.4.0",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.overdrive.mobile.android.libby"
        },
        {
            "id": 684,
            "name": "com.android.imsserviceentitlement",
            "bundle_size": null,
            "app_version": "16",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.android.imsserviceentitlement"
        },
        {
            "id": 498,
            "name": "com.android.simappdialog",
            "bundle_size": null,
            "app_version": "16",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.android.simappdialog"
        },
        {
            "id": 524,
            "name": "com.android.egg",
            "bundle_size": null,
            "app_version": "1.0",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.android.egg"
        },
        {
            "id": 1539,
            "name": "com.google.android.soundpicker",
            "bundle_size": null,
            "app_version": "3.3 (825845788)",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.google.android.soundpicker"
        },
        {
            "id": 991,
            "name": "com.google.android.pixel.setupwizard",
            "bundle_size": null,
            "app_version": "1.36.834129332",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.google.android.pixel.setupwizard"
        },
        {
            "id": 1704,
            "name": "PixelSetupWizardOverlayExpressive",
            "bundle_size": null,
            "app_version": "16",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.google.android.pixel.setupwizard.overlay.expressive"
        },
        {
            "id": 1047,
            "name": "com.google.android.pixel.avatarpicker",
            "bundle_size": null,
            "app_version": "16",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.google.android.pixel.avatarpicker"
        },
        {
            "id": 617,
            "name": "com.google.android.packageinstaller",
            "bundle_size": null,
            "app_version": "16",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.google.android.packageinstaller"
        },
        {
            "id": 1002,
            "name": "com.android.sdm.plugins.diagmon",
            "bundle_size": null,
            "app_version": "16",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.android.sdm.plugins.diagmon"
        },
        {
            "id": 1060,
            "name": "com.google.android.avatarpicker",
            "bundle_size": null,
            "app_version": "16",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.google.android.avatarpicker"
        },
        {
            "id": 1102,
            "name": "android.autoinstalls.config.google.nexus",
            "bundle_size": null,
            "app_version": "1",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "android.autoinstalls.config.google.nexus"
        },
        {
            "id": 541,
            "name": "Android System Intelligence",
            "bundle_size": null,
            "app_version": "B.26.playstore.pixel8.940155965",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.google.android.as"
        },
        {
            "id": 985,
            "name": "com.android.angle",
            "bundle_size": null,
            "app_version": "100",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.android.angle"
        },
        {
            "id": 333,
            "name": "Audible: Audiobooks & Podcasts",
            "bundle_size": null,
            "app_version": "26.25.07",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.audible.application"
        },
        {
            "id": 764,
            "name": "com.android.traceur",
            "bundle_size": null,
            "app_version": "1.0",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.android.traceur"
        },
        {
            "id": 839,
            "name": "NYTimes: US and Global News",
            "bundle_size": null,
            "app_version": "11.82.0",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.nytimes.android"
        },
        {
            "id": 620,
            "name": "Google Play services",
            "bundle_size": null,
            "app_version": "26.25.32 (260400-944114241)",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "com.google.android.gms"
        },
        {
            "id": 1092,
            "name": "com.google.RilConfigService",
            "bundle_size": null,
            "app_version": "16",
            "mandatory_app": false,
            "black_listed": false,
            "managed": false,
            "status": "Installed",
            "identifier": "c

[Message truncated - exceeded 50,000 character limit]

---

**Claude**

This response actually resolves the bigger mystery — and changes the plan. Look at `"id": 8` in that payload: **Hexnode device ID 8 is the same one already stored in our DB as `mdm_vendor_id`.** It wasn't deleted or replaced — the same Hexnode enrollment record got reassigned to new hardware. Compare to what we restored:

| Field | Our restored DB (old snapshot) | Live Hexnode right now |
|---|---|---|
| Model | Samsung SM-S127DL | **Google Pixel 8 Pro** |
| Serial | R58RB0XZD4E | 41030DLJG001JC |
| IMEI | 350214925992706 | 355522574747089 |
| Enrolled | 2025-12-05 | 2025-12-20 |

Same Hexnode device ID, completely different physical hardware, re-enrolled two weeks later. So this wasn't really "Hexnode wiped and orphaned" — it was a **phone swap**: Sara got a new Pixel, someone reassigned the existing Hexnode device slot to it, cleaned up the old app group/policy along the way (explaining the dead `app_group=57`), and then the process stalled before a new app group/policy got created — and somewhere in there the DB row also got deleted. `mdm_vendor_id='8'` was actually correct all along; we just need `imei`/`serial`/`model_name`/`manufacturer`/`os_version` corrected to the Pixel's values.

That also means our restored `android_device_apps` (431 rows) describes the **old Samsung's** app inventory, not this Pixel's — a lot of those package IDs (Samsung/Verizon-specific system apps) almost certainly don't exist on this phone at all, while genuinely new Pixel-specific ones are missing. A simple "add what's missing" isn't quite enough — we should probably also flag/remove the old-phone-only entries so category/schedule logic doesn't reference phantom apps. Want me to do both (add what's new, clean up what's stale), or just add for now and leave cleanup for later?

Either way, your paste got cut off at 50,000 characters mid-`applications` array — I only have a partial list. Could you save the full API response to a file (e.g. in `C:\Users\JEREMI~1\AppData\Local\Temp\claude\c--Git-AprilAire-BI\b864b6c2-ceaa-47a5-adbf-697dab5165b1\scratchpad`) so I can read the whole thing? I don't want to reconcile the app table off a truncated list.
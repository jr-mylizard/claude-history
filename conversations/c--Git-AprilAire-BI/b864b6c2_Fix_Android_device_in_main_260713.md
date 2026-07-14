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
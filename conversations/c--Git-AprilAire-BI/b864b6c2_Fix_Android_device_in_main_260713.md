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
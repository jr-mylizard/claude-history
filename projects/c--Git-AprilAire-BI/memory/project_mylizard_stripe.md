---
name: project-mylizard-stripe
description: MyLizard Stripe migration scripts and live/sandbox product alignment status
metadata: 
  node_type: memory
  type: project
  originSessionId: af6fba9f-7f58-46ed-9658-e02c4833b97f
---

Stripe migration scripts live in `c:\Git\MyLizard-Supabase\stripe\`:
- `copy_stripe_to_test.py` — copies Live → Sandbox (original, for test setup)
- `copy_sandbox_to_live.py` — copies Sandbox → Live with dry-run mode, image upload, duplicate detection
- `update_stripe_images.py` — copies images from live products to matching test products by name
- `.env` — contains `STRIPE_LIVE_KEY` and `STRIPE_TEST_KEY`

**Why:** As of 2026-06-17, Sandbox is the source of truth. Live was updated to match it.

**Live product status as of 2026-06-17:** Fully aligned with Sandbox. Key changes made:
- Consolidated Qin/Sonim/Sunbeam/Wisephone from separate Purchase+Rental products into single products with Buy+Rent prices
- Migrated 12 sandbox-only products (books, lockboxes, etc.) to live with images
- Firewalla Purple SE Router intentionally stays in Sandbox only
- Stripe MCP tool connects to the test/sandbox account (all products return `livemode: false`); use Python scripts with `.env` keys to access Live

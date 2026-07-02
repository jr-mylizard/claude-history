---
name: feedback-stripe-source-of-truth
description: "Sandbox is Stripe source of truth — always update Live to match Sandbox, never the reverse"
metadata: 
  node_type: memory
  type: feedback
  originSessionId: af6fba9f-7f58-46ed-9658-e02c4833b97f
---

When syncing Stripe environments, **Sandbox (test) is always the source of truth**. Live should be updated to match Sandbox — never rename or change Sandbox to match Live.

**Why:** Jeremiah explicitly corrected this when I renamed sandbox products to match live names. The workflow is: develop/iterate in sandbox, then push changes to live.

**How to apply:** Any time there's a name or data discrepancy between Sandbox and Live, update the Live product. Never touch the Sandbox product to "align" it with Live.

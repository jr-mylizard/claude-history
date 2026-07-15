---
name: project-device-audit
description: Periodic device app monitoring — GHA workflow using GitHub Environments to target dev vs prod Supabase project
metadata: 
  node_type: memory
  type: project
  originSessionId: 16fe29c6-8197-4675-b815-9803aaf8a610
---

Periodic Mosyle device app audit built 2026-07-07.

GHA workflow uses GitHub **Environments** (not repo variables) to resolve the Supabase project ref:
- `development` environment → `vars.SUPABASE_PROJECT = xxinwdxprvkvpmhdelnp` (dev branch)
- `production` environment → `vars.SUPABASE_PROJECT = gmovbfjefbxoiruomgum` (master branch)

Merging to master automatically targets prod — no manual variable flip needed.

Set in GitHub: Settings → Environments → <env name> → Variables → SUPABASE_PROJECT

**Why:** Environment variables correctly model branch-based deployment context; a repo variable would require manual flipping and could be forgotten.

**How to apply:** When this workflow comes up in future sessions, remind user to set SUPABASE_PROJECT in both GitHub environments if not already done.

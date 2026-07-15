---
name: feedback-supabase-mcp-over-docker
description: "For MyLizard-Supabase, use the Supabase MCP against the branch DB instead of starting Docker/local stack"
metadata: 
  node_type: memory
  type: feedback
  originSessionId: 32d1d047-b0d4-48b7-8b0d-b0602baf1dbb
---

When a MyLizard-Supabase task needs DB access (e.g. regenerating `db-types.ts`, inspecting views), Jeremiah prefers the Supabase MCP against the branch's remote database over starting Docker Desktop + the local stack.

**Why:** Docker isn't normally running on this machine; the MCP can query/generate against the exact branch DB directly (git `dev` branch → Supabase branch project ref `xxinwdxprvkvpmhdelnp`, parent project `gmovbfjefbxoiruomgum`).

**How to apply:** Use `mcp__supabase__generate_typescript_types` / `execute_sql` with the branch project ref. When regenerating db-types this way, still apply the `bin/gen-db-types.sh` patch (make `customer_devices.Insert.device_id` optional) before writing the file. Never deploy or commit — file changes only (see repo CLAUDE.md). Note: Brevo objects like `vw_product_block_email` exist only on `main`, so dev-branch types legitimately omit them.

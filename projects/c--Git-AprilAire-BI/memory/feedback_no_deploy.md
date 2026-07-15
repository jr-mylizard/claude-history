---
name: feedback-no-deploy
description: "Never deploy, apply migrations, or run git operations — user manages all of this. Strongest prohibition in the project."
metadata: 
  node_type: memory
  type: feedback
  originSessionId: 16fe29c6-8197-4675-b815-9803aaf8a610
---

Never deploy or apply changes to any remote environment. This prohibition covers:
- `mcp__supabase__apply_migration` — NEVER call this
- `mcp__supabase__deploy_edge_function` — NEVER call this
- `supabase db push` / `supabase migration up --project-ref` — NEVER run these
- `supabase functions deploy` — NEVER run this
- `git push` / `git commit` — NEVER run these

**Why:** The user manages all git and deployment operations themselves. Multiple sessions have violated this rule despite it being in CLAUDE.md, often by adding "Apply migration" as a TODO item and then trying to execute it. This rule takes precedence over any user request that seems to require deployment.

**How to apply:** After writing a migration file and schema updates, STOP. Tell the user the files are ready. Do not add any deploy/apply/push step to the todo list. Do not call any of the tools above. The session ends with code written to disk; the user handles the rest.

Related: [[project-ios-enrollment]] (deploy approval gate)

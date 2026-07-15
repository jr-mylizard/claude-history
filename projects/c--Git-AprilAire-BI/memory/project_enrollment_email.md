---
name: project-enrollment-email
description: enrollment-summary.html path change and Deno.readTextFile bundling hazard follow-up
metadata: 
  node_type: memory
  type: project
  originSessionId: a67b079d-bb60-4c6e-ad0e-ae7cb2626126
---

`enrollment-summary.html` is loaded at runtime by `buildEnrollmentEmail.ts` via `Deno.readTextFile`. It was moved from `email-templates/` (hyphen) to `email_templates/` (underscore) with the path updated. The old `email-templates/` directory is gone.

**Why:** The file is genuinely needed at runtime, not just build time, so it can't be simply removed.

**Hazard:** `Deno.readTextFile` on a relative path is a deploy-bundling risk — the file may not be present in the deployed edge function bundle. CLAUDE.md warns about this pattern.

**How to apply:** When touching `buildEnrollmentEmail.ts` or anything in `email_templates/`, flag this hazard. The right fix is to inline the HTML as a template literal in the TypeScript file, eliminating the runtime file read. This was deferred as more surgery than the original task called for — treat it as a worthwhile follow-up when that file is next touched.

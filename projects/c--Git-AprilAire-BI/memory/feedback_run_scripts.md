---
name: feedback-run-scripts
description: User is OK with Claude running local scripts and commands in PowerShell during iterative development tasks. No deployment.
metadata: 
  node_type: memory
  type: feedback
  originSessionId: 16fe29c6-8197-4675-b815-9803aaf8a610
---

During iterative development tasks (running Python scripts, testing, debugging), Claude may run commands directly in PowerShell without asking first.

**Why:** Speeds up iterative loops (script tweaks, test runs) without back-and-forth.

**How to apply:** Run local scripts, Python, git status/diff, file edits freely. The no-deploy rule still applies absolutely — no git push, git commit, supabase deploy, apply_migration, etc.

Related: [[feedback-no-deploy]]

---
name: feedback-session-titles
description: User uses the first message of a session as a title — do not take action on it
metadata: 
  node_type: memory
  type: feedback
  originSessionId: 54f6bee0-a43b-40c5-9ab5-de342fd8a507
---

When the first message of a session looks like a short title or label (e.g., "AprilAire (title)", "MyLizard — Sprint 12"), do NOT try to interpret it as a task or explore the codebase. Just acknowledge and wait for the user's actual instructions.

**Why:** Claude Code displays the first message as the session title in the UI. The user types a descriptive title for their own organization, not as a prompt to act on.

**How to apply:** If the first message is short (1–2 lines), contains no verb or imperative, and reads like a label/title, respond with a brief "Ready when you are" or similar and stop. Do not call any tools.

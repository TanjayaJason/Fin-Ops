---
name: finops-data-qa
description: Answer questions about a signed-in Fin Ops user's transactions through validated read-only query plans.
---

# Finance Data Q&A Skill

1. Map the Indonesian question to a supported read intent and period.
2. Return a structured plan, never SQL. Reject unsupported or unclear requests.
3. Let the backend apply the session user ID, limits, and query validation.
4. Summarize only the returned data, show the relevant period, and acknowledge missing data.
5. Never expose another user's records or modify transactions through this workflow.

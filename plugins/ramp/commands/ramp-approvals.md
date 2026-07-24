---
description: Surface and clear your pending Ramp approval queue (transactions, bills, reimbursements, requests).
---

# /ramp-approvals

Review and act on everything awaiting your approval in Ramp.

Use the `ramp-approval-dashboard` skill to:

1. Fetch every pending queue and paginate each to completion — pending transactions, bills,
   reimbursements, and requests (POs / fund requests).
2. Present a single combined queue sorted by highest dollar amount first, grouped by type,
   with a total. Convert amounts correctly (transactions are formatted strings, bills and
   reimbursements are numeric dollars).
3. For any item the user wants to act on, fetch full details first, then confirm before
   executing. Rejections must include a clear reason. Never blind-approve, and confirm
   before any bulk action.

Follow the steps and tool/command syntax defined in the `ramp-approval-dashboard` skill, and the
`ramp-safety` guardrails for all write actions.

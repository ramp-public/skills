# Safety and permissions

Ramp tools can act on live financial data. Agents must distinguish read-only analysis from writes that approve, reject, submit, pay, or modify records.

This page describes two different things: what Ramp's skills instruct an agent to do, and what Ramp enforces regardless of the agent. Skill guidance is not a technical control — a model can ignore it.

## What the skills instruct

- Show the affected records and material details before acting.
- Ask for explicit confirmation immediately before a write.
- Never infer approval for a bulk action from approval of one item.
- Require a reason when rejecting an item.
- Report partial failures instead of silently retrying or skipping records.

Individual skills may define stricter requirements for their workflows. Follow those requirements in addition to this baseline.

## What Ramp enforces

- Every operation runs with the authenticated user's role and OAuth scopes. Employees may only see their own data, while administrators and business owners may have broader access. A skill cannot elevate those permissions.
- Every tool call requires a `rationale` describing why the operation is being performed. The rationale is part of the account's audit trail and should be specific to the user's request.

Ramp tools are reachable without loading a skill — for example through the MCP connector alone. In that case the guidance above is absent, so keep your client's per-tool-call confirmation prompts enabled for write operations.

## Data accuracy

- Paginate list operations to completion before claiming a queue or report is complete.
- Preserve currencies and units when displaying or totaling amounts.
- Distinguish card transactions from bills and vendors from merchants.
- Fetch current details before performing an approval or other state-changing action.

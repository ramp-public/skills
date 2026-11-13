---
name: ramp-manage-procurement
area: Procurement
supported_surfaces: [cli, mcp]
description: |-
  Search, inspect, track, and safely approve submitted procurement requests and
  purchase orders, and issue a purchase order directly for already-decided spend.
  Use when: 'find a PO', 'show procurement request details', 'purchase order
  status', 'what procurement requests need approval', 'approve this PO request',
  or 'issue a PO for this bill'. Do NOT use to create, continue, edit, review before
  submission, or submit a draft request (use ramp-submit-procurement-request), to
  track or award a sourcing event or RFP (use ramp-manage-sourcing), or for
  a cross-product approval queue (use ramp-approval-dashboard).
---

# Manage Procurement

Use this skill for purchase order lookup, unified procurement request lookup,
status tracking after submission, pending procurement approvals, and safe
request approval or rejection.

For a new or existing draft that the user wants to fill, review before
submission, or submit, use `ramp-submit-procurement-request`. For an approval queue
that spans requests, bills, reimbursements, or transactions, use
`ramp-approval-dashboard`.

Do not use it for bill approval/payment, card transaction cleanup,
reimbursements, vendor document upload, accounting recoding, or drafting
contract edits.

## Rules

- Use `purchase_orders` for PO lookup/detail. Use `requests` for unified request
  search, request detail, pending approval queue, approval, and rejection.
- Keep identifiers labeled. `purchase_order_id`, `unified_request_id`, and
  `spend_request_id` are distinct. Approval acts on
  the unified request UUID, not the PO UUID.
- Use `--agent` for responses you need to parse.
- Use `--json` for search filters. The CLI validates unknown JSON keys and enum
  values, so use `--dry_run --json ...` when checking payload shape.
- Use exact pagination cursors returned by the API. Do not trim or rewrite them.
- Procurement schemas do not return deep-link URLs. Present returned identifiers
  and statuses instead of inventing links.
- Do not approve or reject without first showing details and confirming the
  user's intent.
- When `requests get` returns a non-null `original_request`, treat the request as
  a change request; before approval, verify that `original_request` identifies the
  intended original approved request and review every entry in
  `change_request_diff` fields, including each old and new value. Stop if the
  source or diff is absent or unexpected.
- A `LINE_ITEM` diff includes only changed values. Overlay a present `new_value`
  on `old_value`: a null property means unchanged; the whole `new_value` is null
  when the line was removed. When a line item's UUID matches the original, its
  tracking categories, custom fields, withholding rates, and external IDs
  carried over unchanged; do not report them as changes.
- For rejection, include the user-supplied or user-accepted reason in
  `--reason`.
- PO amount fields are numeric currency units with a `currency` code. Display
  them directly; do not apply cents conversion.

## Workflow

1. Search POs first when the user gives a PO number, vendor, or procurement
   record. `filters.search` matches PO numbers, request names, and owner names;
   it does not match vendor names. For a vendor search, resolve the vendor UUID
   first, then filter POs by `payee_uuids`:

   ```bash
   ramp vendors search --search_term "Figma" --limit 10 --rationale "Find the vendor UUID for the purchase order search" --agent
   ramp purchase_orders search --json '{"rationale":"Searching purchase orders for the resolved vendor","filters":{"payee_uuids":["{vendor_uuid}"]},"limit":10}' --agent
   ```

   For a PO number, request name, or owner name, use `filters.search` directly:

   ```bash
   ramp purchase_orders search --json '{"rationale":"Searching purchase orders by PO number","filters":{"search":"PO-1234"},"limit":10}' --agent
   ```

   If a known PO number is not returned, the spend request may still be pending
   and the PO may not be issued yet. Search unified requests scoped to purchase
   orders:

   ```bash
   ramp requests search --json '{"rationale":"Searching purchase order requests before PO issuance","filters":{"search":"PO-1234","unified_spend_request_types":["PURCHASE_ORDER"]},"limit":10}' --agent
   ```

   Unified request search and pending rows include `unified_request_id` and may
   include `purchase_order_number`, but they do not include
   `purchase_order_id`. To move from a unified request row to PO detail, first
   call `requests get` with `unified_request_id` and use `purchase_order_id`
   from that detail response if it is present.

2. Get detail before summarizing or taking action:

   ```bash
   ramp purchase_orders get {purchase_order_id} --rationale "Reviewing purchase order details" --agent
   ramp requests get {unified_request_id} --rationale "Reviewing request before action" --agent
   ```

   The unified request identifier is named `unified_request_id` in request
   search, pending, detail, approve, and reject responses. Do not call
   `purchase_orders get` directly from a request search or pending row; list
   rows do not expose `purchase_order_id`.

3. Review pending approvals through the request queue:

   ```bash
   ramp requests pending --rationale "Reviewing pending procurement approvals" --page_size 50 --request_types PURCHASE_ORDER --agent
   ```

   Paginate with the returned cursor:

   ```bash
   ramp requests pending --rationale "Continuing pending procurement approval review" --page_size 50 --request_types PURCHASE_ORDER --start "{next_page_cursor}" --agent
   ```

4. Before approval or rejection, check the current approval step:

   ```bash
   ramp requests get {unified_request_id} --rationale "Pre-approval workflow check" --agent
   ```

   If `approval_workflow.needs_user_action` is not `true`, do not call
   `requests approve`; hand off with the identifiers and the current approval
   step from `approval_workflow.steps`.

   If `original_request` is non-null, also show its original spend request
   UUID, unified request UUID, and PO number, followed by the complete
   `change_request_diff` fields and old/new values. Confirm that the source is the
   original approved request the user intended to amend and that the diff
   contains only the expected changes. Do not infer a change request from names or
   amounts when `original_request` is null.

5. Act only after the user confirms the exact `unified_request_id`:

   ```bash
   ramp requests approve {unified_request_id} --action APPROVE --rationale "Approving confirmed purchase order request" --agent
   ramp requests approve {unified_request_id} --action REJECT --rationale "Rejecting confirmed purchase order request" --reason "{confirmed_reason}" --agent
   ```

## Issue a purchase order directly

Issue directly only when the spend is already decided: a bill with no PO, a signed
contract, a purchase approved outside Ramp, or a copy of an existing PO. For new
spend that needs approval, submit a request with `ramp-submit-procurement-request`.
These commands need the Issue purchase orders permission and may not be offered to
every user.

- `ramp purchase_orders issue`: a new PO from scratch.
- `ramp purchase_orders issue-for-bill`: copies a bill or draft bill and matches each
  PO line to its bill line.
- `ramp purchase_orders issue-copy`: copies an existing PO; fields you pass replace the
  copied ones.

Resolve the vendor and entity first:

```bash
ramp vendors search --search_term "Acme" --limit 10 --rationale "Find the vendor for the PO" --agent
ramp business list --rationale "Find the entity for the PO" --agent
```

A new PO from scratch looks like this:

```bash
ramp purchase_orders issue --json '{"rationale":"Issuing the PO the user confirmed","vendor_uuid":"{vendor_uuid}","entity_uuid":"{entity_uuid}","currency":"USD","name":"Acme analytics","line_items":[{"description":"Platform license, 12 months","unit_price":"4800.00"},{"description":"Implementation hours","unit_price":"150.00","quantity":"10"}]}' --agent
```

Leave `owner_uuid` out when the user owns the PO; it defaults to the caller. For other
fields, check the command's `--help` (or `ramp tools schema`) rather than guessing.

Issuing creates a live, approved PO that can go to the vendor and sync to the ERP.
Before running an issue command, show the vendor, entity, owner, lines and total, and
get the user's confirmation. If a command fails without a clear error, search POs
before retrying so you don't issue a duplicate.

On errors:

- Missing fields: ask the user for the listed fields, then issue again.
- Verification required: give the user the returned link to review and issue the PO in
  Ramp. Nothing was issued.

To keep a quote or contract with the PO (terminal agents such as Claude Code, Cursor
or Codex, with the file on disk):

1. After the user confirms the PO, request a one-time upload link
   (`request-issue-purchase-order-document-upload-link` on MCP):

   ```bash
   ramp purchase_orders upload-link --rationale "Uploading the quote to keep with the PO" --agent
   ```

2. Run the returned `curl` command with the local file path. Each link is single use
   and expires shortly, so request one per file. Never give the URL to the user.
3. Pass the returned `document_id` to the issue command. The document stays internal
   and is never sent to the vendor.

Without a shell, ask the user to attach the document on the Issue purchase order form
in Ramp.

## Output

For search results, keep rows compact and use fields from the command response:

```text
PO number | vendor | amount | currency | PO status | id
```

For detail summaries, include available status and relationship fields:

```text
PO number:
Vendor:
Amount:
PO status:
Request status:
Change-request source:
Change-request diff:
Promise date:
Linked bills:
Linked transactions:
Linked item receipts:
Unified request ID:
```

Before approval, state that approval acts on the unified request UUID, not the
PO ID. For change requests, include the original approved request identifiers and
old/new diff in the confirmation prompt. After action, surface returned
`unified_request_id`, `action`, `success`, and `message`.

## Handoff

Hand off instead of guessing when the CLI cannot complete the workflow, the
current user is not the active approver, the user asks for unsupported edits, or
the task belongs to bills, reimbursements, transactions, vendor documents, or
contracts.

Include the identifiers and statuses you have:

```text
PO number:
Purchase order ID:
Unified request ID:
Spend request ID:
Request status:
Current approval step:
```

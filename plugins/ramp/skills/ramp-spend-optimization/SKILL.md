---
name: ramp-spend-optimization
area: Cards and Spend
supported_surfaces: [cli, mcp]
description: |-
  Find credible potential savings and account-control opportunities across Ramp spend,
  explain the evidence and confidence, and safely configure supported fund controls after
  an exact preview and explicit approval. Use when: "optimize my Ramp account", "find
  savings", "clean up recurring vendor spend", "consolidate vendor spend", "duplicate
  subscriptions", "oversized funds", or "move recurring vendor spend onto a controlled
  card". Do NOT use for a general spend report (use ramp-spend-analysis), expense cleanup
  or missing receipts (use ramp-complete-expenses), approvals (use ramp-approval-dashboard),
  subscription cancellation, or vendor-site payment changes.
---

# Spend Optimization

## Non-Negotiables

- **Pass `--rationale` on every command.** It is required on all agent-tool calls. Omitting it returns `HTTP 422`.
- **Pass `--agent`** for machine-readable JSON output on all CLI commands.
- **Read-only first.** The analysis phase requires no user confirmation. Action requires explicit approval of an exact preview.
- **Potential savings only.** Frame every dollar estimate as potential savings backed by observed charges. Never claim guaranteed savings, additional Ramp rewards, or vendor-negotiated discounts without an authoritative, account-specific source.
- **Missing products are normal.** If the account does not use Bill Pay or Procurement, continue with available card, fund, and transaction data. Do not treat missing products as setup problems or upsell requirements.
- **Separate currencies.** Do not add amounts across currencies unless an authoritative conversion method and conversion date are available and disclosed.
- **Account for refunds.** Negative transaction amounts are refunds. Subtract them from the vendor's observed spend rather than presenting them as separate savings.
- **Cleared spend only.** Base savings calculations on cleared or completed activity. Exclude declined, deleted, and draft items.
- **Paginate to completion.** A single page may not return everything. Repeat with `--cursor` / `cursor` using the next cursor value from the prior response until it is null.
- **Secure vendor-side handoff.** When directing the customer to update a vendor's payment method, tell them to retrieve the new virtual card credentials through the Ramp app or another secure first-party surface. Never ask the customer to paste full card details, OTPs, or credentials into this chat.

## Runtime Capability Check

Before starting, discover which tools are available:

1. **Analyst query** — `ramp analyst query` / `ExecuteAnalystQuery`. Required for account-wide spend correlation. If unavailable, explain that the account-wide optimization scan cannot be completed reliably and offer per-vendor analysis using transaction and bill listing instead.
2. **Fund listing** — `ramp funds list` / `GetUserFunds`. Required for fund-level analysis.
3. **Merchant search** — `ramp merchant search` / `SearchMerchants`. Required before any merchant-restriction write. If unavailable, merchant-restriction actions remain advisory.
4. **Fund creation** — `ramp funds create` / `IssueOneOffFunds`. Required for vendor-restricted fund creation. This tool does not accept an idempotency key; a retry after an ambiguous failure (timeout, network error) can create a duplicate fund and virtual card. See Safety Gate 1 for the required pre- and post-creation duplicate checks.
5. **Merchant restrictions** — `ramp funds update-merchant-restrictions` / `UpdateMerchantRestrictions`.
6. **Category restrictions** — `ramp funds update-category-restrictions` / `UpdateCategoryRestrictions`.
7. **Periodic limit increase** — `ramp funds edit` / `LimitIncrease`. Increase-only: sets a periodic limit to a higher positive value. Requires explicit user confirmation of the increase and choice of temporary vs permanent treatment. Cannot decrease a periodic limit; lowering a periodic limit is advisory/manual until a supported decrease action exists.
8. **Per-transaction limit** — `ramp funds update-transaction-amount-limit` / `UpdateTransactionAmountLimit`.
9. **Fund lock/unlock** — `ramp funds lock` / `LockOrUnlockSpendAllocation`.

If any write tool is unavailable, remain advisory for that action type and tell the user the exact manual steps.

## Analysis Workflow

### Step 1: Gather account-wide spend data

Use the analyst query to pull account-wide spend, grouping by vendor, payment route (card vs bill), and time period. Request at least the last six months of cleared activity.

Before writing any analyst SQL, call `ramp analyst catalog` / `GetAnalystCatalog` once to discover available tables and starter SQL. Then read domain docs for every physical `analyst.*` table that may appear in the query using `ramp analyst spend-docs` / `GetAnalystSpendFactsDomainDocs` and `ramp analyst table-docs` / `GetAnalystTableDomainDocs`. Do not generate SQL until documentation for all candidate tables has been read.

CLI:

```bash
ramp analyst query --sql "<analyst SQL query for spend by vendor and route>" \
  --agent --rationale "Account-wide spend analysis for optimization scan"
```

MCP:

```text
ExecuteAnalystQuery(sql="<analyst SQL query>", rationale="Account-wide spend analysis for optimization scan")
```

If analyst query is unavailable, fall back to paginated `ramp transactions list` and `ramp bills search` (see `ramp-spend-analysis` for exact pagination workflow). Explain that the analysis is limited to these data sources.

### Step 2: List existing funds

```bash
ramp funds list --funds_to_retrieve ALL_FUNDS \
  --include_balance --include_lock_info --include_members --include_restrictions \
  --page_size 100 --agent --rationale "List all funds with details for optimization review"
```

Paginate with `--cursor` using the next cursor from the prior response until it is null. Record each fund's UUID, display name, interval, limit, balance, lock state, merchant/category restrictions, and member list.

### Step 3: Cross-reference and rank opportunities

Apply the opportunity detection rules below. Rank findings by:

1. Credible potential savings (highest first).
2. Confidence level.
3. Customer impact or exposure reduction.
4. Actionability with available tools.

Return only the top three to five findings. Omit low-confidence findings unless the user explicitly requests a deeper scan.

## Opportunity Detection Rules

### Type 1: Potential Duplicate or Overlapping Recurring Spend

Look for repeated charges that plausibly represent spend the customer could eliminate.

**High confidence** — all of:
- Same normalized vendor name and currency.
- Charges occur within seven days of each other in each billing cycle.
- Amounts within 2% of each other.
- Pattern repeats across at least two billing cycles.

**Medium confidence** — any of:
- Amounts within 10% (but not 2%).
- Less precise billing-period alignment.
- Still requires at least two billing cycles.
- State the missing evidence that prevents high confidence.

Savings scope:
- **Observed overlap** — the suspicious charges already recorded. Count only the charges that plausibly disappear, never the customer's total vendor spend.
- **Forecast potential savings** — observed recurring overlap projected forward. Require at least three consistent cycles before annualizing.

State explicitly that separate subscriptions, teams, seats, products, and valid separate payment routes may be legitimate.

### Type 2: Bill-and-Card Fragmentation

The same vendor is paid through both bills and card transactions.

This is an account-optimization opportunity. Do not assign a dollar-savings value unless a separate authoritative source supports that amount. Recommend consolidating future payments under one owner and one controlled payment route.

### Type 3: Recurring Vendor Spend Without Dedicated Controls

Recurring vendor charges use a general-purpose card or fund with no vendor-restricted fund evident.

Recommend a dedicated vendor fund with an appropriate periodic limit, merchant restriction, owner, and per-transaction cap. Frame this as reducing exposure and improving ownership clarity, not as direct savings.

### Type 4: Dormant or Oversized Recurring Funds

**Dormant:** A recurring monthly fund is dormant when:
- It is active (not locked or terminated).
- It was created at least 90 days ago.
- It has no cleared spend during the most recent 90 days.

Do not classify one-time or newly created funds as dormant in V1.

**Oversized:** A recurring monthly fund is oversized when:
- At least three complete months of history exist.
- The current monthly limit is more than twice the highest cleared monthly spend during those months.

Suggested new limit: the highest observed monthly spend plus a 20% buffer. If history is insufficient, seasonal, or irregular, ask the user to choose the amount rather than presenting a false precision. Lowering the periodic fund limit is not supported by the available tools; recommend the target limit and tell the user to apply it through the Ramp dashboard. If the per-transaction cap is also oversized, that can be lowered via `UpdateTransactionAmountLimit`.

## Response Contract

Every numbered finding must include:

1. **Finding** — what was detected and the opportunity type.
2. **Confidence** — `High` or `Medium`, with the reason.
3. **Evidence window** — the date range and spend routes examined.
4. **Observed overlap** — when applicable, the charges that represent the potential waste.
5. **Forecast potential savings** — when supportable (at least three consistent cycles), the annualized projection.
6. **Calculation and assumptions** — how the dollar figures were derived.
7. **Recommended next action** — what the agent can do and what remains manual.
8. **What this will not do** — explicit limitations.
9. **Remaining customer/vendor-side work** — steps the agent cannot perform.

### Follow-up commands

- `explain <N>` — show the underlying dates, amounts, payment routes, fund ownership, and limits.
- `fix <N>` — for duplicate spend, explain the suspected overlap and provide vendor-side review/cancellation guidance. For fund actions, proceed to the preview and approval workflow. "Fix" never means the agent canceled a subscription.
- `ignore <N>` — acknowledge and skip the finding.

## Supported Actions

The following writes are available when the connected user is authorized and the live tool exists:

| Action | CLI | MCP |
|---|---|---|
| Create a vendor-restricted fund | `ramp funds create` | `IssueOneOffFunds` |
| Add merchant restrictions to a fund | `ramp funds update-merchant-restrictions` | `UpdateMerchantRestrictions` |
| Add category restrictions to a fund | `ramp funds update-category-restrictions` | `UpdateCategoryRestrictions` |
| Increase a periodic fund limit | `ramp funds edit` | `LimitIncrease` |
| Set a per-transaction limit | `ramp funds update-transaction-amount-limit` | `UpdateTransactionAmountLimit` |
| Lock or unlock a fund | `ramp funds lock` | `LockOrUnlockSpendAllocation` |

**Not supported in V1:** subscription cancellation, vendor-site payment updates, fund deletion or termination, periodic fund-limit decreases, fund interval changes, Procurement configuration, purchase-order cleanup, reimbursement actions, or approval actions.

Removing restrictions and unlocking funds are rollback/restoration actions only, not optimization recommendations. They require a full preview and separate explicit approval.

## Safety Gates

Before previewing any action:

1. **Duplicate fund check and creation safety.** Before proposing fund creation, resolve the target merchant via `SearchMerchants` to obtain both its integer `id` and `uuid`. Then use the full fund listing from Step 2 (which includes `--include_restrictions`) and compare the merchant's integer `id` against every existing fund's `vendor_whitelist` entries — these are integer merchant IDs, not UUIDs. Do not rely on display-name search alone — a fund named "Marketing subscriptions" with Asana-only restrictions would be missed by a name filter. If any existing fund already restricts to the target merchant ID, do not propose a new fund; recommend using the existing one instead.

   **Retry safety:** `IssueOneOffFunds` does not accept an idempotency key. If a creation call returns an error or times out without a confirmed fund UUID, do **not** retry blindly. Instead, re-list all funds with `--include_restrictions` and check whether a fund with the target merchant ID in its `vendor_whitelist` was created. If a duplicate is found, report both funds and remain advisory for cleanup since fund deletion is not supported. Only retry creation after confirming the first attempt did not persist.
2. **Merchant resolution.** Before any merchant-restriction write, resolve the merchant using `ramp merchant search` / `SearchMerchants`. The result returns both `id` (integer) and `uuid` (string) per merchant. Use the integer `id` when comparing against existing fund `vendor_whitelist`/`vendor_blacklist` restrictions (which contain integer merchant IDs). Use the `uuid` when calling `IssueOneOffFunds` or `UpdateMerchantRestrictions` (which accept merchant UUIDs). If reliable merchant lookup is unavailable, remain advisory.
3. **Pre-lock/unlock checks.** Both locking and unlocking a fund require an exact preview and explicit user approval. Before presenting a lock or unlock preview, show the exact fund and all affected members.
   - **For locking:** explain that locking will block all transactions on the fund until it is unlocked. Before presenting the lock preview, verify:
     - **Pending transactions.** Query pending charges on the fund using `ramp transactions list` / `GetTransactions` with `--transactions_to_retrieve all_transactions_across_entire_business`, `--state pending`, and a `spend_allocation` filter for the target fund. Paginate to completion. If any pending charges exist, report them and remain advisory.
     - **Annual or irregular renewals.** Review at least 13 months of cleared transaction history on the fund to identify annual or irregular charges that would be disrupted by locking. If renewal evidence cannot be established from available data, remain advisory and ask the user about known renewals.
     - Which members are affected.
     - A supported recovery path exists (unlock via the same tools, with its own approval).
     - If pending activity or renewal evidence cannot be established, remain advisory and explain the risk.
   - **For unlocking:** explain that unlocking will re-enable spending for all fund members. Show the fund's current lock state, restrictions, limit, and member list. Require explicit user approval before proceeding. Unlock is a rollback/restoration action only — do not recommend it as an optimization.
4. **Periodic limit increase protocol.** `LimitIncrease` is an increase-only tool. Before calling it:
   - Preview the exact fund, current limit, and proposed new limit.
   - Ask the user to explicitly confirm the increase.
   - Ask the user to choose `temporary` (resets at the next interval boundary) or `permanent`.
   - Pass `user_confirmed_limit_increase: true` and `user_confirmed_increase_type: true` only after both confirmations.
   - Lowering a periodic fund limit is not supported by this tool. For oversized funds, recommend the lower limit and tell the user to apply it manually through the Ramp dashboard.
5. **Limit calculation.** Use observed peak recurring usage plus a 20% buffer only when at least three representative months exist. For seasonal or irregular spend, ask the user to choose the limit.

## Preview, Approval, and Verification

### Preview

Every write preview must show:

- **Account object** — exact fund name, ID, owner, and affected members.
- **Proposed values** — amount, interval, merchant/category restrictions, per-transaction cap.
- **Calculation** — how the values were derived from observed data.
- **Expected improvement** — what changes after the action.
- **Limitations** — what the action will not do.
- **Vendor-side steps** — any remaining customer/vendor work.
- **Reversibility** — whether the change is reversible with currently available tools, and the exact recovery action.

### Approval

- `Fix it` or `do it` is **not** approval until exact values have been previewed.
- Any change to the owner, member list, amount, interval treatment (temporary vs permanent), restriction, or cap requires a revised preview and new approval.
- The user must confirm the exact previewed values before the agent acts.

### Pre-action re-read

Immediately before acting, re-read the affected account state using the exact fund UUID:

```bash
ramp funds list --fund_uuid "<target-fund-uuid>" \
  --include_balance --include_lock_info --include_members --include_restrictions \
  --agent --rationale "Pre-action state check before approved change"
```

For fund creation, re-read the full fund list with `--include_restrictions` and compare the target merchant's integer `id` against all existing `vendor_whitelist` entries to confirm no matching vendor-restricted fund was created since the preview.

For lock actions, also repeat the pending-transaction query immediately before locking:

```bash
ramp transactions list \
  --transactions_to_retrieve all_transactions_across_entire_business \
  --state pending --filters '{"filters": [{"filter": "<fund-name>", "filter_type": "spend_allocation"}]}' \
  --agent --rationale "Pre-lock pending-transaction recheck"
```

If any pending charges appeared since the initial check, **stop** and issue a revised preview.

If the owner, members, limits, restrictions, or other previewed values differ from the preview, **stop** and issue a revised preview. Do not apply a stale preview.

### Execution

Execute only the approved action(s), in the previewed order. When a preview contains multiple ordered steps (for example, fund creation followed by per-transaction cap), verify after each step and stop with `Failed or partial change` if any step fails.

### Post-action verification

After acting, read the resulting state using the exact fund UUID:

```bash
ramp funds list --fund_uuid "<target-fund-uuid>" \
  --include_balance --include_lock_info --include_members --include_restrictions \
  --agent --rationale "Post-action verification of approved change"
```

Compare the result to the approved preview. A successful write response alone is not sufficient proof. The verified state — including restrictions and per-transaction cap — must match. If the write response indicates the change is pending internal Ramp approval (e.g. `request_pending`), the fund will correctly retain its old values until approved — this is not a failure; see the `Pending Ramp approval` completion state.

### Completion states

Report exactly one:

- **Ramp configured** — the approved Ramp control exists and is verified. Customer or vendor-side work remains.
- **Vendor migration pending** — the new Ramp route is ready. The vendor payment method has not been confirmed.
- **Optimization complete** — the Ramp control is verified and every required external step has evidence of completion.
- **Pending Ramp approval** — the write was accepted but requires internal Ramp approval before taking effect. The fund retains its current values until the request is approved. Report the pending request and do not treat the old values as a verification failure.
- **Failed or partial change** — the resulting state does not fully match the approved preview and no pending-approval request explains the difference. Report the exact resulting state.

Do not report the account as optimized while an external migration remains incomplete.

## Example Session

```
User: Optimize my Ramp account.

Agent: Let me scan your account for optimization opportunities.
> ramp analyst catalog --agent --rationale "Discover analyst tables for optimization scan"
> ramp analyst spend-docs --agent --rationale "Read spend facts domain docs"
> ramp analyst query --sql "..." --agent --rationale "Account-wide spend analysis for optimization scan"
> ramp funds list --funds_to_retrieve ALL_FUNDS \
>   --include_balance --include_lock_info --include_members --include_restrictions \
>   --page_size 100 --agent --rationale "List all funds with details for optimization review"
> [paginate with --cursor until null]

I found 4 opportunities, ordered by potential impact:

1. **Recurring vendor spend without dedicated controls — Asana**
   Confidence: High
   Evidence: 6 months of monthly charges averaging $12,100 on a general-purpose
   Marketing card. No vendor-restricted Asana fund exists.
   Recommended action: Create a dedicated Asana fund with a $15,000 monthly limit
   (peak month of $12,500 + 20% buffer), Asana merchant restriction, a single
   owner, and a $15,000 per-transaction cap.
   This will not: update the payment method in Asana, cancel any existing payment
   route, or delete the fund if created in error.
   Remaining work: After fund creation, retrieve the new virtual card credentials
   through the Ramp app and update the payment method inside Asana. Confirm a
   successful charge before cleaning up the old payment route.

2. **Potential duplicate recurring spend — Zoom**
   Confidence: High
   Evidence: Two Zoom charges of $499.00 and $499.00 on different cards, 3 days
   apart, same currency, repeating across 4 billing cycles.
   Observed overlap: $499.00/month × 4 months = $1,996.00
   Forecast potential savings: ~$5,988/year (projected from 4 consistent cycles)
   Calculation: One of the two charges assumed eliminable. Annualized from monthly
   cadence. Separate subscriptions, teams, or products may justify both charges.
   Recommended action: Review both Zoom subscriptions with the respective
   cardholders. If one is redundant, cancel it through the Zoom admin portal. The
   agent cannot cancel Zoom subscriptions.
   Remaining work: Customer must verify which subscription is redundant and cancel
   it directly with Zoom.

3. ...

User: fix 1

Agent: I'll create a vendor-restricted Asana fund. First, let me resolve the
Asana merchant and confirm no existing fund already restricts to this merchant:
> ramp merchant search --query "Asana" --agent --rationale "Resolve Asana merchant UUID for fund restriction"

Merchant resolved: id=42, uuid=abc123-...
Checking all existing funds from the initial scan — comparing merchant integer
ID 42 against every fund's vendor_whitelist entries.

No existing fund has merchant ID 42 in its vendor_whitelist.

**Proposed Ramp changes (2 ordered steps)**

Step 1 — Create a new fund:
- Display name: Asana
- Owner: Sarah Chen (current cardholder for Asana charges)
- Monthly limit: $15,000 (peak recent month $12,500 + 20% buffer)
- Merchant restriction: Asana only (merchant ID 42 / UUID abc123-...)
- Interval: Monthly

Step 2 — Set the per-transaction cap on the new fund:
- Per-transaction cap: $15,000

This gives Asana one owner and one controlled payment route. It does not:
- Update the payment method inside Asana
- Cancel existing bills or card routes
- Delete the newly created fund (fund deletion is not currently supported)

After both steps, retrieve the new virtual card credentials through the Ramp
app or another secure first-party surface. Do not paste full card details into
this chat. Update the payment method in Asana and confirm a successful charge
before cleaning up the old route.

This change is partially reversible: the fund can be locked but not deleted.

Approve creating this fund and setting the per-transaction cap exactly as
described?

User: Yes, approved.

Agent: Confirmed. Re-reading current state before acting...
> ramp funds list --funds_to_retrieve ALL_FUNDS --include_restrictions \
>   --agent --rationale "Pre-action state check — verify no fund with Asana merchant restriction was created"

No fund has merchant ID 42 in its vendor_whitelist. Creating the fund.

Step 1: Create fund.
> ramp funds create --display_name "Asana" --amount 15000 --interval MONTHLY \
>   --user_ids '["sarah-chen-uuid"]' --vendor_whitelist '["asana-merchant-uuid"]' \
>   --agent --rationale "Create vendor-restricted Asana fund per user approval"

Verifying step 1...
> ramp funds list --fund_uuid "<new-fund-uuid>" \
>   --include_balance --include_members --include_restrictions \
>   --agent --rationale "Verify created Asana fund"

Fund created and verified. Applying step 2.

Step 2: Set per-transaction cap.
> ramp funds update-transaction-amount-limit \
>   --spend_allocation_id "<new-fund-uuid>" --new_transaction_amount_limit 15000 \
>   --agent --rationale "Set $15,000 per-transaction cap on Asana fund per user approval"

Verifying step 2...
> ramp funds list --fund_uuid "<new-fund-uuid>" \
>   --include_balance --include_members --include_restrictions \
>   --agent --rationale "Verify per-transaction cap on Asana fund"

**Ramp configured.** The Asana fund is live with a $15,000 monthly limit,
$15,000 per-transaction cap, and Asana-only merchant restriction.
Vendor migration pending — retrieve the new virtual card credentials through
the Ramp app, update the payment method in Asana, and confirm a successful
charge before cleaning up the old payment route.
```

## Gotchas

| Issue | Fix |
|---|---|
| Duplicate vendor names | The API does not normalize merchant names. Run `ramp merchant search` and check for variants ("Asana", "Asana Inc", "Asana, Inc."). Deduplicate by merchant integer ID. |
| Refunds reduce spend | Negative transaction amounts are refunds. Subtract from vendor totals; do not count as separate savings. |
| Multiple currencies | Keep currency totals separate. Do not combine USD and EUR spend. Disclose conversions when used. |
| Incomplete Bill Pay data | Bill `amount` is the full invoice amount, not the paid allocation. Do not sum bill amounts as paid-in-period spend without payment-allocation data. |
| Missing merchant lookup | If `ramp merchant search` is unavailable, merchant-restriction actions remain advisory. Provide the manual steps. |
| Stale previews | Always re-read state immediately before acting. If anything changed, stop and re-preview. |
| Annual or irregular renewals | Before locking a fund, review at least 13 months of cleared history for annual charges. Query pending transactions with `--transactions_to_retrieve all_transactions_across_entire_business --state pending` and a `spend_allocation` filter scoped to the fund. Repeat the pending-transaction query immediately before locking. If either check cannot be completed, remain advisory. |
| Duplicate vendor funds | Before creating a fund, compare the resolved target merchant's integer `id` against every existing fund's `vendor_whitelist` entries (which are integer merchant IDs). Do not compare UUIDs against this field — the types are incompatible. |
| Fund creation retries | `IssueOneOffFunds` has no idempotency key. If a creation call errors or times out, do not retry. Re-list funds with `--include_restrictions` and check whether the target merchant ID already appears in any `vendor_whitelist` before attempting again. |
| Partial writes | If any step in a multi-step approved action fails, stop and report `Failed or partial change` with the exact resulting state. Do not claim the per-transaction cap exists if step 2 failed. |
| Pending Ramp approval | Some writes (e.g. `UpdateTransactionAmountLimit`, `LimitIncrease`) can return `request_pending` when the change requires internal Ramp approval. The fund retains its old values until approved — this is not a failure. Report `Pending Ramp approval` and do not retry. |
| Unverified vendor migration | Do not report optimization as complete while the vendor payment method has not been confirmed on the new route. |
| `LimitIncrease` is increase-only | This tool cannot decrease a periodic fund limit. For oversized funds, recommend the target limit and tell the user to apply it manually. Pass `user_confirmed_limit_increase` and `user_confirmed_increase_type` only after explicit user confirmation of the increase and temporary/permanent choice. |
| Merchant ID vs UUID | `SearchMerchants` returns both `id` (integer) and `uuid` (string). Fund `vendor_whitelist`/`vendor_blacklist` restrictions use integer IDs. `IssueOneOffFunds` and `UpdateMerchantRestrictions` accept UUIDs. Use the correct type for each operation. |
| Per-transaction vs periodic limit | `UpdateTransactionAmountLimit` sets the per-transaction cap (supports increases and decreases). `LimitIncrease` sets the periodic total (increases only). `default_member_limit_amount` on fund creation is a per-member limit, not a per-transaction cap. Do not confuse the three. |
| Something broken? | With the user's consent, run `ramp feedback "<user-approved message>"`. This sends only that message to Ramp support; omit secrets and diagnostic artifacts. |

## When NOT to Use

- **General spend report** — use `ramp-spend-analysis`
- **Missing receipts or memos** — use `ramp-complete-expenses`
- **Approving transactions** — use `ramp-approval-dashboard`
- **Card lock/unlock or status** — use `ramp-card-management`
- **Cancel a subscription** — unsupported vendor-side action; the agent cannot do this
- **Set up Procurement** — use `ramp-manage-procurement`
- **Verifying a single payment** — use `ramp-payment-lookup`

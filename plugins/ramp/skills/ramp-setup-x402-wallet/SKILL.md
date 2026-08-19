---
name: ramp-setup-x402-wallet
area: Agentic Commerce
supported_surfaces: [browser, cli, mcp]
description: >-
  Set up and fund a Ramp x402 wallet from a Ramp Checking account. Use when an
  owner or admin of a company's Ramp account asks to enable, provision, fund,
  or prepare x402 payments. For an already-funded wallet and a specific x402
  purchase, use ramp-make-x402-payment instead.
compatibility: Requires a user who is an owner or admin of the company's Ramp account with Ramp Checking access, an up-to-date Ramp CLI or Ramp MCP connection, Python 3, and browser access for OAuth and any required withdrawal approval.
---

# Set up a Ramp x402 wallet

Provision the business's Ramp-managed Solana wallet and fund it from a Ramp
Checking account. Ask one question at a time and never guess an account, amount,
permission, or approval state.

## Safety rules

- Provisioning and funding are separate mutations. Get explicit confirmation
  immediately before each one.
- Never ask for passwords, one-time codes, banking credentials, private keys, or
  wallet signing material. The user completes Ramp sign-in and approvals in
  Ramp-controlled browser pages.
- Every Ramp agent-tool call needs a non-empty `rationale`.
- Use only an account UUID returned by `treasury accounts`; never infer or reuse
  an identifier from another source.
- Funding is an irreversible transfer from Ramp Checking. Use a new idempotency
  key for every new transfer. A repeated funding call with the same key does not
  replay the original result; reconcile an uncertain call before proceeding.

## 1. Update and connect Ramp

### Ramp CLI

Inspect the installed CLI:

```bash
ramp --version
ramp update --help
```

If the CLI is missing, stop and help the user install it through the trusted
instructions at https://agents.ramp.com/docs/cli/overview. If an update is
available, explain that it changes the local CLI installation and ask permission
before running:

```bash
ramp update
```

Authenticate in the user-controlled browser and verify the session:

```bash
ramp auth login
ramp auth status
```

Refresh the generated tool specification, then inspect the live CLI catalog:

```bash
ramp tools refresh
ramp tools list
```

Confirm that the refreshed CLI exposes `treasury accounts`, `x402 create`, and
`x402 fund`. Tool presence does not override account permissions; the calls
below remain the source of truth for access.

### Ramp MCP

Have the user update or reconnect the Ramp MCP connector using their MCP
client's connection settings, complete Ramp OAuth in the browser, and refresh
the client's tool list. Confirm that the connected tool list includes the Ramp
equivalents of:

- List Ramp Checking accounts (`treasury accounts`)
- Provision an x402 wallet (`x402 create`)
- Fund an x402 wallet (`x402 fund`)

If any required tool is absent, disabled, or returns an authorization,
permission, scope, or availability error, stop. Tell the user:

```text
Ramp x402 setup is not available with your current account access. Please reach
out to your Ramp account team or Ramp Support to request access, then reconnect
Ramp and try again.
```

Do not expose internal rollout names or recommend broader permissions as a
workaround.

## 2. Provision the x402 wallet

Explain that Ramp will create a business x402 wallet on Solana and manage its
signing securely. Ask:

```text
Are you an owner or admin of the company's Ramp account, and do you confirm
that I should provision the company's Ramp x402 wallet?
```

Only after the user confirms their company-level role and provisioning, call the
Ramp MCP tool or run:

```bash
ramp x402 create --confirmed \
  --rationale "User confirmed they are an owner or admin of the company's Ramp account and approved provisioning its x402 wallet"
```

Record the returned wallet address and require `status` to be `sign_ready`.
Repeated calls return the same wallet, but still require confirmation. If Ramp
reports that provisioning is unavailable or the business setup is incomplete,
stop and relay the actionable error without retrying.

## 3. Choose a Ramp Checking account and amount

List every Ramp Checking account visible to the user:

```bash
ramp treasury accounts --agent \
  --rationale "List Ramp Checking accounts and available balances for x402 wallet funding"
```

Present each returned account's `name`, `status`, `available_balance`,
`currency`, and `provider`. Do not display account numbers or unrelated fields.

An eligible funding source must be:

- A Ramp Checking account returned by this call
- Active
- Denominated in USD
- Provided by Increase
- Funded with an `available_balance` at least as large as the requested amount

If there is no Ramp Checking account, stop and say:

```text
Funding a Ramp x402 wallet requires a Ramp Checking account. Please open a Ramp
Checking account in Ramp, then return here to continue setup.
```

If eligible accounts exist, let the user select the exact account and USD
amount of at least $1. Never choose for them. Reject amounts below $1 or above
the selected account's available balance before confirmation. Show a funding
preview:

```text
From: <Ramp Checking account name>
Available balance: <formatted USD balance>
To: Ramp x402 wallet <wallet address>
Amount: <formatted USD amount>
```

Ask for explicit confirmation of those exact details.

## 4. Fund the wallet

After confirmation, generate one UUID locally for this transfer:

```bash
IDEMPOTENCY_KEY="$(python3 -c 'import uuid; print(uuid.uuid4())')"
```

Call the Ramp MCP funding tool with the selected account UUID, exact amount, and
idempotency key, or run:

```bash
ramp x402 fund "<source_wallet_account_uuid>" \
  --amount "<confirmed_usd_amount>" \
  --idempotency_key "$IDEMPOTENCY_KEY" \
  --rationale "User confirmed funding the Ramp x402 wallet from the selected Ramp Checking account"
```

Keep the returned `transfer_uuid`, `status`, source, destination, amount, and
currency. Do not report the wallet as funded merely because the transfer was
created.

If the funding call times out or its response is lost, do not call `x402 fund`
again with the same idempotency key. Search all available pages from
`treasury transfers` and match an entry whose `uuid` equals `IDEMPOTENCY_KEY`;
the funding service uses that UUID as the transfer UUID. If found, continue with
that transfer's status. If the exact UUID cannot be deterministically
reconciled, report the uncertain outcome and stop. Do not retry or create
another transfer until Ramp confirms whether the original transfer exists.

## 5. Complete any Ramp Checking approval

Use the returned transfer status and the selected Ramp Checking account's
activity in Ramp to determine whether its withdrawal needs approval. If Ramp
shows a pending approval, tell the user the exact account and amount, open the
pending withdrawal in the Ramp web app, and ask the assigned approver to review
it. Do not approve on the user's behalf or imply that access to the funding tool
is approval authority.

Wait for the user to confirm that the withdrawal was approved. Then inspect the
transfer again with the Ramp Checking transfer-history tool:

```bash
ramp treasury transfers --agent \
  --rationale "Verify the confirmed x402 wallet funding transfer status"
```

Match the funding result's `transfer_uuid` to the transfer history entry's
`uuid`. If the transfer is still requested, processing, held, or otherwise
non-terminal, report that state, wait 15 seconds, and call `treasury transfers`
again. Continue polling every 15 seconds for at most 15 minutes (60 checks)
without requiring another user prompt. Only notify the user when the status
changes or settlement completes; do not claim the funds are available while the
transfer is non-terminal. If the polling window expires, report the exact
transfer UUID and current status, then stop and let the user request another
monitoring window. If it is rejected, canceled, reversed, blocked, or in error,
report the status and stop.

## 6. Confirm setup

When the exact transfer is completed, tell the user:

- The x402 wallet address
- The Ramp Checking account name used
- The funded USD amount
- The completed transfer UUID and status
- That the wallet is ready for an x402 payment

Offer to continue with `ramp-make-x402-payment`.

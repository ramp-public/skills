# Ramp — Product Reference for AI Agents

A map of Ramp's product areas, key entities, and which skill to load for a task. Read this to orient; load a skill from [`README.md`](README.md) to act.

## What Ramp is

Ramp is a finance operations platform: corporate cards and expense management, bill payments (AP), procurement, travel booking, business banking/treasury, and accounting automation with ERP sync. Businesses run their spend through Ramp; AI agents interact with that data through the Ramp agent tools.

## The tool surface

One OpenAPI "agent-tools" specification, exposed through two equivalent surfaces:

- **Ramp MCP connector** — for Claude Desktop, ChatGPT, Perplexity, and MCP-capable coding agents
- **`ramp` CLI** — for terminal-based agents (Claude Code, Codex, Cursor); OAuth via `ramp auth login`

Tool contracts (parameters, semantics) are identical on both; each operation is tagged with its supported surfaces. Every call requires a `rationale` string (audit trail). Access is scoped by OAuth scopes and the acting user's permissions in Ramp.

## Product areas → entities → skills

| Area | What it covers | Key entities | Load |
|---|---|---|---|
| Cards & Spend | Corporate cards, transactions, attention-feed tasks, funds (spend allocations), receipts, memos, coding | task, card, transaction, fund, receipt, tracking category, merchant | `ramp-complete-tasks`, `ramp-complete-expenses`, `ramp-spend-analysis`, `ramp-spend-optimization`, `ramp-card-management`, `ramp-approval-dashboard` |
| Bill Pay | Vendor bills/invoices, payment status, approvals, recurring bills | bill, draft bill, recurring bill, payment, vendor (payee) | `ramp-manage-bills`, `ramp-payment-lookup`, `ramp-approval-dashboard` |
| Procurement | Purchase requests, purchase orders, request approvals | unified request, purchase order | `ramp-submit-procurement-request`, `ramp-manage-procurement` |
| Travel | Flight and hotel search/booking under company policy | trip, booking, policy | `ramp-book-flight`, `ramp-book-hotel` |
| Banking & Treasury | Ramp checking accounts, investment/managed-portfolio balances, transfers | treasury account (checking / investment / managed portfolio), wallet transfer | *(flagship skill coming — see area folder)* |
| Accounting | Coding hygiene, missing items, ERP sync readiness | tracking category, sync run, missing items | `ramp-complete-expenses` *(close-books coming)* |
| Vendor Management | Vendors (payees), agreements/contracts, vendor documents | vendor, agreement/contract, vendor document | `ramp-manage-vendors` |
| Reimbursements | Out-of-pocket expense reimbursement | reimbursement, receipt | `ramp-submit-reimbursement` |
| Agentic commerce | Agent cards, browser checkout | agent card fund, payment token | `ramp-agentic-purchase` |

### Entity disambiguation

- **Vendor (payee)** — who the business pays via Bill Pay. **Merchant** — who appears on card transactions. The same company can be both; correlate by name.
- **Fund / spend allocation** — card-side budget container (also called cards or budgets). Distinct from **treasury accounts** (actual bank/investment accounts).
- **Bill vs. transaction** — bills are AP documents owed to vendors; transactions are card spend events.
- Amounts vary by endpoint family: bill amounts are numeric major currency units (never divide by 100), reimbursements are in dollars, and transactions are formatted strings — each skill documents its endpoints' conventions.

## Task routing

| User wants to... | Load |
|---|---|
| See what needs their attention or complete overdue tasks | `ramp-complete-tasks` |
| Resolve a card locked until overdue tasks are completed | `ramp-complete-tasks`, then `ramp-card-management` to verify card state |
| Fix missing memos/categories/receipts on card spend | `ramp-complete-expenses` |
| See or act on their approval queue | `ramp-approval-dashboard` |
| Find/track a bill or a bill payment | `ramp-manage-bills`, `ramp-payment-lookup` |
| Analyze spend by vendor/category/team, including broad AI-spend questions | `ramp-spend-analysis` |
| Find potential savings, duplicate recurring spend, fragmented vendor payments, or oversized funds | `ramp-spend-optimization` |
| Book or quote travel | `ramp-book-flight`, `ramp-book-hotel` |
| Submit or track a purchase request | `ramp-submit-procurement-request`, `ramp-manage-procurement` |
| Submit an out-of-pocket expense | `ramp-submit-reimbursement` |
| Upload vendor documents (W-9s, contracts, COIs) | `ramp-manage-vendors` |
| Make a purchase with an agent card | `ramp-agentic-purchase` |
| Apply to Ramp / incorporate a company | `ramp-apply-for-account`, `ramp-incorporate` |
| Set up their agent for Ramp from scratch | `ramp-get-started` |

## Canonical links

- Product docs: https://docs.ramp.com
- Agent playbooks & install: https://agents.ramp.com (skills at `/.well-known/agent-skills/`)
- Developer API: https://docs.ramp.com/developer-api/v1/overview
- Connectors overview: https://agents.ramp.com/docs/connectors/overview — [Claude](https://claude.ai/directory/61bac03c-3f98-4b3c-affb-1b99533fa82c) · [ChatGPT](https://chatgpt.com/apps/ramp/asdk_app_69250fb6281c819195b52a1556b0060c) · [Perplexity](https://www.perplexity.ai/computer/connectors?connector=ramp) · [Cursor](https://agents.ramp.com/docs/connectors/cursor)
- Custom MCP redirect URI whitelist: https://docs.ramp.com/developer-api/v1/mcp-redirect-whitelist-request
- Sandbox access: https://docs.ramp.com/developer-api/v1/sandbox-access · Demo MCP: `https://demo-mcp.ramp.com/mcp`

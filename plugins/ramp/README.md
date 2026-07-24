<p align="center">
  <img src="assets/logo.svg" alt="Ramp" width="96" height="96" />
</p>

<h1 align="center">Ramp for AI Agents</h1>

<p align="center">Search, access, and act on your Ramp financial data from your AI agent.</p>

Connect your Ramp account to your agent to manage company finances through natural conversation.
The plugin bundles Ramp's official agent skills with configuration for the Ramp MCP server.

## What you can do

- **Analyze company spend** — Pull transactions and bills, break spend down by vendor,
  category, team, or date range, and flag vendor-name variants and refunds.
- **Clear approvals** — Review and approve/reject pending transactions, reimbursements, and
  requests (POs and fund requests).
- **Clean up transactions** — Add memos and coding, complete missing details, and resolve
  receipt/memo compliance gaps.
- **Run AP and procurement** — Search and draft bills, look up payments, and move
  procurement requests forward.
- **Handle employee tasks** — Manage cards (activate/lock), submit reimbursements, upload
  receipts and vendor documents, and book travel.
- **Make agent purchases** — Generate Ramp Agent Card credentials with built-in spend
  controls.

## Setup

Use the **Ramp CLI** with terminal-based agents:

```bash
# install (or: brew install ramp-public/ramp/ramp-cli)
curl -fsSL https://agents.ramp.com/install.sh | sh

# authenticate via browser OAuth
ramp auth login
```

The plugin also ships the **Ramp MCP server** (`mcp.json` → `https://mcp.ramp.com/mcp`) for
tool-based, natural-language access. Connect it in your agent's MCP settings; supported agents
run Ramp's OAuth flow. To explore with sample data and no Ramp account, point the server at the
demo URL instead:

```json
{
  "mcpServers": {
    "ramp": { "url": "https://demo-mcp.ramp.com/mcp" }
  }
}
```

Use the exact URL with no trailing slash.

## Sample prompts

- "Analyze our company spend over the last 90 days — break it down by vendor and category,
  and call out the biggest increases vs. the prior 90 days."
- "What needs my approval?" — or run `/ramp-approvals`
- "Which of my transactions are missing receipts or memos? Fix the coding where you can."
- "Find the payment for invoice #4401 and tell me its status."

## What's included

| | |
|---|---|
| **Skills** | Ramp's official agent skills, mirrored from the canonical catalog (see [Skill sync](#skill-sync)) — first sync intentionally pending |
| **MCP server** | `ramp` — connects compatible agents to Ramp over MCP (`https://mcp.ramp.com/mcp`) |
| **Rule** | `ramp-safety` — Cursor-only reinforcement of the skills' money-handling, write-confirmation, and pagination guidance |
| **Command** | `/ramp-approvals` — surface and clear your pending approval queue |

## Skill sync

`skills/` here is a 1:1 mirror of the repository's canonical catalog under
[`../../skills/`](../../skills/). Ramp's publishing pipeline updates both copies together,
so there is no separate plugin sync.

Don't hand-edit mirrored `SKILL.md` files (hand edits are overwritten and rejected by CI);
plugin-specific guidance lives in the `ramp-safety` rule and the `/ramp-approvals` command
instead.

## Safety

Ramp tools act on a live financial account and most writes cannot be undone. Ramp's skills
instruct the agent to:

- Show item details before approving/rejecting — never blind-approve.
- Confirm with the user before any write, especially bulk operations.
- Require a reason on rejections.
- Convert amounts correctly (transactions are formatted strings; bills and reimbursements
  are numeric dollars).
- Paginate every queue to completion.

These are instructions to the agent, not enforcement. Ramp enforces access server-side through
the authenticated user's role and OAuth scopes, and records the `rationale` passed on every call.
Most MCP clients also prompt before running a tool call — keep that prompting enabled.

In Cursor, the bundled `ramp-safety` rule restates the guidance above with `alwaysApply`, so it
stays in context even when no skill has loaded. Claude and Codex do not read Cursor `.mdc` rules;
on those clients the guidance travels with the skills.

## License

[MIT](../../LICENSE)

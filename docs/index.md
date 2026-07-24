# Ramp Skills documentation

Ramp Skills are portable instruction sets that teach AI agents how to use Ramp safely and accurately. They cover cards and spend, bill pay, procurement, travel, banking, accounting, reimbursements, and agentic commerce.

## Start here

1. [Install the skills or plugin](installation.md).
2. [Connect and authenticate your agent](authentication.md).
3. Review the [safety and permissions model](safety-and-permissions.md).
4. Use the [product reference](../PRODUCT.md) to find the right skill for a task.

If an installed skill does not appear or a Ramp tool cannot connect, see [Troubleshooting](troubleshooting.md).

## Ways to use Ramp Skills

- **Skills CLI:** install one or more skills into Claude Code, Codex, Cursor, and other compatible agents.
- **Agent plugin:** install the shared Ramp plugin bundle, which includes the skill catalog and platform-specific metadata.
- **MCP connector:** connect supported agents directly to Ramp tools through `https://mcp.ramp.com/mcp`.
- **Ramp CLI:** use the same Ramp capabilities from terminal-based agents after running `ramp auth login`.

Every Ramp action is performed with the authenticated user's permissions and includes an audit rationale.

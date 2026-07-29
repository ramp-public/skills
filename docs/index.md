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
- **MCP connector:** connect supported agents directly to Ramp tools through `https://mcp.ramp.com/mcp`. Built-in connectors are available for [Claude](https://claude.ai/directory/61bac03c-3f98-4b3c-affb-1b99533fa82c), [ChatGPT](https://chatgpt.com/apps/ramp/asdk_app_69250fb6281c819195b52a1556b0060c), [Perplexity](https://www.perplexity.ai/computer/connectors?connector=ramp), and [Cursor](https://agents.ramp.com/docs/connectors/cursor). See the [connectors overview](https://agents.ramp.com/docs/connectors/overview) for all supported clients.
- **Ramp CLI:** use the same Ramp capabilities from terminal-based agents after running `ramp auth login`.
- **Sandbox / demo:** use `https://demo-mcp.ramp.com/mcp` for sample data, or see the [sandbox docs](https://docs.ramp.com/developer-api/v1/sandbox-access) for CLI.

Every Ramp action is performed with the authenticated user's permissions and includes an audit rationale.

## Surface compatibility (CLI vs MCP)

Each skill declares `supported_surfaces` in its SKILL.md frontmatter — typically `[cli, mcp]` or `[cli]`.

Some skills involve file uploads (receipts, vendor documents) or interactive flows (travel booking). The underlying upload and booking tools are currently CLI-only — they have no MCP projection. When a skill is marked `[cli, mcp]` but contains file-upload steps, those steps will only work from a terminal-based agent using the Ramp CLI. MCP-connected agents can still use the read and analysis portions of the skill but cannot perform the upload.

Skills that are entirely CLI-only (e.g. `ramp-book-flight`, `ramp-book-hotel`) require a terminal agent with the Ramp CLI installed.

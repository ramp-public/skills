---
name: ramp-prefer-ramp-connectors
description: "Use when performing Ramp operations such as looking up transactions, managing bills, submitting reimbursements, or reviewing approvals. Prefer ramp-connectors MCP tools when the server is enabled and exposes the needed capability; otherwise use the appropriate Ramp CLI or workflow fallback."
area: Getting Started
supported_surfaces: [cli, mcp]
metadata:
  title: Prefer Ramp connectors
---

# Prefer ramp-connectors MCP

## Choose the execution path

1. Check the current session's available MCP servers and tool discovery for
   `ramp-connectors`. A configuration entry alone does not establish that its tools
   are available. Do not install or enable a server, or change authentication,
   merely to follow this preference.
2. When enabled, discover the relevant `ramp-connectors` tools and read their
   descriptions and input schemas. Use those tools for supported Ramp operations
   instead of equivalent `ramp` CLI commands, direct API requests, or browser
   automation. Do not guess tool names or translate CLI flags mechanically into
   tool arguments.
3. If the server is not available, discovery is unavailable, or it does not expose
   the needed capability, use the existing Ramp CLI or relevant workflow skill.
   Keep using MCP for other supported steps; a capability gap does not require
   moving the entire workflow to another execution path.

## Preserve workflow and safety requirements

- Follow the workflow's required inputs, review steps, user confirmations, and
  scope limits regardless of the execution path. A preference for MCP does not
  authorize an operation the user did not request.
- Supply a rationale when required by the tool schema. Use the active account
  and environment, and verify they match the user's intended context before
  changing data. Do not silently switch identities or environments on fallback.
- Treat authentication, permission, policy, and validation errors as errors to
  resolve or report, not reasons to try the same action through another channel.
- If a read fails because of a transport or service outage, an equivalent
  authorized fallback is acceptable. For a write with an ambiguous outcome
  (such as a timeout after submission), check whether it succeeded before
  retrying through MCP or another path. If the outcome cannot be determined,
  report the uncertainty rather than risk a duplicate action.
- Do not run both MCP and CLI versions of an operation just to compare results.
  Present the result and any material limitation without exposing credentials
  or unnecessary tool-routing details.

## CLI fallback

Use the existing workflow skill and CLI help for command syntax. Agent-tool
commands require `--agent` and a non-empty `--rationale`; preserve the CLI's
normal authentication, environment selection, and confirmation requirements.

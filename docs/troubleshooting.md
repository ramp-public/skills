# Troubleshooting

## An installed skill does not appear

1. Confirm the skill exists in the repository catalog.
2. Run `npx skills list` to verify the installation target.
3. Run `npx skills check` to detect an outdated installation.
4. Restart or reload the agent so it discovers newly installed skills.
5. Reinstall with an explicit `--agent` value if the skill was written to the wrong client directory.

## The Ramp CLI is unavailable

Reinstall it with Homebrew and open a new shell:

```bash
brew reinstall ramp-public/ramp/ramp-cli
ramp auth login
```

For other platforms, follow the [public installation guide](https://agents.ramp.com).

Confirm the CLI is on the shell's `PATH` before asking the agent to use it.

## MCP cannot connect

- Verify the URL is exactly `https://mcp.ramp.com/mcp` with no trailing slash.
- Complete the OAuth flow opened by the client.
- Restart the client after changing its MCP configuration.
- Use `https://demo-mcp.ramp.com/mcp` only when sample data is intended.

## A tool returns a permission error

The authenticated user may not have the required Ramp role or OAuth scope. Re-authentication can refresh an expired session, but it does not elevate access. Ask a Ramp administrator to grant the appropriate product permission when needed.

## Results appear incomplete

Check whether the agent paginated every response and queried all relevant resource types. For example, card transactions and Bill Pay bills are separate sources and must both be queried for complete company spend.

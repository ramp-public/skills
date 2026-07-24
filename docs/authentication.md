# Authentication

Ramp Skills contain instructions. The agent still needs an authenticated Ramp tool connection before it can read or change Ramp data.

## Ramp CLI

Install the CLI with Homebrew:

```bash
brew install ramp-public/ramp/ramp-cli
```

For other platforms, follow the [public installation guide](https://agents.ramp.com).

Authenticate through browser OAuth:

```bash
ramp auth login
```

Claude Code, Codex, Cursor, and other terminal-based agents can then use the `ramp` commands documented by each skill.

## Ramp MCP connector

Configure an MCP-capable agent with this server:

```json
{
  "mcpServers": {
    "ramp": {
      "url": "https://mcp.ramp.com/mcp"
    }
  }
}
```

Use the exact URL without a trailing slash. The client opens Ramp's OAuth flow when it first connects.

For sample data without a Ramp account, use `https://demo-mcp.ramp.com/mcp` instead.

## Access model

Authentication does not grant additional Ramp permissions. Tools operate with the signed-in user's role, OAuth scopes, and business access. Permission errors should be surfaced to the user rather than bypassed or retried with another identity.

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

For built-in connectors (Claude, ChatGPT, Perplexity, Cursor), OAuth is handled automatically during setup. See the [connectors overview](https://agents.ramp.com/docs/connectors/overview) for per-client setup instructions.

### Custom MCP clients and gateways

If you are building a custom MCP client or connecting through a gateway that is not one of the built-in connectors, you must get your redirect URI whitelisted before OAuth will work:

1. Submit a request via the [redirect URI whitelist form](https://docs.ramp.com/developer-api/v1/mcp-redirect-whitelist-request).
2. Wait for the Ramp team to approve your redirect URI.
3. Once approved, configure your client to point at `https://mcp.ramp.com/mcp` (production) or `https://demo-mcp.ramp.com/mcp` (demo).

### Sandbox / demo

For sample data without a Ramp account, use `https://demo-mcp.ramp.com/mcp` instead of the production URL. Substitute it in any connector setup or MCP config. For CLI sandbox access, see the [Ramp sandbox docs](https://docs.ramp.com/developer-api/v1/sandbox-access).

## Access model

Authentication does not grant additional Ramp permissions. Tools operate with the signed-in user's role, OAuth scopes, and business access. Permission errors should be surfaced to the user rather than bypassed or retried with another identity.

# Installation

## Install with the Skills CLI

Install the Ramp catalog interactively:

```bash
npx skills add ramp-public/skills
```

Install one skill without prompts:

```bash
npx skills add ramp-public/skills --skill <skill-name> --yes
```

Target one or more supported agents:

```bash
npx skills add ramp-public/skills \
  --skill <skill-name> \
  --agent claude-code \
  --agent codex \
  --agent cursor
```

Use `npx skills check` to preview updates and `npx skills update` to install them.

## Install as a plugin

The repository contains one shared plugin bundle at `plugins/ramp/` that conforms to the [Agent Plugins v1.0.0](https://agent-plugins.org) specification. The portable `plugin.json` and `mcp.json` at the plugin root work with any spec-compliant client. Root marketplace manifests expose the bundle to Claude, Codex, and Cursor:

- `.claude-plugin/marketplace.json`
- `.agents/plugins/marketplace.json`
- `.cursor-plugin/marketplace.json`

Each platform reads its own native manifest from `plugins/ramp/` for client-specific metadata. The portable manifest, skills, and assets are shared rather than copied into separate platform bundles.

## Choose a tool connection

After installing the instructions, connect the agent to Ramp using either:

- The Ramp CLI for terminal-based agents.
- The Ramp MCP connector for MCP-capable agents.

See [Authentication](authentication.md) for setup.

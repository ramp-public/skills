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

The repository contains one shared plugin bundle at `plugins/ramp/`. Root marketplace manifests expose that bundle to Claude, Codex, and Cursor:

- `.claude-plugin/marketplace.json`
- `.agents/plugins/marketplace.json`
- `.cursor-plugin/marketplace.json`

Each platform reads its own plugin manifest from `plugins/ramp/`. The skills and assets are shared rather than copied into separate platform bundles.

## Choose a tool connection

After installing the instructions, connect the agent to Ramp using either:

- The Ramp CLI for terminal-based agents.
- The Ramp MCP connector for MCP-capable agents.

See [Authentication](authentication.md) for setup.

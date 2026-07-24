# Ramp Skills

[![skills.sh](https://skills.sh/b/ramp-public/skills)](https://skills.sh/ramp-public/skills)
[![Agent Skills Spec](https://img.shields.io/badge/Agent%20Skills-Specification-blue)](https://agentskills.io)
[![License](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

**Official Ramp skills for AI coding agents.**

> New here? Start with the playbooks at [agents.ramp.com](https://agents.ramp.com).

---

Skills are portable instruction sets that teach compatible agents how to work with
[Ramp](https://ramp.com): cards, bill pay, procurement, travel, banking, and accounting.
They pair with Ramp's agent tools through the MCP connector or `ramp` CLI, and work across
the growing ecosystem of clients that support the [Agent Skills specification](https://agentskills.io).

This repository is Ramp's public skills catalog. Skill content is maintained by Ramp and
published here through an automated pipeline. Long-form setup, authentication, safety,
troubleshooting, and repository guides live in [`docs/`](docs/).

---

## Quickstart

Install skills with the [`skills` CLI](https://github.com/vercel-labs/skills):

```bash
npx skills add ramp-public/skills
```

Install a specific skill without prompts:

```bash
npx skills add ramp-public/skills --skill <skill-name> --yes
```

Target a specific agent:

```bash
npx skills add ramp-public/skills --skill <skill-name> --agent <agent-name>
```

Repeat `--agent` to install into several clients. See the [`skills` CLI supported agents
table](https://github.com/vercel-labs/skills#supported-agents) for current target names.

Keep skills up to date with `npx skills update` (and preview with `npx skills check`).

### Connect your agent to Ramp

- **Terminal-based agents:** install the Ramp CLI using the [public installation guide](https://agents.ramp.com), then run `ramp auth login`.
- **MCP-capable agents:** connect to `https://mcp.ramp.com/mcp` or install Ramp from your client's connector directory where available.

Ramp Skills contain instructions; access to Ramp data requires a separately authenticated Ramp
tool connection.

## Skill catalog

<!-- catalog:start -->

_The catalog will be published by Ramp's skills pipeline; the first sync is intentionally pending._

<!-- catalog:end -->

## Plugins

Agent plugins that bundle these skills with client-specific configuration live under
[`plugins/`](plugins/):

- [`plugins/ramp`](plugins/ramp/) — the official Ramp plugin, including MCP server config,
  the `ramp-safety` rule, the `/ramp-approvals` command, and a mirror of the skill catalog.
  It supports the shared agent plugin format and native client discovery, superseding the
  standalone [ramp-public/cursor-plugin](https://github.com/ramp-public/cursor-plugin) repo.

Each plugin's `skills/` directory is a 1:1 mirror of the root catalog, updated by the same sync job — do not PR those directories (plugin-native content like rules, commands, and manifests is maintained here).

## About this repository

- Skills live under `skills/` in a flat layout: `skills/<name>/SKILL.md`.
- **Skill content is generated — do not PR skill directories here.** Skills are maintained by Ramp and published into `skills/` and `plugins/ramp/skills/` together with regenerated catalog indexes. Catalog-level content (`README.md`, `PRODUCT.md`, `.github/`, docs, and plugin-native files) is maintained directly in this repo.
- `index.json`, `skills.sh.json`, and `provenance.json` are regenerated from skill frontmatter by Ramp's publishing pipeline.
- See [CHANGELOG.md](CHANGELOG.md) for public catalog and plugin release history.
- Found a bug in a skill? Open an issue here — content fixes are routed to the source of truth.
- See [`PRODUCT.md`](PRODUCT.md) for an orientation to Ramp's product areas, entities, and which skill to load for a task.

## License

[MIT](LICENSE)

# Repository architecture

This repository is the public distribution catalog for Ramp Skills. Skill content is authored with the Ramp agent tools it documents and synchronized into this repository.

## Layout

```text
skills/                         Synced skill catalog
plugins/ramp/                   Shared Ramp plugin bundle
  plugin.json                   Portable Agent Plugins v1.0.0 manifest
  mcp.json                      Portable MCP server configuration
  .claude-plugin/plugin.json    Claude native package metadata
  .codex-plugin/plugin.json     Codex native package metadata
  .cursor-plugin/plugin.json    Cursor native package metadata
  skills/                       Mirror of the root skill catalog
.claude-plugin/marketplace.json Claude marketplace registry
.agents/plugins/marketplace.json Shared/Codex marketplace registry
.cursor-plugin/marketplace.json Cursor marketplace registry
provenance.json                 Generated publication metadata
```

## One shared plugin bundle

Claude, Codex, and Cursor do not receive separate Ramp products. Their root marketplace files all point to `plugins/ramp/`, and that bundle provides platform-specific manifests around one shared skill catalog.

The plugin ships a portable `plugin.json` conforming to the [Agent Plugins v1.0.0 specification](https://agent-plugins.org) alongside native manifests for each client. The portable manifest is the source of truth for metadata and version; native manifests add client-specific fields (display metadata, UI hints, commands) that the portable spec does not cover.

This avoids divergent skill content while allowing each platform to read the metadata format it expects.

## Generated content

The following paths are synchronized and should not be edited directly:

- `skills/`
- `plugins/ramp/skills/`
- Generated catalog indexes

Plugin manifests, rules, commands, assets, and repository documentation are maintained in this repository. Ramp's publishing pipeline owns mirror paths, catalog generation, and validation so generated surfaces cannot drift.

The publishing workflow recognizes the Ramp Skills publishing bot directly. All other pull requests are prevented from editing generated paths.

# Changelog

All notable changes to this catalog are documented here. This project adheres to
[Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

## [0.3.0] - 2026-08-06

### Added

- Portable `plugin.json` conforming to the Agent Plugins v1.0.0 specification.
- Portable `mcp.json` with `$schema` and `type: "streamable-http"` per the Agent Plugins MCP
  schema.
- CI workflow (`validate-plugin.yml`) that validates portable manifests against the versioned
  Agent Plugins schemas, checks version consistency across portable and native manifests, and
  validates SKILL.md frontmatter.

### Changed

- Bumped plugin version to 0.3.0 across all manifests (portable and native).

## [0.2.0] - 2026-07-24

### Added

- Initial shared public catalog for Ramp Skills.
- Native plugin marketplace manifests for Claude, Codex, and Cursor, backed by one shared
  `plugins/ramp/` bundle.
- Ramp MCP server configuration, financial-action safety rule, and `/ramp-approvals` command.
- Documentation for installation, authentication, safety, troubleshooting, and catalog
  architecture.

### Changed

- Consolidated the standalone Cursor plugin into this catalog. Ramp's publishing pipeline now
  owns the root skill catalog and the plugin skill mirror together.

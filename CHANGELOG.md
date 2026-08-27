# Changelog

All notable changes to the Tasqr plugin for Claude Code are documented here.
The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and versions follow [Semantic Versioning](https://semver.org/).

## [0.1.0] - 2026-08-27

Initial public release.

### Added

- MCP server entry that launches the [`tasqr-mcp`](https://pypi.org/project/tasqr-mcp/)
  proxy over stdio via `uvx`, exposing the Tasqr API as `mcp__tasqr__*` tools.
- `tasqr` skill teaching Claude when work deserves a durable task, the producer
  and consumer workflows, batch task graphs with dependencies, tag vocabulary
  rules, lease management, multi-agent fan-out, and talking to humans about
  tasks by title rather than bare id.
- Reference guides for org administration (tags, member roles and seats, teams)
  and the analysis tools (insights, standups, runbooks, grounded planning).

# Changelog

All notable changes to the Tasqr plugin for Claude Code are documented here.
The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and versions follow [Semantic Versioning](https://semver.org/).

## [0.1.3] - 2026-10-03

### Changed

- The MCP server runs `tasqr-mcp==0.1.2`. The proxy now passes the Tasqr
  server's instructions through to Claude Code, which tell Claude when to reach
  for Tasqr and which tools to start with. Earlier proxies dropped them.

## [0.1.2] - 2026-10-03

### Changed

- The MCP server runs `tasqr-mcp==0.1.1`, pinned to an exact version. A new
  proxy release reaches the plugin through a plugin release.

### Fixed

- The README's setup step. The proxy reads its API key from
  `~/.config/tasqr/credentials`, written by a one-time `uvx tasqr-mcp` sign-in
  in a terminal. It never read `TASQR_API_KEY`.
- The README and the skill named the tools `mcp__tasqr__*`. Tools from the
  plugin's server are `mcp__plugin_tasqr_tasqr__*`.

### Added

- The README lists everything the proxy reads, writes and contacts.
- `repository` in `plugin.json`.

## [0.1.1] - 2026-09-28

### Changed

- The `tasqr` skill is about 45% shorter. It no longer repeats what the tool
  descriptions already say (parameter lists, response shapes, most code
  examples) and keeps the judgment calls and server rules models get wrong
  without it.
- The canceled task status is spelled `canceled`, matching the server.

### Fixed

- `pending` can move straight to `blocked`. A task blocked on something outside
  Tasqr (no `blocked_by`) never unblocks on its own, and must be moved to
  `in_progress` rather than back to `pending`.
- `ref:` names resolve only in `blocked_by`, not in `parent_task_id`.
- Subagents sharing one API key share its email; `update_tasks` takes an
  `agent_id` that records which one made each change.
- The multi-agent example assigned tasks to made-up emails; assignees must be
  real members.
- A non-admin's `update_profile` silently drops strict tags rather than
  rejecting the call.
- `output` and `metadata` must be JSON objects.
- A task can have at most 90 blockers.
- List pagination and ordering, and runbook `source_task_ids`, are documented.
- On the free tier, team tools return a tool error rather than an HTTP 403.

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

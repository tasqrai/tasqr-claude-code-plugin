# Changelog

All notable changes to the Tasqr plugin for Claude Code are documented here.
The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and versions follow [Semantic Versioning](https://semver.org/).

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

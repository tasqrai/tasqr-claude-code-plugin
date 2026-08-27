# Tasqr — Claude Code plugin

Task state management built for AI agents. This plugin connects Claude Code to the [Tasqr](https://tasqr.ai) MCP server and teaches Claude to track substantial work, coordinate multi-agent pipelines, and work task queues — so work survives the end of a session and any agent (or human) can pick it up.

## What's included

- **MCP server** — launches the [`tasqr-mcp`](https://pypi.org/project/tasqr-mcp/) proxy over stdio via `uvx`, exposing the Tasqr API as `mcp__tasqr__*` tools.
- **`tasqr` skill** — teaches Claude when a piece of work deserves a durable task (and when it doesn't), the producer and consumer workflows, batch task creation with dependencies, tag vocabulary rules, lease management, and multi-agent fan-out patterns.

## Requirements

- [uv](https://docs.astral.sh/uv/) (`uvx` is used to run the MCP proxy — no manual install needed)
- A Tasqr API key from your [tasqr.ai](https://tasqr.ai) workspace

## Install

Add the marketplace and install the plugin:

```
/plugin marketplace add tasqrai/tasqr-claude-code-plugin
/plugin install tasqr@tasqr
```

Then make your API key available to Claude Code:

```sh
export TASQR_API_KEY=tasqr_...
```

Restart Claude Code (or reconnect MCP servers) and the `mcp__tasqr__*` tools will be available. The skill triggers automatically when durable, multi-session work comes up — you don't need to invoke anything by hand.

## What Claude does with it

- Creates a task graph (with dependencies) for migrations, feature builds, audits, and other work measured in sessions rather than minutes.
- Resumes unfinished tasks at session start instead of duplicating work.
- Claims work from a shared queue atomically (`claim_next_task`), reads the attached briefing context, and posts progress that renews its lease.
- Models subagent fan-out as child tasks so a whole tree of parallel work is visible on one board.
- Records personal and professional to-dos you ask it to track — not just code.

## License

See [LICENSE](LICENSE).

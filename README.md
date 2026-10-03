# Tasqr plugin for Claude Code

Task tracking built for AI agents. This plugin connects Claude Code to [Tasqr](https://tasqr.ai) and teaches Claude to track substantial work, coordinate multi-agent pipelines and work through a shared task queue. Work survives the end of a session, and any agent or person can pick it up.

## What's included

- **MCP server.** Runs the [`tasqr-mcp`](https://pypi.org/project/tasqr-mcp/) proxy over stdio with `uvx`, pinned to an exact version. It exposes the Tasqr tools to Claude Code as `mcp__plugin_tasqr_tasqr__*`.
- **`tasqr` skill.** Teaches Claude when a piece of work deserves a durable task, how to plan it as tasks with dependencies in one call, how to use your workspace's tag vocabulary, how to pick up queued work, and how to give each subagent its own child task.

## Requirements

- [uv](https://docs.astral.sh/uv/), which provides `uvx`. It downloads and runs the proxy, so there is nothing else to install.
- A GitHub account, used once to sign in to Tasqr.

## Install

1. Add the marketplace and install the plugin:

   ```
   /plugin marketplace add tasqrai/tasqr-claude-code-plugin
   /plugin install tasqr@tasqr
   ```

2. Sign in once, in a terminal:

   ```sh
   uvx tasqr-mcp==0.1.2
   ```

   Your browser opens GitHub's device authorization page, and the proxy copies the code to your clipboard. After you approve, choose your workspace. The proxy saves your Tasqr API key to `~/.config/tasqr/credentials` (`%APPDATA%\tasqr\credentials` on Windows), readable only by you, and starts serving. Press Ctrl+C to stop it.

   If you already have an API key from your Tasqr dashboard, write the file yourself instead:

   ```ini
   [default]
   api_key = tasqr_...
   ```

3. Restart Claude Code, or reconnect the server from `/mcp`.

The skill starts working on its own when durable, multi-session work comes up. There is nothing to invoke by hand.

## What Claude does with it

- Plans migrations, feature builds, audits and other multi-session work as tasks with dependencies.
- Checks for unfinished tasks at the start of a session and resumes them.
- Takes the next task it is eligible for from a shared queue, and reads the context that comes with it: the parent task, the output of the tasks it waited on, and similar completed work on paid plans.
- Gives each subagent its own child task, so a whole tree of parallel work shows on one board.
- Records personal and professional to-dos you ask it to track, as well as code work.

## Data and network access

The plugin runs the `tasqr-mcp` proxy. Here is everything it reads, writes and contacts:

- **PyPI.** `uvx` downloads `tasqr-mcp` and its dependencies from pypi.org the first time it runs.
- **Sign-in.** The one-time terminal sign-in opens github.com in your browser, receives a GitHub token, and sends it once to `https://auth.tasqr.ai` in exchange for a Tasqr API key. The GitHub token is not saved.
- **Credentials file.** The proxy reads your API key, and any optional settings, from `~/.config/tasqr/credentials`. It writes that file only during sign-in and, for client-side encryption, to cache its wrapped data key.
- **Tasqr.** Every tool call goes to `https://mcp.tasqr.ai/mcp`, authenticated with your API key. The tasks, notes and outputs Claude writes are stored in your Tasqr workspace.
- **AWS KMS, optional.** Workspaces that use client-side encryption add a `kms_key_id` to the credentials file. The proxy then encrypts task fields on your machine and calls AWS KMS with your own AWS credentials to wrap and unwrap its data key. Without `kms_key_id`, it makes no AWS calls.
- **Log file, optional.** Off by default. Setting `log_level` writes a log to `~/.config/tasqr/tasqr-mcp.log`.

The [`tasqr-mcp` README](https://github.com/tasqrai/tasqr-mcp-python) documents every setting, including the profiles and environment variables that change these defaults.

## License

MIT. See [LICENSE](LICENSE).

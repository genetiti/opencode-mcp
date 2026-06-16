# opencode-mcp (fork)

[![license](https://img.shields.io/badge/license-MIT-blue)](LICENSE)
[![node](https://img.shields.io/badge/node-%3E%3D18-brightgreen)](https://nodejs.org/)

> Fork of [AlaeddineMessadi/opencode-mcp](https://github.com/AlaeddineMessadi/opencode-mcp), **run from source** (not published to npm). The instructions below build the fork locally and point your MCP client at the compiled binary — there is no `npx` package to install. Replace `<your-username>` in the commands with your GitHub fork's owner.

**Give any MCP client the power of [OpenCode](https://opencode.ai/).**

opencode-mcp is an MCP server that bridges your AI tools (Claude, Cursor, Windsurf, VS Code, etc.) to OpenCode's headless API. It lets your AI delegate real coding work — building features, debugging, refactoring, running tests — to OpenCode sessions that autonomously read, write, and execute code in your project.

**80 tools** | **10 resources** | **6 prompts** | **Multi-project** | **Auto-start**

## Why Use This?

- **Delegate coding tasks** — Tell Claude "build me a REST API" and it delegates to OpenCode, which creates files, installs packages, writes tests, and reports back.
- **Parallel work** — Fire off multiple tasks to OpenCode while your primary AI keeps working on something else.
- **Any MCP client** — Works with Claude Desktop, Claude Code, Cursor, Windsurf, VS Code Copilot, Cline, Continue, Zed, Amazon Q, and any other MCP-compatible tool.
- **Zero setup** — The server auto-starts the OpenCode HTTP server in-process via the official `@opencode-ai/sdk` if one isn't already running. No manual steps.

## Quick Start (build from source)

This is a source-run fork — there is no npm package. You clone it, build it once, and point your MCP client at the compiled `dist/index.js`.

> **Prerequisites:**
>
> - [OpenCode](https://opencode.ai/) installed — `curl -fsSL https://opencode.ai/install | bash`, `npm i -g opencode-ai`, or `brew install sst/tap/opencode`
> - Node.js >= 18

**1. Clone and build:**

```bash
git clone https://github.com/genetiti/opencode-mcp.git
cd opencode-mcp
npm install
npm run build
```

**2. Register with your client**, pointing at the absolute path to the built `dist/index.js`.

**Claude Code** (user scope = available in all your projects):

```bash
claude mcp add --scope user opencode -- node "$(pwd)/dist/index.js"
```

**Claude Desktop / Cursor / Windsurf / Cline / Continue** (add to your MCP config; replace the path with your absolute clone path):

```json
{
  "mcpServers": {
    "opencode": {
      "command": "node",
      "args": ["/absolute/path/to/opencode-mcp/dist/index.js"]
    }
  }
}
```

That's it. Restart your client and OpenCode's tools will be available.

> Setting up a second machine? See [Install your fork on another machine](#install-your-fork-on-another-machine).
> See [Configuration](docs/configuration.md) for all client configs (VS Code Copilot, Zed, Amazon Q, etc.) and environment variables.

## Install your fork on another machine

Repeat these steps on each machine. Everything runs from your local clone — no npm publish involved.

**Prerequisites (per machine):**

- Node.js >= 18
- OpenCode installed **and a provider authenticated** (e.g. `opencode auth login`). This fork delegates real model calls to OpenCode, so provider credentials live in OpenCode — not in this repo.

**1. Clone and build:**

```bash
git clone https://github.com/genetiti/opencode-mcp.git
cd opencode-mcp
npm install && npm run build
```

**2. Register with Claude Code** (user scope — all projects):

```bash
claude mcp add --scope user opencode -- node "$(pwd)/dist/index.js"
```

**3. (Optional) Register with Codex** — add to `~/.codex/config.toml`:

```toml
[mcp_servers.opencode]
command = "node"
args = ["/path/to/opencode-mcp/dist/index.js"]
```

**4. (Optional) Set a default provider/model** so you don't pass them on every call. For Claude Code, append `-e OPENCODE_DEFAULT_PROVIDER=<provider> -e OPENCODE_DEFAULT_MODEL=<model>` to the `claude mcp add` command; for Codex, add an `[mcp_servers.opencode.env]` block. See [Configuration](docs/configuration.md#environment-variables).

**5. Verify:**

```bash
claude mcp get opencode   # should report: ✔ Connected
```

**Keep it updated.** The fork runs the compiled `dist/`, so after every `git pull` (or local edit) rebuild:

```bash
git -C /path/to/opencode-mcp pull && npm --prefix /path/to/opencode-mcp run build
```

## How It Works

```
MCP Client  <--stdio-->  opencode-mcp  <--HTTP-->  OpenCode Server
(Claude, Cursor, etc.)   (this package)            (in-process via @opencode-ai/sdk,
                                                    or external opencode serve)
```

Your MCP client calls tools over stdio. This server translates them into HTTP requests to the OpenCode headless API. If no OpenCode server is reachable at `OPENCODE_BASE_URL`, one is started in-process via the official `@opencode-ai/sdk`. The `directory` parameter on every tool routes that request to a specific project via the `x-opencode-directory` header, so a single MCP instance can fan out across many project roots.

## Key Tools

The 80 tools are organized into tiers. Start with the workflow tools — they handle the common patterns in a single call.

### Workflow Tools (13) — Start Here

| Tool                         | What it does                                                                 |
| ---------------------------- | ---------------------------------------------------------------------------- |
| `opencode_setup`             | Check server health, providers, and project status. Use first.               |
| `opencode_ask`               | Create session + send prompt + get answer. One call.                         |
| `opencode_reply`             | Follow-up message in an existing session                                     |
| `opencode_run`               | Send a task and wait for completion (session + async send + polling)         |
| `opencode_fire`              | Fire-and-forget: dispatch a task, return immediately                         |
| `opencode_check`             | Compact progress report for a running session (status, todos, files changed) |
| `opencode_conversation`      | Get formatted conversation history                                           |
| `opencode_sessions_overview` | Quick overview of all sessions                                               |
| `opencode_context`           | Project + VCS + config + agents in one call                                  |
| `opencode_review_changes`    | Formatted diff summary for a session                                         |
| `opencode_wait`              | Poll an async session until it finishes                                      |
| `opencode_provider_test`     | Quick-test whether a provider is working                                     |
| `opencode_status`            | Health + providers + sessions + VCS dashboard                                |

### Recommended Patterns

**Quick question:**

```
opencode_ask({ prompt: "Explain the auth flow in this project" })
```

**Build something and wait:**

```
opencode_run({ prompt: "Add input validation to POST /api/users", maxDurationSeconds: 300 })
```

**Parallel background tasks:**

```
opencode_fire({ prompt: "Refactor the auth module to use JWT" })
→ returns sessionId immediately
opencode_check({ sessionId: "..." })
→ check progress anytime
```

### All Tool Categories

| Category                                              | Count | Description                                           |
| ----------------------------------------------------- | ----- | ----------------------------------------------------- |
| [Workflow](docs/tools.md#workflow-tools)              | 13    | High-level composite operations                       |
| [Session](docs/tools.md#session-tools)                | 20    | Create, list, fork, share, abort, revert, permissions |
| [Message](docs/tools.md#message-tools)                | 6     | Send prompts, execute commands, run shell             |
| [File & Search](docs/tools.md#file--search-tools)     | 6     | Search text/regex, find files/symbols, read files     |
| [System](docs/tools.md#system--monitoring-tools)      | 13    | Health, VCS, LSP, MCP servers, agents, logging        |
| [TUI Control](docs/tools.md#tui-control-tools)        | 9     | Remote-control the OpenCode terminal UI               |
| [Provider & Auth](docs/tools.md#provider--auth-tools) | 6     | List providers/models, set API keys, OAuth            |
| [Config](docs/tools.md#config-tools)                  | 3     | Get/update configuration                              |
| [Project](docs/tools.md#project-tools)                | 3     | List, inspect, and initialize projects                |
| [Events](docs/tools.md#event-tools)                   | 1     | Poll real-time SSE events                             |

### Resources (10)

Browseable data endpoints — your client can read these without tool calls:

| URI                          | Description               |
| ---------------------------- | ------------------------- |
| `opencode://project/current` | Current active project    |
| `opencode://config`          | Current configuration     |
| `opencode://providers`       | Providers with models     |
| `opencode://agents`          | Available agents          |
| `opencode://commands`        | Available commands        |
| `opencode://health`          | Server health and version |
| `opencode://vcs`             | Version control info      |
| `opencode://sessions`        | All sessions              |
| `opencode://mcp-servers`     | MCP server status         |
| `opencode://file-status`     | VCS file status           |

### Prompts (6)

Guided workflow templates your client can offer as selectable actions:

| Prompt                     | Description                                     |
| -------------------------- | ----------------------------------------------- |
| `opencode-code-review`     | Review diffs from a session                     |
| `opencode-debug`           | Step-by-step debugging workflow                 |
| `opencode-project-setup`   | Get oriented in a new project                   |
| `opencode-implement`       | Have OpenCode build a feature                   |
| `opencode-best-practices`  | Setup, tool selection, monitoring, and pitfalls |
| `opencode-session-summary` | Summarize what happened in a session            |

## Multi-Project Support

Every tool accepts an optional `directory` parameter to target a different project. No restarts needed.

```
opencode_ask({ directory: "/home/user/mobile-app", prompt: "Add navigation" })
opencode_ask({ directory: "/home/user/web-app", prompt: "Add auth" })
```

Use `opencode_project_init` to scaffold a new project directory (or open a preexisting one) before the first call, so the OpenCode server registers it as a project:

```
opencode_project_init({ path: "/home/user/new-project" })
// → "Successfully initialized project directory at: /home/user/new-project"

opencode_run({ directory: "/home/user/new-project", prompt: "Set up a Vite + React app" })
```

## Environment Variables

All optional. Only needed if you've changed defaults on the OpenCode server.

| Variable                    | Default                 | Description                                                                                                   |
| --------------------------- | ----------------------- | ------------------------------------------------------------------------------------------------------------- |
| `OPENCODE_BASE_URL`         | `http://127.0.0.1:4096` | OpenCode server URL                                                                                           |
| `OPENCODE_SERVER_USERNAME`  | `opencode`              | HTTP basic auth username                                                                                      |
| `OPENCODE_SERVER_PASSWORD`  | _(none)_                | HTTP basic auth password (enables auth when set)                                                              |
| `OPENCODE_AUTO_SERVE`       | `true`                  | Auto-start an in-process OpenCode server (via `@opencode-ai/sdk`) if none is reachable at `OPENCODE_BASE_URL` |
| `OPENCODE_DEFAULT_PROVIDER` | _(none)_                | Default provider ID when not specified per-tool (e.g. `anthropic`)                                            |
| `OPENCODE_DEFAULT_MODEL`    | _(none)_                | Default model ID when not specified per-tool (e.g. `claude-sonnet-4-5`)                                       |

## Development

```bash
git clone https://github.com/genetiti/opencode-mcp.git
cd opencode-mcp
npm install
npm run build
npm start        # run the MCP server
npm run dev      # watch mode
npm test         # 328 tests
```

### Smoke Testing

End-to-end test against a running OpenCode server:

```bash
npm run build && node scripts/mcp-smoke-test.mjs
```

## Documentation

- [Getting Started](docs/getting-started.md) — step-by-step setup
- [Configuration](docs/configuration.md) — env vars and all client configs
- [Tools Reference](docs/tools.md) — all 80 tools in detail
- [Resources](docs/resources.md) — 10 MCP resources
- [Prompts](docs/prompts.md) — 6 guided workflow templates
- [Examples](docs/examples.md) — real workflow examples
- [Architecture](docs/architecture.md) — system design and data flow

## References

- [OpenCode](https://opencode.ai/) | [OpenCode Docs](https://opencode.ai/docs/) | [OpenCode Server API](https://opencode.ai/docs/server/)
- [Model Context Protocol](https://modelcontextprotocol.io/)

## License

[MIT](LICENSE)

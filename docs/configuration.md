# Configuration

## Environment Variables

All environment variables are **optional**. You only need to set them if you've changed the defaults on the OpenCode server side.

| Variable | Description | Default | Required |
|---|---|---|---|
| `OPENCODE_BASE_URL` | URL of the OpenCode headless server | `http://127.0.0.1:4096` | No |
| `OPENCODE_SERVER_USERNAME` | HTTP basic auth username | `opencode` | No |
| `OPENCODE_SERVER_PASSWORD` | HTTP basic auth password | *(none — auth disabled)* | No |
| `OPENCODE_AUTO_SERVE` | Auto-start `opencode serve` if not running | `true` | No |
| `OPENCODE_DEFAULT_PROVIDER` | Default provider ID when not specified per-tool | *(none)* | No |
| `OPENCODE_DEFAULT_MODEL` | Default model ID when not specified per-tool | *(none)* | No |

### Notes

- **Authentication is disabled by default.** It only activates when `OPENCODE_SERVER_PASSWORD` is set on both the OpenCode server and the MCP server.
- **Username and password are both optional.** The default username is `opencode`, matching the OpenCode server's default. You only need to set these if you've explicitly enabled auth on the server.
- **The base URL** should point to where `opencode serve` is listening. If running on the same machine with default settings, you don't need to set this.
- **Default provider/model** are optional. When set, tools that accept `providerID`/`modelID` will use these as fallbacks when not specified per-call. Both must be set together. Example: `OPENCODE_DEFAULT_PROVIDER=anthropic` + `OPENCODE_DEFAULT_MODEL=claude-sonnet-4-5`.
- **Directory validation** — The `directory` parameter on all tools must be an absolute path to an existing directory. Relative paths, non-existent paths, and trailing slashes are handled automatically (resolved or rejected with a helpful error).

## Build the fork first

This is a source-run fork ([<your-username>/opencode-mcp](https://github.com/<your-username>/opencode-mcp)) with no npm package, so every config below points your client at the compiled `dist/index.js` rather than at `npx`. Build it once per machine:

```bash
git clone https://github.com/<your-username>/opencode-mcp.git
cd opencode-mcp
npm install
npm run build
```

Note the absolute path to the build output (`<clone-dir>/dist/index.js`) — you'll substitute it for `/absolute/path/to/opencode-mcp/dist/index.js` in the examples. After any `git pull` or local edit, re-run `npm run build`.

## MCP Client Configurations

Below are complete configuration examples for every supported MCP client. All examples assume the OpenCode server is running on the default `http://127.0.0.1:4096` with no auth, and that you've built the fork as above.

### Claude Desktop

**Config file location:**
- macOS: `~/Library/Application Support/Claude/claude_desktop_config.json`
- Windows: `%APPDATA%\Claude\claude_desktop_config.json`
- Linux: `~/.config/Claude/claude_desktop_config.json`

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

### Claude Code (CLI)

```bash
# Add at user scope (available in all your projects); run from your clone dir so $(pwd) resolves
claude mcp add --scope user opencode -- node "$(pwd)/dist/index.js"

# Or use an absolute path + custom env
claude mcp add --scope user opencode --env OPENCODE_BASE_URL=http://192.168.1.10:4096 -- node /absolute/path/to/opencode-mcp/dist/index.js

# Remove
claude mcp remove opencode -s user
```

### Cursor

**Config file:** `.cursor/mcp.json` in your project root

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

### Windsurf

**Config file:** `~/.windsurf/mcp.json`

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

### VS Code — GitHub Copilot

**Config file:** `.vscode/settings.json` or user `settings.json`

```json
{
  "github.copilot.chat.mcp.servers": [
    {
      "name": "opencode",
      "type": "stdio",
      "command": "node",
      "args": ["/absolute/path/to/opencode-mcp/dist/index.js"]
    }
  ]
}
```

### Cline (VS Code extension)

Cline manages MCP servers through its own settings UI. Add a new server with:

- **Command:** `node`
- **Args:** `/absolute/path/to/opencode-mcp/dist/index.js`
- **Transport:** stdio

### Continue

**Config file:** `.continue/config.json` in your project root or `~/.continue/config.json` globally

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

### Zed

**Config file:** `~/.config/zed/settings.json` or project `settings.json`

```json
{
  "context_servers": {
    "opencode": {
      "command": {
        "path": "node",
        "args": ["/absolute/path/to/opencode-mcp/dist/index.js"]
      }
    }
  }
}
```

### Amazon Q

**Config file:** VS Code `settings.json`

```json
{
  "amazon-q.mcp.servers": [
    {
      "name": "opencode",
      "type": "stdio",
      "command": "node",
      "args": ["/absolute/path/to/opencode-mcp/dist/index.js"]
    }
  ]
}
```

### With authentication (optional)

Add `env` to any config above. This is only needed if you've enabled auth on the OpenCode server:

```json
{
  "mcpServers": {
    "opencode": {
      "command": "node",
      "args": ["/absolute/path/to/opencode-mcp/dist/index.js"],
      "env": {
        "OPENCODE_BASE_URL": "http://127.0.0.1:4096",
        "OPENCODE_SERVER_USERNAME": "myuser",
        "OPENCODE_SERVER_PASSWORD": "mypass"
      }
    }
  }
}
```

### Keeping the fork updated

This fork runs the compiled `dist/`, so changes only take effect after a rebuild. After pulling updates (or editing the source), rebuild and restart your client:

```bash
git -C /absolute/path/to/opencode-mcp pull
npm --prefix /absolute/path/to/opencode-mcp run build
```

## Permissions (Headless Mode)

In headless mode, OpenCode may pause sessions waiting for permission to use tools (file writes, shell commands, etc.). This blocks progress silently.

**Recommended: Auto-allow all permissions** by adding to your `opencode.json`:

```json
{
  "permission": "allow"
}
```

Or set it at runtime:

```
opencode_config_update({ config: { permission: "allow" } })
```

If you prefer manual control, use the permission tools to detect and unblock stuck sessions:

| Tool | Description |
|---|---|
| `opencode_permission_list` | List all pending permission requests across sessions |
| `opencode_session_permission` | Reply to a permission request (`once`, `always`, `reject`) |

## Auto-Start

By default, the MCP server **automatically starts** `opencode serve` if it's not already running. To disable this:

```json
{
  "env": {
    "OPENCODE_AUTO_SERVE": "false"
  }
}
```

## Manual OpenCode Server Setup

If you prefer to manage the server yourself:

```bash
# Default (no auth, port 4096)
opencode serve

# Custom port
opencode serve --port 8080

# With authentication (optional)
OPENCODE_SERVER_USERNAME=myuser OPENCODE_SERVER_PASSWORD=mypass opencode serve
```

The server exposes an OpenAPI 3.1 spec at `http://<host>:<port>/doc`.

# Usage Examples

Real-world examples of using opencode-mcp tools from any MCP client.

## Quick Question

Ask OpenCode something and get an answer in one call:

```json
opencode_ask({
  "prompt": "Explain the authentication flow in this project",
  "providerID": "anthropic",
  "modelID": "claude-opus-4-6"
})
```

## Build a Feature

Have OpenCode implement something and wait for it to finish:

```json
opencode_run({
  "prompt": "Add input validation to POST /api/users. Validate email format, non-empty name, and positive integer age. Return 400 with descriptive errors.",
  "providerID": "anthropic",
  "modelID": "claude-opus-4-6",
  "maxDurationSeconds": 300
})
```

Then review what it did:

```json
opencode_review_changes({ "sessionId": "<session-id>" })
```

## Multi-Turn Conversation

Start a session and iterate:

**1.** `opencode_ask`:
```json
{ "prompt": "What testing framework does this project use?", "title": "Testing exploration" }
```

**2.** `opencode_reply`:
```json
{ "sessionId": "<session-id>", "prompt": "Show me an example of writing a new test" }
```

**3.** `opencode_reply`:
```json
{ "sessionId": "<session-id>", "prompt": "Now add a test for UserService.create" }
```

## Background Tasks (Fire-and-Forget)

Dispatch a long-running task and keep working on something else:

**1. Fire the task** — `opencode_fire`:
```json
{
  "prompt": "Refactor the entire authentication module to use JWT tokens",
  "providerID": "anthropic",
  "modelID": "claude-opus-4-6",
  "title": "JWT refactor"
}
```
Returns immediately with a session ID.

**2. Check progress** — `opencode_check`:
```json
{ "sessionId": "<session-id>" }
```
Returns: status (running/idle), todo progress (e.g. "3/7 done, current: Add JWT middleware"), files changed count.

**3. Get full results when done** — `opencode_conversation`:
```json
{ "sessionId": "<session-id>" }
```

## Build a Full App

Chain multiple tools to build an entire project:

```json
// 1. Set up the project
opencode_run({
  "prompt": "Create a new Express.js API with TypeScript, SQLite, and Vitest. Include health check, CRUD for users, and error handling middleware.",
  "title": "Build API",
  "maxDurationSeconds": 600
})

// 2. Add features in parallel
opencode_fire({ "prompt": "Add JWT authentication with login/register endpoints", "title": "Add auth" })
opencode_fire({ "prompt": "Add rate limiting middleware and request logging", "title": "Add middleware" })

// 3. Check both are done
opencode_check({ "sessionId": "<auth-session-id>" })
opencode_check({ "sessionId": "<middleware-session-id>" })

// 4. Review all changes
opencode_review_changes({ "sessionId": "<auth-session-id>" })
opencode_review_changes({ "sessionId": "<middleware-session-id>" })
```

## Multi-Model Parallel Review

Get independent opinions on the same code from multiple models simultaneously. Each call creates a fresh session (no `sessionId`) so model contexts never bleed into each other.

**1. Fire the same prompt to several models at once** — each `opencode_fire` returns immediately:
```json
// Model A — Anthropic Claude
opencode_fire({
  "prompt": "Review the authentication module for security issues. Be specific about file names and line numbers.",
  "providerID": "anthropic",
  "modelID": "claude-opus-4-6",
  "agent": "plan",
  "title": "Security review — Claude"
})

// Model B — Kimi K2.7 Code via OpenRouter
opencode_fire({
  "prompt": "Review the authentication module for security issues. Be specific about file names and line numbers.",
  "providerID": "openrouter",
  "modelID": "moonshotai/kimi-k2.7-code",
  "agent": "plan",
  "title": "Security review — Kimi"
})

// Model C — a third model
opencode_fire({
  "prompt": "Review the authentication module for security issues. Be specific about file names and line numbers.",
  "providerID": "openrouter",
  "modelID": "google/gemini-2.5-pro",
  "agent": "plan",
  "title": "Security review — Gemini"
})
```
Each returns a `sessionId`. No session shares context with another. The `providerID`/`modelID` pairs above are illustrative — enumerate exact model IDs for an authed provider with `opencode_provider_models({ "providerId": "openrouter" })`.

**2. Poll until all are done** — `opencode_check` is cheap; call it every ~30 s per session:
```json
opencode_check({ "sessionId": "<claude-session-id>" })
opencode_check({ "sessionId": "<kimi-session-id>" })
opencode_check({ "sessionId": "<gemini-session-id>" })
```
`status: "idle"` means the model finished. `status: "running"` means keep polling.

**3. Collect and compare results** — `opencode_conversation`:
```json
opencode_conversation({ "sessionId": "<claude-session-id>" })
opencode_conversation({ "sessionId": "<kimi-session-id>" })
opencode_conversation({ "sessionId": "<gemini-session-id>" })
```
Look for findings that appear in multiple responses — cross-model agreement is a strong signal that an issue is real.

> **Tip:** Use `agent: "plan"` (read-only) for review tasks so no model accidentally edits files. Confirm with `opencode_review_changes` after — all three should show no changes.

## Get Project Context

Understand a project you've never seen:

```json
opencode_context({})
```

Returns: project info, VCS details (branch, status), configuration, and available agents.

## Code Review

Review changes from a coding session:

**1.** Find the session — `opencode_sessions_overview`:
```json
{}
```

**2.** Review the diff — `opencode_review_changes`:
```json
{ "sessionId": "<session-id>" }
```

**3.** Read the conversation — `opencode_conversation`:
```json
{ "sessionId": "<session-id>" }
```

## Search the Codebase

```json
// Find TODOs and FIXMEs
opencode_find_text({ "pattern": "TODO|FIXME|HACK" })

// Find config files
opencode_find_file({ "query": "config" })

// Find a function or class
opencode_find_symbol({ "query": "handleAuth" })
```

## Multi-Project Workflow

Work on multiple projects from one client:

```json
// Mobile app
opencode_ask({
  "directory": "/home/user/projects/mobile-app",
  "prompt": "Set up React Navigation with a tab navigator"
})

// Web app (same server, different project)
opencode_ask({
  "directory": "/home/user/projects/web-app",
  "prompt": "Add authentication to the Next.js app"
})
```

## Provider Management

```json
// Check what's available
opencode_provider_list({})

// Set an API key (one-time, global)
opencode_auth_set({
  "providerId": "anthropic",
  "type": "api",
  "key": "sk-ant-..."
})

// Test that it works
opencode_provider_test({ "providerId": "anthropic" })
```

## Using Prompts

MCP prompts are guided workflow templates your client offers as selectable actions.

| Prompt | Arguments | What it does |
|---|---|---|
| `opencode-code-review` | `sessionId` | Reviews diffs for correctness, style, performance, security |
| `opencode-debug` | `issue`, `context?` | Step-by-step debugging: finds files, reads code, identifies root cause |
| `opencode-project-setup` | *(none)* | Reads README, configs, entry points, summarizes the project |
| `opencode-implement` | `description`, `requirements?` | Sends to OpenCode's build agent, reviews changes, reports results |
| `opencode-best-practices` | *(none)* | Guides on setup, tool selection, monitoring, and common pitfalls |
| `opencode-session-summary` | `sessionId` | Summarizes discussion, actions, files modified, remaining work |

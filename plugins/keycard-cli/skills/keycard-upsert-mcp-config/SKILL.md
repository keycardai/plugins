---
name: keycard-upsert-mcp-config
description: |
  Add or update an MCP server entry in .mcp.json.
  TRIGGER when: user wants to add or update an MCP server in .mcp.json.
  DO NOT TRIGGER when: user wants to set a keycard.toml field (use keycard-upsert-config).
argument-hint: "add \"<name>\" type=http url=\"<url>\" | add \"<name>\" command=\"<cmd>\" args=[...] env={...}"
license: Apache-2.0
metadata: {}
---

# keycard-upsert-mcp-config

Add an MCP server entry to `.mcp.json`. Writes to the project-local `.mcp.json` only (user-global config is out of scope). Supports stdio and HTTP transport.

The arguments for this invocation are: `$ARGUMENTS`

## Step 1 — Read existing config

Read `.mcp.json`. If the file is absent or empty, start from:
```json
{
  "mcpServers": {}
}
```

## Step 2 — Parse arguments

Parse `name`, `type`, `command`, `url`, `args`, and `env` from `$ARGUMENTS`.

Two invocation forms are supported:

- **HTTP:** `add "<name>" type=http url="<url>"` — for HTTP MCP servers (e.g. Keycard gateway URLs)
- **Stdio:** `add "<name>" command="<cmd>" args=[...] env={...}` — for local process-based servers

`type` defaults to `stdio` when not specified.

Validate `name` matches `[A-Za-z0-9._-]{1,64}`. If it does not, stop with:
> Server name `<name>` contains invalid characters. Use only letters, digits, `.`, `_`, and `-` (max 64 chars).

## Step 3 — Check for duplicates

If `mcpServers["<name>"]` already exists, stop with this error:

> MCP server `<name>` already exists in `.mcp.json`. Use a different name or remove the existing entry first.

Do not overwrite silently.

## Step 4 — Merge new entry

**HTTP transport:** Set `mcpServers["<name>"]` to:
```json
{
  "type": "http",
  "url": "<url>"
}
```

**Stdio transport:** Set `mcpServers["<name>"]` to:
```json
{
  "command": "<cmd>",
  "args": [...],
  "env": {...}
}
```

Omit the `args` key entirely when the array is empty. Omit the `env` key entirely when the object is empty.

## Step 5 — Write back

Write the updated config to `.mcp.json` with 2-space indentation.

## Step 6 — Confirm

Re-read `.mcp.json` and show the user the added `mcpServers["<name>"]` entry to confirm the write succeeded.

## Examples

**User asks:**
- "add an MCP server called my-server using npx with args -y @my/mcp-server"
- "add a local MCP tool at /usr/local/bin/my-mcp"
- "add an HTTP MCP server called granola at https://granola.mcp.example.com/sse"
